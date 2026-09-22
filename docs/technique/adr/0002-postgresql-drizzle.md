# ADR-0002 — PostgreSQL + Drizzle ORM

- **Statut** : accepté
- **Date** : 2026-09-22

## Contexte
Données relationnelles fortement contraintes (baux, échéances, imputations), argent, besoin d'isolation stricte par compte, contraintes d'exclusion (chevauchement de baux).

## Décision
**PostgreSQL 16+** et **Drizzle ORM** (`node-postgres`), avec migrations SQL versionnées générées par `drizzle-kit` et complétées à la main (RLS, CHECK, EXCLUDE).

## Alternatives écartées
- **Prisma** : plus de magie, moins de contrôle sur le SQL, intégration RLS et `SET LOCAL` par transaction moins naturelle.
- **Kysely** : excellent, mais Drizzle fournit en plus le schéma déclaratif et les migrations.

## Conséquences
- Le schéma TypeScript est la source de vérité ; le SQL reste lisible en revue.
- Fonctionnalités avancées (RLS, `btree_gist`) dans des migrations SQL personnalisées, à documenter.
