# Conventions

## 1. Langue

| Où | Langue |
|----|--------|
| Interface, URL visibles, e-mails, PDF | Français |
| Code, noms de tables, API, commits, commentaires | Anglais |
| Documentation (`docs/`) | Français |

La correspondance entre termes français et noms du code est **uniquement** celle du [glossaire](../produit/03-glossaire.md). On ne crée pas de synonymes : c'est `Tenant`, jamais `Renter` ou `Occupant`. Le mot « tenant » au sens SaaS est proscrit : on dit `Account`.

## 2. Nommage

| Élément | Convention | Exemple |
|---------|------------|---------|
| Fichiers | `kebab-case` | `rent-due-status.ts` |
| Composants React | `PascalCase` | `RentDuesTable` |
| Hooks | `useXxx` | `useRecordPayment` |
| Fonctions du domaine | verbe + nom, règle en JSDoc | `/** R2.1 */ prorate()` |
| Montants | suffixe `Cents` | `rentCents` |
| Dates civiles | suffixe `On` ou `Date`, type `IsoDate` | `paidOn`, `dueDate` |
| Horodatages | suffixe `At` | `voidedAt` |
| Booléens | `is`/`has`/`accepts` | `acceptsEmailDocuments` |
| Codes d'erreur | `SCREAMING_SNAKE_CASE` | `LEASE_OVERLAP` |

## 3. TypeScript

- Pas de `any` ; `unknown`, puis affinage par un schéma Zod aux frontières.
- Types de marque (*branded types*) pour `Cents`, `IsoDate`, `AccountId`, et les identifiants d'entités (évite de passer un `tenantId` à la place d'un `leaseId`).
- Pas d'`enum` TypeScript : unions littérales dérivées des schémas Zod (`z.enum([...])`).
- Pas d'exceptions dans le domaine pour les cas métier attendus : résultats discriminés (`{ ok: false, code: 'DEPOSIT_CAP_EXCEEDED', rule: 'R3' }`). Le service les traduit en `DomainError` → 422.

## 4. Git

- Branche principale `main`, protégée ; branches courtes `feat/loyer-3-record-payment`, `fix/…`.
- Pull request obligatoire, CI verte, auto-revue avec la checklist du modèle de PR.
- Conventional Commits, avec la portée du module : `feat(rent): allocate payment to oldest due (R4.4)`.
- Versionnement SemVer par tag `vX.Y.Z` ; `CHANGELOG.md` généré.

## 5. Documentation vivante

- Une règle métier modifiée : mettre à jour `06-regles-metier.md` **dans la même PR** que le code et les tests.
- Une décision technique structurante : nouvel ADR dans `docs/technique/adr/` (modèle : [0000-modele.md](adr/0000-modele.md)).
- Un nouveau terme du domaine : l'ajouter au glossaire avant de l'utiliser dans le code.
