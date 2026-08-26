# Spec macro — Import « manuel » d'une série manga

> **Document interne** (exclu du site publié, comme `spec-macro-recherche-tris-galerie.md`).
> Créé le 2026-08-26. Vue d'ensemble d'une évolution : ajouter une série manga en la cherchant par
> titre, sans en posséder aucun tome.
> Sprints d'exécution : `acervatim-backend/docs/interne/sprints/` et
> `acervatim-app/docs/interne/sprints/` (répertoires **gitignorés**).

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
  pas une liste vide.

## Points ouverts

- **Séries en cours.** `EditionMapping.ongoing` existe déjà. Que crée-t-on à l'import — tous les tomes
  connus à cet instant ? — et comment rattrape-t-on ceux qui paraissent ensuite : re-énumération
  manuelle, périodique, à l'ouverture de la série ? Hors périmètre, à instruire séparément.
- **Recherche par titre d'un tome isolé.** Le même parcours pourrait servir à ajouter un tome sans le
  scanner. Recoupe le parcours `edition-mapping` existant ; à ne pas mélanger avec la série.
- **Réconciliation des doublons déjà en base.** L'arbitrage sur l'édition empêche de *créer* de
  nouveaux conflits ; il ne répare pas les séries qui en portent déjà.

## Découpage en sprints

| # | Sprint | Capacité livrée | Dépend de |
|---|---|---|---|
| SD1 | Découverte série + éditions | Lister les éditions disponibles d'un titre FR (projection sur `fetchEditionRecords`, même cache, aucun appel sortant nouveau) ; exposer un `titleFr` exploitable pour un candidat choisi | — |
| SD2 | L'édition dans le contrat | `edition` dans `MangaSerieSchema` (et sur le tome), pour que le choix survive à la synchronisation et à un second appareil | — |
| AD1 | Écran de recherche de série | Entrée dans `SerialAddBottomSheet`, à côté de « scanner un tome » ; saisie debouncée ; choix du candidat, puis de son édition | SD1 |
| AD2 | Import sans tome possédé | Variante de `ImportScannedSeriesUseCase` (tous les tomes en `WISHLIST`) ; migration Room v8 : l'édition entre dans la clé naturelle de la série | SD2 |

```
SD1 ──► AD1        (l'écran a besoin de la liste des éditions)
SD2 ──► AD2        (l'import a besoin que l'édition survive à la sync)
```

Aucun ordre imposé entre les deux branches. AD1 sans AD2 donne un écran qui trouve et ne sait pas
importer ; AD2 sans AD1 n'a pas de point d'entrée : les deux doivent être livrés pour que la capacité
existe.

## Documents liés

| Document | Contenu |
|---|---|
| [agent/backend/pivot-vs-pipeline-enrichissement.md](agent/backend/pivot-vs-pipeline-enrichissement.md) | Le pivot ISBN → BnF → MangaDex → MAL, dont ce chantier réutilise la seconde moitié |
| [agent/backend/recherche-textuelle-discogs.md](agent/backend/recherche-textuelle-discogs.md) | Le précédent vinyle (SD1/AD1) : charge utile d'un résultat texte, cache et quota **mesurés** |
| [spec-macro-recherche-tris-galerie.md](spec-macro-recherche-tris-galerie.md) | § 1 — la désambiguïsation des pressages, dont la sélection d'édition est le pendant manga |
| [spec-repli-google-books-pivot.md](spec-repli-google-books-pivot.md) | Les trous de catalogage BnF, et pourquoi le repli existant ne couvre pas le chemin par titre |
