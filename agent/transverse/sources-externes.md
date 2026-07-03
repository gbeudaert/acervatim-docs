# Sources de données externes (Discogs, MAL, TMDB, BnF)

> Remonter : [../INDEX.md](../INDEX.md) · Doc humaine associée : [../../utilisateur/sources-de-donnees.md](../../utilisateur/sources-de-donnees.md), [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md)
> Toute la communication avec les providers passe par le **backend** (jamais d'appel direct depuis l'app).

## Vue d'ensemble

| Provider | Type de collection | Auth | Usage |
|---|---|---|---|
| **Discogs** | vinyl | OAuth 1.0a (compte user) | Recherche texte + barcode (`database/search?barcode=`), métadonnées albums |
| **MyAnimeList (MAL)** | manga | OAuth2 PKCE (compte user) | Métadonnées séries (le nœud série porte la source MAL) |
| **TMDB** | movie (futur) | API key / flow dédié | Prévu pour les films |
| **BnF (SRU, sans auth)** | manga | Aucune (API publique) | Pivot **ISBN → série/tome FR** ; énumération des tomes d'une édition (`edition-mapping`) |

## Architecture côté backend

- **Contrat commun** : `SourceAdapter` / `UnifiedItem` dans `src/oauth/providers/types.ts` (méthodes `search`, `searchByBarcode?`, `mediaType`).
- **Registre** : token `SOURCE_ADAPTERS` fourni par `src/common/sources/sources.module.ts` ; `SearchService` construit une `Map<typeCode, adapter>`.
- **Adapters** : `src/oauth/providers/discogs.adapter.ts`, `mal.adapter.ts`, `tmdb.adapter.ts`.
- **BnF** : `src/common/sources/bnf/` — `bnf.service.ts` (SRU, énumération d'édition), `unimarc.parser.ts` (parsing notices UNIMARC), `bnf.types.ts` (`EditionMapping`).
- **Snapshots** : `src/common/sources/source-snapshot.service.ts` — normalisation des `sources[]` (`{ provider, externalId, rawData, fetchedAt }`) attachées aux items/nodes.
- **Protections** : cache DB (`common/cache/api-cache.service.ts`), rate limit token-bucket (`common/rate-limit/`), client HTTP commun (`common/http/http-client.service.ts`).

## Pivot ISBN → MAL (manga)

Problème : MAL ne connaît pas les ISBN ; la BnF connaît les éditions FR. Chaîne :

```
ISBN scanné → BnF (notice tome FR : titre série, n° tome, mention d'édition)
           → titre série → MAL (métadonnées série) [source attachée au nœud]
           → /v1/search/edition-mapping (BnF) → liste complète des tomes de l'édition
```

Doc de conception : [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md).

## OAuth utilisateur (connexion de comptes externes)

- Flows : `src/oauth/oauth.service.ts` + `oauth-flow.registry.ts` ; primitives `providers/oauth1.ts` (Discogs) et `providers/pkce.ts` (MAL).
- Endpoints : voir [../backend/api-endpoints.md](../backend/api-endpoints.md) § OAuth.
- Stockage : `oauth_credentials`, tokens chiffrés AES-256-GCM ([../backend/securite-et-privacy.md](../backend/securite-et-privacy.md)).
- Côté app : `ui/account/AccountScreen.kt` + `AcervatimApi.startOAuth/connectedProviders/revokeOAuth`.
