---
title: D'où viennent les informations
layout: default
parent: Utilisateur
nav_order: 3
---

# D'où viennent les informations de vos collections

Acervatim fonctionne **hors-ligne par défaut** : vos collections vivent sur votre appareil, sans
compte ni réseau. Quand vous le souhaitez, l'application peut **pré-remplir** une fiche (pochette,
synopsis, auteurs, date…) en interrogeant des **bases de données spécialisées**. C'est toujours le
serveur Acervatim qui interroge ces bases, jamais l'application directement.

Cette page liste **toutes les sources**, type de collection par type de collection : ce que chacune
apporte, à quel moment elle intervient, et où elle en est.

> L'enrichissement est **optionnel** et **à votre initiative** : l'application reste pleinement
> utilisable sans jamais interroger ces sources.

## Où en est chaque source

| Statut | Signification |
|---|---|
| **en service** | Le serveur Acervatim l'interroge aujourd'hui. |
| **prête** | Le serveur sait l'interroger, mais ce type de collection n'est pas encore dans l'application, ou le serveur qui la porte n'est pas encore en production. |
| **pending** | En attente d'une réponse ou d'un accord : clé d'accès, licence, autorisation d'usage. |

## Les sources par type de collection

### Vinyles

| Source | Ce qu'elle apporte | Quand elle intervient | Statut |
|---|---|---|---|
| [Discogs](https://www.discogs.com) | Album, artistes, année, label, pays, format, numéro de catalogue, pochette | Scan du code-barres ; recherche par titre ou par artiste, avec les pressages regroupés par album | en service |

Discogs demande un accès : reliez votre compte Discogs dans **Réglages → Gérer les collections →
Gérer les sources**, ou utilisez l'accès partagé inclus avec premium.

### Mangas

| Source | Ce qu'elle apporte | Quand elle intervient | Statut |
|---|---|---|---|
| [Bibliothèque nationale de France (BnF)](https://catalogue.bnf.fr) | Le tome et son **édition française** : titre de série, numéro, éditeur, ISBN, titre d'origine ; la liste de tous les tomes d'une édition | Scan d'un ISBN ; choix de l'édition française quand vous ajoutez une série par son titre ; recherche des nouveaux tomes | en service |
| [MangaDex](https://mangadex.org) | La **série** : synopsis en français, note, genres, nombre de tomes, **jaquette de chaque tome** | Scan d'un ISBN ; recherche d'une série par son titre | en service |
| [MyAnimeList](https://myanimelist.net) | La série, quand MangaDex ne la reconnaît pas (résumé en anglais) | En secours seulement, si vous avez relié votre compte MyAnimeList ou si vous êtes premium | en service |
| [Google Books](https://books.google.com) | Une jaquette ou un résumé de tome qui manque ; le titre d'un livre que la BnF ne connaît pas | En secours seulement, au scan et à l'affichage des tomes | en service |

MangaDex, la BnF et Google Books ne demandent aucun accès : le scan d'un manga s'enrichit même sans
compte relié.

### Films

| Source | Ce qu'elle apporte | Quand elle intervient | Statut |
|---|---|---|---|
| [TMDB (The Movie Database)](https://www.themoviedb.org) | Titre, affiche, année, synopsis | Recherche d'un film par son titre | prête : le serveur sait chercher un film, l'application n'a pas encore de collection de films |

### LEGO

| Source | Ce qu'elle apporte | Quand elle intervient | Statut |
|---|---|---|---|
| [Rebrickable](https://rebrickable.com) | Le catalogue des sets : numéro, nom, année, thème, nombre de pièces et de figurines, image | Recherche d'un set par numéro ou par nom (le filtre par thème viendra plus tard) | prête : le catalogue est importé chaque jour par le nouveau serveur, pas encore en production |
| [Brickset](https://brickset.com) | Le code-barres imprimé sur chaque boîte | Scan d'une boîte | pending : accord de Brickset pour cet usage |

Un code-barres que Brickset ne connaît pas encore, souvent celui d'un set tout juste sorti, n'est
redemandé qu'au bout de 7 jours. En attendant, cherchez le set par le numéro imprimé sur la boîte.

Les images des sets appartiennent au groupe LEGO. Leur affichage dans une application qui aura un
palier payant reste à trancher : **pending**.

### Jeux de société

| Source | Ce qu'elle apporte | Quand elle intervient | Statut |
|---|---|---|---|
| [GameUPC](https://gameupc.com) | Le jeu qui correspond à un code-barres de boîte | Scan d'une boîte | prête : clé d'accès de production obtenue, le nouveau serveur n'est pas encore en production |
| [BoardGameGeek](https://boardgamegeek.com) | La fiche du jeu : nom et titres français connus, année, auteurs, éditeurs, nombre de joueurs, durée, âge, description, image, note | Après le scan, une fois le jeu identifié | pending : licence demandée |

Quand vous choisirez le bon jeu parmi plusieurs propositions, ce choix sera renvoyé à GameUPC, sous
un pseudonyme qui ne permet pas de vous identifier, pour améliorer la reconnaissance des boîtes
françaises. L'application affichera le logo « Powered by BGG », lié à la fiche du jeu.

### Livres et jeux vidéo

Le serveur sait stocker ces deux types de collection, mais aucune source ne les enrichit pour
l'instant, et l'application ne les propose pas encore.

## Scanner un manga : retrouver la série et tous ses tomes

Pour les mangas, vous pouvez **scanner le code-barres (ISBN)** d'un tome. À partir de ce seul
ISBN, Acervatim combine **plusieurs sources complémentaires**, chacune avec son rôle :

1. **BnF** — identifie l'ouvrage dans le catalogue officiel des éditions françaises (titre de
   série, n° de tome, éditeur, ISBN) et sait **énumérer tous les tomes** d'une édition ;
2. **MangaDex** — retrouve la **série** et ses informations d'ensemble : **synopsis en français**,
   note, genres, nombre de tomes, et la **jaquette de chaque tome** (souvent celle de l'édition
   française). C'est la source principale ;
3. **MyAnimeList** — n'est sollicité **qu'en secours**, si MangaDex ne reconnaît pas la série ;
4. **Google Books** — complète une **jaquette de tome** que MangaDex n'aurait pas, et donne le titre
   d'un livre que la BnF ne connaît pas encore.

Acervatim vous propose alors la série pour **valider les tomes** d'un coup, chaque tome avec sa
propre jaquette. Ce scan fonctionne **même sans compte relié** : MangaDex et la BnF sont des
sources ouvertes.

### Comment l'app retrouve la bonne série

La BnF connaît, pour chaque édition française, le **titre d'origine** du manga (celui sous lequel
l'œuvre est référencée internationalement). Acervatim s'en sert pour interroger MangaDex (puis
MyAnimeList en secours), et **confirme le résultat avec l'auteur** — pas seulement avec le titre.
C'est ce double critère qui évite les confusions entre deux séries au nom proche, ou entre une série
et l'un de ses **spin-offs du même auteur** : un titre peut ressembler à un autre, mais l'auteur (et
le titre exact) tranchent.

- **Titres proches, bonne série.** Si le titre recherché est **contenu** dans celui d'une série
  MangaDex *et* que l'auteur correspond, la correspondance est considérée comme **certaine**.
- **Titres gardés en version originale.** Beaucoup de mangas conservent leur titre d'origine en France
  (*Black Torch*, *One Piece*…). Même quand la BnF n'indique pas de titre d'origine séparé, Acervatim
  sait retomber sur le titre français — à condition, là encore, que **l'auteur corresponde**.
- **En cas de doute, rien n'est inventé.** Si aucun résultat ne correspond de façon fiable (ni par le
  titre, ni par l'auteur), l'app garde les informations de la BnF et n'attribue pas de série au hasard.

**Les éditions spéciales sont reconnues.** Si vous possédez par exemple *L'Attaque des Titans* en
**édition Colossale** (un grand format qui regroupe plusieurs tomes de l'édition classique), le scan
le détecte : Acervatim sait qu'il s'agit de la Colossale et **à quels tomes de l'édition standard**
ce volume correspond. Vous ne mélangez donc pas une édition collector avec l'édition classique.

### La langue du résumé

Le **résumé** affiché est le plus souvent **en français** : MangaDex fournit un synopsis français pour
la plupart des séries reconnues. À défaut, Acervatim utilise le résumé de la BnF (français) puis, en
dernier recours, celui de MyAnimeList — **en anglais** (MyAnimeList ne propose pas de résumés
traduits). Vous pouvez toujours le **modifier** après coup.

> Selon les ouvrages, l'information peut être partielle (séries en cours de publication, petits
> éditeurs, catalogage incomplet). Dans ce cas, l'application pré-remplit ce qu'elle a trouvé et vous
> complétez le reste — rien n'est jamais imposé.

## Ajouter une série par son titre {#ajouter-une-serie-par-son-titre}

Pas besoin de posséder un tome pour ajouter une série — c'est tout l'intérêt d'une wishlist.
*Fonctionnalité prête dans l'application ; elle sera disponible quand le serveur de production aura
été mis à jour.*

1. **Vous tapez le titre**, en français ou dans sa version originale (« L'Attaque des Titans »
   comme « Shingeki no Kyojin »). La recherche passe par **MangaDex**, sans compte à relier.
2. **Vous choisissez la série** parmi les résultats, chacun avec sa jaquette, ses auteurs et son
   année : c'est vous qui tranchez entre deux séries au nom proche.
3. **Vous choisissez l'édition française**, d'après la BnF : éditeur, édition standard, colossale,
   perfect…, avec son nombre de tomes. Deux éditions d'une même série restent deux séries distinctes
   dans votre collection.
4. Tous les tomes de l'édition rejoignent votre **wishlist**. Si la série est introuvable, vous
   pouvez la créer à la main, déjà pré-remplie.

### Les nouveaux tomes d'une série en cours

Quand vous ouvrez une série encore en cours de parution, Acervatim vérifie (au plus une fois par
semaine) si de nouveaux tomes ont paru et vous propose de les ajouter à la wishlist — sans rien
ajouter d'office. Ces tomes apparaissent **quelques semaines après leur sortie** : c'est le temps
qu'il faut à la BnF pour les cataloguer après le dépôt légal. Comme l'ajout par titre, cette
vérification attend la mise à jour du serveur de production.

## Et votre vie privée ?

Acervatim est conçu **privacy by design**. Concernant ces sources de données :

- Les recherches portent sur **l'objet** (un disque, un livre, une boîte), **jamais sur vous**.
  Aucun nom, email ni identifiant personnel n'est transmis aux sources.
- Pour les sources qui demandent un accès (Discogs, MyAnimeList, TMDB), le serveur utilise **vos
  propres jetons d'accès**, que vous reliez dans les réglages ; en **premium**, ce sont les jetons
  Acervatim, gardés côté serveur, qui sont utilisés à votre place. Les autres sources ne demandent
  aucun accès.
- Vous restez **maître de vos données** : ce qui est récupéré sert uniquement à pré-remplir une fiche,
  que vous pouvez modifier ou ignorer.

Pour le détail du traitement des données personnelles, voir la [politique de
confidentialité](../legal/confidentialite.md).

## Pour les développeurs

Le fonctionnement technique (d'où vient chaque champ, les débits, les accès, le « pivot »
ISBN → BnF → MangaDex → MyAnimeList, le format des réponses) est documenté côté développeur :
[Sources de données](../developpeur/sources-et-pivot-isbn.md).
