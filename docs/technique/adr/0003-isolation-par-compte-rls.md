# ADR-0003 — Isolation des données par compte : `account_id` + RLS + FK composites

- **Statut** : accepté
- **Date** : 2026-09-22

## Contexte
Décision produit : chaque propriétaire a des données étanches dès le MVP, pour permettre un SaaS plus tard sans migration. Les données (pièces d'identité, revenus) sont sensibles : une fuite entre comptes est l'incident le plus grave possible.

## Décision
Base partagée, schéma partagé, avec **trois couches** :
1. `account_id NOT NULL` sur toute table métier ; `accountId` résolu depuis la session, jamais depuis la requête.
2. **Row-Level Security** PostgreSQL (`FORCE`), avec `set_config('app.account_id', …, true)` dans chaque transaction ; l'application se connecte avec un rôle sans `BYPASSRLS`.
3. **Clés étrangères composites** `(account_id, x_id)` : impossible de relier deux comptes.

## Alternatives écartées
- **Une base ou un schéma par compte** : complexité opérationnelle (migrations × N) disproportionnée.
- **Filtrage applicatif seul** : un oubli de `WHERE` suffit à fuiter.

## Conséquences
- Chaque requête passe par une transaction (léger surcoût, acceptable).
- Les tâches multi-comptes utilisent un rôle système distinct, dont l'usage est limité et contrôlé par le lint.
- Tests d'isolation obligatoires pour chaque route (voir sécurité §6).
