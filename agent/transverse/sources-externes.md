# Sources de données externes (toutes collections)

> Remonter : [../INDEX.md](../INDEX.md) · Doc humaine associée : [../../utilisateur/sources-de-donnees.md](../../utilisateur/sources-de-donnees.md), [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md)
> Toute la communication avec les providers passe par le **backend** (jamais d'appel direct depuis l'app).
> Mis à jour le 2026-11-16. Statuts : **en service** (appelée par la production, le monolithe) · **prête**
> (branchée dans le code, pas en production ou sans écran app) · **pending** (attend un tiers).
> Code : monolithe `be:common/sources/` + `be:oauth/providers/` ; constellation
> `acervatim/backend-services/search/src/sources/` (mêmes adapters, portés) et `rebrickable-data/` (LEGO).
> Conditions d'usage et ce qui change au premium : [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md) § Conditions d'usage.

## Vue d'ensemble

| Provider | Type de collection | Auth (jeton résolu, cf. §Résolution des jetons) | Usage | Statut |
|---|---|---|---|---|
| **Discogs** | vinyl | BYOT OAuth 1.0a (compte user) → repli premium : personal token Acervatim (`DISCOGS_ACERVATIM_TOKEN`) | Recherche texte + barcode (`database/search?barcode=`), métadonnées albums | en service |
| **MangaDex** (**principal manga**) | manga | Aucune (API publique) | **Identité série** : synopsis **FR**, note, statut, nb tomes, genres, **jaquettes par tome**, `links.mal` / `links.al`. Source nominale du scan (nœud série `source='mangadex'`) **et de la recherche texte de séries** (`GET /v1/search?type=manga&q=`, sans jeton) | en service |
| **MyAnimeList (MAL)** (repli) | manga | BYOT OAuth2 PKCE (compte user) → repli premium : `X-MAL-CLIENT-ID` (`MAL_CLIENT_ID`) | **Repli d'identité** si MangaDex n'identifie pas ; joignable par id via `links.mal` (sync bibliothèque). **Repli** de la recherche texte manga : MangaDex sans résultat et jeton MAL résolu, ou MangaDex indisponible | en service |
| **Google Books** | manga (livre) | Clé API optionnelle (`GOOGLE_BOOKS_API_KEY`) | **Repli jaquette par ISBN/tome** (quand MangaDex n'a pas le visuel du tome) ; expose aussi `description` (résumé par édition) et le **titre par ISBN** (job `volume-info`), maillon d'entrée du repli quand la BnF ignore l'ISBN | en service |
| **TMDB** | movie | BYOT clé API perso (`PUT /v1/sources/tmdb/token`, pas d'OAuth) → repli premium : clé serveur (`TMDB_API_KEY`) | Recherche + détails films | prête (aucun type film dans l'app) |
| **GameUPC** | jeu de société | Clé serveur `x-api-key` (`GAMEUPC_API_KEY`) ; votes sous pseudonyme (`GAMEUPC_VOTER_PEPPER`) | **Code-barres → id BGG** (communautaire) ; les choix des utilisateurs lui sont renvoyés. Constellation, service `search` : [../backend/gameupc-bgg.md](../backend/gameupc-bgg.md) | prête : clé de production accordée le 2026-11-18, ni coût ni attribution ; 20 jobs/min (serveur `/test` en dev) ; constellation seulement |
| **BoardGameGeek** (XML API2) | jeu de société | Jeton Bearer d'application (`BGG_API_TOKEN`), **vide tant que la licence BGG n'est pas accordée** (non commerciale d'abord, commerciale avant le premium) ; **pas de BYOT** (BGG interdit de faire obtenir un jeton à l'utilisateur final) | Fiche `thing` d'un jeu ; aujourd'hui file `bgg` de `search` à ≤ 1 req / 2 s ; **cible décidée le 2026-11-14** : `bgg-data` seul appelant, 1 req / 5 s (F7, F8) ; `boardgamegeek.com` sans `www`, logo « Powered by BGG » dans l'app | pending : licence demandée le 2026-11-14 |
| **BnF (SRU, sans auth)** | manga | Aucune (API publique) | Pivot **ISBN → série/tome FR** (titre romaji, auteur, édition) ; énumération des tomes d'une édition (`edition-mapping`) ; **liste des éditions FR** d'une série (`search/editions`) ; **rattrapage** des tomes catalogués depuis une date (`since`) | en service |
| **Rebrickable** (CSV) | LEGO | Aucune : CSV publics `cdn.rebrickable.com/media/downloads/` (l'API n'est plus utilisée par la cible) | Catalogue de référence `rebrickable-data` : sets, thèmes, figurines, URL d'image ; import quotidien 03:00 UTC, garde `Last-Modified` | prête : `rebrickable-data` testé en dev, entré au déploiement le 2026-11-17 (C3), pas en production |
| **Brickset** (API v3) | LEGO | Clé `BRICKSET_API_KEY` (lue par `rebrickable-data` seulement) | Codes-barres `ean`/`upc`/`brickset_id` uniquement ; appel `getSets {query: code}` **au scan d'un code absent du cache** seulement (décision du 2026-11-16, codé le 2026-11-17, action C8) : un appel au plus par code, codes gardés 90 j puis effacés, code inconnu redemandé après 7 j (validé le 2026-11-17), budget 90 appels/jour UTC (`BRICKSET_DAILY_CALL_BUDGET`, quota Brickset 100) ; plus de moissonnage | pending : clé reçue le 2026-11-20 (plafond quotidien levable sur demande), accord pour l'usage commercial à demander (courrier F2 pas encore envoyé) ; la doc de l'API ne dit pas que `query` cherche dans les codes-barres, vérifié le 2026-11-20 sur un EAN (`5702015869935` → `75192-1`), pas sur un UPC seul |

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
2. sinon **premium** → repli sur les **jetons Acervatim** (voir colonne *Auth* ci-dessus), protégés par le limiteur de la file BullMQ de la source (les anciens buckets `acervatim:<source>` sont retirés).
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

## Recherche d'une série par titre (SD1 · SD3)

Implémentée le 2026-11-02, fusionnée dans `main` le 2026-11-14 (monolithe, app et constellation),
**pas encore en production** (la production tourne sur un monolithe antérieur). Mesures et
sources : [../../etude-leviers-recherche-manga.md](../../etude-leviers-recherche-manga.md) ;
décisions : [../../spec-macro-import-serie-manga.md](../../spec-macro-import-serie-manga.md).

```
saisie → GET /v1/search?type=manga&q=   MangaDex, sans jeton (tri par suivis, doujinshi/anthologies/
       │                                 colorisés exclus) → candidats { mangaId, titleFr, titleRomaji… }
       ▼ choix du candidat
         GET /v1/search/editions?mangaId=  identité MangaDex relue par id → BnF texte imprimé →
       │                                 une ligne par (série, éditeur, 205$a)
       ▼ choix de l'édition
         POST /v1/search/import-jobs { title, edition, publisher, mangaId } → edition-mapping
```

- **MAL en repli seulement**, en **première page** et sur une seule page (`nextCursor: null`) :
  MangaDex a répondu sans résultat **et** un jeton MAL est résolu (sinon liste vide, jamais de 403),
  ou MangaDex est indisponible (chemin MAL historique).
- **Énumération corrigée** (`common/sources/bnf/edition-records.ts`) : une édition = **(éditeur
  normalisé, `205$a`)**. Un `publisher` explicite filtre strictement dans le monolithe ; dans la
  constellation, il garde aussi ses **libellés successifs**, ceux qui ne portent aucun de ses n°
  (« Delcourt-Akata » puis « Delcourt », `publisherFamilyOf`), et la liste des éditions les réunit
  en une ligne ; un catalogue qui reprend ses n° (romans, 2-en-1) reste écarté. Sans lui, l'éditeur
  **dominant** ne gagne que les **collisions** de n° (les 2-en-1 France Loisirs ne remplacent plus les
  tomes Pika ; un n° que seul un libellé voisin porte comble le trou). Appartenance d'une notice :
  **`461$t` fait autorité** quand il existe ; sans `461`, **l'un des** `500$a`, `200$a`, `225$a` égal
  à la clé de série — ce qui garde les **pré-notices** des tomes récents (leur `225$a` est la
  collection de l'éditeur, « Shonen manga »). Auteurs lus aussi en `702` (traducteurs exclus) ;
  pré-notice sans `205` dont le titre porte un mot-clé d'art **hors des mots du titre de série** →
  pas rangée en standard (*Perfect World* reste standard) ; requête filtrée `bib.doctype any "a"` ;
  page vide cachée 24 h. `search/editions` renvoie `complete`, un `publisher` éventuellement `null`,
  et `502` si aucune notice n'a pu être lue.
- **Rattrapage** : `edition-mapping?since=AAAAMMJJ` (`bib.creationdate`). Délai mesuré entre sortie et
  notice : 3-4 semaines chez les grands éditeurs, ~3 mois hors de France. Côté serveur, file
  `edition-watch` hebdomadaire sur les séries synchronisées des comptes premium →
  `GET /v1/series-updates` (cf. [../backend/scan-asynchrone-bullmq.md](../backend/scan-asynchrone-bullmq.md)
  § 4ter) ; côté app, vérification à l'ouverture d'une série (cf.
  [../app/ecrans-et-navigation.md](../app/ecrans-et-navigation.md)).
- **Licence** : la politique d'usage MangaDex interdit les « paid services » — exception écrite à
  obtenir avant le lancement du palier payant ; l'app crédite désormais MangaDex (écran de recherche,
  sources de « Gérer les collections »).

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

## LEGO (Rebrickable · Brickset)

- **Cible** : `rebrickable-data` (port 3190, schéma `acv_rebrickable`), **hors gateway** : `/internal/sets`,
  `/internal/sets/by-barcode/:code`, `/internal/sets/:setNum`, `/internal/themes`, `/admin/*`. Doc :
  `acervatim/backend-services/rebrickable-data/README.md`.
- **Type de collection `lego`** (constellation, action C7, 2026-11-17) : type plat, servi par
  `catalog-lego` (port 3220, clone de `catalog-vinyl` sur `catalog-core`). La recherche passe par
  `GET /v1/search?type=lego` : l'adapter `rebrickable` de `search`
  (`search/src/sources/rebrickable/rebrickable.adapter.ts`) lit `rebrickable-data` en `/internal/*`,
  sans appel externe ; au scan, `meta.barcodeLookup` dit d'où vient la réponse. Le numéro canonique
  (`42131-1`) va dans `sources[]` avec `provider: 'rebrickable'`. Conception :
  `_bmad-output/implementation-artifacts/spec-collection-lego.md` (F10).
- **Ancien `catalog-lego`** (stack autonome, port 3010, moissonnage de l'API Rebrickable page par page,
  contraire aux CGU 5.3, clés personnelles) : supprimé le 2026-11-17, jamais déployé.
- **Images** : URL Rebrickable servies telles quelles (hotlink autorisé, téléchargement automatisé
  interdit) ; ce sont des images LEGO, Fair Play non commerciale → décision **pending** avant le premium.
