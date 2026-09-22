# ADR-0007 — Montants en centimes entiers, dates civiles sans fuseau

- **Statut** : accepté
- **Date** : 2026-09-22

## Contexte
Erreurs d'arrondi interdites (quittances, dépôts, régularisations). Les dates de bail et d'échéance sont des **dates civiles** françaises, pas des instants.

## Décision
- Argent : `integer` en **centimes** partout (base, API, domaine), type de marque `Cents`. Arrondi au plus proche (demi vers le haut), une seule fois, en fin de calcul. Répartitions : reste d'arrondi affecté au dernier élément.
- Pourcentages : points de base (`10000 = 100 %`).
- Dates civiles : colonnes `date`, chaînes `YYYY-MM-DD` (`IsoDate`) en TypeScript ; « aujourd'hui » calculé en `Europe/Paris` par un `Clock` injecté.
- Instants : `timestamptz`, ISO 8601 UTC.

## Alternatives écartées
- `numeric` / décimales : conversions en chaîne, risque d'arithmétique flottante côté JS.
- Bibliothèque monétaire (Dinero.js) : inutile pour une seule devise.
- `Date` JS pour les dates civiles : décalages de fuseau garantis.

## Conséquences
- Formatage uniquement à l'affichage (`Intl.NumberFormat('fr-FR')`).
- Plafond `integer` : 21 474 836,47 € par montant, largement suffisant.
