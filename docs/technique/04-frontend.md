# Frontend (`apps/web`) — TanStack Start

## 1. Stack

| Besoin | Choix | Pourquoi |
|--------|-------|----------|
| Framework | **TanStack Start** (React, build Vite) | SSR avec streaming, routes fichiers typées, server functions pour le BFF |
| Routage | **TanStack Router** (intégré à Start) | Paramètres et `search params` validés par Zod (filtres en URL), `loader` + `beforeLoad` |
| Données serveur | **TanStack Query** + intégration SSR Router/Query | Données préchargées au rendu serveur, puis hydratées ; cache et invalidation côté client |
| Formulaires | **TanStack Form** + validateurs Zod partagés (`packages/contracts`) | Mêmes règles que le serveur |
| Tableaux | **TanStack Table** | Grille des loyers : sélection multiple, tri, filtres |
| UI | **shadcn/ui** (Radix UI) + **Tailwind CSS v4** | Composants accessibles, code possédé, thème clair et sombre |
| Auth | Client **Better Auth** (`better-auth/react`) dans le navigateur ; lecture de session côté serveur via NestJS | L'authentification reste dans NestJS |
| Dates | `date-fns` + `@date-fns/tz` | Formatage `fr`, calculs `Europe/Paris` |
| Tests | Vitest + Testing Library ; Playwright (e2e) | Voir [07-tests-qualite.md](07-tests-qualite.md) |

> Rôle de Start : **présentation + BFF minimal**. Il ne se connecte **jamais** à la base et ne contient **aucune règle métier propre** (il importe seulement `packages/domain` pour les aperçus). Voir [ADR-0001](adr/0001-monorepo-tanstack-nestjs.md).
>
> Les noms exacts des API de TanStack Start (`createServerFn`, `createIsomorphicFn`, routes serveur, lecture des en-têtes de requête) évoluent entre versions : à vérifier avec la version installée.

## 2. Structure

```
apps/web/src/
├── routes/                       # Routes fichiers (pages + routes serveur)
│   ├── __root.tsx                # Document HTML, providers, gestion d'erreurs, nonce CSP
│   ├── api/$.ts                  # Route serveur catch-all : proxy /api/* → NestJS (§3)
│   ├── (public)/connexion.tsx · inscription.tsx · mot-de-passe-oublie.tsx
│   └── _app/                     # Layout authentifié (beforeLoad : session requise)
│       ├── index.tsx             # Tableau de bord (E1)
│       ├── loyers.tsx            # ?mois=2026-09&bien=…&statut=… (E2)
│       ├── biens/index.tsx · $bienId.tsx
│       ├── colocataires/index.tsx · $colocataireId.tsx
│       ├── baux/index.tsx · nouveau.tsx · $bailId.tsx · $bailId.depart.tsx
│       ├── documents.tsx
│       └── parametres.tsx
├── server/                       # Code exécuté uniquement sur le serveur Start
│   ├── session.ts                # getSession (server function) : transmet le cookie à NestJS
│   ├── api-proxy.ts              # Relais HTTP vers API_INTERNAL_URL
│   └── security-headers.ts       # CSP avec nonce, Cache-Control
├── features/                     # Un dossier par épopée : composants, hooks de requête, formulaires
│   ├── rent/  leases/  properties/  tenants/  documents/  deposits/  charges/  dashboard/
├── components/ui/                # shadcn/ui
├── lib/
│   ├── api.ts                    # Client ts-rest isomorphe (§4)
│   ├── auth-client.ts            # Client Better Auth (navigateur)
│   ├── format.ts                 # formatMoney(cents), formatDate(iso)…
│   └── query-keys.ts
└── router.tsx                    # Création du routeur + QueryClient par requête
```

Les **URL sont en français** (côté utilisateur) ; le code est en anglais.

## 3. Proxy `/api/*` (même origine)

```
Navigateur ── /api/v1/rent-dues ──► Start (routes/api/$.ts) ──► NestJS (API_INTERNAL_URL)/api/v1/rent-dues
           ◄─ réponse + Set-Cookie ─┘                        ◄─┘
```

Règles du relais :
- Transmet la méthode, le chemin, la query, le corps **en flux** et les en-têtes utiles (`cookie`, `content-type`, `idempotency-key`, `origin`), et ajoute `X-Forwarded-For`, `X-Forwarded-Host`, `X-Forwarded-Proto` et `X-Internal-Token`.
- Renvoie le statut, le corps et **tous les `Set-Cookie`** sans les modifier (sessions Better Auth).
- Ne met **rien** en cache ; délai d'expiration de 30 s ; en cas d'échec amont → `502` au format Problem Details.
- `/api/auth/*` passe par le même relais : le client Better Auth du navigateur parle à `/api/auth`, comme s'il était sur la même origine que NestJS.

## 4. Client d'API isomorphe

Un seul client ts-rest, dont la résolution diffère selon l'environnement d'exécution :

| Exécution | URL de base | Cookies |
|-----------|-------------|---------|
| Serveur Start (SSR, `loader` au premier rendu, server functions) | `API_INTERNAL_URL` (réseau privé) | En-tête `cookie` **de la requête entrante** transmis, plus `X-Internal-Token` |
| Navigateur (navigation, mutations) | `/api` (même origine → proxy) | `credentials: 'include'` |

Implémenté avec `createIsomorphicFn()` (ou équivalent) dans `lib/api.ts`. Les `loader` utilisent `queryClient.ensureQueryData(...)` : les données chargées côté serveur sont sérialisées dans le HTML et hydratées dans le cache TanStack Query. Au premier rendu, aucune requête n'est refaite dans le navigateur.

**Un `QueryClient` par requête** côté serveur (jamais partagé entre utilisateurs), créé dans `router.tsx`.

## 5. Authentification côté Start

- `routes/_app` → `beforeLoad` appelle la server function `getSession()`. Celle-ci relaie le cookie à `GET {API_INTERNAL_URL}/api/auth/get-session`. Sans session → `redirect({ to: '/connexion', search: { redirect } })`.
- Le résultat (utilisateur, compte, rôle) est placé dans le **contexte du routeur**, disponible pour toutes les routes enfants.
- Connexion et déconnexion : client Better Auth dans le navigateur (via le proxy), puis `router.invalidate()`.
- Start ne vérifie ni ne signe jamais les sessions lui-même : **NestJS fait autorité**.

## 6. Données et cache

- Un hook par opération dans `features/*/queries.ts` : `useRentDues(month, filters)`, `useRecordPayment()`, et les `queryOptions` réutilisées par les `loader`.
- Clés hiérarchiques : `['rent-dues', { month, propertyId, status }]`, `['tenants', id, 'ledger']`.
- **Invalidation après mutation** : un paiement invalide `rent-dues`, `dashboard` et `tenants/:id/ledger`.
- **Optimiste** uniquement pour « Marquer payé », avec retour arrière si erreur, et un toast « Annuler » (10 s).
- Chaque `POST /payments` porte une `Idempotency-Key` (`crypto.randomUUID()`) : un double tap sur mobile ne crée pas deux paiements.
- Mutations : toujours depuis le navigateur via le proxy (pas de server function qui écrit), pour un seul chemin d'écriture vers NestJS.

## 7. Règles métier côté client

Le web importe `@easycoloc/domain` pour les **aperçus instantanés** : prorata dans l'assistant de bail, plafond du dépôt sous le champ, date de fin d'un congé, nouveau loyer IRL. **NestJS recalcule toujours** et fait autorité.

## 8. Formats et textes

- Montants : `Intl.NumberFormat('fr-FR', { style: 'currency', currency: 'EUR' })` à partir des centimes. Saisie tolérante (`480`, `480,5`, `480.50`) convertie par `parseMoneyInput`.
- **Fuseau fixé à `Europe/Paris`** pour tout formatage, côté serveur comme côté navigateur : évite les écarts d'hydratation et les dates décalées.
- Chiffres tabulaires (`tabular-nums`) dans les colonnes de montants.
- Textes en français, centralisés par fonctionnalité (`features/*/messages.ts`). Pas de bibliothèque i18n au MVP.
- Erreurs métier : table `code → message` dans `lib/errors.ts`, avec un repli sur `detail`.

## 9. Accessibilité et responsive

- Composants Radix ; `eslint-plugin-jsx-a11y` ; `@axe-core/playwright` en e2e.
- Mobile (< 768 px) : onglets en bas, cartes à la place des tableaux. Bureau (≥ 1024 px) : barre latérale.
- Le SSR affiche le contenu avant l'hydratation : les actions critiques (liens, formulaires) restent utilisables le plus tôt possible.

## 10. Performance et en-têtes

- SSR en **streaming** : l'en-tête et les KPI du tableau de bord s'affichent pendant que les listes se chargent (`Suspense`).
- Découpage du code par route (automatique).
- HTML : `Cache-Control: private, no-store` (données personnelles). Fichiers statiques avec empreinte : `public, max-age=31536000, immutable`.
- Objectif : premier affichage utile du tableau de bord < 1,5 s en 4G, moins de 200 ko de JS compressé.
