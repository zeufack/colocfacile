# ADR-0009 — Contrat d'API partagé avec ts-rest + Zod

- **Statut** : accepté
- **Date** : 2026-09-22

## Contexte
Front et back en TypeScript dans le même dépôt : on veut des appels typés de bout en bout, une validation unique, et une API REST lisible (et documentable en OpenAPI).

## Décision
Contrat **ts-rest** dans `packages/contracts` (schémas Zod) ; implémentation côté NestJS avec `@ts-rest/nest` ; client typé côté web, appelé depuis les hooks TanStack Query ; OpenAPI générée depuis le contrat.

## Alternatives écartées
- **tRPC** : moins naturel avec NestJS, API non REST.
- **OpenAPI d'abord + génération de code** : plus de cérémonie pour un seul client.
- **DTO class-validator de NestJS** : validations non partageables avec le front.

## Conséquences
- Un changement de contrat casse la compilation des deux côtés (voulu).
- Les routes Better Auth (`/api/auth/*`) restent hors contrat.
- Surveiller la compatibilité de ts-rest avec les versions de NestJS et de Zod.
