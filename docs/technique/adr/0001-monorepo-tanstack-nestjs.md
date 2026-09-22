# ADR-0001 — Monorepo : TanStack Start (web) + NestJS (API)

- **Statut** : accepté (révisé le 2026-09-22 : TanStack Start remplace la SPA Vite)
- **Date** : 2026-09-22

## Contexte
Développeur solo, TypeScript de bout en bout. Des règles métier (prorata, plafonds, IRL) sont nécessaires côté client (aperçus) et côté serveur (autorité). Choix d'équipe : **TanStack Start** pour le web (SSR, routage typé, streaming) et **NestJS** pour l'API (modules, injection de dépendances, adapté à un domaine riche).

## Décision
- Monorepo **pnpm workspaces + Turborepo** : `apps/web` (TanStack Start), `apps/api` (NestJS), `packages/{domain,contracts,db,config}`.
- **Répartition des rôles** :
  - **NestJS est la seule autorité métier** : règles, persistance, authentification (Better Auth), documents, e-mails, tâches.
  - **TanStack Start est la couche de présentation** : SSR, routage, et un **BFF minimal** (lecture de session, proxy `/api`). Aucun accès à la base, aucune règle métier propre : le lint interdit à `apps/web` d'importer `packages/db`.
- **Même origine pour le navigateur** : le serveur Start est le seul point d'entrée public (`app.easycoloc.fr`) et **relaie `/api/*` vers NestJS** par une route serveur catch-all. Cookies de session `SameSite=Lax` simples, pas de CORS.
- **Appels isomorphes** : les `loader` s'exécutent sur le serveur Start au premier rendu (appel direct à l'URL interne de NestJS, avec les cookies de la requête transmis) et dans le navigateur lors des navigations (via `/api`).
- **Deux conteneurs** : `web` (Start) et `api` (NestJS + worker pg-boss).

## Alternatives écartées
- **SPA Vite servie par NestJS** (version initiale de cet ADR) : plus simple (un seul conteneur), mais pas de SSR. Écartée par choix d'équipe.
- **Sous-domaines séparés (`app.` / `api.`) avec cookies inter-sous-domaines et CORS** : fonctionne, mais multiplie les pièges (CORS avec credentials, domaine des cookies, CSRF).
- **Logique métier dans les server functions de Start** (sans NestJS) : deux lieux de vérité ; NestJS est retenu comme backend.
- **Reverse proxy dédié (Caddy) devant les deux services** : un composant de plus à héberger ; à reconsidérer si le proxy dans Start devient un goulot.

## Conséquences
- Un saut réseau supplémentaire pour les appels navigateur → API (Start → NestJS), négligeable sur un réseau privé.
- Le proxy doit transmettre fidèlement `Set-Cookie`, `X-Forwarded-For/Host/Proto` et le corps en flux (streaming) ; les fichiers ne transitent pas par lui (upload direct vers S3 par URL signée).
- NestJS n'est pas exposé publiquement : réseau privé, ou à défaut un secret partagé `X-Internal-Token` vérifié par un garde global.
- Les pages HTML rendues côté serveur contiennent des données personnelles : `Cache-Control: private, no-store`.
- Types et validations partagés sans publication de paquets ; frontières vérifiées par le lint.
