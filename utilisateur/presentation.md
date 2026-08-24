---
title: Présentation
layout: default
parent: Utilisateur
nav_order: 1
---

# Guide d'utilisation de l'application
{: .no_toc }

## Sommaire
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Vue d'ensemble

Acervatim s'inspire de l'application Contacts de Google : une longue liste sobre et lisible, une barre de recherche flottante toujours accessible, et un bouton d'ajout en bas à droite. Chaque type de collection (vinyle, manga…) vit dans son propre espace, accessible depuis le tiroir de navigation.

## Changer de collection

Touchez l'icône **☰** à gauche de la barre de recherche pour ouvrir le tiroir de navigation. Il liste toutes les collections disponibles avec leur compteur :

- **Vinyle · 128** — nombre de disques ;
- **Manga · 12 séries** — nombre de séries suivies.

L'application mémorise la dernière collection ouverte et la restaure au prochain lancement.

## L'écran de liste

### Recherche

La barre de recherche flottante en haut de l'écran filtre la liste en temps réel sur l'artiste et le titre. Touchez-la, tapez quelques lettres : les résultats s'affichent instantanément, sans connexion.

### Filtres avancés

Le bouton réglages (⛭) à droite de la barre de recherche ouvre le panneau de filtres :

- **Statut** : tout afficher / collection uniquement / wishlist uniquement ;
- **Genre** et **format** : listes déroulantes construites à partir de vos propres fiches ;
- **Mode d'affichage** (collections qui le permettent, comme le manga) : vue à plat ou vue par séries ;
- **Réinitialiser** pour revenir à la liste complète.

### Liste, grille ou galerie

L'icône d'affichage, à droite de la barre de recherche, propose trois densités — le choix est mémorisé **par collection** :

- **Liste** : une ligne par élément, vignette et texte ;
- **Grille** : des jaquettes en tuiles, toute la collection d'un bloc ;
- **Galerie** : de grandes jaquettes classées par rubriques d'écoute — *Derniers écoutés*, *Les plus écoutés*, *Jamais écoutés ou oubliés*, puis toute la collection. Une rubrique sans élément n'apparaît pas.

Les rubriques d'écoute s'appuient sur le compteur d'écoutes : chaque *Je viens de l'écouter* incrémente un total, en plus de la date de dernière écoute. Le compteur suit votre compte quand la synchronisation est active.

### Lecture de la liste

- Les entrées sont triées par artiste, avec des en-têtes alphabétiques collants.
- Un bandeau discret sous la recherche récapitule la collection : `128 disques · 12 souhaités`.
- Les éléments en **wishlist** apparaissent **grisés avec une icône cœur** : vous voyez d'un coup d'œil ce que vous possédez et ce que vous cherchez encore.
- Pour les albums à plusieurs artistes, un réglage permet de choisir entre une entrée par artiste (l'album apparaît sous chaque nom) ou une seule entrée sous l'artiste principal.

## Ajouter un élément

Le bouton **+ Ajouter** en bas à droite propose les modes d'ajout adaptés à la collection active.

### Saisie manuelle

Formulaire complet, organisé en sections :

- **Jaquette** : prise de photo, choix dans la galerie, ou récupération en ligne ;
- **Identité** : artiste(s), titre, année — la saisie des artistes propose une autocomplétion sur ceux déjà enregistrés, et accepte plusieurs artistes par disque (duos, collaborations) ;
- **Édition** : label, format (33t / 45t / 78t…), vitesse de gravure (distincte du format : certains 33 cm sont gravés à 45 tr/min), pays de pressage ;
- **Classification** : genre (autocomplétion), note de 0 à 5 étoiles ;
- **Achat** : prix, code-barres ;
- **Statut** : possédé ou wishlist.

Seuls l'artiste et le titre sont obligatoires — tout le reste est optionnel.

### Scan de code-barres

Le scan utilise la caméra et fonctionne **entièrement hors-ligne** (reconnaissance embarquée) :

1. Visez le code-barres EAN/UPC du disque ;
2. si le disque est **déjà dans votre collection**, l'application vous l'indique et ouvre sa fiche (pas de doublon possible) ;
3. sinon, si une source externe est configurée (compte Discogs) et que vous êtes connecté à internet, la fiche est **pré-remplie automatiquement** (titre, artiste, année, jaquette…) ;
4. sans connexion, le formulaire s'ouvre avec le code-barres pré-rempli — complétez à la main.

Pour le détail des sources interrogées, voir [D'où viennent les informations](sources-de-donnees.md).

### Recherche Discogs (vinyle)

En renseignant un jeton d'accès personnel Discogs dans **Réglages → Comptes & sources**, vous pouvez rechercher un disque par texte ou par code-barres dans la base Discogs et importer sa fiche complète, jaquette comprise. Cette connexion est facultative : l'application est pleinement fonctionnelle sans.

## La fiche détail

Touchez un élément de la liste pour ouvrir sa fiche :

- grande jaquette, artiste et titre ;
- **note** : touchez les étoiles pour la modifier ;
- chips d'informations : genre, format, année, label, pays, prix ;
- **dernière écoute** : « Il y a 3 jours » et le nombre d'écoutes, avec un bouton *Je viens de l'écouter* et un bouton *Réinitialiser* (qui efface la date **et** le compteur) — pratique pour repérer les disques délaissés ;
- **statut** : interrupteur pour basculer entre collection et wishlist ;
- menu ⋮ : éditer, supprimer, voir sur Discogs (si la fiche en provient).

## Collection manga : la vue par séries

La collection manga s'organise par défaut **par séries** plutôt que tome par tome.

### Liste des séries

Chaque série occupe une ligne (ou une tuile en grille) avec sa jaquette, son auteur et sa **progression** : `Berserk · 28 / 42 tomes`. Un badge cœur signale qu'au moins un tome est en wishlist. Le total peut être inconnu pour une série en cours (`28 / ?`).

### Fiche série

Touchez une série pour ouvrir sa fiche : un en-tête rétractable (jaquette, auteur, statut de publication, synopsis, progression) au-dessus de la **grille des tomes**. L'en-tête se replie au défilement pour laisser la place aux tomes.

### Les trois états d'un tome

Un **appui long** sur un tome fait basculer son état :

| État | Affichage | Signification |
|---|---|---|
| **Possédé** | Tuile pleine, coche | Le tome est dans votre collection. |
| **Wishlist** | Tuile grisée, cœur | Vous le cherchez. |
| **Ignoré** | Tuile très atténuée | Vous savez qu'il existe, vous n'en voulez pas. |

### Ajouter une série et des tomes

- Bouton **+** en vue séries → création d'une série (titre, auteur, nombre de tomes annoncés, statut de publication, synopsis, jaquette).
- Depuis la fiche série, **+ Ajouter un tome** : le numéro du tome suivant est suggéré automatiquement. La fiche tome accepte ISBN, éditeur français, public visé (shōnen, seinen…), nombre de pages, note et statut.
- Les doublons sont évités : un même ISBN ou un même couple série + numéro de tome ne peut pas être saisi deux fois.

Si vous préférez voir chaque tome individuellement, basculez en **vue à plat** depuis le panneau de filtres — le choix est mémorisé par collection.

La bascule **liste / grille / galerie** s'applique aussi à la vue par séries : une série y devient une tuile, et les rubriques d'écoute cumulent les écoutes de ses tomes.

## Import, export et sauvegarde

Vos données ne sont jamais captives :

- **Export** de la collection en JSON ou CSV ;
- **Sauvegarde complète** au format ZIP, jaquettes incluses — à conserver où vous voulez (stockage local, cloud personnel…) ;
- **Import** d'une sauvegarde ou d'un export : les fiches existantes sont fusionnées, les nouvelles créées.

## Réglages

- **Thème** : trois modes — *Vert d'eau* (couleur de marque, par défaut), *Adapté au système* (couleurs de votre fond d'écran, Android 12+), *Expressif*. Tous existent en clair et en sombre.
- **Albums multi-artistes** : une entrée par artiste, ou une seule sous l'artiste principal.
- **Comptes & sources** : connexion des sources de métadonnées externes (Discogs pour le vinyle), avec indication claire de ce qui est configuré.

## Confidentialité et permissions

- Aucune permission n'est demandée au lancement. La **caméra** n'est demandée qu'au premier scan ou à la première photo de jaquette ; l'accès aux **images** qu'au premier import depuis la galerie.
- En mode hors-ligne, l'application **ne collecte aucune donnée** et n'émet aucune requête réseau.
- Les appels réseau (recherche Discogs, et demain le serveur Acervatim) sont **toujours déclenchés explicitement** par vous — jamais en arrière-plan.

Détail du traitement des données : [politique de confidentialité](../legal/confidentialite.md).
