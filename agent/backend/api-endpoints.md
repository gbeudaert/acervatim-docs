# acervatim-backend — Catalogue des endpoints HTTP

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-backend/src/`
> Préfixe global `/v1` (posé dans `main.ts`) sauf mention contraire. Toutes les routes sont JWT-protected (`JwtAuthGuard`) sauf mention **[public]** ou **[admin]**.
> Conventions (pagination cursor, filtres `[in]`/`[all]`, Problem Details) : [../conventions.md](../conventions.md).

## Infra

| Route | Fichier | Notes |
|---|---|---|
| `GET /health` **[public, hors /v1]** | `app.controller.ts` | Healthcheck infra ; expose aussi `redis: up\|down` (le `status` reste `ok` : Redis non vital au boot) |
| `GET /docs` **[hors prod]** | `main.ts` | Swagger UI (OpenAPI généré au bootstrap) |
| `GET /v1/admin/queues` **[admin]** | `metrics/queue-metrics.controller.ts` | Snapshot des compteurs BullMQ par file (backlog/débit/échecs) pour surveiller la passerelle sortante et calibrer les limiters. Protégé par `admin-token.guard.ts` |

## Auth & utilisateur

| Route | Controller | Service | Notes |
|---|---|---|---|
| `POST /v1/auth/google` **[public]** | `auth/auth.controller.ts` | `auth/auth.service.ts` | idToken Google → JWT interne ; résolution/création user via `identity-resolver.service.ts` |
| `GET /v1/me` | `users/users.controller.ts` | `users/users.service.ts` | Profil (id opaque, dates, premium) |
| `GET /v1/me/export` | idem | idem | Export RGPD des données : `user`, `collections`, `collectionNodes`, `items`, `oauthCredentials` (chiffrés), `subscription`, `premiumGrant`, `invitationRedemptions`, `auditLogs` |
| `DELETE /v1/me` | idem | idem | Suppression compte (cascade Prisma) |

## Collections / Nodes / Items — **premium-only depuis S2**

> Ces trois familles portent la synchronisation, donc un coût serveur : elles sont **réservées au
> premium**. Deux guards distincts, et confondre les deux casserait le partage (S4) :
>
> | Opération | Guard | Statut vérifié |
> |---|---|---|
> | Écriture (POST/PATCH/DELETE) | `premium/premium.guard.ts` | Le **requérant** — seul le propriétaire écrit |
> | Lecture (GET d'une ressource) | `premium/collection-premium.guard.ts` + `@CollectionRef(via, param)` | Le **propriétaire de la collection**, pour qu'un membre non-premium puisse lire ce qu'on lui partage |
>
> Refus → `402` `/probs/payment-required`. Une ressource d'autrui reste un `404` (on ne confirme pas
> son existence). `CollectionAccessService.resolveAccess` est le point d'extension de S4 (rôle
> `shared` + `scope`).

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/collections` | `collections/collections.controller.ts` | `typeCode` requis (vinyl/manga/…) ; DTO `dto/create-collection.dto.ts` |
| `GET /v1/collections` | idem | Filtres : `type[in]`, `items[in]`/`items[all]` (recherche substring CI dans le contenu — SQL `COLLATE utf8mb4_general_ci`, cf. `CollectionsService.matchingCollectionIds`) |
| `GET /v1/collections/:id` | idem | |
| `PATCH /v1/collections/:id` | idem | `dto/update-collection.dto.ts` |
| `DELETE /v1/collections/:id` | idem | Cascade items/nodes |

## Nodes (hiérarchie — séries manga)

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/collections/:collectionId/nodes` | `nodes/nodes.controller.ts` | Créer une série ; `dto/create-node.dto.ts` |
| `GET /v1/collections/:collectionId/nodes` | idem | `?level=serie` |
| `GET /v1/nodes/:id` | idem | |
| `GET /v1/nodes/:id/sources` | idem | Sources avec `rawData` |
| `PATCH /v1/nodes/:id` | idem | `unifiedData` (re-validé par le niveau) + `note`/`comment` **à plat** (pas sous `userData` — forme historique du contrat V1, schéma `dto/node-user-data.schema.ts`) + `isWishlist`. Peut purger un nœud vide non-wishlist → `204` |
| `POST /v1/nodes/:id/sources` | idem | Attacher une source externe (ex. MAL) ; `dto/attach-source.dto.ts` |

## Items

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/collections/:collectionId/items` | `items/items.controller.ts` | Validation `unifiedData` par le profil du type ; plafond technique vérifié |
| `GET /v1/collections/:collectionId/items` | idem | Projection légère dépendante du type ; filtres `provider[in]`, `nodeId`. Toutes les projections exposent `status` (statut de possession résolu, `OWNED` par défaut) |
| `GET /v1/items/:id` | idem | |
| `GET /v1/items/:id/sources` | idem | |
| `POST /v1/items/:id/sources` | idem | Attacher une source (ex. ISBN/BnF) |
| `PATCH /v1/items/:id` | idem | `unifiedData` (remplacé en bloc) / `userData` (mergé) — schéma `dto/item-user-data.schema.ts`, dont `status: OWNED\|WISHLIST\|IGNORED\|null` |
| `DELETE /v1/items/:id` | idem | |

## Recherche externe

| Route | Controller | Notes |
|---|---|---|
| `GET /v1/search` | `search/search.controller.ts` | `type` obligatoire ; `q` XOR `barcode` ; routé vers l'adapter du type (Discogs pour vinyl, MAL/BnF pour manga) via `SOURCE_ADAPTERS`. Jeton résolu par `TokenResolverService` : BYOT → repli premium → mode dégradé → `403 source-token-required` — voir [../transverse/sources-externes.md](../transverse/sources-externes.md#résolution-des-jetons-byot--repli-premium--mode-dégradé) |
| `GET /v1/search/edition-mapping` | idem | `title` (+ `edition`, `malId`, `mangaId` optionnels) → énumération des tomes d'une édition via BnF (`common/sources/bnf/bnf.service.ts`) ; chaque tome porte `coverUrl` **et `description`** (résumé par tome : `330$a` BnF → repli Google Books → `null`), résolus en réseau à **concurrence bornée** (`COVER_RESOLUTION_CONCURRENCY`, anti-livelock du bucket `gbooks:global`). La série MangaDex est jointe par `mangaId` › `malId` › auteur BnF, sinon repli ISBN (cf. [sources-externes](../transverse/sources-externes.md)) |
| `GET /v1/search/cover` | idem | `isbn` (+ `title`/`volume`/`edition` optionnels pour **armer le repli `intitle:`**) → `{ coverUrl }` : résout la jaquette d'un tome via Google Books (`common/sources/googlebooks/googlebooks.service.ts`) et alimente le cache |
| `POST /v1/search/import-jobs` | `search/import/edition-import.controller.ts` | Lance l'import **asynchrone** d'une édition (`{ title, edition?, malId?, mangaId? }`) → `202 { jobId, state }`. Job BullMQ `edition-import` (fond, concurrence 1) qui énumère la BnF + réchauffe les jaquettes ; `jobId` **déterministe** (hash titre+édition) → dédup inter-user. Sort le fan-out lent du cycle requête-réponse (cf. incident 0.5.3) |
| `GET /v1/search/import-jobs/:id` | idem | Statut + avancement du job → `{ jobId, state: queued\|running\|done\|failed, progress: { phase, done, total } \| null }`. `404` si inconnu. L'app poll puis, à `done`, re-tape `edition-mapping` (cache chaud) |

## OAuth externe (comptes user Discogs / MAL)

| Route | Controller | Notes |
|---|---|---|
| `GET /v1/oauth/:provider/start` | `oauth/oauth.controller.ts` | Démarre le flow (OAuth1 Discogs, PKCE MAL). **TMDB non concerné** (pas d'OAuth → `/start` renvoie 404) |
| `GET /v1/oauth/:provider/callback` | idem | Callback ; tokens chiffrés AES-256-GCM en base |
| `GET /v1/oauth` | idem | Providers connectés |
| `DELETE /v1/oauth/:provider` | idem | Révocation |

## Sources externes (clé personnelle — BYOT hors OAuth)

| Route | Controller | Notes |
|---|---|---|
| `PUT /v1/sources/tmdb/token` | `sources/sources.controller.ts` | Saisit/remplace la clé API TMDB perso (`{ token }`), stockée chiffrée (`oauth_credentials`, provider `tmdb`). Pas de validation à la saisie |
| `DELETE /v1/sources/tmdb/token` | idem | Révoque la clé TMDB perso (idempotent, `204`) |

## Premium & abonnements

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/subscriptions/verify` | `subscriptions/subscriptions.controller.ts` | Vérifie un achat Play (`google-play.service.ts`), upsert `Subscription` |
| `POST /webhooks/google-play/rtdn` **[public, hors JWT]** | idem | Webhook RTDN Pub/Sub ; signature vérifiée par `pubsub-verifier.service.ts` |

Cron de réconciliation : `subscriptions/subscriptions.cron.ts`. Statuts : `subscriptions/subscription-status.ts`. Gates premium : `premium/premium.guard.ts` (requérant) et `premium/collection-premium.guard.ts` (propriétaire, cf. § Collections).

## Invitations

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/admin/invitations` **[admin]** | `invitations/invitations.controller.ts` | Protégé par `admin-token.guard.ts` (token admin, pas JWT user). Code retourné une seule fois |
| `POST /v1/invitations/redeem` | idem | JWT requis ; code haché HMAC comparé à `Invitation.codeHash` ; peut créer un `PremiumGrant` |
