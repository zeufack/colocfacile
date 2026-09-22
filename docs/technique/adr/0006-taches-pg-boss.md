# ADR-0006 — Tâches asynchrones et planifiées avec pg-boss

- **Statut** : accepté
- **Date** : 2026-09-22

## Contexte
Besoins : génération quotidienne des échéances, envoi d'e-mails avec réessais, anonymisation RGPD, exports. Faible volume.

## Décision
**pg-boss** : file de tâches et planification (cron) stockées dans PostgreSQL, exécutées par le processus NestJS (`APP_ROLE=all|worker`).

## Alternatives écartées
- **BullMQ + Redis** : un service de plus à héberger et sauvegarder.
- **Cron de l'hébergeur appelant un endpoint** : pas de réessais ni de visibilité.

## Conséquences
- Une mise en file peut participer à la **même transaction** que l'écriture métier (pas de tâche fantôme).
- Toutes les tâches doivent être idempotentes.
- Surveiller la table de la file ; archivage automatique configuré.
