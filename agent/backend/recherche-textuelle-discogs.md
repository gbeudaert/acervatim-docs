# Recherche textuelle Discogs — désambiguïsation des pressages (SD1)

> Remonter : [INDEX.md](INDEX.md) · Document d'**index agent** (pas une page du site public).
> Sprint d'origine : `acervatim-backend/docs/interne/sprints/decouverte/sprint-sd1-recherche-textuelle.md`
> (gitignoré). Spec macro : `acervatim-docs/spec-macro-recherche-tris-galerie.md` § 1.
> Dernière mise à jour : 2026-08-24. Branche : `feat/sd1-recherche-textuelle`.

## TL;DR

`GET /v1/search?type=vinyl&q=…` fonctionnait déjà. Le manque n'était pas la recherche mais
l'**exploitabilité de son résultat** : un texte renvoie 20-30 pressages du même album, et la charge
utile ne portait pas de quoi les distinguer. Trois champs ont été ajoutés au mapping des résultats
(`metadata.catno`, `metadata.barcodes`, `metadata.masterId`), le découpage artiste/titre a été
durci, et le taux de hit du cache est désormais **mesuré** par famille de clés au lieu d'être
supposé.

## 1. Ce que porte un résultat de recherche

`DiscogsAdapter.mapSearchResult` → `UnifiedItem` :

| Champ | Origine Discogs | Rôle pour choisir un pressage |
|---|---|---|
| `title` / `creators` | `title` (`"Artist - Title"`) | identification de base |
| `releaseDate` | `year` → `YYYY-01-01` | **année de pressage** |
| `coverUrl` | `cover_image` › `thumb` | reconnaissance visuelle |
| `metadata.label` | `label[]` | label / sous-label |
| `metadata.country` | `country` | pays de pressage |
| `metadata.format` | `format[]` (plat : nom + descriptions) | LP / 45 tours / réédition |
| `metadata.recordingSpeed` | dérivé de `format[]` | `RPM_33` / `RPM_45` / `RPM_78` |
| `metadata.catno` | `catno` | **numéro de catalogue** — le discriminant décisif |
| `metadata.barcodes` | `barcode[]`, normalisé digits + dédupliqué | comparable à un EAN scanné (détection de doublon) |
| `metadata.masterId` | `master_id` | commun à **tous** les pressages d'un album → permet de **regrouper** la liste |
| `metadata.genres` / `styles` | `genre[]` / `style[]` | classement |

Les champs absents chez Discogs restent `undefined` (pas de `null` bruyant).

`masterId` est le levier UX principal côté app (AD1) : sans lui, une recherche « Bagatelle » affiche
deux lignes identiques ; avec lui, on sait que ce sont deux pressages (2017 / 2021, catnos `ENLP07`
et `LP07`) du **même** album.

### Découpage artiste / titre

Discogs ne renvoie **pas** de tableau `artists` en recherche : seulement `"Artist - Title"`. Le
découpage (`extractCreatorsFromTitle` / `stripArtistFromTitle`) :

- coupe au **premier** `" - "` — un titre qui contient lui-même un tiret reste intact
  (`"Various - Rock - The Early Years"` → `Various` + `Rock - The Early Years`) ;
- scinde les multi-artistes sur `" / "` (`"Bob Dylan / The Band"` → deux entrées) ;
- retire le suffixe d'homonymie Discogs `" (N)"` (`"Nirvana (2)"` → `Nirvana`), comme le fait déjà
  `cleanArtistName` sur le chemin `fetchDetails`.

**Limite assumée** : un nom d'artiste contenant lui-même `" - "` est indécidable ici. Seul
`fetchDetails` (tableau `artists` réel), appelé à l'ajout de l'item, tranche.

## 2. Pagination

Le curseur de `/v1/search` est **opaque côté contrat** et vaut, pour Discogs, le **numéro de page** :

- `limit` → `per_page` (1-100, défaut 50) ;
- `cursor` → `page` (`parsePage` : absent ou illisible → page 1, jamais d'erreur) ;
- `meta.pagination.nextCursor` = page suivante, `null` en dernière page
  (`pagination.page < pagination.pages`) ;
- la page **fait partie de la clé de cache** (`discogs:search:q:<q>:<page>:<limit>`) : deux pages
  d'une même requête ne se recouvrent pas.

Rien à changer côté contrat : la brique existait, elle est atteignable depuis la route.

## 3. Cache et quota — mesuré, pas supposé

Une recherche texte est **beaucoup moins cachable** qu'un code-barres : la clé porte la requête
libre, rarement identique d'un utilisateur à l'autre, là où un EAN scanné donne la même clé pour
tout le monde.

`ApiCacheService` compte donc les hit/miss par **famille de clés** déclarée par l'appelant
(`discogs:search:q`, `discogs:search:barcode`, `discogs:release`, …) — jamais dérivée de la clé
complète, qui porterait un cardinal non borné. Lecture : `GET /v1/admin/queues` → `cache[]`, et log
périodique `api-cache …` (`QueueMetricsService`, toutes les 60 s). Compteurs en mémoire du
processus, remis à zéro au redémarrage : deux relevés se soustraient pour obtenir le taux sur
l'intervalle.

**Relevé du 2026-08-24** (dev, repli premium, 9 requêtes) :

| Famille | hits | misses | taux |
|---|---|---|---|
| `discogs:search:q` | 2 | 6 | 0,25 |
| `discogs:search:barcode` | 2 | 1 | 0,67 |

File `discogs` : 6 jobs terminés — soit **6 appels sortants réels pour 9 requêtes**, et les 2 hits
texte n'existent que parce que la même requête a été rejouée à l'identique.

Lecture : en recherche texte, **une requête utilisateur ≈ un appel Discogs**. Avec le limiter à
`max: 1 / 1000 ms` (≈ 60 req/min, la limite Discogs authentifiée), le débit texte consomme le quota
quasiment au rapport 1:1, alors que le scan par code-barres est largement absorbé par le cache
partagé. La file absorbe le profil (elle sérialise), mais c'est le **délai d'attente** qui monte
sous rafale, pas le taux d'erreur : `DISCOGS_WAIT_MS` = 15 s, au-delà → `502`. C'est le point à
surveiller quand AD1 ouvrira la recherche texte à tous les utilisateurs — le signal est
`discogs[w… d…]` dans le log des files.

## 4. Absence de jeton

Une recherche texte Discogs **exige un jeton** (BYOT ou repli premium). Sans jeton et sans hit de
cache, la route renvoie un `403` actionnable, pas une liste vide :

```json
{ "type": "https://api.acervatim/probs/source-token-required",
  "title": "Source token required", "status": 403,
  "detail": "No access token available for source \"discogs\". …",
  "provider": "discogs", "requestId": "…" }
```

`provider` est l'extension member RFC 9457 qui permet à l'app de router vers le bon parcours
(connecter Discogs **ou** passer premium) — cf.
[../transverse/sources-externes.md](../transverse/sources-externes.md).

## 5. Hors périmètre

- L'écran de saisie et l'UI de sélection → sprint app **AD1**.
- La recherche multi-sources / la fusion de résultats entre providers.
- Le côté manga : `MalAdapter.search` renvoie des **séries**, pas des éditions — la sélection
  d'édition y passe par `edition-mapping` (BnF), qui a son propre chantier
  ([pivot-vs-pipeline-enrichissement.md](pivot-vs-pipeline-enrichissement.md)). Rien à
  désambiguïser au niveau de `/v1/search` pour ce type.

## Fichiers

- `src/oauth/providers/discogs.adapter.ts` — `mapSearchResult`, `runSearch`, `normalizeBarcodes`,
  `extractCreatorsFromTitle`, `parsePage`.
- `src/common/cache/api-cache.service.ts` — familles de clés, `stats()`.
- `src/metrics/queue-metrics.{service,controller}.ts` — exposition et log du taux de hit.
- `src/search/dto/search.query.ts` — `q` XOR `barcode`, `cursor`, `limit`.
