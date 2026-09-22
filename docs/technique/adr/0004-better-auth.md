# ADR-0004 — Authentification avec Better Auth

- **Statut** : accepté
- **Date** : 2026-09-22

## Contexte
Besoin : e-mail + mot de passe, vérification de l'e-mail, réinitialisation, sessions sûres ; plus tard 2FA et co-gestionnaires. Les données d'identité doivent rester dans l'UE.

## Décision
**Better Auth**, bibliothèque TypeScript open source, avec **sessions en base** (tables `auth_*` dans notre PostgreSQL, adaptateur Drizzle) et cookie `HttpOnly`. Montée sur `/api/auth/*` dans NestJS ; un garde NestJS lit la session.

## Alternatives écartées
- **Fournisseur hébergé (Clerk, Auth0)** : un sous-traitant de plus, données hors de notre base, coût.
- **Auth.js** : pensé pour Next.js, support e-mail + mot de passe limité.
- **Passport + développement maison** : trop de code sensible à écrire et maintenir.

## Conséquences
- Tables préfixées `auth_` pour éviter la collision avec `accounts` (compte propriétaire).
- L'intégration NestJS se fait à la main (ou avec un paquet communautaire) : à vérifier à chaque montée de version.
- 2FA et plugins disponibles plus tard sans changer de solution.
