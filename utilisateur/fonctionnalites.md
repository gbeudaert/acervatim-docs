---
title: Fonctionnalités à venir
layout: default
parent: Utilisateur
nav_order: 2
---

# Fonctionnalités : disponibles et à venir

Acervatim est en pré-alpha et n'est pas encore publiée pour le grand public sur le Play Store. Cette
page dit ce que l'application sait faire, ce qui attend une mise à jour du serveur, et la direction
générale — les priorités peuvent évoluer.

## Dans l'application

- Collections **vinyle** (vue à plat) et **manga** (vue par séries, trois états par tome), avec bascule
  entre collections via le tiroir de navigation.
- Recherche instantanée, filtres, **tri** au choix (artiste, titre, année, dernière écoute, ajout
  récent, note), affichage en **liste, grille ou galerie**, wishlist, note, dernière écoute et
  **compteur d'écoutes**.
- **Sélection multiple** : édition groupée, suppression groupée, rafraîchissement des fiches.
- Ajout manuel et **scan de code-barres hors-ligne**, avec déduplication par code-barres et par ISBN.
- Multi-artistes, vitesse de gravure.
- Import/export JSON et CSV, **sauvegarde complète ZIP** avec jaquettes.
- Thèmes Material You (marque, système, expressif), clair et sombre.

Avec le **mode connecté** (connexion Google, voir [Le mode connecté](mode-connecte.md)) :

- **Scan enrichi** : un vinyle se pré-remplit depuis Discogs, un manga depuis la BnF et MangaDex
  (série, synopsis souvent en français, jaquette de chaque tome, éditions spéciales comprises) — voir
  [D'où viennent les informations](sources-de-donnees.md).
- **Recherche d'un disque par son titre**, pressages regroupés par album, pour les disques sans
  code-barres.
- **Premium** : synchronisation des collections, vinyles et mangas, et **partage de collections** avec
  d'autres membres, en lecture seule. Le premium s'obtient aujourd'hui par un code d'invitation ;
  l'achat dans l'application n'est pas encore ouvert.

## Prêt, en attente de la mise à jour du serveur

Ces fonctionnalités sont développées dans l'application et dans le serveur cible ; elles seront
disponibles quand le serveur de production aura été mis à jour.

- **Ajouter une série manga par son titre**, sans en posséder un tome : vous tapez le titre (français
  ou original), choisissez la série puis son **édition française** (éditeur, édition standard,
  colossale, perfect…), et tous ses tomes rejoignent votre wishlist — voir
  [D'où viennent les informations](sources-de-donnees.md#ajouter-une-serie-par-son-titre).
- **Nouveaux tomes signalés** pour les séries en cours : à l'ouverture d'une série, l'application vous
  propose les tomes parus depuis votre dernier passage. Ils apparaissent **quelques semaines après leur
  sortie**, le temps que la Bibliothèque nationale de France les catalogue.

## Court terme

- **Publication Play Store** de la première version publique.
- **Films** : le serveur sait déjà chercher un film dans TMDB et le stocker ; il reste la collection
  dans l'application (ajout, liste, détail).
- **LEGO** : la collection, la recherche d'un set par son nom ou son numéro (Rebrickable) et le scan
  du code-barres d'une boîte (Brickset) sont prêts côté serveur ; reste à les ajouter à
  l'application.
  L'usage des codes-barres attend l'accord de Brickset : **pending**.

## Moyen terme

- **Jeux de société** : le scan d'une boîte (GameUPC) et la fiche du jeu (BoardGameGeek) sont
  développés côté serveur. La clé de production de GameUPC est obtenue ; reste la licence de
  BoardGameGeek : **pending**.
- **Premium** ouvert à l'achat : enrichissement avec les **jetons Acervatim** quand vous n'avez pas
  renseigné les vôtres (vos jetons restent utilisés en priorité), synchronisation sans limite, d'abord
  comme sauvegarde, ensuite pour la consultation en ligne. Le socle gratuit reste complet pour un usage
  personnel.
- **Autres collections** : BD (vue à plat ou par séries au choix), livres, jeux vidéo. Le serveur sait
  déjà stocker des livres et des jeux vidéo, sans source pour les enrichir.
- **Confort** : recherche dans la vue séries, transitions animées liste → détail, scan adapté
  automatiquement à la collection active, création de collections personnalisées depuis le tiroir.

## Plus tard

- Notification « nouveau tome disponible » hors de l'application et vue des tomes manquants pour les
  séries suivies.
- Statistiques de collection (valeur, répartition par genre, habitudes d'écoute).

## Hors périmètre

Volontairement exclus du projet, aujourd'hui comme demain :

- marketplace ;
- lecture audio, scans... ;
- publicité et revente de données — le modèle est un palier gratuit complet + premium optionnel, rien d'autre.
