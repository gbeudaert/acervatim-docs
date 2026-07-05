---
title: Sources de données & pivot ISBN
layout: default
parent: Développeur
nav_order: 7
---

# Sources de données & pivot ISBN → MyAnimeList

D'où viennent les métadonnées renvoyées par `GET /v1/search`, et comment le backend résout un **ISBN
de manga** en une œuvre MyAnimeList. Source de vérité = le code du dépôt
`acervatim-backend` ; cette page en donne la vue normalisée.

## Le proxy de recherche unifié

Le backend ne sert jamais une source externe directement à l'app : il expose un **proxy unifié**
`GET /v1/search?type=<type>&q=<texte>` **ou** `?type=<type>&barcode=<code>` (les deux paramètres sont
mutuellement exclusifs) et renvoie un `UnifiedItem` indépendant de la source.

Chaque type média est servi par un **adapter** (`SourceAdapter`) :

| `type` | Source | `search` (texte) | `searchByBarcode` | `fetchDetails` |
|---|---|---|---|---|
| `vinyl` | Discogs | ✅ | ✅ (EAN via `barcode=`) | ✅ |
| `manga` | MyAnimeList (+ BnF) | ✅ | ✅ (**ISBN → pivot BnF**) | ✅ |
| `movie` | TMDB | ✅ | — | ✅ |

Code : `src/common/sources/` et `src/oauth/providers/` (dépôt `acervatim-backend`).

## Résolution des jetons (BYOT / repli premium / mode dégradé)

Chaque source à jeton (Discogs, MAL, TMDB) résout son jeton **de la même façon**, via
`TokenResolverService` (`src/oauth/token-resolver.service.ts`) :

1. **Jeton utilisateur (BYOT)** — si l'utilisateur a connecté la source, on utilise **son** jeton
   (répartition des quotas fournisseur sur son compte, résultats liés à son compte source).
2. **Repli premium** — sinon, si l'utilisateur est **premium**, repli sur les **jetons Acervatim** :
   `X-MAL-CLIENT-ID` (MAL), personal access token (Discogs, `DISCOGS_ACERVATIM_TOKEN`), clé serveur
   (TMDB, `TMDB_API_KEY`). Ce repli est plafonné par un bucket de rate-limit **partagé**
   `acervatim:<source>` (tous premiums confondus), pour ne pas faire throttler le compte Acervatim.
3. **Mode dégradé** — sinon (non-premium sans jeton) : **aucun appel sortant** vers une source à
   jeton. On sert les sources **sans clé** (BnF) et les réponses déjà en **cache partagé** ; à défaut,
   l'erreur actionnable **`403 source-token-required`** (avec `provider`), qui invite à connecter la
   source **ou** à passer premium.

Le cache de recherche est indexé sur la **requête publique** (`<source>:<requête>`, sans `userId`) :
une réponse obtenue via jeton Acervatim est identique à une réponse jeton-utilisateur (endpoints
publics interrogés ici), donc **partageable sans fuite**. Le mode dégradé ne sert donc que des clés
partagées.

**TMDB n'a pas d'OAuth** : la clé personnelle se saisit via `PUT /v1/sources/tmdb/token` (stockée
chiffrée dans `oauth_credentials`, provider `tmdb`) ; `DELETE` pour la révoquer.

## Vue d'ensemble : trois sources pour un manga

À partir d'un seul **ISBN**, trois sources se répartissent le travail — chacune à sa granularité :

| Source | Rôle | Granularité |
|---|---|---|
| **BnF** (SRU) | identifie le tome (titre de série FR, n° de tome, éditeur, ISBN) et **énumère tous les tomes** d'une édition | tome / édition |
| **MyAnimeList** | **pivot série** : métadonnées d'ensemble attachées au nœud série (synopsis, statut, note, nb de tomes, visuel de série) | série |
| **Google Books** | **jaquette par tome** (par ISBN) — seule source d'image par volume | tome |

BnF + MAL forment le **pivot ISBN → série** (étapes A→D ci-dessous) ; **Google Books** est un
**enrichissement complémentaire au tome** (jaquettes), sans aucune notion de série — il ne remplace
donc pas MAL. Détails : [Jaquettes par tome](#jaquettes-par-tome-google-books).

## Le problème : MyAnimeList ne connaît pas les ISBN

MyAnimeList (API v2) **n'a aucun lookup par ISBN**, n'indexe que des titres **romaji / anglais /
japonais** (jamais le titre français), et renvoie `num_volumes = 0` pour les séries en cours. Il est
donc inutilisable seul pour partir d'un ISBN d'édition française.

La **BnF** (dépôt légal) comble exactement ces manques : autorité sur les éditions FR, et — c'est la
clé — ses notices manga portent le **titre original romaji**, qui est *précisément* ce que MyAnimeList
indexe. D'où un **pivot en deux étapes**.

## Le pipeline

```
ISBN (EAN-13)
 └─(A) BnF SRU ── recherche par ISBN ─────────────▶ notice UNIMARC
        └─(B) extraction ──▶ titre original (454$t), titre FR (200$a), titre série (461$t),
               │             édition (205), plage de tomes source (454$h), éditeur (210/214$c),
               │             auteur (700), note FR (330), ISBN (010), ARK (003)
               └─(C) MyAnimeList ── q = <romaji 454$t> sinon <titre FR 200$a> ──▶ candidats
                      └─(D) consolidation ── score = titre + AUTEUR + type ──▶ œuvre
                             └─▶ UnifiedItem (série) + metadata.scannedTome
```

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
| `461$t` (fallback `225$a`, `200$a`) | `seriesTitle` | **Titre de la série** (lien « fait partie de ») — clé d'énumération des tomes. |
| `210$c` \| `214$c` | `publisherFr` | Éditeur FR (les notices RDA-fr récentes utilisent `214`). |
| `700$a/$b` | `authors` | Auteur structuré (nom / prénom). |
| `330$a` | `noteFr` | **Note de résumé en français** (souvent absente pour les mangas) → synopsis. |
| `003` | `ark` | Identifiant pérenne BnF. |

### Étape C+D — MyAnimeList
`MalAdapter.searchByBarcode` (`src/oauth/providers/mal.adapter.ts`) :

- Jeton MAL **résolu** (cf. [Résolution des jetons](#résolution-des-jetons-byot--repli-premium--mode-dégradé)) :
  Bearer de l'utilisateur s'il a connecté MAL, sinon repli `X-MAL-CLIENT-ID` serveur (`MAL_CLIENT_ID`)
  pour un premium ; un non-premium sans jeton obtient la **notice BnF seule** (aucun appel MAL). Les
  endpoints publics MAL renvoient un corps **identique** en Bearer et en `X-MAL-CLIENT-ID` (vérifié),
  donc un scan ISBN ne requiert pas d'OAuth pour un premium.
- Requête `q=<titre>` envoyée **brute** : le `q=` de MyAnimeList est *fuzzy côté serveur* et absorbe
  les écarts de romanisation (ex. `jyouou` ↔ `joou`). Aucune normalisation romaji n'est nécessaire pour
  interroger. Le titre interrogé est le **titre original romaji** (`454$t`) ; à défaut, le **titre FR**
  (`200$a`) en repli (voir *Repli titre FR* ci-dessous).
- **Consolidation** : parmi les candidats, on retient celui de **meilleur score** (voir §Scoring de
  confiance) — jamais un simple « premier résultat ». Le **match auteur** (nom BnF `700` vs `authors`
  MAL, insensible casse / accents / ordre) est le validateur robuste : il confirme un titre qui pourrait
  matcher par hasard. Sans signal suffisant → pas de sélection (`bnf_only`).
- **Piège type LN vs manga** : l'œuvre source japonaise peut être un **light novel** alors que
  l'édition FR est une **adaptation manga** (cas réel : *Kakuzetsu Toshi no Joou* / *Attack on Titan:
  The Harsh Mistress of the City* est un LN sur MAL, catalogué adaptation manga côté BnF). Pour
  l'adaptation, l'auteur MAL peut être le **mangaka adaptateur** et non l'auteur original → accepter un
  match sur **l'un OU l'autre**.

### Scoring de confiance

Le candidat retenu est celui qui **maximise** un score ∈ [0,1] combinant trois signaux, calculé côté
backend (`choosePivotCandidate`, dans `mal.adapter.ts`) :

```
confidence = 0.45 · similarité_titre  +  0.40 · match_auteur  +  0.15 · type_manga
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

**Règle d'acceptation** (garde-fou anti faux-positif — on préfère `bnf_only` à un faux match) :

| Chemin | Requête MAL | Condition de rétention | `resolutionPath` |
|---|---|---|---|
| Titre original présent (`454$t`/`500$a`) | romaji brut | match auteur **OU** (type manga **ET** similarité_titre ≥ 0.85) | `bnf+mal` |
| **Repli titre FR** (pas de titre original) | titre FR `200$a` | match auteur **obligatoire** | `bnf+mal-fr` |

Le **repli titre FR** sert les mangas dont le titre est **conservé en graphie latine** en France
(*Black Torch*, *One Piece*…) : le titre FR = l'original, donc la recherche MAL aboutit. Mais un titre
FR étant plus générique (homonymes), **seul l'auteur valide** — sans match auteur, rien n'est retenu.

### Sortie
Un `UnifiedItem` au niveau **série** (pour le fan-out des tomes), enrichi de
`metadata.scannedTome` = `{ isbn, titleFr, volume, edition, publisherFr, sourceVolumeRange }` :
les données du **tome précis** scanné, à reporter dans l'item créé. Le champ `sourceVolumeRange` a une
place dédiée dans le schéma du tome (`MangaItemSchema`).

Le **synopsis** (`description`) privilégie la **note de résumé BnF** (`330$a`, en français) quand elle
existe, avec repli sur le synopsis MyAnimeList (**anglais** — MAL ne fournit aucun résumé localisé, ni
paramètre de langue, ni champ `synopsis_fr`). `metadata.pivot` porte `{ confidence, authorMatched,
resolutionPath }` pour l'audit.

## Éditions collector (ex. Colossale)

C'est le pivot BnF qui rend les éditions spéciales correctes :

- L'**ISBN identifie l'édition** (chaque édition a le sien) ; `205` indique laquelle (`Éd. colossale`).
- `454$h` donne le **mapping de volumes** : une Colossale T.1 = tomes **1-3** de l'édition standard.
- **MyAnimeList ne sait pas** compter une édition FR (`num_volumes` = compte de l'œuvre japonaise, ou
  `0` si en cours) → le comptage fiable vient de la BnF, pas de MAL. Le code **loggue un avertissement**
  quand la série MAL est `currently_publishing` / `num_volumes=0`.

## Énumération des tomes (`edition-mapping`)

`GET /v1/search/edition-mapping?title=<série>&edition=<205>` renvoie **tous les tomes** d'une édition
(« récupérer toute la série d'un coup »).
`BnfService.enumerateEdition` recherche
`bib.title all "<titre de série>"` (paginé), puis pour chaque notice :

1. **Appartenance à la série** — on ne garde que les notices dont la série (`461$t` → `225$a` →
   `200$a`, normalisée) **égale** le titre demandé. Exclut les spin-offs et homonymes qui matchent le
   titre (ex. *… : Before the Fall*) et les notices d'ensemble.
2. **Édition** — filtre sur `205$a` (standard vs `Éd. colossale`…).
3. **N° de tome** — lu dans `461$v` → `225$v` → `200$h` (dans cet ordre), puis **dédup** par numéro.

> ⚠️ **Le titre de série est décisif.** Le catalogage varie : chez Pika (SNK, Maid sama) `200$a` **est**
> le titre de série, répété sur chaque tome ; chez **Ki-oon** (Jujutsu Kaisen) `200$a` est le titre du
> **tome** (« Je vais te tuer ») et le n° est en `225$v`. Énumérer sur `200$a` ne remontait alors que
> le tome scanné. La série exposée dans `metadata.scannedTome.seriesTitleFr` (= `461$t`) est donc la
> clé à passer à `edition-mapping` — pas `titleFr`. Idem pour le n° de tome : lire `200$h` seul ratait
> les éditeurs qui numérotent en `225$v`/`461$v`.

### Ce que l'app fait des données par tome

Chaque `EditionTome` porte `editionVolume`, `isbn`, `titleFr`, `sourceVolumeRange` (454$h) **et
`coverUrl`** (jaquette Google Books résolue par ISBN — cf. [Jaquettes par
tome](#jaquettes-par-tome-google-books) ; `null` si non résolue). À l'import, l'app crée un tome
(`Record`) et **stocke dans ses extras** l'édition (205), l'éditeur et la plage source
(`sourceVolumeRange`), et met en cache la **jaquette du tome**. Le **détail d'un tome manga** (écran
dédié) affiche alors n° de tome, édition, éditeur, ISBN, et le libellé dérivé « **contient les tomes 1
à 3** » pour les éditions compilées. Le **résumé** montré est celui de la **série**
(`Series.synopsis`) : ni MAL (résumé d'œuvre) ni la BnF (`330` quasi jamais renseigné par volume) n'ont
de synopsis par tome. Google Books, lui, expose un `description` **par édition** — disponible mais
**pas encore exploité** (voir section suivante).

## Jaquettes par tome (Google Books)

Ni la BnF (bibliographique, sans image) ni MyAnimeList (uniquement `main_picture`, **au niveau
série**) ne fournissent de jaquette **par volume**. La seule source d'image par tome est **Google
Books**, interrogée **par ISBN** — c'est le troisième pilier du scan manga.

- **Service** : `GoogleBooksCoverService` (`src/common/sources/googlebooks/googlebooks.service.ts`),
  module `@Global` (miroir de `BnfModule`). Réutilise l'infra commune : `HttpClientService`,
  `ApiCacheService`, `TokenBucketService`.
  - `resolveCover(isbn)` — appelle `GET https://www.googleapis.com/books/v1/volumes?q=isbn:<isbn>&country=FR`
    (clé `GOOGLE_BOOKS_API_KEY` optionnelle), extrait `volumeInfo.imageLinks.thumbnail` (repli
    `smallThumbnail`), force `https://` et retire `&edge=curl`. **Best-effort** : rate-limit / échec /
    absence → `null`, jamais d'exception. **Cache** positif 90 j, négatif 7 j (évite de re-taper un
    ISBN sans jaquette).
  - `cachedCover(isbn)` — lecture **cache-only** (jamais de réseau).
- **Endpoints** :

  | Route | Comportement |
  |---|---|
  | `GET /v1/search/cover?isbn=` | résout (réseau) **et** met en cache → `{ coverUrl }`. Source de vérité de la résolution ; appelé explicitement par l'app (politique réseau : pas d'appel en arrière-plan). |
  | `GET /v1/search/edition-mapping` | chaque tome porte `coverUrl`, peuplé **depuis le cache uniquement** (`cachedCover`) → **latence inchangée**, aucun appel Google Books pendant l'énumération. Le cache est alimenté par `/cover`. |

- **Réponse enrichie** : `edition-mapping` renvoie un type dédié (`EditionMappingResponse` /
  `EditionTomeWithCover`) — `coverUrl` est peuplé côté `SearchService`, ce qui laisse `EditionTome`
  (couche BnF) pur.
- **Ne remplace pas MAL** : Google Books ignore la notion de **série** (pas de `num_volumes`, pas de
  statut de publication, pas d'agrégation). Il expose bien un résumé (`description`, par édition) et une
  note (`averageRating`), mais au **tome**, pas à la série. MAL reste la source du **nœud série** ;
  Google Books enrichit le **tome** (jaquette, éventuellement résumé par volume plus tard).

## Observabilité (logs)

Le process est tracé explicitement (préfixes `bnf:` et `pivot:`) :

```
[BnfService] bnf: extracted isbn=... titleFr="..." edition="Éd. colossale"
             originalTitle="Shingeki no kyojin"[454$t] 454h="1-3" author="Isayama"
[BnfService] WARN bnf: serie likely ONGOING ... -> comptage de tomes / 454$h partiels
[MalAdapter] pivot: bnf->mal query="Tokyo toritsu" via=454$t requireAuthor=false author="Akutami"
[MalAdapter] pivot: mal candidates count=2 titles=[Tokyo Ghoul:re | Jujutsu Kaisen 0: Tokyo Toritsu…]
[MalAdapter] pivot: retained mal_id=115710 titleSim=1.00 authorMatch=true typeOk=true confidence=1.00
```

On loggue : ce que la BnF a parsé, ce qu'on en a extrait, les séries en cours (comptage partiel), et
le pivot — la requête et sa provenance (`via=454$t` titre original, ou `via=titleFr(fallback)`), si le
match auteur est exigé (`requireAuthor`), les candidats bruts, et ce qu'on retient (`titleSim` /
`authorMatch` / `typeOk` / `confidence`).

## Modes d'échec & fallbacks

| Situation | Comportement |
|---|---|
| ISBN absent du catalogue BnF | `unresolved` (`bnf_not_found`). |
| Notice sans `454` ni `500` (titre conservé en graphie latine, ou catalogage incomplet) | **repli sur le titre FR** (`200$a`) envoyé à MAL, **match auteur obligatoire** → `bnf+mal-fr`. Si l'auteur ne matche aucun candidat → `bnf_only` (repli possible sur **data.bnf.fr**). |
| Notice sans titre FR **ni** original | `bnf_only`, aucune requête MAL. |
| MAL renvoie 0 résultat / que des homonymes sans auteur | `bnf_only` (on ne retient rien plutôt qu'un faux positif). |
| MAL renvoie N candidats sans match auteur/type net | renvoyer top-N + scores, **ne pas auto-sélectionner**. |
| Romaji BnF ≠ romaji MAL malgré le fuzzy serveur | dernier recours : **pivot AniList** (champ `idMal`) — fuzzy-match sur AniList puis récupérer l'ID MAL canonique. |

> **Pas d'identifiant pivot natif** : la BnF expose un ARK, MAL ne porte pas d'ISBN. Le pont est le
> **titre original romaji** (`454$t`) envoyé au `q=` fuzzy de MAL, **confirmé par auteur + type**.

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
- **Couverture MyAnimeList** : le match par titre romaji est fiable mais pas garanti ; l'auteur valide.
- **Catalogage BnF irrégulier** : géré mais heuristique — n° de tome en `461$v`/`225$v`/`200$h` selon
  l'éditeur ; titre de série en `461$t` (fiable) car `200$a` peut être le titre du tome et `225$a` la
  collection éditeur ; `210` vs `214` selon l'ancienneté.
- **Repli titre FR** : n'aboutit que si le titre FR ≈ le titre indexé par MAL (titres latins). Un titre
  FR traduit (« L'Attaque des Titans ») ne matchera pas MAL — mais ces notices portent un `454$t`, donc
  n'empruntent pas le repli.
- **Note de résumé FR** : la BnF renseigne rarement le `330$a` pour les mangas → la description reste
  souvent le synopsis **anglais** de MyAnimeList.
- **Score de confiance** : similarité de titre (contenance / Dice sur bigrammes) + auteur + type ; pas
  de normalisation romaji dédiée ni de distance de Levenshtein (le Dice suffit en pratique).

## Voir aussi

- Vue utilisateur : [D'où viennent les informations](../utilisateur/sources-de-donnees.md).
- Flux scan macro : [architecture.md](architecture.md#3-enrichissement-au-scan).
- Contrat API : `openapi.json` (dépôt `acervatim-backend`).
