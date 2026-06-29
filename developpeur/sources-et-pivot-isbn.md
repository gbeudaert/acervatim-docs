---
title: Sources de données & pivot ISBN
layout: default
parent: Développeur
nav_order: 7
---

# Sources de données & pivot ISBN → MyAnimeList

D'où viennent les métadonnées renvoyées par `GET /v1/search`, et comment le backend résout un **ISBN
de manga** en une œuvre MyAnimeList. Source de vérité = le code du dépôt
[`acervus-backend`](../../acervus-backend) ; cette page en donne la vue normalisée.

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

Code : [`src/common/sources/`](../../acervus-backend/src/common/sources/) et
[`src/oauth/providers/`](../../acervus-backend/src/oauth/providers/).

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
        └─(B) extraction ──▶ titre original (454$t), édition (205),
               │             plage de tomes source (454$h), éditeur (210/214$c),
               │             auteur (700), ISBN (010), ARK (003)
               └─(C) MyAnimeList ── recherche publique q=<romaji> ──▶ candidats
                      └─(D) consolidation ── filtre media_type + match AUTEUR ──▶ œuvre
                             └─▶ UnifiedItem (série) + metadata.scannedTome
```

### Étape A+B — BnF
[`BnfService`](../../acervus-backend/src/common/sources/bnf/bnf.service.ts) interroge le **catalogue
général BnF** en SRU (HTTP, sans authentification) :

- Endpoint `https://catalogue.bnf.fr/api/SRU`, `recordSchema=unimarcxchange` (impératif :
  `dublincore` écrase série/tome dans le titre).
- Index `bib.fuzzyISBN` (accepte ISBN-10/13 et EAN indifféremment).
- Parsing UNIMARC **sans dépendance externe**
  ([`unimarc.parser.ts`](../../acervus-backend/src/common/sources/bnf/unimarc.parser.ts)) — extraction
  ciblée par regex, tolérante au préfixe de namespace `mxc:`.

Zones extraites → `BnfNotice` :

| Zone UNIMARC | Champ | Rôle |
|---|---|---|
| `454$t` (fallback `500$a`) | `originalTitle` | **Titre original romaji** — le pont vers MyAnimeList. |
| `454$h` | `sourceVolumeRange` | **Plage de tomes de l'édition source** (ex. `1-3`) → mapping inter-édition. |
| `205$a` | `edition` | Mention d'édition (ex. `Éd. colossale`) ; absente = standard. |
| `200$a` / `200$h` | `titleFr` / `volume` | Titre FR et n° de tome. |
| `210$c` \| `214$c` | `publisherFr` | Éditeur FR (les notices RDA-fr récentes utilisent `214`). |
| `700$a/$b` | `authors` | Auteur structuré (nom / prénom). |
| `003` | `ark` | Identifiant pérenne BnF. |

### Étape C+D — MyAnimeList
[`MalAdapter.searchByBarcode`](../../acervus-backend/src/oauth/providers/mal.adapter.ts) :

- Recherche **publique** via l'en-tête `X-MAL-CLIENT-ID` (pas d'OAuth utilisateur requis pour un
  scan ISBN). Le Client ID vit côté serveur (`MAL_CLIENT_ID`).
- Requête `q=<titre original romaji>` envoyé **brut** : le `q=` de MyAnimeList est *fuzzy côté
  serveur* et absorbe les écarts de romanisation (ex. `jyouou` ↔ `joou`). Aucune normalisation
  romaji n'est nécessaire pour interroger.
- **Consolidation** sur les candidats : on retient le meilleur via **filtre `media_type`** (manga /
  manhwa / one_shot…) **et match auteur** (le nom BnF `700` vs `authors` MAL, insensible casse /
  accents / ordre). L'auteur est le **validateur robuste** : il rejette les faux positifs quand le
  titre matche par hasard. Sans signal fort (ni auteur ni type+rang 0) → pas de sélection (`bnf_only`).
- **Piège type LN vs manga** : l'œuvre source japonaise peut être un **light novel** alors que
  l'édition FR est une **adaptation manga** (cas réel : *Kakuzetsu Toshi no Joou* / *Attack on Titan:
  The Harsh Mistress of the City* est un LN sur MAL, catalogué adaptation manga côté BnF). Pour
  l'adaptation, l'auteur MAL peut être le **mangaka adaptateur** et non l'auteur original → accepter un
  match sur **l'un OU l'autre**.

### Sortie
Un `UnifiedItem` au niveau **série** (pour le fan-out des tomes), enrichi de
`metadata.scannedTome` = `{ isbn, titleFr, volume, edition, publisherFr, sourceVolumeRange }` :
les données du **tome précis** scanné, à reporter dans l'item créé. Le champ `sourceVolumeRange` a une
place dédiée dans le schéma du tome (`MangaItemSchema`).

## Éditions collector (ex. Colossale)

C'est le pivot BnF qui rend les éditions spéciales correctes :

- L'**ISBN identifie l'édition** (chaque édition a le sien) ; `205` indique laquelle (`Éd. colossale`).
- `454$h` donne le **mapping de volumes** : une Colossale T.1 = tomes **1-3** de l'édition standard.
- **MyAnimeList ne sait pas** compter une édition FR (`num_volumes` = compte de l'œuvre japonaise, ou
  `0` si en cours) → le comptage fiable vient de la BnF, pas de MAL. Le code **loggue un avertissement**
  quand la série MAL est `currently_publishing` / `num_volumes=0`.

## Observabilité (logs)

Le process est tracé explicitement (préfixes `bnf:` et `pivot:`) :

```
[BnfService] bnf: extracted isbn=... titleFr="..." edition="Éd. colossale"
             originalTitle="Shingeki no kyojin"[454$t] 454h="1-3" author="Isayama"
[BnfService] WARN bnf: serie likely ONGOING ... -> comptage de tomes / 454$h partiels
[MalAdapter] pivot: bnf->mal query="Shingeki no kyojin" author="Isayama" 454h="1-3"
[MalAdapter] pivot: retained mal_id=23390 authorMatch=true typeOk=true confidence=1.00
```

On loggue : ce que la BnF a parsé, ce qu'on en a extrait, les séries en cours (comptage partiel), et
le pivot (ce qu'on interroge / ce qu'on retient / le score de confiance).

## Modes d'échec & fallbacks

| Situation | Comportement |
|---|---|
| ISBN absent du catalogue BnF | `unresolved` (`bnf_not_found`). |
| Notice sans `454` ni `500` (essai FR original, pas une traduction, ou catalogage incomplet) | `bnf_only`, `originalTitle: null` → repli possible sur **data.bnf.fr** (entité Œuvre, titre original mieux structuré, accès RDF/SPARQL). |
| MAL renvoie 0 résultat | retry sans normalisation destructive, puis avec le titre EN si dispo ; sinon `bnf_only`. |
| MAL renvoie N candidats sans match auteur/type net | renvoyer top-N + scores, **ne pas auto-sélectionner**. |
| Romaji BnF ≠ romaji MAL malgré le fuzzy serveur | dernier recours : **pivot AniList** (champ `idMal`) — fuzzy-match sur AniList puis récupérer l'ID MAL canonique. |

> **Pas d'identifiant pivot natif** : la BnF expose un ARK, MAL ne porte pas d'ISBN. Le pont est le
> **titre original romaji** (`454$t`) envoyé au `q=` fuzzy de MAL, **confirmé par auteur + type**.

## Note romanisation

BnF et MAL romanisent différemment (`jyouou` vs `joou`). Ce n'est **pas bloquant** : le `q=` de MAL
est fuzzy côté serveur et absorbe l'écart sur le titre brut. Une normalisation romaji (macrons,
digraphes `jyo→jo`/`sy→sh`, voyelles longues `ou→o`) n'est utile que pour **affiner le
`confidence_score`** côté application (fuzzy Levenshtein / Dice — **pas** de trigramme SQL, la base est
MariaDB), jamais pour interroger MAL.

## Limites connues

- **ISBN-10 avec clé `X`** : non accepté par le `barcode` de `/v1/search` (chiffres uniquement). Sans
  impact pratique : les mangas portent un EAN-13.
- **Couverture MyAnimeList** : le match par titre romaji est fiable mais pas garanti ; l'auteur valide.
- **Catalogage BnF irrégulier** : n° de tome tantôt en `200$h` tantôt en `225$v` ; `225$a` parfois la
  collection éditeur et non l'œuvre ; `210` vs `214` selon l'ancienneté. L'extraction est heuristique.
- **Score de confiance** volontairement simple (auteur + type + rang), sans fuzzy Levenshtein.

## Voir aussi

- Vue utilisateur : [D'où viennent les informations](../utilisateur/sources-de-donnees.md).
- Flux scan macro : [architecture.md](architecture.md#3-enrichissement-au-scan).
- Contrat API : [`openapi.json`](../../acervus-backend/openapi.json).
