# UX & écrans

## 1. Principes de design

| Principe | Ce que ça veut dire concrètement |
|----------|----------------------------------|
| **Le mois d'abord** | L'accueil répond à « où en est-on ce mois-ci ? » avant tout le reste |
| **Du statut à l'action** | Chaque statut affiché (retard, pièce manquante…) est cliquable et mène à l'action qui le résout |
| **Calculer, montrer, laisser corriger** | Chaque montant calculé affiche « Comment c'est calculé ? » et reste modifiable (sauf plafonds légaux) |
| **Progressif** | Les formulaires montrent le strict nécessaire ; les options avancées sont repliées |
| **Mobile pour agir, bureau pour gérer** | Pointer les loyers et déposer une photo de document : mobile d'abord. Configurer les baux et les charges : bureau d'abord |
| **Rassurant** | Ton calme et précis, pas d'alarme inutile. Rouge réservé aux retards et aux blocages légaux |

## 2. Architecture de l'information

```
EasyColoc
├── Tableau de bord            (accueil)
├── Loyers                     (vue du mois, filtres par bien et statut)
│   └── Échéance ► paiements, quittance ou reçu
├── Biens
│   └── Bien ► chambres, baux actifs, documents, charges
│       └── Chambre ► occupation actuelle et historique
├── Colocataires
│   └── Colocataire ► baux, paiements, solde, documents, garants
├── Baux
│   └── Bail ► résumé, signataires, échéances, dépôt, documents, historique
├── Documents                  (toutes les pièces, filtres par type et statut)
└── Paramètres                 (profil, bailleurs, délai de grâce, pièces obligatoires, IRL, données)
```

**Navigation** : barre latérale sur ordinateur ; barre d'onglets en bas sur mobile (Accueil · Loyers · Biens · Colocataires · Plus).

**Action globale** : un bouton « + » donne accès à : Enregistrer un paiement · Ajouter un colocataire · Nouveau bail · Déposer un document.

## 3. Écrans clés (maquettes filaires)

### E1 — Tableau de bord

```
┌─────────────────────────────────────────────────────────────────┐
│ Bonjour Claire                                 Septembre 2026 ▾ │
├───────────────┬───────────────┬───────────────┬─────────────────┤
│ Attendu       │ Encaissé      │ Reste         │ En retard       │
│ 3 450,00 €    │ 2 480,00 €    │ 970,00 €      │ 🔴 2             │
│               │ ▓▓▓▓▓▓▓░░░ 72%│               │                 │
├───────────────┴───────────────┴───────────────┴─────────────────┤
│ À faire (5)                                                     │
│ 🔴 Loyer en retard — Thomas B. · T4 Lyon · 450 €     [Pointer] │
│ 🔴 Loyer en retard — Sofia K. · T5 Villeurb. · 520 € [Pointer] │
│ 🟠 Attestation d'assurance expirée — Inès M.        [Déposer]  │
│ 🟠 Dépôt à restituer avant le 15/10 — Paul D.       [Traiter]  │
│ 🔵 Révision IRL possible le 01/10 — Bail Ch. 3       [Voir]    │
├─────────────────────────────────────────────────────────────────┤
│ Mes biens                                                       │
│ T4 Lyon           3/3 chambres occupées   ✓ 3/3 payés           │
│ T5 Villeurbanne   4/4 chambres occupées   ⚠ 2/4 payés           │
└─────────────────────────────────────────────────────────────────┘
```

### E2 — Loyers du mois

```
┌─────────────────────────────────────────────────────────────────┐
│ Loyers   ◀ Septembre 2026 ▶    Bien: Tous ▾   Statut: Tous ▾    │
├──┬──────────────┬──────────────┬─────────┬──────────┬───────────┤
│☐ │ Colocataire  │ Bien · Ch.   │ Dû      │ Payé     │ Statut    │
├──┼──────────────┼──────────────┼─────────┼──────────┼───────────┤
│☐ │ Inès M.      │ T4 · Ch.1    │ 480,00  │ 480,00   │ 🟢 Payé  ⋮│
│☐ │ Thomas B.    │ T4 · Ch.2    │ 450,00  │ 0,00     │ 🔴 Retard ⋮│
│☐ │ Léa P.       │ T4 · Ch.3    │ 470,00  │ 200,00   │ 🟠 Partiel⋮│
│☐ │ Sofia K.     │ T5 · Ch.1    │ 520,00  │ 0,00     │ 🔴 Retard ⋮│
├──┴──────────────┴──────────────┴─────────┴──────────┴───────────┤
│ 2 sélectionnés   [Marquer payés (montant exact)]                │
└─────────────────────────────────────────────────────────────────┘
Menu ⋮ : Enregistrer un paiement · Quittance / Reçu · Envoyer par e-mail · Voir le bail
```

Sur mobile, les lignes deviennent des cartes. Balayer vers la droite = « Marquer payé ».

### E3 — Enregistrer un paiement (fenêtre ou panneau)

```
┌──────────────────────────────────────┐
│ Paiement — Thomas B. · Septembre     │
│ Reste dû : 450,00 €                  │
│                                      │
│ Montant    [ 450,00 € ]              │
│ Date       [ 22/09/2026 ]            │
│ Mode       (•) Virement ( ) Espèces  │
│            ( ) Chèque  ( ) CAF/APL   │
│ Note       [                      ]  │
│                                      │
│ ☑ Générer la quittance               │
│ ☐ L'envoyer par e-mail à Thomas      │
│                                      │
│          [Annuler]  [Enregistrer]    │
└──────────────────────────────────────┘
```
- Si le montant est inférieur au reste dû : la case devient « Générer un reçu », avec le message « Paiement partiel : reste 250,00 € ».
- Si le montant est supérieur : « 50,00 € seront gardés en avance sur la prochaine échéance ».

### E4 — Fiche bien

```
┌─────────────────────────────────────────────────────────────────┐
│ T4 Lyon · 12 rue X, 69003 Lyon · Meublé · 78 m² · DPE C    [⋮]  │
│ Onglets : [Chambres] Baux  Documents  Charges                   │
├─────────────────────────────────────────────────────────────────┤
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐              │
│ │ Chambre 1    │ │ Chambre 2    │ │ Chambre 3    │              │
│ │ 14 m²        │ │ 12 m²        │ │ 11 m²        │              │
│ │ Inès M.      │ │ Thomas B.    │ │ 🟢 Libre      │              │
│ │ jusqu'au     │ │ 🟠 Préavis    │ │ depuis 01/09 │              │
│ │ 31/08/2027   │ │ fin 03/12    │ │ [Louer]      │              │
│ └──────────────┘ └──────────────┘ └──────────────┘              │
│ Pièces du bien : ✓ DPE (valide jusqu'en 2031)                   │
└─────────────────────────────────────────────────────────────────┘
```

### E5 — Assistant de création de bail (4 étapes)

```
① Logement & forme  →  ② Colocataire(s)  →  ③ Conditions financières  →  ④ Pièces & récap
```

| Étape | Contenu | Points d'attention |
|-------|---------|--------------------|
| ① | Bien, forme (unique ou individuel, avec une explication d'une ligne pour chaque), chambre, type (meublé, vide, étudiant, mobilité) | Le type est pré-rempli d'après le bien |
| ② | Choisir ou créer un colocataire ; pour un bail unique, plusieurs signataires et leurs quote-parts (répartition égale par défaut) ; garant facultatif | La somme des quote-parts est contrôlée en direct (= 100 %) |
| ③ | Date de début, durée (valeur légale par défaut), loyer HC, charges et leur mode, dépôt, jour d'échéance, clause IRL | Plafond du dépôt affiché sous le champ : « Maximum 960,00 € (2 mois HC, meublé) » ; aperçu de la première échéance au prorata |
| ④ | Checklist des pièces avec dépôt de fichiers (facultatif maintenant), récapitulatif | Bouton principal : « Activer le bail ». Secondaire : « Enregistrer en brouillon » |

### E6 — Fiche colocataire

En-tête : nom, contact, solde (vert si à jour, rouge si débiteur), badge « Solidaire jusqu'au … » le cas échéant.
Onglets : **Paiements** (timeline des échéances et paiements) · **Baux** · **Documents** (checklist) · **Garants**.

### E7 — Départ et restitution du dépôt

Écran en étapes depuis le bail : ① Congé (date de réception, origine, motif, date de fin calculée) → ② Dernière échéance (prorata) → ③ État des lieux de sortie (fichier, conforme ou non) → ④ Retenues (lignes motif / montant / justificatif) → ⑤ Récapitulatif : montant à restituer, date limite, décompte PDF.

## 4. États et micro-textes

| Situation | Comportement |
|-----------|--------------|
| **État vide — aucun bien** | Illustration et « Ajoutez votre premier bien pour commencer », avec le bouton [Ajouter un bien] |
| **État vide — aucun retard** | « Tout est à jour ce mois-ci 🎉 » |
| **Blocage légal** | Message sous le champ, avec la règle et un lien « En savoir plus » : « Le dépôt de garantie d'un bail meublé ne peut pas dépasser 2 mois de loyer hors charges (960,00 €). » |
| **Action irréversible** (annuler une quittance, terminer un bail) | Fenêtre de confirmation qui explique la conséquence |
| **Suppression** | Préférer « Archiver ». La suppression n'est possible que pour un brouillon |
| **Enregistrement** | Retour optimiste avec un toast « Paiement enregistré · Annuler » (10 s) |

## 5. Système visuel (orientations)

| Élément | Orientation |
|---------|-------------|
| Ton | Sérieux et chaleureux : un outil de confiance, pas une banque froide |
| Couleurs de statut | 🟢 payé / à jour · 🟠 partiel / à surveiller · 🔴 retard / bloquant · 🔵 information · ⚪ à venir. Toujours accompagnées d'un libellé : jamais la couleur seule |
| Typographie | Une sans-serif lisible, chiffres tabulaires pour les montants (alignement des colonnes) |
| Montants | Alignés à droite, format `1 234,56 €` |
| Thème | Clair et sombre |
| Composants | Bibliothèque de composants accessible (choix technique à préciser dans la doc technique) |

## 6. Accessibilité

- Contraste minimum AA (4,5:1 pour le texte).
- Toutes les actions sont accessibles au clavier, avec un focus visible.
- Les statuts sont annoncés par du texte, pas seulement par la couleur ou une icône.
- Les champs de formulaire ont des libellés explicites ; les erreurs sont reliées au champ (`aria-describedby`).
- Cibles tactiles ≥ 44 × 44 px sur mobile.
