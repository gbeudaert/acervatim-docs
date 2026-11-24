---
title: Sources de données
layout: default
parent: Développeur
nav_order: 8
---

# Sources de données, type par type, et pivot ISBN manga

D'où viennent les métadonnées que le serveur renvoie, comment chaque source est appelée, et où elle
en est. La seconde moitié de la page détaille le cas le plus complexe : la résolution d'un **ISBN de
manga** en une série. Le code fait foi : la constellation `acervatim-services` (services `search` et
`rebrickable-data`), en production depuis le 23 novembre 2026, et le monolithe `acervatim-backend`
(`src/common/sources/`, `src/oauth/providers/`), dont elle a repris les adapters.

**Statuts** : **en service** = appelée par la production ; **prête** = branchée dans le code, pas en
production ou sans écran dans l'application ; **pending** = attend une réponse ou un accord d'un
tiers.

## Toutes les sources

| Source | Collection | Rôle | Accès | Débit sortant | Statut |
|---|---|---|---|---|---|
| [Discogs](https://www.discogs.com) ([API](https://www.discogs.com/developers)) | vinyle | Recherche par code-barres et par texte, détail d'un pressage | Compte Discogs de l'utilisateur (OAuth 1.0a) ; en premium, jeton personnel Acervatim (`DISCOGS_ACERVATIM_TOKEN`) | File `discogs` : 1 requête/s (Discogs : environ 60/min par identité) | en service |
| [BnF](https://catalogue.bnf.fr) ([SRU](https://api.bnf.fr/fr/api-sru-catalogue-general)) | manga | ISBN → tome et édition française ; liste des tomes d'une édition ; éditions d'une série ; tomes catalogués depuis une date | Aucun | File `bnf` : 10 requêtes/s, 2 en parallèle | en service |
| [MangaDex](https://mangadex.org) ([API](https://api.mangadex.org/docs/)) | manga | Identité de la série, synopsis français, jaquettes par tome, recherche par titre | Aucun | File `mangadex` : 4 requêtes/s (MangaDex : environ 5/s par adresse IP) | en service |
| [MyAnimeList](https://myanimelist.net) ([API](https://myanimelist.net/apiconfig/references/api/v2)) | manga | Repli d'identité et de recherche quand MangaDex échoue | Compte de l'utilisateur (OAuth 2 PKCE) ; en premium, `MAL_CLIENT_ID` | File `mal` : 3 requêtes/s, 2 en parallèle | en service |
| [Google Books](https://books.google.com) ([API](https://developers.google.com/books)) | manga | Repli de jaquette et de résumé par tome ; titre d'un ISBN inconnu de la BnF | Clé facultative (`GOOGLE_BOOKS_API_KEY`) | File `gbooks` : 4 requêtes/s, 1 en parallèle | en service |
| [TMDB](https://www.themoviedb.org) ([API](https://developer.themoviedb.org/docs)) | film | Recherche et détail d'un film | Clé personnelle de l'utilisateur (`PUT /v1/sources/tmdb/token`) ; en premium, `TMDB_API_KEY` | File `tmdb` : 20 requêtes/s, 4 en parallèle | prête : pas de collection film dans l'application |
| [Rebrickable](https://rebrickable.com) ([téléchargements](https://rebrickable.com/downloads/)) | LEGO | Catalogue des sets, thèmes, figurines, images | Aucun : fichiers CSV publics | Un import par jour, 03:00 UTC | prête : service `rebrickable-data`, en production depuis le 23 novembre 2026, sans écran LEGO dans l'app |
| [Brickset](https://brickset.com) ([API](https://brickset.com/article/52664/api-version-3-documentation)) | LEGO | Codes-barres EAN et UPC des boîtes | Clé `BRICKSET_API_KEY` | Brickset autorise 100 appels par jour sur une clé gratuite ; le code en consomme 90 au plus (`BRICKSET_DAILY_CALL_BUDGET`), un par code scanné absent du cache | pending : clé reçue le 20 novembre 2026, accord pour un usage commercial à demander |
| [GameUPC](https://gameupc.com) | jeu de société | Code-barres → identifiant BoardGameGeek ; retour des choix des utilisateurs | Clé `GAMEUPC_API_KEY` (production) ; serveur de test en développement | Files `gameupc` (15/min) et `gameupc-feedback` (5/min), sous les ~30 requêtes/min que GameUPC indique | prête : clé de production accordée le 18 novembre 2026 ; service `search`, en production depuis le 23 novembre 2026, sans écran jeux de société dans l'app |
| [BoardGameGeek](https://boardgamegeek.com) ([XML API2](https://boardgamegeek.com/wiki/page/BGG_XML_API2)) | jeu de société | Fiche d'un jeu | Jeton d'application `BGG_API_TOKEN`, vide : aucun appel ne part | File `bgg` : 1 requête toutes les 2 s ; cible : `bgg-data` seul appelant, 1 toutes les 5 s | pending : licence demandée |

Les livres (`book`) et les jeux vidéo (`game`) sont des types de collection du serveur sans aucune
source : saisie manuelle seulement, et aucun écran dans l'application.

Chaque appel sortant passe par une **file BullMQ** propre à la source (Redis) : le limiteur borne le
débit tous utilisateurs confondus, et les requêtes identiques simultanées n'en font qu'une. Les
réponses vont dans un **cache partagé** indexé sur la requête publique, jamais sur l'utilisateur.

## Conditions d'usage et palier premium

Acervatim est aujourd'hui gratuit, édité par une personne physique, sans publicité : un usage non
commercial. Le palier premium, payant, changera ce statut pour plusieurs sources. Ce tableau
recense ce qui a été relu et ce qui devra être obtenu avant le lancement du premium.

| Source | Ce que disent les conditions | Avant le premium | Statut |
|---|---|---|---|
| Discogs | [Conditions de l'API](https://support.discogs.com/hc/en-us/articles/360009334593) : licence limitée, non exclusive et révocable | Relire les conditions pour un service payant | à vérifier |
| BnF | [Conditions de réutilisation](https://www.bnf.fr/fr/conditions-de-reutilisations-des-donnees-de-la-bnf) : métadonnées sous Licence Ouverte, citation de la source obligatoire ; l'extraction d'une part substantielle à des fins commerciales est encadrée | Vérifier que l'interrogation à la demande n'est pas concernée | à vérifier |
| MangaDex | [Politique d'usage](https://api.mangadex.org/docs/2-limitations/) : pas de publicité ni de service payant, sauf exception accordée ; créditer MangaDex | Obtenir une exception écrite | à demander |
| MyAnimeList | Accord de licence accepté à l'enregistrement du client, non relu | Relire l'accord | à vérifier |
| Google Books | [Conditions](https://developers.google.com/books/terms) : faire payer l'usage de l'application demande un accord écrit de Google ; [règles de marque](https://developers.google.com/books/branding) : attribution à Google Books | Obtenir l'accord, ou ne pas servir Google Books dans le palier payant | à demander |
| TMDB | [Conditions de l'API](https://www.themoviedb.org/api-terms-of-use) : attribution obligatoire ; usage commercial seulement sous accord écrit (page à relire en entier) | Obtenir l'accord avant d'utiliser la clé Acervatim pour le premium | à demander |
| Rebrickable | Conditions relues le 2 novembre 2026 : usage commercial autorisé, automatisation limitée à la page des téléchargements, images servies par lien | Rien | acquis |
| Brickset | 100 appels par jour sur une clé gratuite, plafond levable sur demande ; API réservée aux applications complémentaires, conditions commerciales non publiées | Accord écrit | pending |
| Images LEGO | [Fair Play de LEGO](https://www.lego.com/fr-fr/legal/notices-and-policies/fair-play) : usage non commercial seulement, mention de marque exigée | Décision sur l'affichage des images dans une application payante | pending |
| GameUPC | Aucune condition publiée ; le mainteneur a écrit le 18 novembre 2026 : ni coût ni attribution, y compris pour un palier payant, environ 30 requêtes par minute au plus | Rien | acquis |
| BoardGameGeek | [Conditions de l'API XML](https://boardgamegeek.com/xmlapi/termsofuse) : licence par application, jeton jamais demandé à l'utilisateur, logo « Powered by BGG » | Licence commerciale, en général gratuite jusqu'à 100 utilisateurs payants | pending |

## Le proxy de recherche unifié

Le serveur ne laisse jamais l'application parler à une source : il expose un **proxy unifié**
`GET /v1/search?type=<type>&q=<texte>` **ou** `?type=<type>&barcode=<code>` (les deux paramètres sont
mutuellement exclusifs) et renvoie un `UnifiedItem` indépendant de la source.

Chaque type est servi par un **adapter** (`SourceAdapter`) :

| `type` | Source | `search` (texte) | `searchByBarcode` | `fetchDetails` |
|---|---|---|---|---|
| `vinyl` | Discogs | ✅ | ✅ (EAN via `barcode=`) | ✅ |
| `manga` | MangaDex (+ BnF), MAL en repli | ✅ (**MangaDex**, MAL en repli) | ✅ (**ISBN → pivot BnF → MangaDex**) | ✅ (MAL) |
| `movie` | TMDB | ✅ | — | ✅ |

> L'adapter manga reste `MalAdapter` (il porte l'OAuth MAL). Son `search` interroge **MangaDex sans
> jeton** et ne passe à MAL qu'en repli (cf. [Recherche d'une série par titre](#recherche-serie-titre)) ;
> son `searchByBarcode` orchestre le pivot **BnF → MangaDex → MAL repli**. MangaDex n'a **pas
> d'adapter** ni de jeton (API publique, comme la BnF), il est injecté via `MangaDexCoverService`.

Deux types passent par des routes à part, présentes **dans la constellation seulement** :

- **Jeux de société** : `GET /v1/search/boardgame/barcode/:code` (`?q=` pour affiner, `?format=` pour
  le format du code scanné) et `POST /v1/search/boardgame/barcode/:code/choice`. Réservées aux
  utilisateurs connectés, premium ou non ; le choix est plafonné à 30 par minute. Voir
  [Jeux de société](#jeux-de-societe).
- **LEGO** : aucune route `/v1` pour l'instant. Le catalogue vit dans `rebrickable-data`, qui ne
  répond qu'aux autres services. Voir [LEGO](#lego).

## Vinyle : Discogs {#vinyle}

- **Scan** : `barcode=` part vers `database/search?barcode=` ; la recherche texte vers
  `database/search?q=`. Un résultat texte porte le numéro de catalogue, les codes-barres et le
  `master_id`, commun à tous les pressages d'un album : l'application regroupe ainsi les 20 à 30
  pressages d'un même album.
- **Accès** : le compte Discogs de l'utilisateur (OAuth 1.0a, signé côté serveur) ; à défaut, en
  premium, le jeton personnel Acervatim, qui donne aussi accès aux images. Sans l'un ni l'autre :
  `403 source-token-required`.
- **Cache** : en texte, une requête vaut environ un appel Discogs (taux de succès du cache relevé
  à 0,25 en développement, contre 0,67 par code-barres).

## Film : TMDB {#film}

- `GET /v1/search?type=movie&q=` puis le détail du film ; pas de recherche par code-barres.
- **Accès** : TMDB n'a pas d'OAuth. La clé personnelle se saisit par `PUT /v1/sources/tmdb/token`
  (chiffrée, provider `tmdb`), `DELETE` pour la retirer ; en premium, la clé serveur `TMDB_API_KEY`.
- **État** : le serveur stocke déjà des collections `movie`. L'application n'a encore ni type film
  ni écran ; c'est le prochain type de collection prévu dans l'application, avec le LEGO.

## LEGO : Rebrickable et Brickset {#lego}

Le catalogue LEGO de référence est le service `rebrickable-data` de la constellation, sans route
`/v1` : il ne répond qu'aux services voisins (`/internal/sets`, `/internal/sets/by-barcode/:code`,
`/internal/sets/:setNum`, `/internal/themes`).

- **Rebrickable** : import quotidien, à 03:00 UTC, des CSV publics (`sets`, `themes`,
  `inventories`, `inventory_minifigs`) depuis `cdn.rebrickable.com/media/downloads/`. Pas d'API, pas
  de clé : les conditions de Rebrickable n'autorisent l'automatisation que sur la page des
  téléchargements. Les images restent servies depuis Rebrickable, jamais recopiées.
- **Brickset** : seule source connue des codes-barres de boîtes. Il n'est appelé qu'au scan d'un code
  absent du cache (décision du 16 novembre 2026, codée le 17) : un appel `getSets` au
  plus par code, dans un budget de 90 appels par jour UTC sur les 100 que Brickset autorise. Seuls
  `ean`, `upc` et `brickset_id` sont gardés, 90 jours, puis effacés ; un code que Brickset ne connaît
  pas est redemandé après 7 jours (choix validé le 17 novembre 2026 : un set neuf obtient vite son
  code chez Brickset, et un délai plus court userait le budget en scans répétés d'un code inconnu).
  Sans `BRICKSET_API_KEY`, le catalogue est complet mais le scan ne
  trouve rien. La documentation de l'API ne dit pas que la recherche porte sur les codes-barres ;
  le premier appel réel, le 20 novembre 2026, l'a vérifié sur un EAN (`5702015869935` → `75192-1`),
  pas encore sur un UPC seul.
- **État** : `rebrickable-data` a tourné contre une vraie base en développement, il n'est pas en
  production. Le type de collection `lego` existe côté serveur depuis le 17 novembre 2026 :
  `catalog-lego` refait sur `catalog-core`, recherche par `GET /v1/search?type=lego` (texte ou
  code-barres), lue par `search` dans `rebrickable-data`, sans appel externe. L'ancien
  `catalog-lego`, qui interrogeait l'API Rebrickable page par page, est supprimé. L'application n'a
  pas encore d'écran LEGO.

## Jeux de société : GameUPC et BoardGameGeek {#jeux-de-societe}

- **Code-barres → jeu** : GameUPC, base communautaire, associe un code-barres à un identifiant
  BoardGameGeek. Le code est normalisé avant l'appel (`400` s'il est invalide), car GameUPC ne valide
  rien. Statuts rendus : `verified` (un seul jeu), choix parmi des candidats, `unknown`,
  `unavailable` ; la dernière réponse connue est servie si GameUPC est injoignable.
- **Retour des choix** : le jeu choisi par l'utilisateur est renvoyé à GameUPC (file
  `gameupc-feedback`) sous un pseudonyme HMAC (`GAMEUPC_VOTER_PEPPER`). C'est ce vote qui fera la
  couverture des boîtes françaises.
- **Fiche du jeu** : BoardGameGeek (XML API2, `thing`). Tant que `BGG_API_TOKEN` est vide, aucun appel
  ne part et la réponse vit des seules données GameUPC. Pas de jeton par utilisateur : BoardGameGeek
  l'interdit. Logo « Powered by BGG » obligatoire dans l'application.
- **Cible décidée le 14 novembre 2026** : un service `bgg-data` sera le seul à appeler BoardGameGeek,
  à une requête toutes les 5 s, et `search` y lira les fiches ; il attend la licence. Le choix de
  l'édition d'un jeu sera proposé en option, et le scan restera ouvert à tous les utilisateurs
  connectés.

## Résolution des jetons (BYOT / repli premium / mode dégradé)

Chaque source à jeton (Discogs, MAL, TMDB) résout son jeton **de la même façon**, via
`TokenResolverService` (`src/oauth/token-resolver.service.ts` dans le monolithe ; dans la
constellation, le service `connections` porte cette résolution et `search` la lui demande) :

1. **Jeton utilisateur (BYOT)** — si l'utilisateur a connecté la source, on utilise **son** jeton
   (répartition des quotas fournisseur sur son compte, résultats liés à son compte source).
2. **Repli premium** — sinon, si l'utilisateur est **premium**, repli sur les **jetons Acervatim** :
   `X-MAL-CLIENT-ID` (MAL), personal access token (Discogs, `DISCOGS_ACERVATIM_TOKEN`), clé serveur
   (TMDB, `TMDB_API_KEY`). Le limiteur de la file de chaque source, commun à tous les utilisateurs,
   protège ces jetons partagés : il a remplacé les anciens buckets `acervatim:<source>`.
3. **Mode dégradé** — sinon (non-premium sans jeton) : **aucun appel sortant** vers une source à
   jeton. On sert les sources **sans clé** (BnF, **MangaDex**) et les réponses déjà en **cache
   partagé** ; à défaut, l'erreur actionnable **`403 source-token-required`** (avec `provider`), qui
   invite à connecter la source **ou** à passer premium.

> **Conséquence de l'inversion du pivot** : MangaDex étant **sans jeton** (API publique, comme la
> BnF), le chemin nominal du scan manga (identité + synopsis FR + jaquettes) fonctionne **même en mode
> dégradé** — un utilisateur non-premium sans compte MAL obtient désormais une fiche enrichie, là où
> l'ancien pivot MAL retombait en `bnf_only`. MAL (à jeton) n'intervient plus qu'en repli.

Le cache de recherche est indexé sur la **requête publique** (`<source>:<requête>`, sans `userId`) :
une réponse obtenue via jeton Acervatim est identique à une réponse jeton-utilisateur (endpoints
publics interrogés ici), donc **partageable sans fuite**. Le mode dégradé ne sert donc que des clés
partagées.

**TMDB n'a pas d'OAuth** : la clé personnelle se saisit via `PUT /v1/sources/tmdb/token` (stockée
chiffrée dans `oauth_credentials`, provider `tmdb`) ; `DELETE` pour la révoquer.

## Manga : la résolution d'un ISBN

> **Ordre des sources.** Le chemin nominal du scan manga est **BnF → MangaDex → MAL en repli**.
> MangaDex identifie l'œuvre (titre romaji/FR + auteur), fournit le **synopsis en français**, la note,
> le nombre de tomes, les genres, les **jaquettes par tome** et le **`links.mal`** (l'id MyAnimeList,
> porté en `metadata` pour la sync bibliothèque). MAL n'est appelé **qu'en repli** quand MangaDex
> n'identifie pas.

## Vue d'ensemble : quatre sources pour un manga

À partir d'un seul **ISBN**, plusieurs sources se répartissent le travail — chacune à sa granularité :

| Source | Rôle | Granularité |
|---|---|---|
| **BnF** (SRU) | identifie le tome (titre de série FR + romaji, n° de tome, éditeur, ISBN) et **énumère tous les tomes** d'une édition | tome / édition |
| **MangaDex** (principal) | **identité série** : synopsis **FR**, note, statut, nb de tomes, genres, **jaquettes par tome**, et le **`links.mal`** (id MyAnimeList) | série + tome |
| **MyAnimeList** (repli) | identité série **si MangaDex n'identifie pas** (recherche floue historique) ; sinon **joignable par id** via `links.mal` (sync bibliothèque) | série |
| **Google Books** (repli jaquette) | **jaquette par tome** (par ISBN) quand MangaDex n'a pas la couverture du tome | tome |

BnF + **MangaDex** forment le **pivot ISBN → série** (étapes A→D ci-dessous). MangaDex apporte, en un
seul point, ce qui demandait auparavant MAL **et** Google Books : identité + synopsis FR + jaquettes
par tome. **MAL** est le repli d'identité ; **Google Books** le repli de jaquette au tome. Détails :
[Jaquettes par tome](#jaquettes-par-tome-mangadex--google-books).

## Le problème : ni MyAnimeList ni Google Books ne suffisent seuls

MyAnimeList (API v2) **n'a aucun lookup par ISBN**, n'indexe que des titres **romaji / anglais /
japonais** (jamais le titre français), renvoie `num_volumes = 0` pour les séries en cours, et — décisif
pour un public FR — **n'a aucun synopsis en français**. Google Books, lui, résout mal les ISBN papier
FR (Ki-oon, Kana, Glénat… surtout les `979…`) : couverture jaquettes mesurée **~20-33 %**.

La **BnF** (dépôt légal) est l'autorité sur les éditions FR et — c'est la clé du pont — ses notices
manga portent le **titre original romaji**. **MangaDex** (base communautaire) prend le relais sur ce
titre : indexée par série, elle couvre **~80 %** des tomes des séries reconnues avec la jaquette de
l'édition FR, fournit le **synopsis FR** (`description.fr`), et expose sur chaque œuvre le **`links.mal`
/ `links.al`** — un join **fiable** vers MAL/AniList, sans recherche floue. D'où un **pivot
BnF → MangaDex**, MAL n'étant plus sollicité qu'en dernier recours.

## Le pipeline

```
ISBN (EAN-13)
 └─(A) BnF SRU ── recherche par ISBN ─────────────▶ notice UNIMARC
        │
        │  ISBN inconnu de la BnF (bnf_not_found) ─▶ (A') Google Books  q=isbn:<isbn> ──▶ titre
        │        └─(B') échelle de troncature ──▶ préfixes candidats ──▶ (C)
        │
        └─(B) extraction ──▶ titre original (454$t), titre FR (200$a), titre série (461$t),
               │             édition (205), plage de tomes source (454$h), éditeur (210/214$c),
               │             auteur (700), note FR (330), ISBN (010), ARK (003)
               └─(C) MangaDex ── q = <romaji 454$t> sinon <titre FR 200$a> sinon <préfixe B'> ──▶ candidats
                      └─(D) désambiguïsation ── score = titre + AUTEUR ──▶ identité
                             │   { mangaId, synopsis.fr, links.mal, links.al, note, genres,
                             │     jaquettes par tome } → item source='mangadex' (mal_id porté)
                             └─(repli) MangaDex n'identifie pas ──▶ MyAnimeList (recherche floue,
                                        score titre+auteur+type) ──▶ item source='mal'
                                        └─(repli) ──▶ notice BnF seule (source='bnf', bnf_only)
                                        └─(repli, voie A') ──▶ item minimal (source='gbooks', gbooks_only)
                             ─────────▶ UnifiedItem (série) + metadata.scannedTome
```

Le chemin (D) principal (**MangaDex**) est décrit ci-dessous ; le repli MAL — inchangé — l'est en
[Repli MyAnimeList](#repli-c-d--myanimelist-identité-floue). Les deux partagent les **mêmes helpers de
rapprochement** (`src/common/sources/manga-matching.ts` : normalisation, tokens auteur, similarité de
titre, seuil `PIVOT_TITLE_STRONG`).

### Étape A+B — BnF
`BnfService` (`src/common/sources/bnf/bnf.service.ts`) interroge le **catalogue
général BnF** en SRU (HTTP, sans authentification) :

- Endpoint `https://catalogue.bnf.fr/api/SRU`, `recordSchema=unimarcxchange` (impératif :
  `dublincore` écrase série/tome dans le titre).
- Index `bib.fuzzyISBN` (accepte ISBN-10/13 et EAN indifféremment).
- Parsing UNIMARC **sans dépendance externe**
  (`src/common/sources/bnf/unimarc.parser.ts`) — extraction
  ciblée par regex, tolérante au préfixe de namespace `mxc:`.

Zones extraites → `BnfNotice` :

| Zone UNIMARC | Champ | Rôle |
|---|---|---|
| `454$t` (fallback `500$a`) | `originalTitle` | **Titre original romaji** — le pont vers MyAnimeList. |
| `454$h` | `sourceVolumeRange` | **Plage de tomes de l'édition source** (ex. `1-3`) → mapping inter-édition. |
| `205$a` | `edition` | Mention d'édition (ex. `Éd. colossale`) ; absente = standard. |
| `200$a` / `200$h` | `titleFr` / `volume` | Titre FR et n° de tome **du volume scanné** (`200$a` peut être le titre du *tome*, pas de la série — cf. Ki-oon). |
| `461$t`, sinon `225$a` ou `200$a` | `seriesTitle` | **Titre de la série** (lien « fait partie de ») — clé d'énumération des tomes. Sans `461`, le monolithe prend `225$a`, puis `200$a` ; la constellation ne prend `225$a` que s'il numérote le tome (cf. ci-dessous). |
| `210$c` \| `214$c` | `publisherFr` | Éditeur FR (les notices RDA-fr récentes utilisent `214`). |
| `700$a/$b` | `authors` | Auteur structuré (nom / prénom). |
| `330$a` | `noteFr` | **Note de résumé en français** (souvent absente pour les mangas) → synopsis. |
| `003` | `ark` | Identifiant pérenne BnF. |

### Étape A'+B' — repli Google Books quand la BnF est muette {#repli-google-books}

La BnF ne connaît pas tous les ISBN : une **parution récente** (retard de catalogage du dépôt légal) ou
un **éditeur non français** (préfixes belges, suisses…) donnent `numberOfRecords 0`. Tout le pivot étant
conditionné à la notice, le scan renvoyait alors une **liste vide** alors que la série existe chez
MangaDex — seul le maillon d'entrée manquait. Le repli le rouvre :

1. **Titre par ISBN** — `GET /books/v1/volumes?q=isbn:<isbn>` donne le **titre commercial** du tome, ses
   auteurs (présents dans **42 %** des notices manga FR mesurées) et sa date. Même file `gbooks` que les
   jaquettes : un seul throttle sortant, un seul cache, single-flight par ISBN.
2. **Échelle de troncature** — le titre Google n'est pas normalisé (`One Piece Tome 112`, `Frieren`,
   `Chainsaw Man Tome 18 . Edition collector`, voire du déchet de coffret) et le champ structuré
   `seriesInfo.bookDisplayNumber` est **vide sur les 137 tomes mesurés** : le n° de tome ne peut venir
   que du titre. Comme MangaDex ne tolère aucun bruit (`title=One Piece Tome 112` → `total=0`), on
   retire les tokens **par la fin**, du titre complet au plus court, et on interroge à chaque échelon
   jusqu'au **premier préfixe accepté**. Le n° de tome retenu est celui de la queue retirée du préfixe
   **gagnant** : il n'est cru que parce que le préfixe, lui, a matché.
3. **Un seul garde-fou, non linguistique** : on ne tronque que si la queue retirée **contient un
   chiffre**. Pas de liste de mots (« Tome », « T. », « vol. ») — les titres ne sont pas normalisés, une
   telle liste serait fausse pour la moitié du corpus. Sans ce garde-fou, `Instinct grégaire` (tome
   Ki-oon) se tronquerait en `Instinct` et matcherait une série réellement nommée *Instinct*.

Seul `bnf_not_found` emprunte cette voie. Une panne BnF (`bnf_unavailable`, `bnf_unparsable`) est
**transitoire** : y greffer un repli dégraderait la qualité pendant un incident au lieu de laisser
l'app retenter.

**Acceptation renforcée en l'absence d'auteur.** Sans notice BnF, le validateur habituel (match auteur)
manque une fois sur deux. L'identification expose donc un drapeau **`ambiguous`** — un autre candidat
atteint le même score de titre — et le repli **refuse** quand `ambiguous` et qu'aucun auteur n'a matché.
C'est ce qui écarte le crossover *Frieren Cinnamoroll Kamigata* sur la requête d'un seul token
« Frieren », **sans** toucher au seuil `PIVOT_TITLE_STRONG` du chemin nominal. Quand Google fournit les
auteurs, ils redeviennent l'arbitre et la vraie série (*Sousou no Frieren*) est retenue.

**Taux mesuré** sur 137 tomes numérotés passés dans le chemin réel : **98 résolus (~72 %)**, ~84 % en
écartant les coffrets/goodies et les 6 ISBN absents de Google Books. Réserve de méthode : l'échantillon
est fait d'ISBN que la BnF **possède**, alors que le repli ne se déclenche que quand elle est muette —
sur les nouveautés et les éditeurs non français, la distribution des formes de titre est probablement
moins favorable.

### Étape C+D — MangaDex (identité principale) {#identite-mangadex}
`MangaDexResolver.identify` (`src/common/sources/mangadex/mangadex.resolver.ts`), orchestré par
`MangaDexCoverService.identifySeries` (cache + file BullMQ `mangadex`) et appelé en premier par
`MalAdapter.searchByBarcode` :

- **API publique, sans jeton** (comme la BnF) : `GET /manga?title=<romaji 454$t sinon titre FR 200$a>`
  avec `includes[]=author&includes[]=artist&includes[]=cover_art` (les relations portent alors le nom
  d'auteur et le `fileName` de couverture).
- **Désambiguïsation titre + auteur** (mêmes règles que le pivot MAL, via `manga-matching`) : parmi les
  candidats, on retient le **meilleur score** ; le **match auteur** (nom BnF `700` vs relations
  `author`/`artist`) est le validateur robuste. Un **bonus d'égalité exacte de titre** départage les
  **spin-offs du même mangaka** (ex. *Jigokuraku* qui remonte plusieurs entrées). Acceptation : auteur
  matché **et** titre proche, **ou** titre fortement similaire seul (sans auteur BnF exploitable).
- **Identité produite** : `{ mangaId, titre (FR préféré), synopsis.fr / .en, links.mal, links.al,
  status, year, lastVolume, genres (tags), auteurs, jaquette principale, jaquettes par tome, note }`.
  La **note bayésienne** vient de `GET /statistics/manga/{id}` (best-effort, non bloquant).
- **Sortie** : `MalAdapter.buildMangaDexItem` mappe cette identité en `UnifiedItem`
  **`source: 'mangadex'`**, `resolutionPath: 'bnf+mangadex'` (titre romaji) ou `'bnf+mangadex-fr'`
  (repli titre FR). Le **`mal_id`** (`links.mal`) et l'`anilistId` sont portés dans
  `metadata.pivot` — on **ne rappelle pas MAL**, on porte simplement son id pour la sync ultérieure.
- **Best-effort** : MangaDex indisponible ou **aucun candidat validé** → l'adapter **replie sur MAL**
  (ci-dessous). MangaDex reste donc **remplaçable** (source communautaire).
- **Réchauffage gratuit** : à l'identification, dès qu'un `malId` est connu, les jaquettes par tome
  sont **mises en cache** sous la clé série `mal` → l'endpoint `edition-mapping` les sert en cache-only
  sans re-appeler MangaDex. Le `mangaId` (et le `malId`) sont par ailleurs portés dans `metadata.pivot`
  et **renvoyés par l'app à `edition-mapping`/`import-jobs`** : le refresh série retrouve alors la même
  œuvre par `mangaId` (join direct), sans re-chercher par titre.

### Repli C+D — MyAnimeList (identité floue) {#repli-c-d--myanimelist-identité-floue}
`MalAdapter.malPivotFallback` (`src/oauth/providers/mal.adapter.ts`) — **appelé seulement si MangaDex
n'identifie pas**. C'est le pivot flou historique, inchangé :

- Jeton MAL **résolu** (cf. [Résolution des jetons](#résolution-des-jetons-byot--repli-premium--mode-dégradé)) :
  Bearer de l'utilisateur s'il a connecté MAL, sinon repli `X-MAL-CLIENT-ID` serveur (`MAL_CLIENT_ID`)
  pour un premium ; un non-premium sans jeton obtient la **notice BnF seule** — mappée en item avec
  jaquette Google Books, cf. §[Repli `bnf_only`](#repli-bnf-only) (aucun appel MAL). Les
  endpoints publics MAL renvoient un corps **identique** en Bearer et en `X-MAL-CLIENT-ID` (vérifié),
  donc un scan ISBN ne requiert pas d'OAuth pour un premium.
- Requête `q=<titre>` envoyée **brute** : le `q=` de MyAnimeList est *fuzzy côté serveur* et absorbe
  les écarts de romanisation (ex. `jyouou` ↔ `joou`). Aucune normalisation romaji n'est nécessaire pour
  interroger. Le titre interrogé est le **titre original romaji** (`454$t`) ; à défaut, le **titre FR**
  (`200$a`) en repli (voir *Repli titre FR* ci-dessous).
- **Consolidation** : parmi les candidats, on retient celui de **meilleur score** (voir §Scoring de
  confiance) — jamais un simple « premier résultat ». Le **match auteur** (nom BnF `700` vs `authors`
  MAL, insensible casse / accents / ordre) est le validateur robuste : il confirme un titre qui pourrait
  matcher par hasard. Sans signal suffisant → pas de sélection MAL, la **notice BnF est renvoyée**
  (`bnf_only`, cf. §Sortie).
- **Piège type LN vs manga** : l'œuvre source japonaise peut être un **light novel** alors que
  l'édition FR est une **adaptation manga** (cas réel : *Kakuzetsu Toshi no Joou* / *Attack on Titan:
  The Harsh Mistress of the City* est un LN sur MAL, catalogué adaptation manga côté BnF). Pour
  l'adaptation, l'auteur MAL peut être le **mangaka adaptateur** et non l'auteur original → accepter un
  match sur **l'un OU l'autre**.

### Scoring de confiance

Les deux chemins (MangaDex `chooseIdentity`, MAL `choosePivotCandidate`) partagent la **même** boîte à
outils (`manga-matching.ts`) et une logique de score analogue ∈ [0,1]. Côté **repli MAL**
(`choosePivotCandidate`), trois signaux :

```
confidence = 0.45 · similarité_titre  +  0.40 · match_auteur  +  0.15 · type_manga
```

Côté **MangaDex** (`chooseIdentity`), pas de `media_type` mais un **bonus d'égalité exacte de titre**
pour départager les spin-offs du même auteur :

```
confidence = 0.50 · similarité_titre  +  0.40 · match_auteur  +  0.10 · titre_exact
```

- **similarité_titre** ∈ [0,1] — **contenance** : si le titre interrogé est inclus dans le titre MAL
  (ou l'inverse), ex. « Tokyo toritsu » ⊂ « Jujutsu Kaisen 0: Tokyo Toritsu Jujutsu Koutou Senmon
  Gakkou » → **1.0** ; sinon **coefficient de Dice** sur bigrammes de caractères (fuzzy, tolère
  `jyouou` ↔ `joou`). Normalisation minuscules / sans accents / espaces compactés.
- **match_auteur** — 1 si un token de nom BnF (`700$a/$b`) est commun avec un `authors{}` MAL.
- **type_manga** — 1 si `media_type` ∈ { manga, manhwa, manhua, one_shot, doujinshi }.
- Le **rang MAL** ne sert que de départage infinitésimal entre scores égaux (`− 0.001 · rang`).

Un « **titre contenu + auteur** » vaut donc ≈ 0.85, +type → **1.0** : match certain. C'est ce qui
distingue le bon candidat d'un homonyme mieux classé par MyAnimeList (le titre seul ne suffit pas ;
l'auteur tranche).

**Règle d'acceptation** (garde-fou anti faux-positif — on préfère le repli suivant à un faux match) :

| Étape | Requête | Condition de rétention | `resolutionPath` |
|---|---|---|---|
| **MangaDex** (principal) | romaji `454$t` | auteur matché (**et** titre proche) **OU** similarité_titre ≥ 0.85 | `bnf+mangadex` |
| **MangaDex**, repli titre FR | titre FR `200$a` | idem | `bnf+mangadex-fr` |
| **MAL** (repli), titre original | romaji brut | match auteur **OU** (type manga **ET** similarité_titre ≥ 0.85) | `bnf+mal` |
| **MAL** (repli), titre FR | titre FR `200$a` | match auteur **obligatoire** | `bnf+mal-fr` |
| aucune source | — | — | `bnf_only` |

Le **repli titre FR** sert les mangas dont le titre est **conservé en graphie latine** en France
(*Black Torch*, *One Piece*…) : le titre FR = l'original, donc la recherche MAL aboutit. Mais un titre
FR étant plus générique (homonymes), **seul l'auteur valide** — sans match auteur, aucun candidat MAL
n'est retenu (repli `bnf_only`).

### Sortie
Un `UnifiedItem` au niveau **série** (pour le fan-out des tomes), enrichi de
`metadata.scannedTome` = `{ isbn, titleFr, volume, edition, publisherFr, sourceVolumeRange }` :
les données du **tome précis** scanné, à reporter dans l'item créé. Le champ `sourceVolumeRange` a une
place dédiée dans le schéma du tome (`MangaItemSchema`).

Le **synopsis** (`description`) dépend du chemin :

- **MangaDex** (nominal) : synopsis **FR** (`description.fr`) en priorité → à défaut note BnF `330$a`
  → à défaut synopsis EN MangaDex. Le public FR obtient donc majoritairement un **résumé français**.
- **MAL** (repli) : note BnF `330$a` (FR) si présente, sinon synopsis MyAnimeList (**anglais** — MAL
  ne fournit aucun résumé localisé).

`metadata.pivot` porte `{ confidence, authorMatched, resolutionPath }` et, sur le chemin MangaDex,
**`malId`** (`links.mal`), **`anilistId`** (`links.al`) et **`mangaId`** ; `metadata` porte aussi
`rating`, `num_volumes` (dérivé de `lastVolume`), `status` et `genres`.

`resolutionPath` dit par où l'identité est passée : `bnf+mangadex` / `bnf+mangadex-fr` (nominal),
`bnf+mal` / `bnf+mal-fr` (repli MAL), `bnf_only`, et — voie Google Books —
**`gbooks+mangadex`** ou **`gbooks_only`**. Sur cette voie, `scannedTome` est renseigné par ce que
Google fournit (`titleFr` = titre commercial, `seriesTitleFr` = préfixe gagnant, `volume` = n° extrait)
et les champs proprement BnF (`edition`, `publisherFr`, `sourceVolumeRange`) restent `null` : personne
ne les a fournis, on ne fabrique pas de fausse notice.

### Repli `bnf_only` — la notice BnF n'est jamais jetée {#repli-bnf-only}

Quand MyAnimeList **n'enrichit pas** — pas de jeton ni de hit de cache (mode dégradé), aucun candidat
retenu (homonyme sans auteur…), ou pas de titre pour interroger MAL — le backend ne renvoie **pas** une
liste vide : `MalAdapter.bnfOnlyResult` **mappe la notice BnF** en `UnifiedItem` (`source: 'bnf'`,
`resolutionPath: 'bnf_only'`), avec le **même `metadata.scannedTome`** que le chemin enrichi, la **note
FR** (`330$a`) en description si présente, et la **jaquette du tome scanné résolue par ISBN via Google
Books** (best-effort — `null` si absente). Un scan manga **sans jeton MAL affiche donc la fiche**
(série + tome), enrichissable plus tard. Le résultat n'est **vide** que si la notice n'a **aucun titre
exploitable** (ni série `461$t` ni tome `200$a`). `bnf` **et `mangadex`** sont des codes source à part
entière (`ITEM_SOURCE_CODES`) **sans adapter dédié** : à la création d'item, `SourceSnapshotService`
stocke la référence sans snapshot (le `mangaId` / l'ARK suffit).

> Avec le pivot MangaDex, `bnf_only` devient **plus rare** : MangaDex étant sans jeton, l'identification
> aboutit désormais même sans compte MAL. `bnf_only` ne subsiste que si **ni** MangaDex **ni** MAL
> n'identifient (aucun candidat validé, ou titre inexploitable).

### Repli `gbooks_only` — un item minimal plutôt qu'un écran vide {#repli-gbooks-only}

Sur la voie Google Books, quand **aucun préfixe** n'identifie de série (titre du tome seul, coffret,
goodies) ou que l'identification est refusée pour ambiguïté, le backend rend un **item minimal**
`source: 'gbooks'`, `resolutionPath: 'gbooks_only'` : titre Google, auteurs, jaquette et n° de tome
extrait. Objectif : que l'app enchaîne sur la **saisie manuelle** au lieu d'un « aucun résultat » muet.
Aucun nom de série n'est deviné — aucun préfixe n'ayant été validé, en inventer un ferait passer une
hypothèse pour une donnée. Si **Google non plus** ne connaît pas l'ISBN, le résultat est vide.

`gbooks` rejoint `bnf` et `mangadex` parmi les codes source **sans adapter dédié** (`ITEM_SOURCE_CODES`).

## Éditions collector (ex. Colossale)

C'est le pivot BnF qui rend les éditions spéciales correctes :

- L'**ISBN identifie l'édition** (chaque édition a le sien) ; `205` indique laquelle (`Éd. colossale`).
- `454$h` donne le **mapping de volumes** : une Colossale T.1 = tomes **1-3** de l'édition standard.
- **MyAnimeList ne sait pas** compter une édition FR (`num_volumes` = compte de l'œuvre japonaise, ou
  `0` si en cours) → le comptage fiable vient de la BnF, pas de MAL. Le code **loggue un avertissement**
  quand la série MAL est `currently_publishing` / `num_volumes=0`.

## Énumération des tomes (`edition-mapping`) {#enumeration-des-tomes}

`GET /v1/search/edition-mapping?title=<série>[&edition=<205>][&publisher=<éditeur>][&malId=<id>][&mangaId=<id>][&since=<AAAAMMJJ>]`
renvoie **tous les tomes** d'une édition (« récupérer toute la série d'un coup »). `malId` et
**`mangaId`** (optionnels, issus de `metadata.pivot` du scan ou du candidat choisi en recherche)
fiabilisent le join MangaDex des jaquettes (cf. [Jaquettes par
tome](#jaquettes-par-tome-mangadex--google-books)) ; `POST /v1/search/import-jobs` prend les mêmes
champs (sauf `since`). `BnfService.enumerateEdition` recherche
`bib.title all "<titre de série>" and bib.doctype any "a"` (texte imprimé : DVD et CD écartés,
paginé), puis pour chaque notice (`common/sources/bnf/edition-records.ts`) :

1. **Appartenance à la série** — l'ensemble **`461$t` fait autorité** quand il existe : la notice
   appartient à la série s'il **égale** le titre demandé (normalisé), et seulement dans ce cas — un
   spin-off qui répète le titre en `200$a` mais se rattache à un autre ensemble est écarté. **Sans
   `461`** (pré-notice, catalogage partiel), il suffit que **l'un de** `500$a` (titre uniforme),
   `200$a` (titre propre) ou `225$a` (collection) l'égale : la pré-notice d'un tome récent, dont
   `225$a` est la collection de l'éditeur (« Shonen manga »), reste membre par son `200$a`.
2. **Édition** — une édition est le couple **(éditeur, `205$a`)**. Un `publisher` **explicite** est
   comparé à l'éditeur normalisé (« Pika éd. » ≡ « Pika édition » ; « Pika roman » reste distinct).
   Le monolithe en fait un filtre strict. La constellation (`publisherFamilyOf`) garde aussi les
   **libellés successifs** de cet éditeur : la BnF catalogue le libellé imprimé au dépôt légal, et
   *Alice in Borderland* paraît sous « Delcourt-Akata » (T1-4), « Delcourt » (T5-8) puis « Éditions
   Delcourt » (T9-18) ; le filtre strict n'en rendait que 4 tomes au scan du T1. Un libellé rejoint
   celui demandé s'il ne porte **aucun** n° de tome déjà paru ; un autre catalogue reprend des n°
   (romans « Ki-oon roman », 2-en-1 France Loisirs) et reste écarté. Le préfixe ISBN ne départage
   pas : les romans Ki-oon partagent celui des mangas. La liste des éditions applique la même règle
   et réunit ces libellés en une ligne. **Absent**, l'éditeur **dominant** (le plus de tomes
   numérotés) ne gagne que les **collisions** de numéro : les 2-en-1 France Loisirs de *L'attaque
   des titans* (« 11-12 ») ne prennent plus la place des tomes 11 et 17 de Pika, mais un n° que seul un libellé voisin porte
   comble le trou (changement de raison sociale en cours de série, Tonkam → Delcourt-Tonkam). Une
   pré-notice sans `205` n'est pas rangée en standard si son titre porte un mot-clé d'art **hors des
   mots du titre de série** (« Hellsing perfect edition » → perfect ; *Perfect World* reste standard).
3. **N° de tome** — lu dans `461$v` / `225$v` seulement si leur titre est celui de la série (le `$v`
   d'une collection éditeur est un n° de collection), sinon `200$h` ; puis **dédup** par numéro.

La réponse indique l'éditeur retenu (`publisher`), et **`complete: false`** si l'énumération a été
tronquée ou qu'une page SRU a échoué. Chaque tome porte la date de création de sa notice
(`noticeCreatedAt`, `100$a`) ; `since` ne renvoie que les notices créées depuis cette date (cf.
[Nouveaux tomes](#nouveaux-tomes)).

> ⚠️ **Le titre de série est décisif.** Le catalogage varie : chez Pika (SNK, Maid sama) `200$a` **est**
> le titre de série, répété sur chaque tome ; chez **Ki-oon** (Jujutsu Kaisen) `200$a` est le titre du
> **tome** (« Je vais te tuer ») et le n° est en `225$v`. Énumérer sur `200$a` ne remontait alors que
> le tome scanné. La série exposée dans `metadata.scannedTome.seriesTitleFr` (= `461$t`) est donc la
> clé à passer à `edition-mapping` — pas `titleFr`. Idem pour le n° de tome : lire `200$h` seul ratait
> les éditeurs qui numérotent en `225$v`/`461$v`.
>
> **Sans `461`, `225$a` n'est pas toujours la série.** Sur une pré-notice, c'est la collection de
> l'éditeur. Exemple : *The strange house* T2 (Kana, ISBN 9782505129035) n'a ni `461` ni `410`, son
> `225$a` vaut « Big Kana » et son n° est en `200$h`. Le monolithe en fait une série « Big Kana », que
> l'app remplit des autres titres de la collection. La constellation (`scanSeriesTitle`, service
> `search`) ne prend `225$a` que s'il porte le n° du tome (`225$v` sans `200$h`, cas Ki-oon sans
> `461`) et jamais quand il est repris en `410$t` (lien « collection ») ; sinon elle prend `200$a`.

### Ce que l'app fait des données par tome

Chaque `EditionTome` porte `editionVolume`, `isbn`, `titleFr`, `sourceVolumeRange` (454$h),
**`coverUrl`** (jaquette résolue en cascade **MangaDex → Google Books** — cf. [Jaquettes par
tome](#jaquettes-par-tome-mangadex--google-books) ; `null` si non résolue), **`coverSource`**
(`'mangadex'` | `'google_books'` | `null`, purement informatif) **et `description`** (résumé **propre
au tome**). À l'import, l'app crée un tome (`Record`) et **stocke dans ses extras** l'édition (205),
l'éditeur, la plage source (`sourceVolumeRange`) **et le résumé du tome** (clé `tomeSynopsis`), et met
en cache la **jaquette du tome**. Le **détail d'un tome manga** (écran dédié) affiche alors n° de tome,
édition, éditeur, ISBN, le libellé dérivé « **contient les tomes 1 à 3** » pour les éditions compilées,
et le **résumé propre au tome**.

Le **résumé par tome** est résolu côté backend par ordre de coût croissant : **`330$a` BnF** (rare,
mais authentiquement par tome) → à défaut le **`description` du volume Google Books** *déjà récupéré
pour la jaquette* (donc **sans appel supplémentaire** ; il s'agit souvent du résumé de série côté FR —
repli assumé) → sinon **`null`**. On **ne retombe pas** sur le synopsis de série au niveau du tome : si
aucune source par tome n'existe, le détail du tome n'affiche **pas** de résumé (le synopsis de série
reste sur le détail *de série*, `Series.synopsis`).

## Recherche d'une série par titre {#recherche-serie-titre}

Pour ajouter une série **sans en posséder de tome** (le besoin wishlist), l'app enchaîne trois appels,
tous **sans jeton** :

1. `GET /v1/search?type=manga&q=<texte>` — **MangaDex** cherche aussi dans ses titres alternatifs, dont
   les titres français. Tri par nombre de suivis, tags Doujinshi / Anthology / Official Colored / Fan
   Colored exclus : sur « Frieren », la vraie série passe en tête au lieu d'un crossover. Chaque
   résultat porte `metadata` = `{ mangaId, titleFr, titleRomaji, malId, anilistId, status,
   num_volumes, year, contentRating }`. MAL n'intervient qu'en repli, **sur la première page
   seulement** et en une seule page (pas de page suivante) : MangaDex n'a rien trouvé **et** un jeton
   MAL est disponible, ou MangaDex est indisponible.
2. `GET /v1/search/editions?mangaId=<uuid>[&title=<saisie>]` — le backend relit la série MangaDex par
   son id (titres, auteurs), interroge la BnF et renvoie ses **éditions françaises**, une ligne par
   (série, éditeur, mention d'édition) : `seriesTitleFr`, `publisher` (`null` si la notice n'en porte
   pas), `edition`, `isArtEdition`, `tomeCount`, `firstYear`, `lastYear`, `ongoing`,
   `hasSourceVolumeRange`, `sampleIsbn` — plus `complete` pour la liste entière. Liste vide si la BnF
   ne connaît pas la série (l'app propose alors la création manuelle) ; `502` si aucune notice n'a pu
   être lue parce qu'une lecture BnF a échoué.
3. `POST /v1/search/import-jobs { title: seriesTitleFr, edition, publisher, mangaId }` puis
   `edition-mapping` avec les mêmes valeurs : l'app crée la série (édition et éditeur compris) et
   **tous ses tomes en wishlist**.

Le choix de la série revient à l'utilisateur : son `mangaId` rend inutile tout rapprochement par
titre pour les jaquettes.

## Nouveaux tomes d'une série en cours {#nouveaux-tomes}

Un tome n'apparaît à la BnF qu'une fois sa notice créée : **3 à 4 semaines** après la sortie chez les
grands éditeurs français, **environ 3 mois** pour un éditeur hors de France (mesuré sur peu de cas).
`edition-mapping?since=<AAAAMMJJ>` (`bib.creationdate`, format strict — `AAAA-MM-JJ` renverrait 0
notice sans erreur) donne les tomes catalogués depuis une date. Deux déclencheurs :

- **l'app**, à l'ouverture d'une série en cours dont la dernière vérification date d'au moins 7 jours
  (fenêtre `since` avec 7 jours de marge) ;
- **le serveur**, chaque lundi, pour les séries synchronisées des comptes premium qui portent un
  titre et un éditeur ; le résultat se lit sur `GET /v1/series-updates`.

Dans les deux cas, les tomes trouvés sont **proposés**, jamais ajoutés d'office ; une énumération
incomplète ne fait pas avancer la date de vérification.

## Jaquettes par tome (MangaDex → Google Books) {#jaquettes-par-tome-mangadex--google-books}

Ni la BnF (bibliographique, sans image) ni MyAnimeList (uniquement `main_picture`, **au niveau
série**) ne fournissent de jaquette **par volume**. Deux sources la fournissent, en **cascade** :

1. **MangaDex (prioritaire)** — indexée **par série + n° de tome**, elle couvre ~80 % des tomes des
   séries reconnues avec la jaquette de l'**édition FR** (préférence locale `fr` > `ja` > `en`). Un
   seul appel résout **toute** l'édition (là où Google Books coûte un appel par ISBN). Résolu par
   `MangaDexCoverService.resolveSeriesCovers` (cache + file BullMQ `mangadex`), servi cache-only par
   l'endpoint `edition-mapping`. Si MangaDex a la jaquette du tome, **Google Books n'est pas appelé**.
2. **Google Books (repli)** — interrogée **par ISBN** quand MangaDex n'a pas la couverture du tome.

> **Quelle œuvre MangaDex ? — `mangaId` › `malId` › auteur, sinon repli ISBN.** La série dont on tire
> les jaquettes est résolue par `MangaDexResolver.findManga` **par ordre de fiabilité décroissante** :
> 1. **`mangaId`** (identité produite au scan, porté par `metadata.pivot.mangaId` et transmis à
>    `edition-mapping`/`import-jobs`) → utilisé **directement**, sans recherche par titre — le join le
>    plus sûr, ni homonyme ni variante possible ;
> 2. **`malId`** → recherche par titre puis retenue du candidat dont `links.mal` égale le mal_id ;
> 3. **sinon** (chemin bnf_only) → recherche par titre **validée par l'auteur** BnF (auteurs agrégés
>    par l'énumération d'édition), avec **préférence pour l'entrée canonique** (`links.mal` présent).
>    C'est le garde-fou du cas **One Piece** : le titre seul retenait une **variante** (édition
>    colorisée, `mal=-`) dont les jaquettes sont d'une **autre édition** et partiellement couvertes.
>
> Si **aucun** candidat n'est validé (bnf_only sans auteur exploitable, ou aucun match auteur), la
> résolution MangaDex ressort **`absent`** et la cascade résout **chaque tome par ISBN** (Google Books,
> édition-exact) — c'est le « **full scan ISBN** » du chemin bnf_only.

> ⚠️ **Garde-fou éditions d'art.** MangaDex indexe les couvertures par **(n° de tome)**, **sans
> dimension « édition »**. Sur une **édition d'art** (Colossale, Perfect, Kanzenban, Prestige,
> deluxe/luxe, collector, intégrale, coffret, grand format — visuel **et** numérotation distincts du
> standard), la jaquette du tome N renverrait le **visuel du standard** (faux). Donc on **saute
> MangaDex par tome** pour ces éditions (`isSpecialArtEdition` dans `manga-matching.ts`) et on passe
> par la cascade ISBN **édition-consciente** (Google Books `intitle:` filtré par édition). ⚠️ « édition
> d'art » ≠ « `edition != null` » : un **retirage** (`Quatorzième éd.`, `2e éd.`, ordinaux de tirage) a
> le **même visuel** standard → traité comme standard (on garde MangaDex). Le gate teste donc un
> **mot-clé d'art**, pas la simple présence d'une mention d'édition.

Détail de la source de repli **Google Books** (par ISBN) ci-dessous.

- **Service** : `GoogleBooksCoverService` (`src/common/sources/googlebooks/googlebooks.service.ts`),
  module `@Global` (miroir de `BnfModule`). Réutilise l'infra commune (`HttpClientService`,
  `ApiCacheService`) ; le débit sortant passe par la **file BullMQ `gbooks`** (limiter global +
  *single-flight*), qui a remplacé l'ancien token-bucket SQL.
  - `resolveCover(isbn, hint?)` — résout la **jaquette**. **Étape 1** :
    `GET https://www.googleapis.com/books/v1/volumes?q=isbn:<isbn>&country=FR` (clé
    `GOOGLE_BOOKS_API_KEY` optionnelle), extrait `volumeInfo.imageLinks.thumbnail` (repli
    `smallThumbnail`), force `https://` et retire `&edge=curl`. **Étape 2 (repli par titre)** : si la
    notice ISBN **n'a pas d'image** et qu'un `hint` BnF est fourni (titre de série + n° de tome +
    édition), requête `intitle:<série> T<nn>` (tome zéro-padté, ex. `T06`) puis rétention de la 1ʳᵉ
    notice **illustrée** dont le n° (`seriesInfo.bookDisplayNumber`, sinon extrait du titre) **et**
    l'édition correspondent **strictement** — indispensable pour les ISBN papier FR (Ki-oon…), souvent
    sans notice illustrée par ISBN. **Best-effort** : rate-limit / échec / absence → `null`, jamais
    d'exception. **Cache** positif 90 j, négatif 7 j.
  - `resolveCoverAndDescription(isbn, hint?)` — même résolution, mais renvoie **aussi** le
    `volumeInfo.description` du volume retenu, **dans le même appel réseau** (aucune requête
    supplémentaire → pas de coût quota) et mis en cache **avec** la jaquette. C'est ce que consomme
    `edition-mapping` pour le **résumé par tome**.
  - `cachedCover(isbn)` — lecture **cache-only** (jamais de réseau).
- **Endpoints** :

  | Route | Comportement |
  |---|---|
  | `GET /v1/search/cover?isbn=[&title=&volume=&edition=]` | résout (réseau) **et** met en cache → `{ coverUrl }`. Source de vérité de la résolution ; appelé explicitement par l'app (politique réseau : pas d'appel en arrière-plan). `title` + `volume` (et `edition` si spéciale) **arment le repli `intitle:`** — sans eux, seule la résolution `isbn:` est tentée (donc `null` récurrent pour les ISBN papier FR). |
  | `GET /v1/search/edition-mapping` | résout la **jaquette *et* le résumé** de chaque tome (réseau, best-effort, **concurrence bornée**, cache), le repli par titre armé depuis la BnF (série + n° de tome + édition). La série MangaDex est identifiée par `mangaId` › `malId` › auteur (sinon repli ISBN). |

- **Réponse enrichie** : `edition-mapping` renvoie un type dédié (`EditionMappingResponse` /
  `EditionTomeWithCover`) — `coverUrl` **et `description`** sont peuplés côté `SearchService`, ce qui
  laisse `EditionTome` (couche BnF) pur. Le `description` du tome = `330$a` BnF ⟶ à défaut le résumé
  Google Books ⟶ sinon `null` (cf. [Ce que l'app fait des données par tome](#ce-que-lapp-fait-des-données-par-tome)).
- **Rôle limité au tome** : Google Books ignore la notion de **série** (pas de `num_volumes`, pas de
  statut de publication, pas d'agrégation). Il expose un résumé (`description`) et une note
  (`averageRating`) au **tome**, pas à la série. Le **nœud série** (synopsis FR, statut, nb de tomes,
  genres, note) vient désormais de **MangaDex** (repli MAL) ; Google Books n'est qu'un **repli de
  jaquette + résumé par volume** quand MangaDex n'a pas la couverture du tome.

### Robustesse des appels (concurrence bornée & rate-limit) {#robustesse-des-appels}

`edition-mapping` résout la jaquette (et le résumé) de **tous** les tomes d'une édition (jusqu'à ~30).
Ces résolutions sont menées avec une **concurrence bornée** (`COVER_RESOLUTION_CONCURRENCY = 4`, un
petit pool qui préserve l'ordre) — **et non** un `Promise.all` de 30+ appels simultanés. Deux raisons,
apprises en production :

1. **Le bucket de rate-limit `gbooks:global` livelockait sous rafale.** `TokenBucketService` est une
   **unique ligne SQL** mise à jour par **CAS optimiste** (`updateMany … where lastRefill = <lu>`).
   Avec 30+ writers concurrents, tous lisent le même `lastRefill`, **un seul** commit par tour, les
   autres réessaient puis **abandonnent** au bout de `CAS_MAX_ATTEMPTS` (`token-bucket: CAS gave up
   after N attempts on gbooks:global`) et renvoient « rate limited » **alors que le bucket est plein**
   → l'énumération résolvait **zéro** jaquette. Le petit pool sérialise assez les accès pour qu'ils
   aboutissent ; `CAS_MAX_ATTEMPTS` a été relevé **5 → 10** en ceinture-bretelles.
2. **Google Books renvoyait des `429`/`503` en rafale** sous ~60 requêtes quasi simultanées ; le pool
   lisse la charge sortante.

Sur `429`/`5xx` (après les retries de `HttpClientService`), `resolveCover*` renvoie `null` **sans mise
en cache** — l'échec est **transitoire et re-tentable** au prochain appel. Seule une **absence réelle**
(HTTP 200 sans image) est mise en **cache négatif** (7 j) ; un repli titre infructueux compte comme
absence réelle.

## Observabilité (logs)

Le process est tracé explicitement (préfixes `bnf:`, `pivot:`, `mangadex:`) :

```
[BnfService] bnf: extracted isbn=... titleFr="..." edition="Éd. colossale"
             originalTitle="Shingeki no kyojin"[454$t] 454h="1-3" author="Isayama"
[MalAdapter] pivot: retained via mangadex isbn=... mangaId=... mal_id=23390 title="L'Attaque des Titans"
             matchedBy=title+author confidence=0.95 synopsisFr=y
# … ou, si MangaDex n'identifie pas :
[MalAdapter] pivot: mangadex no match isbn=... -> repli MAL
[MalAdapter] pivot: bnf->mal query="Tokyo toritsu" via=454$t requireAuthor=false author="Akutami"
[MalAdapter] pivot: retained mal_id=115710 titleSim=1.00 authorMatch=true typeOk=true confidence=1.00
# … ou, si la BnF ne connaît pas l'ISBN :
[MalAdapter] pivot: bnf_not_found isbn=9782808703437 -> repli Google Books
[MalAdapter] pivot: gbooks title isbn=... title="Sentenced to be a Hero Tome 1" authors=0 echelons=3
[MalAdapter] pivot: retained via gbooks+mangadex isbn=... prefix="Sentenced to be a Hero" cut=2
             tome=1 requetes=3 mangaId=... mal_id=151361 authorMatch=false confidence=0.49
```

On loggue : ce que la BnF a parsé, l'identification MangaDex (mangaId, mal_id porté, `matchedBy`,
`confidence`, `ambiguous`, présence du synopsis FR), le **repli MAL** le cas échéant (requête +
provenance `via=454$t` / `via=titleFr(fallback)`, candidats bruts, ce qu'on retient), et sur la voie
Google Books le **préfixe essayé**, le nombre de requêtes MangaDex et la raison du refus — c'est ce qui
rend le taux de succès du repli mesurable en production.

## Modes d'échec & fallbacks

| Situation | Comportement |
|---|---|
| ISBN absent du catalogue BnF (`bnf_not_found`) | **repli Google Books** : titre par ISBN → échelle de troncature → MangaDex (`gbooks+mangadex`), sinon item minimal (`gbooks_only`), sinon liste vide. |
| BnF indisponible / réponse illisible (`bnf_unavailable`, `bnf_unparsable`) | Liste vide, **sans** repli : panne transitoire, l'app retente. |
| Voie Google Books, identification ambiguë sans auteur | **Refus** (`gbooks_only`) : on préfère la saisie manuelle à un faux positif. |
| MangaDex indisponible / aucun candidat validé | **repli MAL** (pivot flou), puis `bnf_only` si MAL n'aboutit pas non plus. |
| Notice sans `454` ni `500` (titre conservé en graphie latine, ou catalogage incomplet) | **repli sur le titre FR** (`200$a`) pour MangaDex puis MAL → `bnf+mangadex-fr` / `bnf+mal-fr`. Sur MAL, **match auteur obligatoire**. |
| Notice sans titre FR **ni** original | `bnf_only` via le titre de série (`461$t`) si présent, sinon résultat vide ; aucune requête MangaDex/MAL. |
| Ni MangaDex ni MAL n'identifient (homonymes sans auteur…) | `bnf_only` : on **retient la notice BnF** (mappée en item + jaquette Google Books) plutôt qu'un faux positif. |
| Romaji BnF ≠ romaji MangaDex/MAL malgré le fuzzy | MangaDex expose `links.al` (AniList) sur l'entrée trouvée ; dernier recours possible côté MAL : **pivot AniList** (`idMal`). |

> **Pas d'identifiant pivot natif** : la BnF expose un ARK, MangaDex/MAL ne portent pas d'ISBN. Le pont
> est le **titre original romaji** (`454$t`), **confirmé par auteur**. MangaDex ajoute ensuite le
> **`links.mal`**, qui rend MAL joignable **par id** (fini le fuzzy) pour la sync bibliothèque.

## Note romanisation

BnF et MAL romanisent différemment (`jyouou` vs `joou`). Ce n'est **pas bloquant** : le `q=` de MAL
est fuzzy côté serveur et absorbe l'écart sur le titre brut → on interroge toujours avec le titre
**non normalisé**. Pour le **scoring** (§Scoring de confiance), le backend applique une normalisation
légère (minuscules, sans accents, espaces compactés) puis un **coefficient de Dice** sur bigrammes —
qui tolère l'écart `jyouou`/`joou` sans avoir à coder les digraphes Hepburn. Ce calcul est fait **côté
serveur** (pas de trigramme SQL : la base est MariaDB).

## Limites connues

- **ISBN-10 avec clé `X`** : non accepté par le `barcode` de `/v1/search` (chiffres uniquement). Sans
  impact pratique : les mangas portent un EAN-13.
- **Couverture MangaDex** : ~80 % des tomes des séries reconnues ; le match par titre romaji/FR est
  fiable mais pas garanti (l'auteur valide). MangaDex est une **source communautaire** (scanlation) →
  gardée **best-effort remplaçable** (down → repli MAL + Google/BnF).
- **Éditions non signalées par la BnF** : certaines éditions FR titrées (ex. Dragon Ball Glénat
  « orange ») sont cataloguées « standard » sans n° de tome → écartées de l'énumération, et **aucune
  API gratuite** (MangaDex, Google, OL, AniList, Kitsu) n'en fournit le visuel. On assume alors la
  **jaquette absente** plutôt qu'un visuel faux (couvrir vraiment = scraping éditeur FR, hors scope).
- **Catalogage BnF irrégulier** : géré mais heuristique — n° de tome en `461$v`/`225$v`/`200$h` selon
  l'éditeur ; appartenance à la série par `461$t` quand il existe, sinon par `500$a`/`200$a`/`225$a`,
  car `200$a` peut être le titre du tome et `225$a` la collection éditeur ; `210` vs `214` selon l'ancienneté. Les tomes récents
  n'existent d'abord qu'en **pré-notice** (sans mention d'édition, auteur en `702`), et seulement
  quelques semaines après leur sortie.
- **Repli titre FR** : n'aboutit que si le titre FR ≈ le titre indexé (titres latins). Un titre FR
  traduit (« L'Attaque des Titans ») ne matchera pas par ce repli — mais ces notices portent un
  `454$t` romaji, donc n'empruntent pas le repli.
- **Note de résumé FR** : sur le chemin **MangaDex**, la description est majoritairement en **français**
  (`description.fr`). Sur le **repli MAL**, elle reste souvent le synopsis **anglais** (MAL ne fournit
  aucun résumé localisé) faute de `330$a` BnF.
- **Score de confiance** : similarité de titre (contenance / Dice sur bigrammes) + auteur (+ type côté
  MAL, + bonus titre exact côté MangaDex) ; pas de distance de Levenshtein (le Dice suffit en pratique).
- **Repli Google Books** : ne peut rien pour les ISBN dont Google renvoie le **titre du tome** sans nom
  de série (Ki-oon, vieux One Piece Glénat) ni pour les ISBN qu'il ne connaît pas du tout (6/137 sur
  l'échantillon) — l'information n'existe pas, aucun parsing ne la fabriquera. Ces scans ressortent en
  `gbooks_only` (item minimal) ou vides.

## Voir aussi

- Vue utilisateur : [D'où viennent les informations](../utilisateur/sources-de-donnees.md).
- Flux scan macro : [architecture.md](architecture.md#3-enrichissement-au-scan).
- Contrat API : `openapi.json` (dépôts `acervatim-backend` et `acervatim-services`).
- Code des sources : `src/common/sources/` et `src/oauth/providers/` du monolithe ; `search/src/sources/`
  et `rebrickable-data/` dans la constellation.
