# ADR-0005 — Hébergement chez Scaleway (France)

- **Statut** : accepté
- **Date** : 2026-09-22

## Contexte
RGPD : données personnelles sensibles, hébergement souhaité en France. Petite équipe : services managés privilégiés. Besoins : conteneur, PostgreSQL, stockage S3, e-mails transactionnels.

## Décision
**Scaleway, région fr-par** : Serverless Containers, Managed Database for PostgreSQL, Object Storage, Transactional Email, Container Registry, Secret Manager, Cockpit.

## Alternatives écartées
- **OVHcloud** : offre équivalente ; Scaleway retenu pour ses conteneurs serverless et son service d'e-mails transactionnels intégré.
- **VPS auto-géré** : moins cher, mais sauvegardes, mises à jour et sécurité à notre charge.
- **Fournisseurs américains en région UE** : exposition au droit extraterritorial, moins aligné avec le positionnement.

## Conséquences
- Code du stockage et des e-mails derrière des interfaces (`FileStorage`, `Mailer`) : S3 standard, et SMTP en repli.
- Conteneur à 1 instance minimum pour le worker (pas de mise à zéro).
