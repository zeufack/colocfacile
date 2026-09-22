# Architecture

> Statut : brouillon v1 · 22/09/2026 · Décisions détaillées dans [`adr/`](adr/)

## 1. Vue d'ensemble

EasyColoc est une application web **TypeScript de bout en bout**, organisée en **monorepo** :

- **Web** : **TanStack Start** (React, SSR en streaming) avec Router, Query, Form et Table. C'est le point d'entrée public ; il relaie `/api/*` vers NestJS et ne contient aucune logique métier.
- **Backend** : API REST **NestJS**, qui porte l'autorité métier, l'authentification, la persistance, les documents, les e-mails et les tâches planifiées. Non exposée publiquement.
- **Domaine partagé** : les règles métier (prorata, plafonds, préavis, IRL, statuts…) sont des **fonctions pures** dans un paquet partagé. Le backend s'en sert pour faire autorité, le frontend pour les aperçus en direct.
- **Données** : PostgreSQL (Drizzle ORM), fichiers dans un stockage objet S3, e-mails transactionnels. Tout est hébergé **en France** chez Scaleway.

## 2. Contexte (C4 — niveau 1)

```
                 ┌──────────────────┐
                 │   Propriétaire   │  navigateur (ordinateur / mobile)
                 └────────┬─────────┘
                          │ HTTPS
                          ▼
                 ┌──────────────────┐        ┌─────────────────────────┐
                 │    EasyColoc     │───────►│ Scaleway TEM (e-mails)  │──► Colocataire
                 │ (web Start + API)│        └─────────────────────────┘    (quittances)
                 └──┬────────────┬──┘
                    │            │
          ┌─────────▼───┐   ┌────▼──────────────┐
          │ PostgreSQL  │   │ Object Storage S3 │
          │ (managé)    │   │ (documents, PDF)  │
          └─────────────┘   └───────────────────┘
```

## 3. Conteneurs (C4 — niveau 2)

```
                         https://app.easycoloc.fr  (seul point d'entrée public)
                                       │
┌──────────────── Conteneur « web » ───▼──────────────────────┐
│  TanStack Start (Node)                                      │
│  ├─ SSR des pages (Router + Query, streaming)               │
│  ├─ server functions BFF : getSession                       │
│  └─ routes/api/$  : proxy /api/* ───────────────┐           │
└─────────────────────────────────────────────────┼───────────┘
            loaders SSR : appel direct + cookie   │ réseau privé + X-Internal-Token
                                                  ▼
┌──────────────── Conteneur « api » (non public) ─────────────┐
│  NestJS                                                     │
│  ├─ /api/auth/*  → Better Auth                              │
│  ├─ /api/v1/*    → modules métier                           │
│  └─ worker pg-boss (tâches planifiées, e-mails)             │
└──────────────┬──────────────────────────────────────────────┘
               │
   ┌───────────┼─────────────────────┬──────────────────┬────────────────────┐
   ▼                                 ▼                  ▼                    ▼
PostgreSQL managé             Object Storage       Scaleway TEM        Cockpit / Sentry
(données + file pg-boss)      (bucket privé) ◄──── upload/download direct du navigateur (URL signées)
```

**Pourquoi un proxy dans Start ?** Le navigateur ne voit qu'une origine (`app.easycoloc.fr`) : pas de CORS, cookies de session `SameSite=Lax` simples, NestJS reste privé. Les fichiers ne passent pas par le proxy : ils vont directement vers S3 par URL signée. Le worker tourne dans le conteneur `api` (`APP_ROLE=all`) ; on pourra le séparer plus tard (`APP_ROLE=api` et `APP_ROLE=worker`, même image). Voir [ADR-0001](adr/0001-monorepo-tanstack-nestjs.md).

## 4. Organisation du monorepo

```
easycoloc/
├── apps/
│   ├── web/                    # TanStack Start (SSR, BFF minimal, proxy /api)
│   └── api/                    # NestJS (API + worker)
├── packages/
│   ├── domain/                 # Règles métier pures (R1…R10), sans I/O — testées à 100 %
│   ├── contracts/              # Schémas Zod + contrat d'API (ts-rest) partagés front/back
│   ├── db/                     # Schéma Drizzle, migrations, seeds, politiques RLS
│   └── config/                 # tsconfig, ESLint, Prettier partagés
├── docs/                       # Cette documentation
├── docker/                     # Dockerfile, compose de dev
├── pnpm-workspace.yaml
└── turbo.json
```

**Règles de dépendance** (vérifiées par ESLint `import/no-restricted-paths` ou dependency-cruiser) :

```
apps/web  ──►  contracts, domain
apps/api  ──►  contracts, domain, db
contracts ──►  domain (types de valeurs uniquement)
db        ──►  (rien d'interne)
domain    ──►  (rien : zéro dépendance I/O)
```

Le web n'importe **jamais** `db` : même ses server functions passent par l'API NestJS. Le domaine n'importe **jamais** NestJS, Drizzle ou React.

## 5. Architecture du backend (NestJS)

Un module NestJS par contexte métier ; chaque module suit la même découpe en couches :

```
Controller (ts-rest)  →  Service applicatif  →  Domaine (packages/domain)
      │                        │
  validation Zod          Repository (Drizzle, transaction scopée au compte)
```

| Module | Responsabilité | Épopées |
|--------|----------------|---------|
| `auth` | Montage de Better Auth, garde de session | AUTH |
| `accounts` | Compte, membres, paramètres, export et suppression RGPD | AUTH |
| `landlords` | Bailleurs (personne physique ou SCI) | BIEN-1 |
| `properties` | Biens et chambres | BIEN |
| `tenants` | Colocataires, garants | COLOC |
| `leases` | Baux, signataires, congés, avenants | BAIL |
| `rent` | Échéances, paiements, imputations | LOYER |
| `receipts` | Quittances, reçus (PDF), numérotation | LOYER-6 |
| `notifications` | Envoi d'e-mails, suivi des envois | LOYER-7 |
| `deposits` | Dépôts de garantie, retenues, restitution | DEPOT |
| `charges` | Dépenses réelles, régularisation | CHARGE |
| `indexation` | IRL, révisions | IRL |
| `documents` | Upload, stockage, pièces obligatoires | DOC |
| `dashboard` | Agrégats du mois, liste « à faire » | DASH |
| `audit` | Journal des modifications | Traçabilité |
| `jobs` | Enregistrement des tâches pg-boss | — |

Voir [05-backend.md](05-backend.md).

## 6. Isolation des données par compte

Décision produit : chaque propriétaire a son **compte** (`accounts`), étanche aux autres. On applique **deux couches** de défense ([ADR-0003](adr/0003-isolation-par-compte-rls.md)) :

1. **Applicative** : chaque requête authentifiée résout son `accountId` ; les repositories l'exigent et filtrent systématiquement.
2. **Base de données** : Row-Level Security PostgreSQL. Chaque transaction exécute `SET LOCAL app.account_id = '<uuid>'`, et les politiques RLS filtrent toutes les tables métier. Une requête oubliée ne renvoie rien, au lieu de fuiter.

En plus, des **clés étrangères composites** `(account_id, id)` empêchent de relier deux objets de comptes différents.

## 7. Parcours d'une requête

**Premier chargement (SSR)** — `GET /loyers?mois=2026-09`
```
Navigateur ──► Start : route _app/loyers
                beforeLoad → getSession() → NestJS /api/auth/get-session (cookie transmis)
                loader     → ensureQueryData → client isomorphe → NestJS /api/v1/rent-dues (cookie transmis)
◄── HTML rendu en streaming + état TanStack Query sérialisé (hydratation sans nouvelle requête)
```

**Navigation ou mutation (navigateur)** — `GET /api/v1/rent-dues?month=2026-09`
```
Navigateur ──► Start routes/api/$ (proxy) ──► NestJS
   SessionGuard      → auth.api.getSession(headers) → user
   AccountGuard      → membership(user) → accountId → stocké en CLS (AsyncLocalStorage)
   ZodValidation     → query validée par le contrat ts-rest
   RentController    → RentService.listForMonth(month, filters)
   RentService       → db.tx(accountId) : SET LOCAL app.account_id ; SELECT … (RLS active)
                     → domain.rentDueStatus(…) pour chaque échéance
◄── 200 JSON typé (contrat partagé), relayé tel quel par le proxy
```

## 8. Tâches asynchrones (pg-boss)

La file de tâches vit dans PostgreSQL : pas de Redis à opérer ([ADR-0006](adr/0006-taches-pg-boss.md)).

| Tâche | Planification | Idempotence |
|-------|---------------|-------------|
| `rent-dues.generate` | Tous les jours à 02:00 Europe/Paris | Contrainte unique `(lease_id, tenant_id, period_start, kind)` |
| `email.send-receipt` | À la demande | Clé = `receipt_id` ; réessais avec délai croissant (5 max) |
| `privacy.anonymize` | Tous les jours à 03:00 | Ne traite que les colocataires éligibles non encore anonymisés (R9.3) |
| `account.export` | À la demande | Un export en cours par compte |

Les **alertes** du tableau de bord (retards, pièces manquantes, IRL, dépôts) sont **calculées à la lecture**, pas stockées : c'est plus simple, et toujours juste.

## 9. Décisions d'architecture (ADR)

| ADR | Décision |
|-----|----------|
| [0001](adr/0001-monorepo-tanstack-nestjs.md) | Monorepo pnpm + Turborepo, TanStack Start (web, proxy /api) + API NestJS privée |
| [0002](adr/0002-postgresql-drizzle.md) | PostgreSQL + Drizzle ORM |
| [0003](adr/0003-isolation-par-compte-rls.md) | Isolation par compte : `account_id` + RLS + FK composites |
| [0004](adr/0004-better-auth.md) | Authentification avec Better Auth, sessions en base |
| [0005](adr/0005-hebergement-scaleway.md) | Hébergement Scaleway (France) |
| [0006](adr/0006-taches-pg-boss.md) | Tâches asynchrones avec pg-boss |
| [0007](adr/0007-montants-en-centimes-et-dates.md) | Montants en centimes entiers, dates civiles sans fuseau |
| [0008](adr/0008-generation-pdf.md) | Génération des PDF côté serveur avec pdfmake |
| [0009](adr/0009-contrat-api-ts-rest.md) | Contrat d'API partagé avec ts-rest + Zod |
