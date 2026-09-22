# API

> Contrat de référence : `packages/contracts` (ts-rest + Zod). La spécification OpenAPI est générée à partir du contrat et servie sur `/api/docs` hors production. Voir [ADR-0009](adr/0009-contrat-api-ts-rest.md).

## 1. Principes

| Sujet | Règle |
|-------|-------|
| Style | REST, JSON, préfixe `/api/v1`. Ressources au pluriel en `kebab-case` |
| Accès | Le navigateur appelle `/api/*` sur l'origine du web ; le serveur TanStack Start relaie vers NestJS (non public). Voir [04-frontend.md §3](04-frontend.md#3-proxy-api-même-origine) |
| Authentification | Cookie de session Better Auth (`HttpOnly`, `Secure`, `SameSite=Lax`). Pas de jeton dans `localStorage` |
| Portée | L'`accountId` n'apparaît **jamais** dans l'URL : il vient de la session |
| Validation | Toute entrée est validée par le schéma Zod du contrat. Les champs inconnus sont rejetés (`.strict()`) |
| Montants | Entiers en centimes (`rentCents: 48000`) |
| Dates | `YYYY-MM-DD` pour les dates civiles, ISO 8601 UTC pour les horodatages |
| Pagination | Par curseur : `?limit=50&cursor=…` → `{ items, nextCursor }` |
| Idempotence | En-tête `Idempotency-Key` accepté (et conseillé) sur `POST /payments` et `POST /receipts/:id/send` |
| Actions métier | Les transitions d'état sont des sous-ressources verbales : `POST /leases/:id/activate`, `POST /payments/:id/cancel` |
| Versionnement | `/v1`. Un changement cassant = `/v2`, jamais de rupture silencieuse |

## 2. Format des erreurs

[RFC 9457 — Problem Details](https://www.rfc-editor.org/rfc/rfc9457), avec un **code métier stable** qui renvoie à une règle :

```json
{
  "type": "https://easycoloc.fr/errors/deposit-cap-exceeded",
  "title": "Dépôt de garantie supérieur au plafond légal",
  "status": 422,
  "code": "DEPOSIT_CAP_EXCEEDED",
  "rule": "R3",
  "detail": "Le dépôt d'un bail meublé ne peut pas dépasser 2 mois de loyer hors charges (960,00 €).",
  "errors": [{ "path": "depositCents", "message": "Maximum 96000" }]
}
```

| HTTP | Quand |
|------|-------|
| 400 | Requête mal formée (JSON invalide) |
| 401 | Pas de session |
| 403 | Session valide, mais rôle insuffisant |
| 404 | Ressource absente **ou appartenant à un autre compte** (on ne révèle pas son existence) |
| 409 | Conflit d'état (ex. : activer un bail déjà actif, chevauchement sur une chambre) |
| 422 | Violation d'une règle métier (`code` + `rule`) ou erreur de validation |
| 429 | Limite de débit atteinte |

Le frontend traduit `code` en message et relie `errors[].path` au champ du formulaire.

## 3. Ressources

### Authentification (Better Auth, hors contrat ts-rest)
| Méthode | Chemin | Rôle |
|---------|--------|------|
| POST | `/api/auth/sign-up/email` | Inscription (AUTH-1) |
| POST | `/api/auth/sign-in/email` | Connexion |
| POST | `/api/auth/sign-out` | Déconnexion |
| POST | `/api/auth/request-password-reset` · `/api/auth/reset-password` | Mot de passe oublié (AUTH-3) |
| GET | `/api/auth/get-session` | Session courante |

### Compte
| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET | `/me` | Utilisateur + compte + rôle |
| GET · PATCH | `/account/settings` | Délai de grâce, pièces obligatoires |
| POST | `/account/export` | Lance un export RGPD (tâche) → `202` |
| DELETE | `/account` | Suppression du compte, avec confirmation par mot de passe (AUTH-4) |

### Bailleurs, biens, chambres
| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET · POST | `/landlords` | |
| GET · PATCH | `/landlords/:id` | |
| GET · POST | `/properties` | `?includeArchived` |
| GET · PATCH | `/properties/:id` | Inclut les chambres et leur statut d'occupation (BIEN-4) |
| POST | `/properties/:id/archive` | |
| POST | `/properties/:id/rooms` | |
| PATCH | `/rooms/:id` · POST `/rooms/:id/archive` | |

### Colocataires et garants
| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET · POST | `/tenants` | `?q=` recherche, `?status=current\|former` |
| GET · PATCH | `/tenants/:id` | |
| GET | `/tenants/:id/ledger` | Échéances, paiements, solde, avance (COLOC-2) |
| GET · POST | `/tenants/:id/guarantors` | |
| PATCH | `/guarantors/:id` | |

### Baux
| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET · POST | `/leases` | Création en `draft` |
| GET · PATCH | `/leases/:id` | Modifiable librement en `draft` ; en `active`, champs restreints |
| POST | `/leases/:id/preview` | Calculs sans enregistrer : durée, plafond du dépôt, première échéance au prorata (pour l'assistant E5) |
| POST | `/leases/:id/activate` | Contrôles + génération des échéances (BAIL-1) |
| POST | `/leases/:id/notices` | Congé → date de fin calculée (BAIL-3) |
| POST | `/leases/:id/tenant-replacements` | Avenant de remplacement (BAIL-5) |
| POST | `/leases/:id/end` | Clôture |
| GET | `/leases/:id/required-documents` | Checklist et pièces manquantes (DOC-2) |

### Loyers
| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET | `/rent-dues` | `?month=2026-09&propertyId=&status=` (LOYER-5) |
| PATCH | `/rent-dues/:id` | Ajuster un montant, avec `adjustedReason` obligatoire (R2.2) |
| POST | `/rent-dues/mark-paid` | Lot : `{ rentDueIds, paidOn, method }` → un paiement par échéance, au montant exact (LOYER-2, LOYER-4) |
| POST | `/payments` | `{ tenantId, amountCents, paidOn, method, rentDueId? }` → imputation R4.4 (LOYER-3) |
| PATCH | `/payments/:id` · POST `/payments/:id/cancel` | Correction ou annulation, avec réimputation (LOYER-8) |

### Quittances et e-mails
| Méthode | Chemin | Rôle |
|---------|--------|------|
| POST | `/rent-dues/:id/receipt` | Génère une quittance si l'échéance est soldée, sinon un reçu (LOYER-6) |
| GET | `/receipts` | Liste (DOC-5) |
| GET | `/receipts/:id/download` | `302` vers une URL signée de courte durée |
| POST | `/receipts/:id/void` | Annulation (R5.5) |
| POST | `/receipts/:id/send` | Envoi par e-mail → `202`, avec suivi dans `email_deliveries` (LOYER-7) |

### Dépôt de garantie
| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET · PATCH | `/leases/:id/deposit` | Réception, remise des clés, état des lieux conforme ou non |
| POST · DELETE | `/leases/:id/deposit/deductions[/:deductionId]` | Retenues |
| POST | `/leases/:id/deposit/refund` | Restitution + décompte PDF (DEPOT-4) |

### Charges et IRL
| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET · POST | `/properties/:id/charge-expenses` | Dépenses réelles (CHARGE-2) |
| POST | `/properties/:id/charge-reconciliations/preview` | `{ periodStart, periodEnd }` → décompte par bail |
| POST | `/properties/:id/charge-reconciliations` | Enregistre et génère les échéances ou avoirs (CHARGE-4) |
| GET · POST | `/rent-indices` | Lecture ; création réservée à l'administration au MVP |
| GET | `/leases/:id/revision-preview` | Nouveau loyer calculé, ou blocage R7.4 |
| POST | `/leases/:id/revisions` | Applique la révision (IRL-3) |

### Documents
| Méthode | Chemin | Rôle |
|---------|--------|------|
| POST | `/documents/upload-intents` | `{ attachTo, type, filename, mimeType, sizeBytes }` → `{ documentId, uploadUrl }` (PUT signé, 5 min) |
| POST | `/documents/:id/confirm` | Vérifie la présence de l'objet, sa taille et son type → `ready` |
| GET | `/documents` | Filtres `attachTo`, `type`, `expiring` |
| GET | `/documents/:id/download` | `302` vers une URL signée (60 s) |
| PATCH · DELETE | `/documents/:id` | Type, date d'expiration ; suppression |

### Tableau de bord et exports
| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET | `/dashboard?month=2026-09` | KPI du mois, liste « à faire », occupation (DASH-1…3) |
| GET | `/exports/payments.csv?year=2026` | Export CSV (`;`, UTF-8 avec BOM pour Excel) (DASH-5) |

## 4. Exemple de contrat (ts-rest)

```ts
// packages/contracts/src/rent.ts
export const rentContract = c.router({
  recordPayment: {
    method: 'POST',
    path: '/api/v1/payments',
    headers: z.object({ 'idempotency-key': z.string().uuid().optional() }),
    body: z.object({
      tenantId: z.string().uuid(),
      rentDueId: z.string().uuid().optional(),
      amountCents: z.number().int().positive(),
      paidOn: isoDate,
      method: paymentMethod,
      note: z.string().max(500).optional(),
    }).strict(),
    responses: { 201: paymentWithAllocations, 422: problem },
  },
});
```

## 5. Limites de débit

| Portée | Limite |
|--------|--------|
| `/api/auth/*` (connexion, réinitialisation) | 10 requêtes / min / IP |
| API authentifiée | 300 requêtes / min / utilisateur |
| `POST /receipts/:id/send` | 30 envois / heure / compte |
| `POST /documents/upload-intents` | 60 / heure / compte |
