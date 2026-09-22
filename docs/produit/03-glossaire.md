# Glossaire du domaine

Ce glossaire est **le langage commun** entre produit, design et code. Un terme défini ici porte le même nom dans l'interface, la documentation et, traduit en anglais, dans le code (colonne « Nom technique »).

## Acteurs

| Terme | Définition | Nom technique |
|-------|------------|---------------|
| **Propriétaire** | L'utilisateur de l'application, titulaire du compte | `Owner` |
| **Bailleur** | La personne physique ou morale (ex. : SCI) qui apparaît sur le bail et les quittances. Souvent le propriétaire lui-même, mais pas toujours | `Landlord` |
| **Colocataire** | Une personne qui loue tout ou partie d'un logement partagé | `Tenant` |
| **Garant** | Une personne qui se porte caution pour un colocataire (acte de cautionnement). Dans l'interface, on évite le mot « caution » pour ne pas le confondre avec le dépôt de garantie | `Guarantor` |

## Patrimoine

| Terme | Définition | Nom technique |
|-------|------------|---------------|
| **Bien** | Un logement loué en colocation (appartement, maison), avec une adresse | `Property` |
| **Chambre** | Une partie privative d'un bien, louable séparément dans le cas de baux individuels | `Room` |
| **Parties communes** | Les espaces partagés d'un bien (cuisine, salon, salle de bain…). Informatif | — |
| **Occupation** | Le fait qu'un colocataire occupe une chambre (ou le bien) sur une période donnée | `Occupancy` |

## Contrats

| Terme | Définition | Nom technique |
|-------|------------|---------------|
| **Bail** | Le contrat de location. Il a un type (meublé, vide, mobilité, étudiant), des dates, un loyer, des charges et un dépôt de garantie | `Lease` |
| **Bail unique** | Un seul bail signé par tous les colocataires pour l'ensemble du bien | `LeaseKind.SHARED` |
| **Bail individuel** | Un bail par colocataire, portant sur une chambre et l'usage des parties communes | `LeaseKind.INDIVIDUAL` |
| **Signataire** | Un colocataire partie à un bail. Un bail unique a plusieurs signataires, un bail individuel un seul | `LeaseTenant` |
| **Quote-part** | Dans un bail unique, la part du loyer et des charges attribuée à un colocataire (convention interne, utile au suivi) | `share` |
| **Clause de solidarité** | Une clause qui rend chaque colocataire redevable de la totalité du loyer | `solidarityClause` |
| **Avenant** | Une modification du bail (ex. : remplacement d'un colocataire dans un bail unique) | `LeaseAmendment` |
| **Congé** | L'avis de départ donné par un colocataire ou par le bailleur | `Notice` |
| **Préavis** | Le délai entre la réception du congé et la fin effective de la location | `noticePeriod` |
| **État des lieux** | Le document décrivant l'état du logement à l'entrée et à la sortie | `Inspection` |

## Argent

| Terme | Définition | Nom technique |
|-------|------------|---------------|
| **Loyer hors charges** | Le montant du loyer sans les charges | `rentAmount` |
| **Charges** | Les charges récupérables payées par le locataire, en **provisions** (ajustées à la régularisation) ou au **forfait** (fixes, sans régularisation) | `chargesAmount`, `chargesMode` |
| **Échéance** | Ce qui est dû pour une période (généralement un mois) au titre d'un bail : loyer + charges, à une date donnée | `RentDue` |
| **Paiement** | Une somme reçue du colocataire, rattachée à une ou plusieurs échéances | `Payment` |
| **Statut d'échéance** | À venir, en attente, partiellement payée, payée, en retard | `RentDueStatus` |
| **Prorata** | Le calcul du montant dû pour une période incomplète (entrée ou sortie en cours de mois) | `proration` |
| **Quittance** | Le document attestant le paiement **complet** d'une échéance | `RentReceipt` |
| **Reçu** | Le document attestant un paiement **partiel** | `PaymentReceipt` |
| **Dépôt de garantie** | La somme versée à l'entrée et restituée à la sortie, moins les retenues justifiées | `SecurityDeposit` |
| **Retenue** | Un montant déduit du dépôt de garantie (dégradation, impayé), avec justificatif | `DepositDeduction` |
| **Régularisation des charges** | Le calcul annuel qui compare les provisions versées aux charges réelles | `ChargesReconciliation` |
| **IRL** | L'indice de référence des loyers publié chaque trimestre par l'INSEE, qui sert à réviser le loyer | `RentIndex` |
| **Révision du loyer** | L'ajustement annuel du loyer selon l'IRL, si le bail le prévoit | `RentRevision` |

## Documents

| Terme | Définition | Nom technique |
|-------|------------|---------------|
| **Document** | Un fichier rattaché à un bien, un bail, un colocataire ou un garant | `Document` |
| **Type de document** | La catégorie : bail signé, état des lieux, attestation d'assurance, pièce d'identité, justificatif de revenus, diagnostic (DPE…), acte de cautionnement, autre | `DocumentType` |
| **Pièce obligatoire** | Un type de document attendu pour qu'un dossier soit considéré comme complet | `required` |
| **Date d'expiration** | La date après laquelle un document n'est plus valable (ex. : attestation d'assurance) | `expiresAt` |

## Termes à ne pas utiliser

| ❌ À éviter | ✅ À la place | Pourquoi |
|------------|--------------|----------|
| « Caution » pour l'argent versé à l'entrée | Dépôt de garantie | « Caution » désigne juridiquement le garant |
| « Locataire » quand il s'agit d'un logement partagé | Colocataire | Cohérence du domaine |
| « Facture » pour le loyer | Échéance / avis d'échéance | On ne facture pas un loyer d'habitation |
| « Tenant » au sens SaaS (multi-tenant) | Compte (`Account`) | Ambiguïté avec le colocataire (`Tenant`) dans le code |
