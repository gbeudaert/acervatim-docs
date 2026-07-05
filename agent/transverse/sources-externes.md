# Sources de données externes (Discogs, MAL, TMDB, BnF)

> Remonter : [../INDEX.md](../INDEX.md) · Doc humaine associée : [../../utilisateur/sources-de-donnees.md](../../utilisateur/sources-de-donnees.md), [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md)
> Toute la communication avec les providers passe par le **backend** (jamais d'appel direct depuis l'app).

## Vue d'ensemble

| Provider | Type de collection | Auth (jeton résolu, cf. §Résolution des jetons) | Usage |
|---|---|---|---|
| **Discogs** | vinyl | BYOT OAuth 1.0a (compte user) → repli premium : personal token Acervatim (`DISCOGS_ACERVATIM_TOKEN`) | Recherche texte + barcode (`database/search?barcode=`), métadonnées albums |
| **MyAnimeList (MAL)** | manga | BYOT OAuth2 PKCE (compte user) → repli premium : `X-MAL-CLIENT-ID` (`MAL_CLIENT_ID`) | Métadonnées séries (le nœud série porte la source MAL) |
| **Google Books** | manga (livre) | Clé API optionnelle (`GOOGLE_BOOKS_API_KEY`) | **Jaquette par ISBN/tome** (seule source par volume — MAL n'a que le visuel de série) ; expose aussi `description` (résumé par édition) |
| **TMDB** | movie | BYOT clé API perso (`PUT /v1/sources/tmdb/token`, pas d'OAuth) → repli premium : clé serveur (`TMDB_API_KEY`) | Recherche + détails films |
| **BnF (SRU, sans auth)** | manga | Aucune (API publique) | Pivot **ISBN → série/tome FR** ; énumération des tomes d'une édition (`edition-mapping`) |

## Architecture côté backend

- **Contrat commun** : `SourceAdapter` / `UnifiedItem` dans `src/oauth/providers/types.ts` (méthodes `search`, `searchByBarcode?`, `mediaType`).
- **Registre** : token `SOURCE_ADAPTERS` fourni par `src/common/sources/sources.module.ts` ; `SearchService` construit une `Map<typeCode, adapter>`.
- **Adapters** : `src/oauth/providers/discogs.adapter.ts`, `mal.adapter.ts`, `tmdb.adapter.ts`.
- **BnF** : `src/common/sources/bnf/` — `bnf.service.ts` (SRU, énumération d'édition), `unimarc.parser.ts` (parsing notices UNIMARC), `bnf.types.ts` (`EditionMapping`).
- **Snapshots** : `src/common/sources/source-snapshot.service.ts` — normalisation des `sources[]` (`{ provider, externalId, rawData, fetchedAt }`) attachées aux items/nodes.
- **Protections** : cache DB (`common/cache/api-cache.service.ts`), rate limit token-bucket (`common/rate-limit/`), client HTTP commun (`common/http/http-client.service.ts`).

## Résolution des jetons (BYOT / repli premium / mode dégradé)

`TokenResolverService` (`src/oauth/token-resolver.service.ts`), consommé par les adapters Discogs / MAL / TMDB, applique **une logique unique** pour tous les profils :

1. **Jeton utilisateur (BYOT)** présent → on l'utilise.
2. sinon **premium** → repli sur les **jetons Acervatim** (voir colonne *Auth* ci-dessus), plafonné par un bucket de rate-limit partagé `acervatim:<source>`.
3. sinon → **mode dégradé** : aucun appel sortant vers une source à jeton ; on sert les sources sans clé (BnF) et le **cache partagé** ; à défaut, `403 source-token-required` (payload `provider`) → l'app propose *connecter la source* **ou** *passer premium*.

Cache de recherche indexé sur la requête publique (`<source>:<requête>`, sans `userId`) → réponses partageables sans fuite (endpoints publics). Doc humaine : [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md#résolution-des-jetons-byot--repli-premium--mode-dégradé).

## Pivot ISBN → MAL (manga)

Problème : MAL ne connaît pas les ISBN ; la BnF connaît les éditions FR. Chaîne :

```
ISBN scanné → BnF (notice tome FR : titre série, n° tome, mention d'édition)
           → titre série → MAL (métadonnées série) [source attachée au nœud]
           → /v1/search/edition-mapping (BnF) → liste complète des tomes de l'édition
```

Doc de conception : [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md).

## Jaquettes par tome (Google Books)

MAL ne fournit qu'une illustration **de série** (`main_picture`) ; la BnF est bibliographique (aucune image). La seule source de jaquette **par volume** est **Google Books**, interrogée **par ISBN** :

- Service `common/sources/googlebooks/googlebooks.service.ts` : `resolveCover(isbn)` (réseau, best-effort, cache positif 90 j / négatif 7 j) et `cachedCover(isbn)` (lecture cache-only). Réutilise `HttpClientService` + `ApiCacheService` + `TokenBucketService` comme la BnF ; module `@Global`.
- `GET /v1/search/cover?isbn=` → `{ coverUrl }` : résout **et** met en cache (source de vérité de la résolution).
- `GET /v1/search/edition-mapping` : chaque tome porte désormais `coverUrl`, peuplé **depuis le cache uniquement** (aucun appel externe pendant l'énumération → latence inchangée ; le cache est alimenté par `/cover`).
- Google Books expose aussi `volumeInfo.description` (résumé par édition, HTML) et `averageRating`, mais **ne remplace pas MAL** : il n'a aucune notion de **série** (pas de `num_volumes`, pas de statut de publication, pas d'agrégation série). MAL reste la source du **nœud série** ; Google Books enrichit au **tome** (jaquette, éventuellement résumé).

## OAuth utilisateur (connexion de comptes externes)

- Flows : `src/oauth/oauth.service.ts` + `oauth-flow.registry.ts` ; primitives `providers/oauth1.ts` (Discogs) et `providers/pkce.ts` (MAL). **TMDB n'a pas de flow OAuth** : sa clé perso se saisit via `PUT /v1/sources/tmdb/token` (`src/sources/sources.controller.ts`), `DELETE` pour révoquer.
- Endpoints : voir [../backend/api-endpoints.md](../backend/api-endpoints.md) § OAuth et § Sources externes.
- Stockage : `oauth_credentials` (providers `discogs` / `mal` / `tmdb`), tokens/clés chiffrés AES-256-GCM ([../backend/securite-et-privacy.md](../backend/securite-et-privacy.md)).
- Côté app : `ui/account/AccountScreen.kt` + `AcervatimApi.startOAuth/connectedProviders/revokeOAuth`.
