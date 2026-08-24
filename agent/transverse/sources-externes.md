# Sources de données externes (Discogs, MangaDex, MAL, TMDB, BnF, Google Books)

> Remonter : [../INDEX.md](../INDEX.md) · Doc humaine associée : [../../utilisateur/sources-de-donnees.md](../../utilisateur/sources-de-donnees.md), [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md)
> Toute la communication avec les providers passe par le **backend** (jamais d'appel direct depuis l'app).

## Vue d'ensemble

| Provider | Type de collection | Auth (jeton résolu, cf. §Résolution des jetons) | Usage |
|---|---|---|---|
| **Discogs** | vinyl | BYOT OAuth 1.0a (compte user) → repli premium : personal token Acervatim (`DISCOGS_ACERVATIM_TOKEN`) | Recherche texte + barcode (`database/search?barcode=`), métadonnées albums |
| **MangaDex** (**principal manga**) | manga | Aucune (API publique) | **Identité série** : synopsis **FR**, note, statut, nb tomes, genres, **jaquettes par tome**, `links.mal` / `links.al`. Source nominale du scan (nœud série `source='mangadex'`) |
| **MyAnimeList (MAL)** (repli) | manga | BYOT OAuth2 PKCE (compte user) → repli premium : `X-MAL-CLIENT-ID` (`MAL_CLIENT_ID`) | **Repli d'identité** si MangaDex n'identifie pas ; joignable par id via `links.mal` (sync bibliothèque). Recherche texte manga |
| **Google Books** | manga (livre) | Clé API optionnelle (`GOOGLE_BOOKS_API_KEY`) | **Repli jaquette par ISBN/tome** (quand MangaDex n'a pas le visuel du tome) ; expose aussi `description` (résumé par édition) et le **titre par ISBN** (job `volume-info`), maillon d'entrée du repli quand la BnF ignore l'ISBN |
| **TMDB** | movie | BYOT clé API perso (`PUT /v1/sources/tmdb/token`, pas d'OAuth) → repli premium : clé serveur (`TMDB_API_KEY`) | Recherche + détails films |
| **BnF (SRU, sans auth)** | manga | Aucune (API publique) | Pivot **ISBN → série/tome FR** (titre romaji, auteur, édition) ; énumération des tomes d'une édition (`edition-mapping`) |

## Architecture côté backend

- **Contrat commun** : `SourceAdapter` / `UnifiedItem` dans `src/oauth/providers/types.ts` (méthodes `search`, `searchByBarcode?`, `mediaType`).
- **Registre** : token `SOURCE_ADAPTERS` fourni par `src/common/sources/sources.module.ts` ; `SearchService` construit une `Map<typeCode, adapter>`.
- **Adapters** : `src/oauth/providers/discogs.adapter.ts`, `mal.adapter.ts`, `tmdb.adapter.ts`. **MangaDex n'a pas d'adapter** (API publique, comme la BnF) : module `src/common/sources/mangadex/` exposant `MangaDexCoverService` (@Global), injecté par `MalAdapter`.
- **BnF** : `src/common/sources/bnf/` — `bnf.service.ts` (SRU, énumération d'édition), `unimarc.parser.ts` (parsing notices UNIMARC), `bnf.types.ts` (`EditionMapping`).
- **Rapprochement partagé** : `src/common/sources/manga-matching.ts` — normalisation, tokens auteur, similarité de titre, `PIVOT_TITLE_STRONG`, `isSpecialArtEdition` ; utilisé par le pivot MAL **et** l'identification MangaDex (DRY).
- **Snapshots** : `src/common/sources/source-snapshot.service.ts` — normalisation des `sources[]` (`{ provider, externalId, rawData, fetchedAt }`) attachées aux items/nodes.
- **Protections** : cache DB (`common/cache/api-cache.service.ts`), throttle sortant par **limiters BullMQ** (une file par source — remplace l'ancien token-bucket SQL ; cf. [../backend/scan-asynchrone-bullmq.md](../backend/scan-asynchrone-bullmq.md)), client HTTP commun (`common/http/http-client.service.ts`).

## Résolution des jetons (BYOT / repli premium / mode dégradé)

`TokenResolverService` (`src/oauth/token-resolver.service.ts`), consommé par les adapters Discogs / MAL / TMDB, applique **une logique unique** pour tous les profils :

1. **Jeton utilisateur (BYOT)** présent → on l'utilise.
2. sinon **premium** → repli sur les **jetons Acervatim** (voir colonne *Auth* ci-dessus), plafonné par un bucket de rate-limit partagé `acervatim:<source>`.
3. sinon → **mode dégradé** : aucun appel sortant vers une source à jeton ; on sert les sources sans clé (BnF, **MangaDex**) et le **cache partagé** ; à défaut, `403 source-token-required` (payload `provider`) → l'app propose *connecter la source* **ou** *passer premium*. **NB** : MangaDex étant sans jeton, le scan manga s'enrichit désormais **même en mode dégradé** (identité + synopsis FR + jaquettes), MAL n'étant qu'un repli à jeton.

Cache de recherche indexé sur la requête publique (`<source>:<requête>`, sans `userId`) → réponses partageables sans fuite (endpoints publics). Doc humaine : [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md#résolution-des-jetons-byot--repli-premium--mode-dégradé).

## Pivot ISBN → MangaDex → MAL repli (manga)

Problème : MangaDex/MAL ne connaissent pas les ISBN ; la BnF connaît les éditions FR (et porte le titre romaji). Chaîne nominale :

```
ISBN scanné → BnF (notice tome FR : titre série + romaji, auteur, n° tome, édition)
           │  └─(BnF muette, bnf_not_found) → Google Books q=isbn: → titre → échelle de troncature
           │        → MangaDex identify (préfixe + auteurs Google) [resolutionPath='gbooks+mangadex']
           │        └─(repli) → item minimal [source='gbooks', gbooks_only] → sinon liste vide
           → MangaDex identify (titre + auteur) → identité série [source='mangadex']
                { synopsis.fr, links.mal, links.al, note, genres, jaquettes/tome }
              └─(repli) MangaDex n'identifie pas → MAL (recherche floue) [source='mal']
                 └─(repli) → notice BnF seule [source='bnf', bnf_only]
           → /v1/search/edition-mapping (BnF) → liste complète des tomes de l'édition
```

`source='mangadex'` est un code source **sans adapter** (comme `bnf` et `gbooks`) : `SourceSnapshotService` stocke la réf (mangaId) sans snapshot. Le **`mal_id`** est porté en `metadata.pivot` → MAL joignable par id plus tard (sync bibliothèque). Doc de conception : [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md).

### Repli Google Books (ISBN inconnu de la BnF)

Déclenché par le **seul** `bnf_not_found` (nouveauté non cataloguée, éditeur non français) — `bnf_unavailable` / `bnf_unparsable` sont transitoires et ne replient pas. Chaîne : `GoogleBooksCoverService.resolveVolumeInfo(isbn)` (job `volume-info` de la file `gbooks`, cache HIT/MISS/FAIL comme la jaquette) → `titleLadder()` (`common/sources/title-ladder.ts`) → `identifySeries(préfixe, auteurs Google)`.

- 🪜 **Échelle** : on retire les tokens par la fin, du titre complet au plus court (≤ 6), et on s'arrête au **premier préfixe accepté** par MangaDex. Le n° de tome est le **dernier entier de la queue retirée** du préfixe gagnant. `seriesInfo.bookDisplayNumber` est vide sur 137/137 tomes mesurés : le tome ne peut venir que du titre.
- 🛡️ **Un seul garde-fou**, non linguistique : ne tronquer que si la queue retirée **contient un chiffre**. Aucune liste de mots (« Tome », « T. »…) — sans le garde-chiffre, `Instinct grégaire` → `Instinct` matcherait une autre série.
- ⚖️ **`MangaDexIdentity.ambiguous`** (un autre candidat à égalité de `titleScore`) : ignoré par le chemin nominal (l'auteur BnF tranche), **rejette** côté repli quand aucun auteur n'a matché — c'est ce qui écarte *Frieren Cinnamoroll Kamigata*. Calculé **toujours** et porté par l'identité cachée (clé `mangadex:identify:v2:` — pas de paramètre de mode, qui empoisonnerait le cache partagé entre les deux chemins).
- 📈 Taux mesuré : **98/137 (~72 %)**, ~84 % hors coffrets/goodies et ISBN absents de Google.

## Jaquettes par tome (MangaDex → Google Books)

Ni MAL (illustration **de série** `main_picture`) ni la BnF (bibliographique) ne fournissent de jaquette **par volume**. Deux sources en **cascade** :

- **MangaDex (prioritaire)** — indexée par **série + n° de tome** (préférence locale `fr` > `ja` > `en`), ~80 % des tomes des séries reconnues, **un seul appel** par édition. `MangaDexCoverService.resolveSeriesCovers` (file BullMQ `mangadex`, cache), servi cache-only par `edition-mapping`. Si MangaDex a le tome, **Google Books n'est pas appelé**.
  - 🎯 **Quelle œuvre MangaDex** (`MangaDexResolver.findManga`) : `mangaId` (identité du scan, porté par `metadata.pivot` et renvoyé à `edition-mapping`/`import-jobs`) → **join direct** ; sinon `malId` (`links.mal`) ; sinon **validation par l'auteur** BnF (auteurs agrégés par l'énumération) avec **préférence `links.mal`** — anti-**variante** (One Piece colorisé `mal=-`, autre édition, partiel). Aucun match validé (bnf_only) → MangaDex `absent` → **full scan ISBN** (Google Books, édition-exact).
  - ⚠️ **Garde-fou éditions d'art** (`isSpecialArtEdition`) : MangaDex indexe **sans dimension édition** → sur une édition d'art (Colossale/Perfect/Kanzenban/Prestige/deluxe/collector/intégrale/coffret/grand format — visuel **et** numérotation distincts) la jaquette du tome N serait **fausse** ⇒ on **saute MangaDex** et on passe par la cascade ISBN Google Books édition-consciente. Les **retirages** (ordinaux : « Quatorzième éd. », « 2e éd. ») restent traités comme standard.
- **Google Books (repli, par ISBN)** — `common/sources/googlebooks/googlebooks.service.ts` : `resolveCover(isbn, hint?)` (réseau, best-effort, cache positif 90 j / négatif 7 j), `resolveCoverAndDescription(isbn, hint?)` (idem + résumé du volume, **même appel**), `cachedCover(isbn)` (cache-only) ; module `@Global`, débit sortant via la **file BullMQ `gbooks`** (limiter global + single-flight, a remplacé le token-bucket SQL). **Repli par titre** : si l'ISBN n'a pas d'image et qu'un `hint` BnF est fourni (série + n° de tome + édition), requête `intitle:<série> T<nn>` puis rétention stricte (n° via `seriesInfo.bookDisplayNumber` + édition) — indispensable pour les ISBN papier FR (Ki-oon…).
- `GET /v1/search/cover?isbn=[&title=&volume=&edition=]` → `{ coverUrl }` : résout **et** met en cache. `title`+`volume` (+`edition`) **arment le repli `intitle:`** ; sans eux, seule la résolution `isbn:` est tentée.
- `GET /v1/search/edition-mapping` : chaque tome porte `coverUrl`, **`coverSource`** (`mangadex`|`google_books`|`null`) **et `description`**, résolus à **concurrence bornée** (`COVER_RESOLUTION_CONCURRENCY`).
- Le **résumé par tome** = `330$a` BnF → repli `volumeInfo.description` Google Books → sinon `null` (jamais le synopsis de série). Le **nœud série** (synopsis FR, statut, nb tomes, genres) vient de **MangaDex** (repli MAL) ; Google Books enrichit au **tome** (jaquette + résumé).

## OAuth utilisateur (connexion de comptes externes)

- Flows : `src/oauth/oauth.service.ts` + `oauth-flow.registry.ts` ; primitives `providers/oauth1.ts` (Discogs) et `providers/pkce.ts` (MAL). **TMDB n'a pas de flow OAuth** : sa clé perso se saisit via `PUT /v1/sources/tmdb/token` (`src/sources/sources.controller.ts`), `DELETE` pour révoquer.
- Endpoints : voir [../backend/api-endpoints.md](../backend/api-endpoints.md) § OAuth et § Sources externes.
- Stockage : `oauth_credentials` (providers `discogs` / `mal` / `tmdb`), tokens/clés chiffrés AES-256-GCM ([../backend/securite-et-privacy.md](../backend/securite-et-privacy.md)).
- Côté app : `ui/account/AccountScreen.kt` + `AcervatimApi.startOAuth/connectedProviders/revokeOAuth`.
