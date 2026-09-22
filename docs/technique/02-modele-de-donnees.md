# Modèle de données

> Source de vérité : `packages/db/src/schema/*.ts` (Drizzle). Ce document en décrit les intentions et les contraintes. Les noms suivent le [glossaire](../produit/03-glossaire.md).

## 1. Conventions

| Sujet | Convention |
|-------|------------|
| Nommage | Tables au pluriel en `snake_case` (`rent_dues`) ; colonnes en `snake_case` ; propriétés TypeScript en `camelCase` (mapping Drizzle `casing: 'snake_case'`) |
| Identifiants | `uuid` v7 (triables dans le temps), générés côté application |
| Isolation | Toute table métier a `account_id uuid not null` + politique RLS (voir §5) |
| Montants | `integer` en **centimes** (`*_cents`). Jamais de `float` ni de `numeric` pour l'argent. Voir [ADR-0007](adr/0007-montants-en-centimes-et-dates.md) |
| Pourcentages | `integer` en **points de base** (`*_bp`) : 10 000 = 100 % |
| Dates civiles | `date` (sans fuseau) pour les dates de bail, les échéances et les paiements. Transportées en chaîne `YYYY-MM-DD` |
| Horodatages | `timestamptz` pour `created_at`, `updated_at`, `*_at` |
| Suppression | Pas de suppression physique des données métier : `archived_at` / `cancelled_at` / `voided_at`. Seuls les brouillons peuvent être supprimés |
| Énumérations | `pgEnum` Drizzle, valeurs en anglais `snake_case` |
| Colonnes communes | `id`, `account_id`, `created_at`, `updated_at` sur toutes les tables métier |

## 2. Diagramme entité-relation (simplifié)

```
accounts ─┬─< account_members >── auth_user
          │
          ├─< landlords ─< properties ─┬─< rooms
          │                            ├─< charge_expenses
          │                            └─< documents (property)
          │
          ├─< tenants ─┬─< guarantors
          │            └─< documents (tenant)
          │
          └─< leases ──┬─< lease_tenants >── tenants
                       ├─< lease_guarantors >── guarantors
                       ├─< notices
                       ├─< rent_dues ─< payment_allocations >── payments
                       │       └──< receipts ─< email_deliveries
                       ├── deposits ─< deposit_deductions
                       ├─< rent_revisions
                       ├─< charge_reconciliations
                       └─< documents (lease)

rent_indices (table globale, non rattachée à un compte)
audit_logs   (par compte)
auth_*       (Better Auth : auth_user, auth_session, auth_account, auth_verification)
```

> ⚠️ **Collision de noms** : Better Auth crée par défaut une table `account` (fournisseurs d'identité). Toutes ses tables sont préfixées `auth_` (option `modelName` de Better Auth), pour garder `accounts` au sens « compte propriétaire ».

## 3. Tables

### 3.1 Comptes et accès

**`accounts`** — le compte propriétaire, unité d'isolation.
| Colonne | Type | Notes |
|---------|------|-------|
| `name` | text | ex. « Claire Martin » |
| `grace_days` | smallint, défaut 5 | R4.3 |
| `timezone` | text, défaut `Europe/Paris` | |
| `required_document_types` | jsonb | Checklist des pièces personnalisée (DOC-2) |
| `receipt_counter` | integer | Compteur de numérotation (R5.5), incrémenté par `UPDATE … RETURNING` |

**`account_members`** — `(account_id, user_id, role)` avec `role ∈ {owner, manager}`. Un utilisateur a un seul compte au MVP ; la table prépare AUTH-5.

### 3.2 Patrimoine

**`landlords`** (bailleurs)
| Colonne | Type | Notes |
|---------|------|-------|
| `kind` | enum `person`, `company` | Personne physique ou morale (SCI…) |
| `display_name` | text | Nom affiché sur les quittances |
| `legal_form` | text null | ex. « SCI » |
| `siren` | char(9) null | Si `company` |
| `address_*` | text | `line1`, `line2`, `postal_code`, `city` |
| `email` | text null | Utilisé en `Reply-To` des e-mails |

**`properties`** (biens)
| Colonne | Type | Notes |
|---------|------|-------|
| `landlord_id` | uuid FK | FK composite `(account_id, landlord_id)` |
| `name` | text | « T4 Lyon » |
| `address_*` | text | |
| `furnished` | boolean | Pré-remplit le type de bail |
| `surface_m2` | numeric(6,2) null | Informatif, clé de répartition |
| `is_condominium` | boolean | R3.6 |
| `dpe_class` | enum `A`…`G` null | R7.4, R10.1 |
| `dpe_valid_until` | date null | R10.2 |
| `charges_allocation` | enum `equal`, `surface`, `custom` | CHARGE-5 |
| `archived_at` | timestamptz null | |

**`rooms`** (chambres) : `property_id`, `name`, `surface_m2`, `reference_rent_cents`, `reference_charges_cents`, `custom_allocation_bp` (si répartition personnalisée), `archived_at`.

### 3.3 Personnes

**`tenants`** (colocataires) : `first_name`, `last_name`, `email`, `phone`, `birth_date` (facultatif), `accepts_email_documents` (boolean, R5.4), `notes`, `anonymized_at`.

**`guarantors`** (garants) : `tenant_id`, `kind` (`person`, `organization` — ex. Visale), `display_name`, `email`, `phone`, `address_*`.

### 3.4 Baux

**`leases`** (baux)
| Colonne | Type | Notes |
|---------|------|-------|
| `property_id` | uuid FK | |
| `room_id` | uuid FK null | Obligatoire si `form = individual`, nul si `shared` (CHECK) |
| `form` | enum `shared`, `individual` | Bail unique / individuel |
| `type` | enum `unfurnished`, `furnished`, `student`, `mobility` | R1 |
| `status` | enum `draft`, `active`, `notice`, `ended` | R1.3 |
| `start_date` / `end_date` | date | Fin initiale (durée légale par défaut) |
| `rent_cents` | integer ≥ 0 | Loyer HC courant |
| `charges_cents` | integer ≥ 0 | |
| `charges_mode` | enum `provisions`, `flat` | R6.1 |
| `deposit_cents` | integer ≥ 0 | Contrôlé par R3 (domaine + CHECK `type <> 'mobility' OR deposit_cents = 0`) |
| `due_day` | smallint 1..28 | R4.2 |
| `payment_term` | enum `advance`, `arrears` | R4.1 |
| `solidarity_clause` | boolean | R8.1, bail unique uniquement |
| `irl_clause` | boolean | R7.1 |
| `irl_reference_quarter` | text null | ex. `2026-T2` |
| `irl_reference_value` | numeric(7,2) null | |
| `ended_at` | date null | Date de fin effective |

Contrainte d'exclusion : pas deux baux individuels **actifs** qui se chevauchent sur la même chambre :
```sql
EXCLUDE USING gist (room_id WITH =, daterange(start_date, coalesce(ended_at, end_date), '[]') WITH &&)
  WHERE (form = 'individual' AND status IN ('active','notice'));
```
*(Nécessite l'extension `btree_gist`. Les baux tacitement reconduits utilisent `ended_at`, ou une date de fin repoussée.)*

**`lease_tenants`** (signataires) : `lease_id`, `tenant_id`, `share_bp` (quote-part, R4.5), `joined_on`, `left_on` null, `solidarity_ends_on` null (R8.2). Unique `(lease_id, tenant_id)`.
Invariant (vérifié par le service, dans la transaction) : Σ `share_bp` des signataires présents = 10 000.

**`lease_guarantors`** : `lease_id`, `guarantor_id`, `tenant_id` (le colocataire garanti, R8.3), `ends_on` null.

**`notices`** (congés) : `lease_id`, `lease_tenant_id` null (départ d'un seul signataire d'un bail unique), `given_by` (`tenant`, `landlord`), `received_on`, `reduced_reason` null (R1.1), `notice_months`, `effective_end_date`.

### 3.5 Loyers

**`rent_dues`** (échéances)
| Colonne | Type | Notes |
|---------|------|-------|
| `lease_id`, `tenant_id` | uuid FK | Une échéance par signataire (R4.5) |
| `kind` | enum `rent`, `charges_regularization`, `deposit_shortfall`, `other` | |
| `period_start` / `period_end` | date | Période couverte (prorata R2) |
| `due_date` | date | |
| `rent_cents` / `charges_cents` | integer | Séparés pour la quittance (R5.2) ; négatif autorisé pour un avoir |
| `total_cents` | integer, générée | `rent_cents + charges_cents` |
| `paid_cents` | integer | Somme des imputations, maintenue dans la transaction du paiement |
| `adjusted_reason` | text null | Si le montant calculé a été modifié (R2.2) |
| `cancelled_at` | timestamptz null | |

Unique `(lease_id, tenant_id, kind, period_start)` — rend la génération idempotente.
Le **statut** (`upcoming`, `pending`, `partial`, `paid`, `late`) n'est **pas stocké** : il est calculé par `domain.rentDueStatus(due, today, graceDays)`, et en SQL par une vue `rent_dues_with_status` pour filtrer.

**`payments`** (paiements) : `tenant_id`, `lease_id`, `amount_cents > 0`, `paid_on`, `method` (`transfer`, `cash`, `cheque`, `housing_benefit`, `other`), `note`, `idempotency_key` (unique par compte), `cancelled_at`.

**`payment_allocations`** (imputations) : `payment_id`, `rent_due_id`, `amount_cents > 0`. L'avance d'un colocataire = Σ paiements − Σ imputations (R4.4).

**`receipts`** (quittances et reçus)
| Colonne | Type | Notes |
|---------|------|-------|
| `rent_due_id` | uuid FK | |
| `kind` | enum `rent_receipt`, `payment_receipt` | Quittance / reçu (R5.3) |
| `number` | text | `Q-2026-000123` ou `R-2026-000045`, unique par compte |
| `document_id` | uuid FK | Le PDF figé (R5.5) |
| `snapshot` | jsonb | Données exactes ayant servi au PDF |
| `issued_at` / `voided_at` / `replaced_by_id` | | |

**`email_deliveries`** : `receipt_id`, `to_email`, `status` (`queued`, `sent`, `failed`), `provider_message_id`, `attempts`, `last_error`, `sent_at`.

### 3.6 Dépôt, charges, indexation

**`deposits`** (1–1 avec `leases`) : `expected_cents`, `received_cents`, `received_on`, `keys_returned_on`, `exit_inspection_conform` (boolean null), `refund_deadline` (calculée, R3.3), `withheld_condo_cents` (R3.6), `refunded_cents`, `refunded_on`.

**`deposit_deductions`** : `deposit_id`, `reason`, `amount_cents`, `document_id` null.

**`charge_expenses`** : `property_id`, `category` (liste du décret 87-713), `label`, `period_start`, `period_end`, `amount_cents`, `document_id` null.

**`charge_reconciliations`** : `lease_id`, `tenant_id`, `period_start`, `period_end`, `provisions_cents`, `actual_share_cents`, `balance_cents`, `status` (`draft`, `sent`, `applied`), `rent_due_id` null, `document_id` null.

**`rent_indices`** (globale, sans `account_id`) : `quarter` (PK, ex. `2026-T2`), `value numeric(7,2)`, `published_on`. Écriture réservée à l'administration (seed + import).

**`rent_revisions`** : `lease_id`, `effective_date`, `old_rent_cents`, `new_rent_cents`, `index_old_quarter`, `index_new_quarter`, `applied_at`.

### 3.7 Documents et audit

**`documents`**
| Colonne | Type | Notes |
|---------|------|-------|
| `property_id`, `lease_id`, `tenant_id`, `guarantor_id` | uuid null | **Exactement un** rattachement : `CHECK (num_nonnulls(property_id, lease_id, tenant_id, guarantor_id) = 1)` |
| `type` | enum | `signed_lease`, `entry_inspection`, `exit_inspection`, `insurance`, `identity`, `income_proof`, `address_proof`, `employment_proof`, `guarantee_deed`, `dpe`, `other_diagnostic`, `generated_receipt`, `generated_statement`, `other` |
| `storage_key` | text | `accounts/{accountId}/documents/{id}` — jamais dérivé du nom de fichier |
| `filename`, `mime_type`, `size_bytes`, `sha256` | | |
| `expires_at` | date null | DOC-3 |
| `status` | enum `pending_upload`, `ready` | |

**`audit_logs`** : `user_id`, `entity`, `entity_id`, `action` (`create`, `update`, `cancel`, `void`…), `before jsonb`, `after jsonb`, `created_at`. En écriture seule pour l'application (pas d'`UPDATE`/`DELETE`).

## 4. Index principaux

| Table | Index | Usage |
|-------|-------|-------|
| `rent_dues` | `(account_id, due_date)` | Vue du mois, tableau de bord |
| `rent_dues` | `(account_id, tenant_id, due_date)` | Solde et historique d'un colocataire |
| `leases` | `(account_id, status)` | Listes de baux |
| `documents` | `(account_id, expires_at) WHERE expires_at IS NOT NULL` | Alertes d'expiration |
| `payments` | `(account_id, paid_on)` | Export annuel |
| Toutes | `(account_id, id)` unique | Cible des FK composites |

## 5. Row-Level Security

Politique appliquée à **chaque** table métier (générée par une fonction utilitaire dans les migrations) :

```sql
ALTER TABLE rent_dues ENABLE ROW LEVEL SECURITY;
ALTER TABLE rent_dues FORCE ROW LEVEL SECURITY;

CREATE POLICY account_isolation ON rent_dues
  USING      (account_id = current_setting('app.account_id', true)::uuid)
  WITH CHECK (account_id = current_setting('app.account_id', true)::uuid);
```

- L'application se connecte avec un rôle **`app_user`**, qui n'est ni propriétaire des tables ni `BYPASSRLS`.
- Les migrations et les tâches système (anonymisation, génération des échéances pour tous les comptes) utilisent un rôle **`app_system`** avec `BYPASSRLS`. Elles itèrent compte par compte en positionnant `app.account_id` quand elles écrivent.
- Si `app.account_id` n'est pas défini, `current_setting(..., true)` renvoie `NULL` et **aucune ligne** n'est visible.

## 6. Clés étrangères composites

```sql
ALTER TABLE rooms ADD CONSTRAINT rooms_property_fk
  FOREIGN KEY (account_id, property_id) REFERENCES properties (account_id, id);
```
Un objet ne peut référencer qu'un objet **du même compte**, même en cas de bug applicatif.

## 7. Migrations et seeds

- `drizzle-kit generate` produit du SQL versionné dans `packages/db/migrations/`. Les migrations sont **relues** en revue de code (RLS, index, CHECK sont écrits à la main dans des migrations SQL personnalisées).
- Migrations **en avant uniquement**. On déploie en deux temps pour les changements cassants (ajouter, migrer les données, puis supprimer).
- `pnpm db:seed` : un compte de démonstration (les personas Claire et Marc), les valeurs IRL historiques.
