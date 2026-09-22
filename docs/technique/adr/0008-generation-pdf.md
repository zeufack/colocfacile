# ADR-0008 — Génération des PDF côté serveur avec pdfmake

- **Statut** : accepté
- **Date** : 2026-09-22

## Contexte
Quittances, reçus, décomptes de dépôt et de régularisation : mise en page simple et tabulaire, contenu légal obligatoire, documents figés et archivés.

## Décision
**pdfmake** dans l'API : chaque modèle est une fonction TypeScript `snapshot → docDefinition`. Polices embarquées (sans-serif libre). Le PDF est stocké dans le stockage objet et référencé par `documents` ; le `snapshot` JSON est conservé.

## Alternatives écartées
- **Navigateur sans interface (Puppeteer/Playwright)** : image lourde (Chromium), mémoire, démarrage lent.
- **@react-pdf/renderer** : agréable, mais ESM uniquement et rendu React côté serveur NestJS ; à reconsidérer si les modèles se complexifient.

## Conséquences
- Modèles testables par instantané du `docDefinition` et par extraction du texte.
- Mise en page moins riche qu'en HTML/CSS : suffisante pour des documents administratifs.
