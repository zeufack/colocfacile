# EasyColoc

**Gestionnaire de colocation open source pour les propriétaires bailleurs en France.**

EasyColoc aide un propriétaire qui loue en colocation à savoir, d'un coup d'œil, **qui a payé, qui doit quoi, et quels documents sont à jour**, sans tableur ni classeur papier.

> 🚧 **Statut : conception.** La documentation produit et technique est prête ; le développement du MVP démarre. Aucune version utilisable pour l'instant.

---

## Pourquoi

Un logement en colocation, c'est plusieurs colocataires, chacun avec ses dates d'entrée et de sortie, sa quote-part, son dépôt de garantie, son garant et ses documents. Les logiciels de gestion locative classiques pensent « un logement = un locataire ». EasyColoc part de la **chambre** et du **colocataire**.

## Fonctionnalités du MVP

| Domaine | Ce que fait EasyColoc |
|---------|-----------------------|
| 🏠 **Biens et chambres** | Biens, chambres, statut d'occupation, bailleur (personne physique ou SCI) |
| 👥 **Colocataires** | Fiches colocataires et garants, solde, historique |
| 📄 **Baux** | Bail unique ou baux individuels ; meublé, vide, étudiant, mobilité ; congés et préavis calculés |
| 📎 **Documents** | Pièces rattachées au bail, au bien ou au colocataire ; checklist des pièces obligatoires ; alertes d'expiration |
| 💶 **Loyers** | Échéances générées automatiquement, prorata, paiements, retards |
| 🧾 **Quittances** | Quittances et reçus en PDF, envoi par e-mail |
| ⚖️ **Charges et dépôt** | Provisions ou forfait, régularisation annuelle, dépôt de garantie et restitution |
| 📈 **Révision IRL** | Calcul et rappel de la révision annuelle du loyer |
| 📊 **Tableau de bord** | Le mois en cours en un écran, et la liste des actions à faire |

Hors MVP (plus tard) : portail colocataire, demandes de maintenance, messagerie, rapprochement bancaire, signature électronique.

## Stack technique

| Couche | Choix |
|--------|-------|
| Langage | TypeScript de bout en bout |
| Web | [TanStack Start](https://tanstack.com/start) (Router, Query, Form, Table), shadcn/ui, Tailwind CSS |
| API | [NestJS](https://nestjs.com), contrat partagé [ts-rest](https://ts-rest.com) + Zod |
| Données | PostgreSQL + [Drizzle ORM](https://orm.drizzle.team), isolation par compte (Row-Level Security) |
| Authentification | [Better Auth](https://www.better-auth.com) |
| Tâches | pg-boss |
| Monorepo | pnpm workspaces + Turborepo |
| Tests | Vitest, fast-check, Testcontainers, Playwright |

Organisation prévue du dépôt :

```
apps/web          TanStack Start (SSR, interface)
apps/api          NestJS (API, règles métier, tâches)
packages/domain   Règles métier pures (prorata, plafonds, préavis, IRL…)
packages/contracts Contrat d'API partagé (Zod + ts-rest)
packages/db       Schéma Drizzle, migrations
docs/             Documentation
```

## Documentation

La documentation est en français, dans [`docs/`](docs/README.md) :

- **Produit** : [glossaire](docs/produit/03-glossaire.md) · [parcours utilisateurs](docs/produit/04-parcours-utilisateurs.md) · [exigences fonctionnelles](docs/produit/05-exigences-fonctionnelles.md) · [UX & écrans](docs/produit/07-ux-ecrans.md)
- **Technique** : [architecture](docs/technique/01-architecture.md) · [modèle de données](docs/technique/02-modele-de-donnees.md) · [API](docs/technique/03-api.md) · [frontend](docs/technique/04-frontend.md) · [backend](docs/technique/05-backend.md) · [tests](docs/technique/07-tests-qualite.md) · [conventions](docs/technique/09-conventions.md)
- **Décisions d'architecture** : [ADR](docs/technique/adr/)

## Avancement

Le travail est découpé en user stories, suivies dans les [issues](https://github.com/zeufack/colocfacile/issues) et regroupées en quatre jalons :

| Jalon | Contenu |
|-------|---------|
| [J1 — Fondations](https://github.com/zeufack/colocfacile/milestones) | Compte, biens, chambres, colocataires |
| [J2 — Baux & documents](https://github.com/zeufack/colocfacile/milestones) | Baux, congés, documents, pièces obligatoires |
| [J3 — Loyers](https://github.com/zeufack/colocfacile/milestones) | Échéances, paiements, quittances, tableau de bord |
| [J4 — Fin de cycle](https://github.com/zeufack/colocfacile/milestones) | Charges, dépôt de garantie, IRL, envoi des quittances par e-mail → **MVP** |

## Démarrage

Le code n'existe pas encore. Les instructions d'installation arriveront avec le jalon J1.

## Contribuer

Les suggestions sont bienvenues via les [issues](https://github.com/zeufack/colocfacile/issues). Avant de proposer du code, lisez le [glossaire](docs/produit/03-glossaire.md) (le vocabulaire du domaine est aussi celui du code) et les [conventions](docs/technique/09-conventions.md).

## Avertissement

EasyColoc applique des règles issues du droit français de la location (loi du 6 juillet 1989, loi ALUR…). Ces règles servent de base de conception et **ne constituent pas un conseil juridique**.

## Licence

À définir.
