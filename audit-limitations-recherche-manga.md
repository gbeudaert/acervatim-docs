# Audit — limitations actuelles des recherches côté manga

> **Document interne** (exclu du site publié, comme `spec-macro-import-serie-manga.md`).
> Audit du 2026-10-27, établi depuis le code (`acervatim-app`, `acervatim-backend`) et la
> documentation, à la révision `0.9.0` backend / `fdd0c80` app.
> Ce document **constate** ; il ne décide rien. Le cadrage de la sortie vit dans
> [spec-macro-import-serie-manga.md](spec-macro-import-serie-manga.md).

## En une phrase

La chaîne d'import manga fonctionne **déjà** à partir d'un titre ; il manque les deux maillons de
bout de chaîne — la **découverte** (choisir la bonne série sans l'auteur que fournissait l'ISBN) et
la **projection** des éditions disponibles — plus un champ `edition` dans le contrat sans lequel le
choix de l'utilisateur ne survivrait pas à la synchronisation.

## 1. Le trou principal : on ne peut pas chercher une série par titre

Le seul chemin vers une série manga est le **scan d'un tome** (`barcode` → pivot ISBN). Il faut donc
déjà posséder un volume pour ajouter une série — l'inverse exact du besoin wishlist.

L'endpoint texte existe pourtant côté serveur (`GET /v1/search?type=manga&q=`), mais **l'app ne
l'expose pas** : l'option « Rechercher » du bottom sheet d'ajout est conditionnée au vinyle.

| Constat | Où |
|---|---|
| Option « Rechercher » affichée seulement `if (activeCollectionType == CollectionType.VINYL)` — commentaire : « le manga a son propre parcours par titre (edition-mapping) », parcours qui exige de connaître déjà le titre FR exact | `acervatim-app` · `ui/list/CollectionScreen.kt:620-626` |
| Écran de recherche texte verrouillé au vinyle : « Défaut : vinyle, **seul type ouvert à ce parcours** » | `acervatim-app` · `ui/search/ExternalSearchViewModel.kt:44` |
| « Nouvelle série » manga = création **100 % manuelle**, aucun appel à une source externe | `acervatim-app` · `ui/series/AddEditSeriesViewModel.kt` |

## 2. Si on branchait la recherche texte telle quelle, elle serait bancale

| Limitation | Détail | Où |
|---|---|---|
| **MAL seul, et sous jeton** | `search()` passe par `malResolvedGet` : sans token BYOT ni premium → cache-only puis `403 source-token-required`. Le **scan** manga, lui, marche en mode dégradé (BnF + MangaDex sans clé). La recherche texte est donc *moins* accessible que le scan | `mal.adapter.ts:253` |
| **Pas de titre français** | `mapNode` renvoie `node.title` (romaji/anglais MAL). Chercher « L'attaque des titans » ne trouve rien ; MangaDex (qui porte `titleFr`) n'est joignable qu'**après** identification | `mal.adapter.ts:838` |
| **Pas de filtre de type** | `MANGA_MEDIA_TYPES` (exclut light novels, etc.) ne sert **que** dans le pivot ISBN, pas dans `search()` — les résultats texte mélangent tout | `mal.adapter.ts:61` |
| **Le titre seul ne départage pas** | `chooseIdentity` pèse titre 0,50 / **auteur 0,40** / bonus exact 0,10. Sans ISBN : pas de notice BnF, donc pas d'auteur. Le commentaire du code nomme le cas — « Frieren » donne un `titleScore` 1.00 au crossover parasite *Frieren Cinnamoroll Kamigata* comme à la vraie série | `mangadex.resolver.ts:187` |
| **Seuils d'acceptation** | `IDENTIFY_TITLE_FLOOR = 0.5` si l'auteur matche, sinon `PIVOT_TITLE_STRONG = 0.85` ; `MANGA_SEARCH_LIMIT = 30` candidats | `mangadex.resolver.ts:26-32`, `manga-matching.ts:15` |
| **Cache peu efficace** | Clé `mal:search:<query>:<offset>:<limit>` sans `userId` — partageable, mais le texte libre se répète peu. Relevé Discogs (SD1) : **0,25 de hit en texte contre 0,67 en code-barres** | `mal.adapter.ts:259` |
| **Aucune liste d'éditions** | `enumerateEdition` charge **toutes** les notices du titre puis jette celles des autres éditions (`rec205 !== wantEdition → continue`). La donnée est déjà chargée et cachée, la projection n'existe pas — aucun `listEditions` dans le code | `bnf.service.ts:232` |

## 3. Limitations du chemin scan / ISBN (celui qui marche)

- **Dépendance BnF** : parution récente ou éditeur non francophone → `numberOfRecords=0`. Le repli
  Google Books couvre **98/137 tomes mesurés (~72 %)**, ~84 % hors coffrets/goodies et ISBN absents
  de Google.
- **Ce repli part d'un ISBN** : il ne secourt donc **pas** une recherche par titre.
- **`seriesInfo.bookDisplayNumber` vide sur 137/137** : le n° de tome ne peut venir que du titre, via
  l'échelle de troncature et son unique garde-fou non linguistique (la queue retirée doit contenir un
  chiffre).
- **Trous de jaquettes irréductibles** : certains tomes n'ont **aucune** image chez Google Books
  (vérifié le 2026-07-10 sur Black Torch T1-5 et SNK Éd. colossale T3/T4) — ni 503, ni matching trop
  strict : un trou de données. Source de repli encore « décision produit à trancher »
  (`acervatim-backend/TODO.md`).
- **Éditions d'art** : `isSpecialArtEdition` fait **sauter MangaDex** (indexé sans dimension édition →
  la jaquette du tome N serait fausse) ⇒ cascade ISBN Google Books, donc plus de trous sur
  Colossale / Perfect / Kanzenban / collector.

## 4. Limitations de l'énumération d'une édition

- **Requête `bib.title all "<titre>"`** : il faut le titre FR **juste**. Un titre approximatif → liste
  vide, sans aucune suggestion (`bnf.service.ts:320`).
- **Troncature silencieuse** : plafond `EDITION_MAX_RECORDS = 1000` → log `TRUNCATED`, tomes
  potentiellement manquants (`bnf.service.ts:52`, `:277`).
- **Séries en cours** : `ongoing` est calculé et remonté au DTO, mais l'app ne s'en sert que pour
  marquer `partial = true` (`ImportScannedSeriesUseCase.kt:67`). **Aucun mécanisme de rattrapage** des
  tomes parus après l'import — point explicitement laissé ouvert dans la spec macro.
- **Une page SRU à 0 notice est cachée 7 jours** (`EDITION_CACHE_TTL_SECONDS`) — noté « reste ouvert,
  hors périmètre » dans [spec-repli-google-books-pivot.md](spec-repli-google-books-pivot.md) § 3.
- **Dépend du catalogage** : appartenance à la série lue en `461$t` → `225$a` → `200$a`, n° de tome en
  `461$v` → `225$v` → `200$h`. Un éditeur qui catalogue autrement sort un `tomeCount` faux ou nul.

## 5. Une limitation de modèle qui bloque la suite

L'édition n'existe nulle part dans le contrat ni dans l'identité de la série :

- `MangaSerieSchema` est en **`.strip()`** et n'a **pas** de champ `edition` : une édition envoyée par
  l'app serait **silencieusement supprimée**. `MangaItemSchema` est `.strict()` et l'ignore aussi
  (`collections/types/manga.ts:17`, `:34`).
- Côté app, ni `Series` ni `SeriesEntity` ne portent `edition` ; `index_series_natural_key` est unique
  sur `(collectionType, normalizedTitle, normalizedAuthor)`.
- **Conséquence concrète** : deux « tome 1 » d'éditions différentes rattachés au même nœud violent
  `@@unique([userId, nodeId, volume])` (`prisma/schema.prisma:106`) → **`409 item déjà ajouté
  (série + volume)`**, et un seul tome en conflit fait échouer la synchro entière (incident du
  2026-08-26).

## 6. Recherche locale dans la collection — incohérente entre les deux vues

| Vue | Implémentation | Effet |
|---|---|---|
| Liste plate (tomes) | SQL `LIKE '%q%'` sur `r.title` + nom d'artiste, `COLLATE NOCASE` | **ASCII seulement** : les accents ne sont pas normalisés |
| Vue séries | Filtre **en mémoire** via `normalizeArtistName` | Insensible à la casse **et** aux accents |

`data/local/db/RecordQueries.kt:47` contre `domain/usecase/ObserveSeriesWithCountsUseCase.kt` : même
requête utilisateur, deux résultats selon le mode d'affichage.

Par ailleurs `GET /v1/search` reste **scopé à un seul `type`** (pas de fédération multi-types) et
n'interroge **jamais les items de l'utilisateur** — c'est un relais provider, pas un index
(`acervatim-backend/docs/interne/sprints/memo-a-analyser.md` § 1).

## 7. Où en est le chantier

Tout est cadré dans [spec-macro-import-serie-manga.md](spec-macro-import-serie-manga.md)
(2026-08-26) et découpé en **SD1 / SD2 / AD1 / AD2** — **aucun des quatre n'est implémenté**. Les
SD1 / AD1 présents dans les dossiers de sprints sont ceux du **vinyle / Discogs**, livrés
(`05e255b` backend, `2976abe` app).

L'arbitrage est acté : deux éditions d'une même série **coexistent** — ce qui corrige au passage la
divergence de modèle à l'origine du `409`.

**Mise à jour du 2026-11-16** : SD1, SD2, SD3, AD1, AD2 et AD3, implémentés le 2026-11-02, sont
fusionnés dans `main` depuis le 2026-11-14 (monolithe, app, constellation) et **pas encore en
production** — état, écarts et ordre de déploiement dans
[spec-macro-import-serie-manga.md](spec-macro-import-serie-manga.md) § « Implémentation ».

## Documents liés

| Document | Contenu |
|---|---|
| [spec-macro-import-serie-manga.md](spec-macro-import-serie-manga.md) | Le cadrage de la sortie : découverte + projection, découpage SD1/SD2/AD1/AD2 |
| [etude-leviers-recherche-manga.md](etude-leviers-recherche-manga.md) | Les leviers mesurés pour lever ces limitations (sources testées, délai BnF, défauts d'énumération) — 2026-10-27 |
| [spec-repli-google-books-pivot.md](spec-repli-google-books-pivot.md) | Les trous de catalogage BnF et les mesures du repli (137 tomes) |
| [agent/backend/pivot-vs-pipeline-enrichissement.md](agent/backend/pivot-vs-pipeline-enrichissement.md) | Le pivot ISBN → BnF → MangaDex → MAL |
| [agent/backend/recherche-textuelle-discogs.md](agent/backend/recherche-textuelle-discogs.md) | Le précédent vinyle : charge utile d'un résultat texte, cache et quota **mesurés** |
| [agent/transverse/sources-externes.md](agent/transverse/sources-externes.md) | Les six providers, la résolution des jetons, le mode dégradé |
