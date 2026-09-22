# Tests et qualité

## 1. Stratégie

```
            ▲  e2e Playwright (parcours P1–P5)            ~10 scénarios, lents, critiques
           ╱ ╲
          ╱   ╲ Intégration API (NestJS + Postgres réel)  par cas d'usage + isolation
         ╱     ╲
        ╱       ╲ Composants (Testing Library)            formulaires, grille des loyers
       ╱─────────╲
      ╱  Domaine  ╲ Unitaires + propriétés (fast-check)   règles R1–R10 — la base de la pyramide
     ╱─────────────╲
```

| Niveau | Outil | Portée | Objectif |
|--------|-------|--------|----------|
| Domaine | Vitest + **fast-check** | `packages/domain` | **100 %** des branches ; chaque règle `Rx` a ses cas nominaux, limites et exemples de la documentation produit |
| Intégration API | Vitest + **Testcontainers** (PostgreSQL) + Supertest | `apps/api` | Chaque cas d'usage, les erreurs métier (`code`), l'isolation entre comptes, RLS actif |
| Composants | Vitest + Testing Library + MSW | `apps/web` | Formulaires (validation, erreurs serveur), grille des loyers, états vides |
| Proxy et SSR | Vitest | `apps/web/src/server` | Le proxy relaie statut, corps et **tous les `Set-Cookie`** ; ajoute `X-Forwarded-*` ; renvoie 502 si l'API est injoignable ; `getSession` redirige sans session |
| e2e | **Playwright** + `@axe-core/playwright` | App complète (docker compose) | Parcours P1 à P5, accessibilité des écrans clés, un profil mobile |
| PDF | Vitest | `receipts` | Instantané du `docDefinition` + extraction du texte du PDF rendu (mentions obligatoires R5.2) |

## 2. Tests du domaine — exemples

```ts
describe('R2 — prorate', () => {
  it('entrée le 12 octobre : 20 jours sur 31', () => {
    expect(prorate(cents(50_000), '2026-10-12', '2026-10-31')).toBe(32_258);
  });

  it('mois complet = montant mensuel', () => {
    fc.assert(fc.property(arbCents, arbMonth, (m, month) =>
      prorate(m, month.first, month.last) === m));
  });
});

describe('R4.5 — splitByShares', () => {
  it('la somme des parts est toujours égale au total', () => {
    fc.assert(fc.property(arbCents, arbShares, (total, shares) =>
      sum(splitByShares(total, shares)) === total));
  });
});
```

**Propriétés à couvrir absolument** : conservation des montants (répartition, imputation, régularisation) ; `rentDueStatus` monotone dans le temps ; `depositCap` jamais dépassé ; `noticeEndDate` toujours postérieure à `receivedOn`.

## 3. Pratique de développement

- **TDD pour le domaine** : un test par exemple chiffré de [06-regles-metier.md](../private/produit/06-regles-metier.md), écrit avant le code.
- Les tests d'intégration utilisent une base **par fichier de test** (template Postgres cloné) : rapides et isolés.
- Horloge injectée (`Clock`) : aucun test ne dépend de la date réelle.
- Données de test construites par des *builders* (`aLease().furnished().startingOn('2026-10-12').build()`).

## 4. Qualité du code

| Outil | Règle |
|-------|-------|
| TypeScript | `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` |
| ESLint | `typescript-eslint` (type-checked), `import/no-restricted-paths` (frontières du monorepo), `jsx-a11y`, interdiction de `sql.raw` et de `Date.now()` hors `Clock` |
| Prettier | Formatage automatique |
| Commits | Conventional Commits (`feat(rent): …`), vérifiés par commitlint |
| Hooks | `lefthook` : lint et type-check sur les fichiers modifiés avant commit |

## 5. Intégration continue (GitHub Actions)

Sur chaque pull request :
1. `pnpm install --frozen-lockfile`
2. `turbo run lint typecheck test` (avec cache Turborepo)
3. Tests d'intégration (Testcontainers)
4. Build des images Docker `web` et `api`, puis e2e Playwright sur les deux conteneurs
5. `gitleaks`, `pnpm audit --prod`, CodeQL
6. Vérification des migrations : application sur une base vide + `drizzle-kit check`

**Définition de « terminé »** pour une story : critères d'acceptation couverts par des tests, règles `Rx` citées dans les tests, documentation mise à jour si le comportement change, revue faite, CI verte.
