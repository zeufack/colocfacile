# Documentation EasyColoc

EasyColoc est une application web qui aide un propriétaire bailleur à gérer ses colocations en France : biens, chambres, colocataires, baux, documents, loyers et charges.

**Stack** : monorepo pnpm/Turborepo · TanStack Start (Router, Query, Form, Table) · NestJS · PostgreSQL + Drizzle · Better Auth · pg-boss.

## Documentation publique

| # | Document | Rôle |
|---|----------|------|
| P3 | [Glossaire du domaine](produit/03-glossaire.md) | Le vocabulaire commun (produit + code) |
| P4 | [Parcours utilisateurs](produit/04-parcours-utilisateurs.md) | Les parcours clés, étape par étape |
| P5 | [Exigences fonctionnelles](produit/05-exigences-fonctionnelles.md) | Épopées, user stories, critères d'acceptation |
| P7 | [UX & écrans](produit/07-ux-ecrans.md) | Principes de design, navigation, maquettes filaires |
| T1 | [Architecture](technique/01-architecture.md) | Vue d'ensemble, monorepo, modules, isolation par compte, tâches |
| T2 | [Modèle de données](technique/02-modele-de-donnees.md) | Tables, contraintes, RLS, conventions |
| T3 | [API](technique/03-api.md) | Principes REST, erreurs, ressources, contrat ts-rest |
| T4 | [Frontend](technique/04-frontend.md) | TanStack Start : SSR, proxy /api, client isomorphe, cache |
| T5 | [Backend et domaine](technique/05-backend.md) | NestJS, Better Auth, transactions, règles pures, cas d'usage |
| T7 | [Tests et qualité](technique/07-tests-qualite.md) | Pyramide de tests, CI, définition de « terminé » |
| T9 | [Conventions](technique/09-conventions.md) | Langue, nommage, TypeScript, Git |
| ADR | [Décisions d'architecture](technique/adr/) | Décisions structurantes + modèle |

## Documentation privée (non publiée)

Le dossier `docs/private/` est **exclu du dépôt** (`.gitignore`) : il n'existe que sur les postes de l'équipe. Les liens ci-dessous ne fonctionnent qu'en local.

| # | Document | Pourquoi privé |
|---|----------|----------------|
| P0 | [Tableau des objectifs](private/00-tableau-objectifs.md) | Stratégie, hypothèses, jalons |
| P1 | [Vision produit](private/produit/01-vision-produit.md) | Positionnement |
| P2 | [Personas](private/produit/02-personas.md) | Stratégie |
| P6 | [Règles métier](private/produit/06-regles-metier.md) | Cadre juridique pas encore relu par un professionnel |
| P8 | [Feuille de route](private/produit/08-feuille-de-route.md) | Stratégie |
| T6 | [Sécurité et RGPD](private/technique/06-securite-rgpd.md) | Modèle de menaces et défenses détaillées |
| T8 | [Infrastructure et déploiement](private/technique/08-infrastructure-deploiement.md) | Topologie, ressources, sauvegardes, budget |

> ⚠️ Ces fichiers ne sont **ni versionnés ni sauvegardés par git**. Il faut les sauvegarder ailleurs (dépôt privé, drive chiffré…).

## Comment lire cette documentation

- **Pour concevoir** : parcours utilisateurs, puis UX & écrans.
- **Pour développer** : glossaire, exigences fonctionnelles, puis architecture. Les termes du glossaire sont ceux qu'on utilise dans le code.
