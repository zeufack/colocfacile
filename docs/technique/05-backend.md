# Backend (`apps/api`) et domaine (`packages/domain`)

## 1. Stack

| Besoin | Choix |
|--------|-------|
| Framework | **NestJS** (adaptateur Express), TypeScript strict, compilation SWC |
| Runtime | Node.js 24 LTS |
| Contrat et validation | `@ts-rest/nest` + Zod (`packages/contracts`) |
| Base de données | PostgreSQL 16+ via **Drizzle ORM** (`node-postgres`), pool partagé |
| Contexte de requête | `nestjs-cls` (AsyncLocalStorage) : `userId`, `accountId`, `requestId` |
| Authentification | **Better Auth** (adaptateur Drizzle), monté sur `/api/auth/*` |
| Tâches | **pg-boss** |
| Stockage de fichiers | `@aws-sdk/client-s3` + `s3-request-presigner` (Scaleway Object Storage) |
| E-mails | Scaleway Transactional Email (API HTTP), derrière une interface `Mailer` |
| PDF | **pdfmake** ([ADR-0008](adr/0008-generation-pdf.md)) |
| Logs | `nestjs-pino` (JSON), `requestId` propagé |
| Erreurs | Sentry (région UE) |
| Sécurité HTTP | `helmet`, `@nestjs/throttler` |

## 2. Structure d'un module

```
apps/api/src/modules/rent/
├── rent.module.ts
├── rent.controller.ts          # Implémente le contrat ts-rest ; aucune logique métier
├── rent.service.ts             # Cas d'usage : transaction, orchestration, audit
├── rent.repository.ts          # Requêtes Drizzle (reçoit un `tx`)
├── payment-allocation.ts       # Adaptateur vers domain.allocatePayment
└── rent.service.spec.ts        # Tests d'intégration (Testcontainers)
```

**Règles**
- Un **controller** valide (via le contrat), appelle un service et fait correspondre les erreurs. Rien d'autre.
- Un **service** = un cas d'usage = **une transaction**. Il lit, appelle le domaine pur, écrit, puis ajoute l'entrée d'audit.
- Un **repository** ne prend jamais `accountId` d'un paramètre utilisateur : il utilise le `tx` scopé au compte.
- Le **domaine** (`packages/domain`) ne connaît ni NestJS, ni Drizzle, ni les dates « maintenant » : `today` est toujours un paramètre.

## 3. Authentification et contexte de compte

```ts
// apps/api/src/auth/auth.ts
export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: 'pg', schema: authSchema }),
  user:         { modelName: 'auth_user' },
  session:      { modelName: 'auth_session', expiresIn: 60 * 60 * 24 * 14, updateAge: 60 * 60 * 24 },
  account:      { modelName: 'auth_account' },
  verification: { modelName: 'auth_verification' },
  emailAndPassword: { enabled: true, requireEmailVerification: true, minPasswordLength: 12,
                      sendResetPassword: ({ user, url }) => mailer.sendPasswordReset(user, url) },
  emailVerification: { sendVerificationEmail: ({ user, url }) => mailer.sendVerification(user, url) },
  baseURL: env.APP_URL,                 // URL publique du web : les appels arrivent via le proxy Start
  trustedOrigins: [env.APP_URL],
  databaseHooks: { user: { create: { after: (user) => accounts.provisionFor(user) } } },
});
```

- **Montage** dans `main.ts` : `app.use('/api/auth', toNodeHandler(auth))`, **avant** le parseur JSON de NestJS (qui est désactivé globalement : `bodyParser: false`, puis réactivé après le montage). Une intégration communautaire (`@thallesp/nestjs-better-auth`) existe ; on l'évalue au démarrage du projet.
- **Derrière le proxy Start** : NestJS n'est joignable que depuis le conteneur web. Un garde global `InternalTokenGuard` vérifie `X-Internal-Token` (défense en profondeur, même sur réseau privé). Express est configuré avec `trust proxy` pour prendre l'IP cliente dans `X-Forwarded-For` (limites de débit par IP correctes).
- **`SessionGuard`** (global) : `auth.api.getSession({ headers })` → 401 si absente. Les routes publiques sont marquées `@Public()`.
- **`AccountGuard`** : charge l'appartenance `account_members` de l'utilisateur et place `accountId` et `role` dans le CLS.
- Création du compte : un **hook** Better Auth crée `accounts`, `account_members (owner)` et un bailleur par défaut à la création de l'utilisateur.
- Les noms exacts des routes et options Better Auth sont à vérifier avec la version installée.

## 4. Transactions scopées au compte

```ts
// apps/api/src/db/account-db.ts
@Injectable()
export class AccountDb {
  constructor(private readonly db: Db, private readonly cls: ClsService) {}

  tx<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
    const accountId = this.cls.get('accountId');
    if (!accountId) throw new Error('AccountDb used outside an account context');
    return this.db.transaction(async (tx) => {
      await tx.execute(sql`select set_config('app.account_id', ${accountId}, true)`);
      return fn(tx);
    });
  }
}
```
`set_config(..., true)` équivaut à `SET LOCAL` : la valeur disparaît à la fin de la transaction, donc aucune fuite entre requêtes d'un même pool.

Les tâches système (`app_system`, `BYPASSRLS`) utilisent un `SystemDb` distinct, qu'on ne peut pas injecter dans les modules métier (vérifié par une règle de lint).

## 5. Le paquet `domain`

Des fonctions pures, testées exhaustivement, nommées d'après les règles :

```ts
// Montants
type Cents = number & { readonly __brand: 'Cents' };
cents(n: number): Cents                       // refuse les non-entiers
splitByShares(total: Cents, sharesBp: number[]): Cents[]   // R4.5 — reste d'arrondi au dernier

// R1 — baux
defaultLeaseEnd(type, start, landlordKind): IsoDate
noticeEndDate({ type, givenBy, receivedOn, reducedReason }): { endDate, months, rule }

// R2 — prorata
prorate(monthly: Cents, periodStart: IsoDate, periodEnd: IsoDate): Cents
monthlyPeriods(lease, from, to): Period[]     // découpe en mois avec 1re et dernière périodes partielles

// R3 — dépôt
depositCap(type, rentCents): Cents | 0
refundDeadline(keysReturnedOn, exitConform): IsoDate
lateRefundPenalty(rentCents, deadline, today): Cents
refundAmount(deposit, deductions): { refund: Cents, shortfall: Cents }

// R4 — échéances
rentDueStatus(due, today, graceDays): RentDueStatus
allocatePayment(amount, openDues, targetDueId?): { allocations, credit }

// R6, R7, R8
reconcileCharges(provisions, expenses, allocation, occupancy): Reconciliation
reviseRent({ rentCents, oldIndex, newIndex, dpeClass }): { newRentCents } | { blocked: 'R7.4' }
solidarityEndDate({ noticeEndDate, replacementDate }): IsoDate
```

Chaque fonction renvoie aussi, quand c'est utile, la **règle appliquée** (`rule: 'R1.1'`), que l'interface affiche dans « Comment c'est calculé ? ».

### Arithmétique

- Tout calcul se fait en **centimes entiers**. Arrondi : au plus proche, demi vers le haut, **une seule fois**, à la fin.
- Prorata : `Math.round(monthly * days / daysInMonth)` — le produit reste bien en deçà de `Number.MAX_SAFE_INTEGER`.
- IRL : indices convertis en centièmes entiers (`142.06 → 14206`), puis `round(rent * newIdx / oldIdx)`.

## 6. Cas d'usage critiques

### Enregistrer un paiement (LOYER-3)
1. Si l'`Idempotency-Key` existe déjà pour ce compte → renvoyer le paiement existant.
2. `SELECT … FOR UPDATE` sur les échéances ouvertes du colocataire (ordre `due_date`).
3. `domain.allocatePayment` → imputations et avance.
4. Insère `payments` et `payment_allocations` ; met à jour `rent_dues.paid_cents`.
5. Audit ; en option, génère la quittance ; en option, met en file `email.send-receipt`.

### Générer les échéances (LOYER-1)
Tâche quotidienne : pour chaque bail `active | notice`, calcule `domain.monthlyPeriods(lease, today, today + 45 j)` puis `INSERT … ON CONFLICT DO NOTHING`. Elle est aussi appelée de façon synchrone à l'activation d'un bail.

### Générer une quittance (LOYER-6)
1. Vérifie `paid_cents >= total_cents` → quittance ; sinon → reçu.
2. Numéro : `UPDATE accounts SET receipt_counter = receipt_counter + 1 RETURNING …` (dans la transaction, sans trou possible en cas d'échec).
3. Construit le `snapshot` (bailleur, colocataire, logement, période, montants, date de paiement), rend le PDF, l'envoie dans le stockage objet, insère `documents` et `receipts`.

### Envoyer par e-mail (LOYER-7)
Worker `email.send-receipt` : vérifie `accepts_email_documents`, télécharge le PDF, appelle `Mailer.send({ from: 'EasyColoc <quittances@easycoloc.fr>', fromName: landlord.displayName, replyTo: landlord.email, to, attachments })`, puis met à jour `email_deliveries`. Au-delà de 5 échecs : `failed`, visible dans l'interface.

## 7. Fichiers

- Bucket **privé**, clés `accounts/{accountId}/documents/{documentId}`.
- Upload direct navigateur → S3 par **PUT signé** (5 min), avec `Content-Type` et `Content-Length` imposés dans la signature.
- `confirm` : `HeadObject` vérifie la taille ; les premiers octets sont lus pour vérifier le type réel (magic bytes). PDF, JPEG, PNG et HEIC sont acceptés, 15 Mo au maximum.
- Téléchargement : `302` vers un GET signé (60 s), avec `Content-Disposition: attachment`.
- Un objet dont l'upload n'est pas confirmé est supprimé au bout de 24 h par une règle de cycle de vie du bucket (préfixe `pending/`, puis déplacement à la confirmation).

## 8. Configuration

Variables d'environnement validées au démarrage par un schéma Zod (`env.ts`) : l'application refuse de démarrer si une variable manque.

| Variable | Exemple |
|----------|---------|
| `APP_ROLE` | `all` \| `api` \| `worker` |
| `APP_URL` | `https://app.easycoloc.fr` (URL publique du web Start) |
| `INTERNAL_TOKEN` | secret partagé avec le web (`X-Internal-Token`) |
| `DATABASE_URL` / `DATABASE_SYSTEM_URL` | rôles `app_user` / `app_system` |
| `BETTER_AUTH_SECRET` | 32+ octets aléatoires |
| `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` | `https://s3.fr-par.scw.cloud`, `fr-par` |
| `MAIL_PROVIDER`, `SCW_TEM_PROJECT_ID`, `SCW_SECRET_KEY` | `scaleway` \| `smtp` (Mailpit en local) |
| `SENTRY_DSN` | |
