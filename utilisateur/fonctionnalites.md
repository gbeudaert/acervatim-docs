---
title: Fonctionnalités à venir
layout: default
parent: Utilisateur
nav_order: 2
---

# Fonctionnalités à venir

Acervatim est en pré-alpha : l'application n'est pas encore publiée sur le Play Store. Cette page donne la direction générale — les priorités peuvent évoluer.

## Disponible aujourd'hui

- Collections **vinyle** (vue à plat) et **manga** (vue par séries, trois états par tome), avec bascule entre collections via le tiroir de navigation.
- Recherche instantanée, filtres, wishlist, note, dernière écoute.
- Ajout manuel et **scan de code-barres hors-ligne** : le scan déduplique sur le code-barres et, s'il est inconnu, pré-remplit le champ code-barres du formulaire. L'enrichissement automatique des métadonnées (titre, artiste, jaquette…) n'est **pas encore disponible** — voir « Court terme ».
- Multi-artistes, vitesse de gravure, déduplication par code-barres et par ISBN.
- Import/export JSON et CSV, **sauvegarde complète ZIP** avec jaquettes.
- Thèmes Material You (marque, système, expressif), clair et sombre.
- Côté serveur (capacités en place mais **pas encore activées dans l'application publiée** — voir « Court terme ») : authentification Google, recherche externe unifiée (Discogs, MangaDex, MyAnimeList, BnF, Google Books, TMDB), synchronisation des collections (**palier premium** — voir [Le mode connecté](mode-connecte.md)).

## Court terme

- **Publication Play Store v1.0** : la version 100 % hors-ligne (vinyle + manga), sans compte ni achat intégré.
- **Activation du mode connecté dans l'application** : connexion Google et recherche dans les bases externes avec **vos propres jetons d'accès** (*bring your own token*). À ce palier, vos collections restent sur l'appareil — la synchronisation cloud relève du palier premium (voir « Moyen terme »). La couche technique est en place côté app et côté serveur ; il reste la configuration de production et la gestion des jetons par source.
- **Enrichissement automatique au scan** : recherche dans les bases externes (via le serveur) pour pré-remplir titre, artiste/auteur, jaquette et métadonnées — par code-barres pour le vinyle (Discogs) et **par ISBN pour le manga** (pivot **BnF → MangaDex**, MyAnimeList en secours, qui retrouve la série et le tome scanné avec un synopsis souvent en français, éditions spéciales comprises — voir [D'où viennent les informations](sources-de-donnees.md)). Aujourd'hui le scan ne fait que pré-remplir le code-barres ; cet enrichissement reste à brancher dans l'app publiée.

## Moyen terme

- **Premium** : enrichissement avec les **jetons Acervatim** quand vous n'avez pas renseigné les vôtres (aucune clé personnelle à créer — vos jetons restent utilisés en priorité), et **synchronisation cloud sans limite** des collections — d'abord comme sauvegarde, ensuite pour la consultation en ligne. Le socle gratuit reste complet pour un usage personnel.
- **Nouvelles collections** : BD (vue à plat ou par séries au choix), LEGO, jeux de société, livres, films — chaque type avec sa source de métadonnées de référence (Rebrickable, BoardGameGeek, Google Books, TMDB…), routée par le serveur.
- **Confort** : recherche dans la vue séries, transitions animées liste → détail, scan adapté automatiquement à la collection active, création de collections personnalisées depuis le tiroir.

## Plus tard

- Alerte « nouveau tome disponible » et vue des tomes manquants pour les séries suivies.
- Statistiques de collection (valeur, répartition par genre, habitudes d'écoute).
- Montée en charge du serveur si l'usage le justifie — le schéma de données est prêt pour ce scénario.

## Hors périmètre

Volontairement exclus du projet, aujourd'hui comme demain :

- marketplace ;
- lecture audio, scans... ;
- publicité et revente de données — le modèle est un palier gratuit complet + premium optionnel, rien d'autre.
