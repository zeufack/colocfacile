# Exigences fonctionnelles

Priorités (MoSCoW) : **M** = indispensable au MVP · **S** = à faire au MVP si possible · **C** = confort, après le MVP · **W** = hors périmètre pour l'instant.

Format : *En tant que propriétaire, je veux… afin de…*, suivi des critères d'acceptation (CA).

---

## ÉPOPÉE AUTH — Compte et accès

| ID | Story | Prio |
|----|-------|------|
| AUTH-1 | Je veux créer un compte avec mon e-mail afin d'accéder à mon espace | M |
| AUTH-2 | Je veux me connecter et me déconnecter de façon sécurisée | M |
| AUTH-3 | Je veux réinitialiser mon mot de passe | M |
| AUTH-4 | Je veux exporter ou supprimer toutes mes données (RGPD) | S |
| AUTH-5 | Je veux inviter un co-gestionnaire (conjoint, associé de SCI) | C |

**CA AUTH-1/2**
- L'e-mail est vérifié avant le premier accès aux données.
- Une session inactive expire (durée définie dans la doc technique).
- Aucune donnée d'un compte n'est visible depuis un autre compte.

---

## ÉPOPÉE BIEN — Biens et chambres

| ID | Story | Prio |
|----|-------|------|
| BIEN-1 | Je veux définir un ou plusieurs **bailleurs** (moi, une SCI) afin qu'ils apparaissent sur les documents | M |
| BIEN-2 | Je veux créer un bien (nom, adresse, type meublé ou vide, surface, bailleur) | M |
| BIEN-3 | Je veux ajouter, renommer et archiver les chambres d'un bien (nom, surface, loyer de référence) | M |
| BIEN-4 | Je veux voir pour chaque chambre son statut : occupée (par qui, jusqu'à quand), libre, bientôt libre | M |
| BIEN-5 | Je veux archiver un bien que je ne loue plus, sans perdre son historique | S |
| BIEN-6 | Je veux noter les équipements et parties communes | C |

**CA BIEN-3/4**
- Une chambre ne peut pas être supprimée si un bail y est ou y a été rattaché : elle est archivée.
- Statut « Bientôt libre » quand un congé est enregistré et que la date de fin tombe dans les 60 jours.

---

## ÉPOPÉE COLOC — Colocataires et garants

| ID | Story | Prio |
|----|-------|------|
| COLOC-1 | Je veux créer un colocataire (nom, prénom, e-mail, téléphone, date de naissance facultative) | M |
| COLOC-2 | Je veux voir la fiche d'un colocataire : baux, historique des paiements, solde, documents | M |
| COLOC-3 | Je veux rattacher un ou plusieurs garants à un colocataire pour un bail | S |
| COLOC-4 | Je veux rechercher un colocataire par nom | S |
| COLOC-5 | Je veux anonymiser un ancien colocataire après la durée de conservation | S |

**CA COLOC-2**
- Le **solde** = total dû − total payé, toutes échéances confondues. Un solde négatif signifie une avance.
- Les anciens colocataires restent consultables (filtre « Anciens »).

---

## ÉPOPÉE BAIL — Baux

| ID | Story | Prio |
|----|-------|------|
| BAIL-1 | Je veux créer un bail avec un assistant : type (meublé, vide, étudiant, mobilité), forme (unique ou individuel), dates, loyer HC, charges et leur mode (provisions ou forfait), dépôt de garantie, jour d'échéance, clause de révision IRL (indice et trimestre de référence) | M |
| BAIL-2 | Pour un bail unique, je veux ajouter plusieurs signataires et définir leur quote-part | M |
| BAIL-3 | Je veux enregistrer un congé et obtenir la date de fin calculée selon le préavis | M |
| BAIL-4 | Je veux voir les baux par statut : brouillon, actif, en préavis, terminé | M |
| BAIL-5 | Je veux remplacer un colocataire dans un bail unique (avenant), en gardant l'historique et la date de fin de solidarité | S |
| BAIL-6 | Je veux renouveler un bail ou le voir tacitement reconduit à l'échéance | S |
| BAIL-7 | Je veux générer le contrat de bail en PDF à partir d'un modèle | W |

**CA BAIL-1**
- Le dépôt de garantie saisi est contrôlé face au plafond légal (voir [règles métier](../private/produit/06-regles-metier.md#r3--dépôt-de-garantie)). En cas de dépassement : blocage, avec explication.
- La date de fin par défaut est calculée à partir du type de bail (durée légale). Elle reste modifiable.
- Un bail individuel exige une chambre, un bail unique porte sur le bien entier.
- Une chambre ne peut pas avoir deux baux individuels actifs qui se chevauchent.
- La création d'un bail actif génère ses échéances (voir LOYER-1).

**CA BAIL-3**
- La date de fin proposée = date de réception du congé + préavis applicable.
- L'utilisateur peut choisir le motif de préavis réduit (bail vide : zone tendue, mutation…), ce qui recalcule la date.

---

## ÉPOPÉE DOC — Documents

| ID | Story | Prio |
|----|-------|------|
| DOC-1 | Je veux déposer un fichier (PDF, image) et le rattacher à un bien, un bail, un colocataire ou un garant, avec un type | M |
| DOC-2 | Je veux voir pour chaque bail la **checklist des pièces obligatoires** et ce qui manque | M |
| DOC-3 | Je veux indiquer une date d'expiration et être alerté avant qu'elle arrive | S |
| DOC-4 | Je veux prévisualiser et télécharger un document | M |
| DOC-5 | Je veux retrouver toutes les quittances et tous les reçus générés | M |

**CA DOC-1**
- Formats acceptés : PDF, JPG, PNG, HEIC. Taille maximale : 15 Mo par fichier.
- Les fichiers ne sont jamais accessibles par une URL publique permanente.

**Pièces obligatoires par défaut (DOC-2)**
- Par bail : bail signé, état des lieux d'entrée, attestation d'assurance habitation de chaque colocataire.
- Par bien : DPE (diagnostic de performance énergétique).
- Si un garant est rattaché : acte de cautionnement.
- La liste peut être modifiée par le propriétaire.

---

## ÉPOPÉE LOYER — Échéances, paiements, quittances

| ID | Story | Prio |
|----|-------|------|
| LOYER-1 | Je veux que les échéances mensuelles de chaque bail soient générées automatiquement | M |
| LOYER-2 | Je veux marquer en lot des échéances passées comme payées lors de la reprise d'un bail existant | M |
| LOYER-3 | Je veux enregistrer un paiement (montant, date, mode : virement, espèces, chèque, CAF/APL, autre) | M |
| LOYER-4 | Je veux marquer plusieurs échéances comme payées en une action | M |
| LOYER-5 | Je veux voir les échéances d'un mois par bien et par colocataire, avec leur statut | M |
| LOYER-6 | Je veux générer une **quittance** PDF pour une échéance entièrement payée, ou un **reçu** pour un paiement partiel | M |
| LOYER-7 | Je veux envoyer une quittance ou un reçu par e-mail au colocataire | M (jalon J4) |
| LOYER-8 | Je veux modifier ou annuler un paiement saisi par erreur | M |
| LOYER-9 | Je veux enregistrer le versement de l'aide au logement (APL) directement au bailleur | S |
| LOYER-10 | Je veux générer un modèle de relance amiable pour un retard | C |

**CA LOYER-1**
- Une échéance par période (mois civil) et par bail. Dans un bail unique : une échéance par signataire, selon sa quote-part.
- Date d'exigibilité = jour d'échéance défini dans le bail (1er du mois par défaut).
- Première et dernière périodes au prorata (voir [R2](../private/produit/06-regles-metier.md#r2--prorata)).
- Les échéances sont générées au moins 1 mois à l'avance. Une modification de loyer ne touche que les échéances futures non payées.

**CA LOYER-5 — statuts**
| Statut | Condition |
|--------|-----------|
| À venir | Date d'exigibilité dans le futur, rien de payé |
| En attente | Date d'exigibilité atteinte, rien de payé, délai de grâce non écoulé |
| Partiellement payée | 0 < payé < dû |
| Payée | payé ≥ dû |
| En retard | Délai de grâce écoulé (5 jours par défaut, réglable) et payé < dû |

**CA LOYER-6**
- La quittance mentionne séparément le loyer et les charges (voir [R5](../private/produit/06-regles-metier.md#r5--quittances-et-reçus)).
- Pas de quittance tant que l'échéance n'est pas entièrement payée : on produit alors un reçu.
- Un document généré est figé : s'il faut le corriger, on annule et on régénère (avec l'historique).

**CA LOYER-7**
- L'envoi n'est possible que si le colocataire a accepté les documents par e-mail (R5.4) et a une adresse e-mail.
- L'e-mail contient le PDF en pièce jointe. Il part de l'adresse d'EasyColoc, avec le nom du bailleur comme expéditeur affiché et l'adresse du propriétaire en « répondre à ».
- La date d'envoi et le statut de chaque envoi (envoyé ou échec) sont visibles sur l'échéance. En cas d'échec, on peut renvoyer.

---

## ÉPOPÉE CHARGE — Charges et régularisation

| ID | Story | Prio |
|----|-------|------|
| CHARGE-1 | Je veux choisir pour chaque bail le mode de charges : provisions ou forfait | M |
| CHARGE-2 | Je veux saisir les charges réelles d'un bien sur une période, par poste (eau, électricité des communs, TEOM, entretien…) | S |
| CHARGE-3 | Je veux obtenir la régularisation par bail (provisions versées − part des charges réelles), au prorata de la durée d'occupation | S |
| CHARGE-4 | Je veux générer le décompte de régularisation en PDF et créer l'échéance (complément) ou l'avoir (trop-perçu) correspondant | S |
| CHARGE-5 | Je veux choisir la clé de répartition des charges entre baux : parts égales, surface, personnalisée | S |

---

## ÉPOPÉE DEPOT — Dépôt de garantie

| ID | Story | Prio |
|----|-------|------|
| DEPOT-1 | Je veux suivre le dépôt de chaque bail : attendu, reçu (date, mode), restitué | M |
| DEPOT-2 | À la sortie, je veux saisir des retenues justifiées et obtenir le montant à restituer | M |
| DEPOT-3 | Je veux voir la date limite de restitution et être alerté avant qu'elle arrive | M |
| DEPOT-4 | Je veux générer un décompte de restitution en PDF | S |

---

## ÉPOPÉE IRL — Révision du loyer

| ID | Story | Prio |
|----|-------|------|
| IRL-1 | Je veux saisir ou consulter les valeurs trimestrielles de l'IRL | S |
| IRL-2 | Je veux être alerté à la date anniversaire d'un bail qui contient une clause de révision | S |
| IRL-3 | Je veux obtenir le nouveau loyer calculé et l'appliquer aux échéances futures | S |
| IRL-4 | Je veux que les valeurs IRL soient importées automatiquement depuis l'INSEE | C |

---

## ÉPOPÉE DASH — Tableau de bord et alertes

| ID | Story | Prio |
|----|-------|------|
| DASH-1 | Je veux voir le mois courant : total attendu, encaissé, restant, nombre de retards | M |
| DASH-2 | Je veux voir la liste des actions à faire : retards, pièces manquantes, documents qui expirent, dépôts à restituer, révisions IRL, fins de bail | M |
| DASH-3 | Je veux voir le taux d'occupation de mes chambres | S |
| DASH-4 | Je veux recevoir un e-mail récapitulatif hebdomadaire des actions à faire | C |
| DASH-5 | Je veux exporter les paiements d'une année en CSV (pour ma comptabilité) | S |

---

## Exigences non fonctionnelles (vue produit)

| Thème | Exigence |
|-------|----------|
| **Responsive** | Tous les écrans utilisables sur mobile (≥ 360 px). Les écrans Loyers et Enregistrer un paiement sont optimisés pour le mobile |
| **Performance** | Tableau de bord affiché en moins de 2 s en 4G |
| **Accessibilité** | Niveau RGAA / WCAG 2.1 AA visé : contrastes, navigation clavier, libellés |
| **Langue et formats** | Français. Montants au format `1 234,56 €`, dates `JJ/MM/AAAA`, fuseau Europe/Paris |
| **RGPD** | Minimisation des données, durées de conservation, export, suppression, hébergement dans l'UE |
| **Fiabilité des montants** | Aucune erreur d'arrondi : calculs au centime, arrondi au centime le plus proche, écarts d'arrondi affectés à la dernière ligne |
| **Traçabilité** | Toute modification d'un paiement, d'une échéance ou d'un bail est historisée (qui, quand, avant, après) |
