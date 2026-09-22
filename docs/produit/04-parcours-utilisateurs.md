# Parcours utilisateurs

Les parcours ci-dessous décrivent les moments clés de Claire (persona principal). Chaque étape renvoie aux user stories des [exigences fonctionnelles](05-exigences-fonctionnelles.md).

---

## P1 — Premiers pas : de zéro à « mes loyers du mois »

**Déclencheur** : Claire crée son compte. **Objectif** : voir l'état de ses loyers en moins de 15 minutes.

| # | Étape | Ce que voit ou fait Claire | Moment clé | Stories |
|---|-------|----------------------------|------------|---------|
| 1 | Inscription | E-mail et mot de passe, ou lien magique | Pas de carte bancaire, pas de formulaire long | AUTH-1 |
| 2 | Bailleur | Nom et adresse du bailleur (elle-même), pré-remplis par défaut pour tous ses biens | Explication : « apparaîtra sur vos quittances » | BIEN-1 |
| 3 | Premier bien | Adresse, type (meublé ou vide), nombre de chambres, qui génère les chambres « Chambre 1…n » à renommer | Les chambres sont créées automatiquement | BIEN-2, BIEN-3 |
| 4 | Colocataires | Pour chaque chambre : nom, e-mail, téléphone | Saisie rapide en ligne, les détails peuvent attendre | COLOC-1 |
| 5 | Baux | Assistant : type, dates, loyer, charges, dépôt, jour d'échéance | Valeurs par défaut reprises du bien, contrôle du plafond de dépôt | BAIL-1 |
| 6 | Historique | « Le bail a commencé avant aujourd'hui : marquer les échéances passées comme payées ? » | Ne pas forcer la ressaisie du passé | LOYER-2 |
| 7 | Tableau de bord | État du mois : payé, en attente, en retard, par colocataire | 🎉 La valeur est visible | DASH-1 |

**Risque d'abandon** : l'étape 5 (trop de champs). **Parade** : seuls les champs obligatoires sont affichés, le reste est replié dans « Plus d'options ».

---

## P2 — Le rituel mensuel : encaisser et envoyer les quittances

**Déclencheur** : le 5 du mois, Claire regarde son relevé bancaire. **Objectif** : tout pointer en moins de 5 minutes.

```
Tableau de bord ──► « 4 loyers en attente » ──► Écran Loyers (mois courant)
                                                     │
          ┌──────────────────────────────────────────┤
          ▼                                          ▼
 [Tout marquer payé]                     Ligne colocataire ► [Enregistrer un paiement]
 (sélection multiple, montant exact,     (montant pré-rempli, date du jour, mode :
  date du jour)                           virement par défaut)
          │                                          │
          └──────────────► Statut « Payé » ◄─────────┘
                                 │
                                 ▼
                 Quittance générée ► [Télécharger] / [Envoyer par e-mail]
```

**Cas alternatifs**
- **Paiement partiel** : statut « Partiellement payé », un **reçu** (et non une quittance) est proposé et le reste dû est affiché.
- **Trop-perçu** : l'excédent est proposé en avance sur l'échéance suivante.
- **Retard** : à J+1 après la date d'échéance, le statut passe à « En retard », un badge rouge apparaît sur le tableau de bord et un modèle de relance est proposé (V1).

---

## P3 — Arrivée d'un nouveau colocataire en cours de mois

**Déclencheur** : Inès emménage le 12 octobre dans la chambre 2.

1. Claire ouvre le bien, puis la chambre 2 (statut « Libre »), puis **[Louer cette chambre]**.
2. Elle crée le colocataire (ou le choisit s'il existe déjà), puis le garant (facultatif).
3. Assistant de bail : date de début le 12/10, loyer, charges, dépôt de garantie.
4. L'application affiche : *« Première échéance au prorata : 20 jours sur 31 = 322,58 € »*. Montant modifiable.
5. Checklist des pièces : bail signé, état des lieux d'entrée, attestation d'assurance, pièce d'identité, avec dépôt de fichiers.
6. Le dépôt de garantie est enregistré comme « Reçu » ou « À recevoir ».

---

## P4 — Départ d'un colocataire

**Déclencheur** : Thomas envoie son congé le 3 novembre (bail meublé, préavis d'1 mois).

1. Fiche du bail : **[Enregistrer un congé]**, avec la date de réception et l'origine (colocataire ou bailleur).
2. L'application calcule la **date de fin** (3 décembre) et l'affiche avec la règle appliquée. Date modifiable.
3. La dernière échéance est calculée au prorata (3 jours de décembre).
4. *Bail unique uniquement* : l'application indique jusqu'à quand la solidarité de Thomas court (remplacement ou 6 mois maximum après la fin du préavis).
5. État des lieux de sortie : dépôt du document, puis choix « conforme » ou « avec dégradations ».
6. **Restitution du dépôt** : l'application affiche la date limite (1 mois ou 2 mois selon l'état des lieux) et permet d'ajouter des retenues avec justificatifs, puis calcule le montant à rendre.
7. La chambre repasse en « Libre à partir du 04/12 ».

---

## P5 — Les obligations annuelles

| Événement | Déclencheur dans l'app | Action de Claire |
|-----------|------------------------|------------------|
| **Révision IRL** | Alerte 30 jours avant la date anniversaire du bail | Saisir le nouvel indice (ou le reprendre de la table IRL), vérifier le nouveau loyer calculé, appliquer aux prochaines échéances |
| **Régularisation des charges** (provisions seulement) | Rappel annuel à la date choisie | Saisir les charges réelles par poste, obtenir le décompte par bail (provisions versées − réel), générer le décompte PDF, créer l'échéance ou l'avoir |
| **Attestation d'assurance** | Alerte à l'expiration | Relancer le colocataire, déposer la nouvelle attestation |
