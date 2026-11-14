# Spec macro — Import « manuel » d'une série manga

> **Document interne** (exclu du site publié, comme `spec-macro-recherche-tris-galerie.md`).
> Créé le 2026-08-26. Vue d'ensemble d'une évolution : ajouter une série manga en la cherchant par
> titre, sans en posséder aucun tome.
> Sprints d'exécution : `acervatim-backend/docs/interne/sprints/` et
> `acervatim-app/docs/interne/sprints/` (répertoires **gitignorés**).
> **Amendé le 2026-10-27** d'après [etude-leviers-recherche-manga.md](etude-leviers-recherche-manga.md) :
> recherche par MangaDex, édition identifiée par (éditeur, mention), séries en cours tranchées,
> sprints SD3 et AD3.
> **Implémenté le 2026-11-02** (SD1, SD2, SD3, AD1, AD2, AD3) sur les branches
> `feat/manga-recherche-serie` de `acervatim-backend` et `acervatim-app` — **non commité, non
> fusionné**. État, écarts retenus et ordre de déploiement : § « Implémentation ».

## Le besoin

« Je cherche *Frieren* pour la mettre dans ma wishlist. »

Aujourd'hui c'est impossible. Le seul chemin vers une série complète est le **scan d'un tome**
(`series/import?barcode=`) : pour ajouter une série, il faut déjà en posséder un volume — soit
l'inverse exact du besoin. Une wishlist sert à noter ce qu'on **n'a pas**.

Le paradoxe est que la mécanique d'import produit déjà le bon résultat : à l'import d'une série
depuis un tome scanné, tous les autres tomes connus sont créés en **`WISHLIST`**, seul le tome scanné
étant `OWNED` (`ImportScannedSeriesUseCase:208`). La wishlist n'est donc pas à construire : c'est le
comportement nominal de l'import existant, privé de sa porte d'entrée.

## L'état réel du code (audit du 2026-08-26)

**La chaîne d'import fonctionne déjà à partir d'un titre, sans ISBN.** Il ne manque pas un pipeline,
il manque une **découverte** et une **projection**.

| Brique | État | Où |
|---|---|---|
| Recherche de séries par texte | ✅ existe | `GET /v1/search?type=manga&q=` → `MalAdapter.search` → `mapNode` (`mal.adapter.ts:838`) : titre, **auteurs**, jaquette, synopsis, `num_volumes` |
| Titre FR d'une série | ✅ existe | `MangaDexResolver.identify(query, authors)` → `titleFr` (`mangadex.resolver.ts:135`) |
| Énumération d'une édition | ✅ existe, **par titre** | `BnfService.enumerateEdition(titleFr, edition)` — la requête SRU est `bib.title all "<titre>"` (`bnf.service.ts:320`) : aucun ISBN dans la boucle |
| Import asynchrone | ✅ existe, **clé sur le titre** | `POST /v1/search/import-jobs {title, edition}` → `edition-import.processor.ts:29` |
| Création série + tomes en wishlist | ✅ existe | `ImportScannedSeriesUseCase:208` |
| **Lister les éditions disponibles d'un titre** | ❌ manque | `fetchEditionRecords` charge **toutes** les notices du titre ; `enumerateEdition` jette ensuite celles des autres éditions (`bnf.service.ts:232`). La donnée est déjà là, la projection non |
| **Choisir la bonne série sans ISBN** | ❌ manque | cf. section suivante |

Deux maillons, pas une fonctionnalité — la même conclusion que l'audit d'AD1 côté vinyle, et la même
consigne : **ne pas réécrire ce qui existe.**

## Le vrai problème de conception : choisir la bonne série

Sur le chemin du scan, l'ISBN donne une notice BnF, donc un titre **et des auteurs**. Ces auteurs ne
sont pas décoratifs : ils tranchent. `chooseIdentity` (`mangadex.resolver.ts:187`) combine titre
0,50 / **auteur 0,40** / bonus de correspondance exacte 0,10, et son commentaire nomme précisément
le cas qui nous occupe :

> « Frieren » (titre FR = troncature de « Sousou no Frieren ») donne un `titleScore` 1.00 au
> crossover parasite *Frieren Cinnamoroll Kamigata* comme à la vraie série — le titre seul ne peut
> plus départager.

Autrement dit : **le titre seul ne suffit pas, et le code le sait déjà.** Une recherche par titre n'a
pas d'ISBN, donc pas de notice BnF, donc pas ce signal d'auteur.

La sortie n'est pas d'inventer un meilleur score, c'est de **rendre le choix à l'utilisateur** :

1. la recherche MAL renvoie déjà, pour chaque candidat, de quoi choisir en connaissance de cause —
   titre, **auteurs**, jaquette, synopsis, nombre de tomes (`mal.adapter.ts:53`) ;
2. l'utilisateur désigne la bonne série ;
3. les auteurs **du candidat retenu** arment `identify(query, authors)`, dont la signature reste
   inchangée. Le signal que l'ISBN fournissait automatiquement est simplement fourni autrement.

C'est la leçon d'AD1 transposée au manga : *la recherche n'est pas le problème, la sélection l'est.*

> **Amendé le 2026-10-27 — la recherche passe par MangaDex, pas par MAL.** MAL exige un jeton
> (`403 source-token-required` en mode dégradé) et ne porte pas de titre français. MangaDex, sans
> jeton, retrouve 11 titres français sur 12 du banc de l'étude et place *Sousou no Frieren* en tête
> sur « Frieren » (tri par suivis, doujinshi et anthologies exclus). Le candidat retenu fournit son
> **`mangaId`** : `edition-mapping` et `import-jobs` le joignent directement (première étape de
> `findManga`), et ses auteurs resserrent l'énumération BnF. MAL ne sert plus qu'en repli, si
> MangaDex ne renvoie rien et qu'un jeton existe.

## Le second choix : l'édition

Une série manga française existe souvent en plusieurs éditions — standard, colossale, perfect,
collector. Elles n'ont ni le même nombre de tomes, ni le même découpage : un tome d'édition colossale
compile plusieurs tomes de l'édition standard, ce que la BnF exprime en `454$h` et que l'app porte
déjà sous `sourceVolumeRange` (« ce tome contient les tomes 1 à 3 »).

Côté BnF, l'édition est la mention `205$a`. `enumerateEdition` la connaît : elle filtre dessus et
**écarte** les notices des autres éditions (`bnf.service.ts:232`). Les notices écartées ont pourtant
déjà été téléchargées et mises en cache par `fetchEditionRecords`. Lister les éditions disponibles
d'un titre est donc une **projection sur un jeu de données déjà chargé** — pas un nouvel appel
sortant, pas un nouveau quota.

> **Amendé le 2026-10-27 — la mention `205$a` ne suffit pas à identifier une édition.** Sur
> *L'attaque des titans*, l'édition « standard » réunit Pika, les 2-en-1 France Loisirs, des romans
> Pika, des DVD… et l'énumération actuelle retient les tomes 11 et 17 **France Loisirs** au lieu de
> Pika. Une édition s'identifie par le couple **(éditeur normalisé, `205$a`)**, sur les seules
> notices de texte imprimé. Les tomes les plus récents arrivent en **pré-notice**, sans `205` ni
> `461` et avec l'auteur en `702` : l'étude (§ 3) détaille les corrections que l'énumération exige
> avant que la projection ne s'appuie dessus.

L'utilisateur choisit donc deux fois : la série, puis son édition. Les deux choix se ressemblent —
départager des candidats visuellement proches — et méritent le même soin d'affichage : ce qui
distingue deux éditions (nom de l'édition, nombre de tomes, découpage) doit être lisible sur la ligne.

## Modèle : l'édition entre dans l'identité de la série

**Arbitrage acté le 2026-08-26 : deux éditions d'une même série doivent pouvoir coexister** dans une
collection. Un collectionneur possède la standard et convoite la colossale ; les confondre serait un
appauvrissement, pas une simplification.

Ce que cet arbitrage implique — et ce qu'il n'implique pas :

- **App : migration nécessaire.** `index_series_natural_key` est unique sur
  `(collectionType, normalizedTitle, normalizedAuthor)` (`SeriesEntity.kt:18`). Deux éditions y sont
  structurellement impossibles. Il faut passer Room v7 → v8, enrichir l'index de l'édition, et donner
  un champ `edition` au modèle `Series` — qui n'en a pas aujourd'hui.
- **Backend : aucune migration.** `prisma/schema.prisma` ne pose **aucune** contrainte d'unicité sur
  les nœuds : deux nœuds « Frieren » (standard, colossale) coexistent déjà sans rien changer.
- **Backend : un champ de contrat.** `MangaSerieSchema` n'a pas de champ `edition`, et le schéma est
  en `.strip()` — une édition envoyée par l'app serait **silencieusement supprimée**, et un second
  appareil reconstruirait deux séries identiques indiscernables. Le tome a le même trou :
  `MangaItemSchema` est `.strict()` et ne connaît pas l'édition, qui ne vit aujourd'hui que dans les
  `extras` locaux (`MangaExtras.KEY_EDITION`).

> **Amendé le 2026-10-27** — ce qui entre dans la clé naturelle est le couple (éditeur, mention
> d'édition), et le contrat porte `publisherFr` au niveau de la série en plus d'`edition`. La
> migration v8 embarque aussi la colonne `records.normalized_title` de la recherche locale
> insensible aux accents (audit § 6), pour ne migrer qu'une fois.

### Ce n'est pas qu'un confort de collectionneur

Les items portent `@@unique([userId, nodeId, volume])` (`prisma/schema.prisma:106`). Deux « tome 1 »
d'éditions différentes rattachés au **même** nœud violent cette contrainte, et le serveur répond
`409 item déjà ajouté (série + volume)`. C'est exactement l'échec de synchronisation observé le
2026-08-26 — un seul tome en conflit suffisant à faire échouer la synchro entière.

Séparer les éditions au niveau de la série n'est donc pas seulement une facilité d'usage : c'est la
correction d'une divergence de modèle entre l'app (qui permet le doublon) et le serveur (qui le
refuse).

## Points d'attention

- **Quota.** Une recherche texte est peu cachable : la clé porte la requête libre, rarement identique
  d'un utilisateur à l'autre. Le relevé SD1 côté Discogs donnait un taux de hit de **0,25 en texte
  contre 0,67 en code-barres** ; il n'y a pas de raison que MAL se comporte mieux. Prévoir le même
  *debounce* qu'AD1 (400 ms, 3 caractères minimum) et ne pas partir en requête à chaque frappe.
  **Amendé le 2026-10-27** : la source est MangaDex (file `mangadex`, 4 req/s partagées), bien moins
  contrainte que Discogs (~60/min) ; le *debounce* reste.
- **Volume d'items créés d'un coup.** Importer une série longue crée des dizaines d'items d'un seul
  geste, tous à pousser. Le plafond technique côté serveur est déjà géré au push (`itemLimitHit`
  reporte les créations refusées), mais l'app doit le dire à l'utilisateur plutôt que de laisser
  croire à un import partiel silencieux.
- **Jaquettes.** Rien à prévoir : `edition-mapping` résout déjà les jaquettes par tome, et l'import
  asynchrone réchauffe le cache avant que l'app ne relise.
- **Trous de catalogage BnF.** Le chemin par titre dépend de la BnF pour les éditions, donc il hérite
  de ses trous — parutions récentes, éditeurs non francophones (cf.
  [spec-repli-google-books-pivot.md](spec-repli-google-books-pivot.md)). Attention : le repli Google
  Books en cours de chantier part d'un **ISBN**, il ne secourt donc pas une recherche par titre. Une
  série absente du catalogue doit se dégrader proprement — proposer la création manuelle de la série,
  pas une liste vide. **Mesuré le 2026-10-27** (étude § 2) : une notice apparaît 22 à 27 jours après
  la sortie chez Glénat et Casterman, 87 jours chez Vega-Dupuis, et Hellsing Perfect T05 n'est
  toujours pas catalogué à J+110 — ce délai vaut aussi pour le rattrapage des séries en cours.
- **Licence MangaDex** (ajouté le 2026-10-27). La politique d'usage de MangaDex interdit les
  « paid services » : une exception écrite est à obtenir avant le lancement du palier payant. Elle
  exige aussi un crédit, que l'app n'affiche pas encore (étude § 6).

## Points ouverts

- **Séries en cours.** `EditionMapping.ongoing` existe déjà. Que crée-t-on à l'import — tous les tomes
  connus à cet instant ? — et comment rattrape-t-on ceux qui paraissent ensuite : re-énumération
  manuelle, périodique, à l'ouverture de la série ? Hors périmètre, à instruire séparément.
  **Tranché le 2026-10-27** : l'import crée tous les tomes connus à cet instant ; les suivants sont
  **proposés**, jamais créés d'office. Deux déclencheurs sur `edition-mapping?since=AAAAMMJJ`
  (`bib.creationdate`) : l'app à l'ouverture d'une série en cours dont la dernière vérification a
  plus de 7 jours, et un job serveur hebdomadaire sur les séries synchronisées. Sprints SD3 et AD3.
- **Recherche par titre d'un tome isolé.** Le même parcours pourrait servir à ajouter un tome sans le
  scanner. Recoupe le parcours `edition-mapping` existant ; à ne pas mélanger avec la série.
- **Réconciliation des doublons déjà en base.** L'arbitrage sur l'édition empêche de *créer* de
  nouveaux conflits ; il ne répare pas les séries qui en portent déjà.

## Découpage en sprints

| # | Sprint | Capacité livrée | Dépend de |
|---|---|---|---|
| SD1 | Découverte série + éditions | Recherche de séries par **MangaDex**, sans jeton (job `search` de la file `mangadex`, tri par suivis, tags de format exclus) ; `titleFr` et `mangaId` par candidat ; lister les éditions d'un titre FR, groupées par (éditeur, `205$a`), sur les notices de texte imprimé (projection sur `fetchEditionRecords`) ; corriger l'énumération (appartenance des pré-notices, auteurs en `702`) | — |
| SD2 | L'édition dans le contrat | `edition` et `publisherFr` dans `MangaSerieSchema`, `edition` sur le tome, pour que le choix survive à la synchronisation et à un second appareil | — |
| SD3 | Rattrapage côté serveur | `edition-mapping?since=AAAAMMJJ` (`bib.creationdate`) ; job hebdomadaire sur les séries synchronisées en cours, une requête SRU par édition et non par utilisateur ; nouveaux tomes signalés à l'app ; journal du délai sortie → notice | SD1 |
| AD1 | Écran de recherche de série | Entrée dans `SerialAddBottomSheet`, à côté de « scanner un tome » ; saisie debouncée ; choix du candidat, puis de son édition (éditeur + mention) | SD1 |
| AD2 | Import sans tome possédé | Variante de `ImportScannedSeriesUseCase` (tous les tomes en `WISHLIST`) ; migration Room v8 : (éditeur, édition) dans la clé naturelle de la série, `records.normalized_title` pour la recherche locale | SD2 |
| AD3 | Rattrapage côté app | À l'ouverture d'une série en cours (dernière vérification de plus de 7 jours) et sur signal du serveur : « N nouveaux tomes » proposés en wishlist, import confirmé par l'utilisateur | SD3, AD2 |

```
SD1 ──► AD1                  (l'écran a besoin de la recherche et de la liste des éditions)
SD2 ──► AD2                  (l'import a besoin que l'édition survive à la sync)
SD1 ──► SD3 ──► AD3 ◄── AD2  (le rattrapage réutilise l'énumération corrigée et la clé d'édition)
```

Aucun ordre imposé entre les deux branches. AD1 sans AD2 donne un écran qui trouve et ne sait pas
importer ; AD2 sans AD1 n'a pas de point d'entrée : les deux doivent être livrés pour que la capacité
existe. SD3 et AD3 viennent ensuite : ils prolongent une série déjà importée.

Hors sprint, avant le lancement du palier payant : obtenir l'exception écrite de MangaDex et
afficher son crédit dans l'app (étude § 6).

## Implémentation (2026-11-02)

Branches `feat/manga-recherche-serie` (backend depuis `main`, app depuis `feat/ad1-recherche-textuelle`),
**non commitées, non fusionnées**. Validation : backend 56 suites / 629 tests unitaires + e2e touchés
(Redis isolé), `tsc` sans erreur nouvelle ; app 450 tests unitaires, `assembleDebug` OK. Ni l'écran de
recherche ni la bannière n'ont été essayés sur appareil contre le backend réel.

| Sprint | Livré | Où (principal) |
|---|---|---|
| SD1 | Recherche MangaDex sans jeton ; `GET /v1/search/editions` ; `edition-mapping` `publisher` / `since` / `complete` / `noticeCreatedAt` ; `import-jobs` `publisher` ; énumération corrigée (éditeur dans l'édition, dominant limité aux collisions de n° ; appartenance par `461$t` s'il existe, sinon l'un de `500$a`/`200$a`/`225$a` ; pré-notices ; mot-clé d'art hors titre de série ; `702` ; `doctype` ; page vide 24 h) ; `search/editions` avec `complete` et `502` si aucune notice lue | `be:oauth/providers/mal.adapter.ts`, `be:common/sources/mangadex/`, `be:common/sources/bnf/edition-records.ts`, `be:search/` |
| SD2 | `edition`, `publisherFr`, `mangaId` sur `MangaSerieSchema` ; `edition` sur `MangaItemSchema` | `be:collections/types/manga.ts` |
| SD3 | File `edition-watch` (planificateur BullMQ, lundi 04:00), `GET /v1/series-updates` (premium), `POST /v1/admin/edition-watch/run` | `be:search/watch/` |
| AD1 | Écran `series/search` : candidats → éditions → import en wishlist ; création manuelle pré-remplie ; crédit MangaDex/BnF | `app:ui/series/SeriesSearch*` |
| AD2 | Room v8 (édition + éditeur dans la clé naturelle, `records.normalized_title`) ; synchro des champs d'édition ; `importFromSearch` ; couche réseau des nouveaux endpoints ; le scan transmet éditeur et `mangaId` | `app:data/local/db/`, `app:data/remote/`, `ImportScannedSeriesUseCase` |
| AD3 | Bannière « N nouveaux tomes » au détail d'une série ; état dans un DataStore dédié | `app:domain/usecase/SeriesCatchUpUseCase.kt`, `app:ui/series/NewVolumesBanner.kt` |

**Écarts retenus** (par rapport au découpage ci-dessus) :

- **Repli MAL si MangaDex est en panne** : la recherche reprend alors le chemin MAL historique, donc
  un `403 source-token-required` reste possible en mode dégradé sans cache. Quand MangaDex a répondu
  sans résultat, MAL n'est tenté qu'avec un jeton — jamais de 403 dans ce cas. Le repli ne joue
  qu'en **première page** et renvoie une seule page (`nextCursor: null`).
- **Éditeur dominant limité aux collisions** : sans `publisher`, un autre libellé d'éditeur n'est
  écarté que s'il porte un n° que le dominant a déjà ; un `publisher` explicite reste un filtre strict.
- **File `mangadex` priorisée** : recherche, lecture par id et identification en priorité 1,
  jaquettes en priorité 10.
- **Nouveautés premium mises en cache côté app** (`SeriesUpdatesCache`, 10 min par compte) : ouvrir
  plusieurs séries n'appelle `series-updates` qu'une fois.
- **`complete`** : `edition-mapping` signale une énumération tronquée ou une page SRU en échec. Le job
  serveur n'avance alors pas sa date de vérification et rejoue la même fenêtre au passage suivant ;
  l'app non plus.
- **Planification BullMQ** (`upsertJobScheduler`, `0 4 * * 1`) plutôt que `@Cron` : un seul job par
  échéance quel que soit le nombre d'instances.
- **Tomes en attente persistés côté app** : la fenêtre `since` avance à chaque vérification réussie,
  donc un tome écarté par « Plus tard » ne reviendrait jamais d'une requête suivante ; l'app garde les
  tomes trouvés et les repropose à la vérification suivante.
- **Première vérification côté app à J+7** après la création de la série, pas dès l'ouverture qui
  suit l'import.
- **Séries d'avant la v8** : une série sans éditeur ni édition, de même titre et auteur, est reprise
  et complétée plutôt que dupliquée — **seulement si ses tomes sont compatibles** (aucun tome, ou des
  tomes dont l'éditeur et l'édition sont ceux de l'import ou inconnus) ; un seul tome d'un autre
  éditeur ou d'une autre édition fait créer une série neuve.
- **État de rattrapage purgé** à la suppression d'une série, à la déconnexion et à la suppression du
  compte (`ClearSeriesCatchUpUseCase`).

**Ordre de déploiement : le backend (SD2) avant l'app.** L'app envoie désormais `edition` sur chaque
tome manga ; `MangaItemSchema` est `.strict()`, donc un serveur sans SD2 rejetterait ces tomes en
`400` et ferait échouer la synchro. Un nœud n'est suivi par le rattrapage serveur que si l'app y a
synchronisé `title` **et** `publisherFr`.

**Restent ouverts** : le plafond `itemLimitHit` n'est signalé qu'à la synchro, pas à l'import ; la
mesure du délai sortie → notice reste à exploiter dans les logs `edition-watch: new tome` ;
l'exception MangaDex (palier payant) n'est pas demandée.

## Documents liés

| Document | Contenu |
|---|---|
| [agent/backend/pivot-vs-pipeline-enrichissement.md](agent/backend/pivot-vs-pipeline-enrichissement.md) | Le pivot ISBN → BnF → MangaDex → MAL, dont ce chantier réutilise la seconde moitié |
| [agent/backend/recherche-textuelle-discogs.md](agent/backend/recherche-textuelle-discogs.md) | Le précédent vinyle (SD1/AD1) : charge utile d'un résultat texte, cache et quota **mesurés** |
| [spec-macro-recherche-tris-galerie.md](spec-macro-recherche-tris-galerie.md) | § 1 — la désambiguïsation des pressages, dont la sélection d'édition est le pendant manga |
| [spec-repli-google-books-pivot.md](spec-repli-google-books-pivot.md) | Les trous de catalogage BnF, et pourquoi le repli existant ne couvre pas le chemin par titre |
| [etude-leviers-recherche-manga.md](etude-leviers-recherche-manga.md) | Les mesures du 2026-10-27 : sources testées, délai de catalogage BnF, défauts d'énumération, licences |
