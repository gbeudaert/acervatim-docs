---
title: D'où viennent les informations
layout: default
parent: Utilisateur
nav_order: 3
---

# D'où viennent les informations de vos collections

Acervatim fonctionne **hors-ligne par défaut** : vos collections vivent sur votre appareil, sans
compte ni réseau. Quand vous le souhaitez (et que vous êtes connecté), l'application peut
**enrichir** un élément — récupérer automatiquement sa pochette, son synopsis, ses auteurs, sa date,
etc. — en interrogeant des **bases de données spécialisées** reconnues.

Cette page explique **quelles sources** sont utilisées et **ce que ça implique pour votre vie privée**.

## Les sources par type de collection

| Type | Source(s) | Ce qu'on y trouve |
|---|---|---|
| **Vinyles / disques** | [Discogs](https://www.discogs.com) | Album, artiste(s), année, pochette, label — recherche par texte **ou par code-barres** (EAN). |
| **Mangas** | [MangaDex](https://mangadex.org) + [Bibliothèque nationale de France (BnF)](https://catalogue.bnf.fr), avec [MyAnimeList](https://myanimelist.net) et [Google Books](https://books.google.com) en secours | L'**œuvre** (synopsis **en français**, note, genres, nombre de tomes, jaquettes) côté MangaDex ; l'**édition française** (éditeur, tome, ISBN) côté BnF ; MyAnimeList prend le relais si MangaDex ne reconnaît pas la série, Google Books complète les jaquettes manquantes — recherche **par ISBN**. |
| **Films** | [TMDB](https://www.themoviedb.org) | Titre, affiche, année, synopsis. |

> L'enrichissement est **optionnel** et **à votre initiative** : l'application reste pleinement
> utilisable sans jamais interroger ces sources.

## Scanner un manga : retrouver la série et tous ses tomes

Pour les mangas, vous pouvez **scanner le code-barres (ISBN)** d'un tome. À partir de ce seul
ISBN, Acervatim combine **plusieurs sources complémentaires**, chacune avec son rôle :

1. **BnF** — identifie l'ouvrage dans le catalogue officiel des éditions françaises (titre de
   série, n° de tome, éditeur, ISBN) et sait **énumérer tous les tomes** d'une édition ;
2. **MangaDex** — retrouve la **série** et ses informations d'ensemble : **synopsis en français**,
   note, genres, nombre de tomes, et la **jaquette de chaque tome** (souvent celle de l'édition
   française). C'est la source principale ;
3. **MyAnimeList** — n'est sollicité **qu'en secours**, si MangaDex ne reconnaît pas la série ;
4. **Google Books** — complète une **jaquette de tome** que MangaDex n'aurait pas.

Acervatim vous propose alors la série pour **valider les tomes** d'un coup, chaque tome avec sa
propre jaquette. Ce scan fonctionne **même sans compte connecté** : MangaDex et la BnF sont des
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

## Et votre vie privée ?

Acervatim est conçu **privacy by design**. Concernant ces sources de données :

- Les recherches portent sur **l'objet** (un disque, un livre), **jamais sur vous**. Aucun nom, email
  ni identifiant personnel n'est transmis à Discogs, MangaDex, MyAnimeList, la BnF, Google Books ou TMDB.
- En mode **connecté**, les sources sont interrogées avec **vos propres jetons d'accès** (que vous
  renseignez dans les réglages) ; en **premium**, ce sont les jetons Acervatim, gardés côté serveur,
  qui sont utilisés à votre place.
- Vous restez **maître de vos données** : ce qui est récupéré sert uniquement à pré-remplir une fiche,
  que vous pouvez modifier ou ignorer.

Pour le détail du traitement des données personnelles, voir la [politique de
confidentialité](../legal/confidentialite.md).

## Pour les développeurs

Le fonctionnement technique (d'où vient chaque champ, le « pivot » ISBN → BnF → MangaDex → MAL, le
format des réponses) est documenté côté développeur :
[Sources de données & pivot ISBN](../developpeur/sources-et-pivot-isbn.md).
