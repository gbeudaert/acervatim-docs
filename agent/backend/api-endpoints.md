# acervatim-backend — Catalogue des endpoints HTTP

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-backend/src/`
> Préfixe global `/v1` (posé dans `main.ts`) sauf mention contraire. Toutes les routes sont JWT-protected (`JwtAuthGuard`) sauf mention **[public]** ou **[admin]**.
> Conventions (pagination cursor, filtres `[in]`/`[all]`, Problem Details) : [../conventions.md](../conventions.md).

## Infra

| Route | Fichier | Notes |
|---|---|---|
| `GET /health` **[public, hors /v1]** | `app.controller.ts` | Healthcheck infra |
| `GET /docs` **[hors prod]** | `main.ts` | Swagger UI (OpenAPI généré au bootstrap) |

## Auth & utilisateur

| Route | Controller | Service | Notes |
|---|---|---|---|
| `POST /v1/auth/google` **[public]** | `auth/auth.controller.ts` | `auth/auth.service.ts` | idToken Google → JWT interne ; résolution/création user via `identity-resolver.service.ts` |
| `GET /v1/me` | `users/users.controller.ts` | `users/users.service.ts` | Profil (id opaque, dates, premium) |
| `GET /v1/me/quota` | idem | idem | Quota d'items (`common/quota/quota.service.ts`) |
| `GET /v1/me/export` | idem | idem | Export RGPD des données |
| `DELETE /v1/me` | idem | idem | Suppression compte (cascade Prisma) |

## Collections

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
| `PATCH /v1/nodes/:id` | idem | Peut purger un nœud vide non-wishlist → `204` |
| `POST /v1/nodes/:id/sources` | idem | Attacher une source externe (ex. MAL) ; `dto/attach-source.dto.ts` |

## Items

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/collections/:collectionId/items` | `items/items.controller.ts` | Validation `unifiedData` par le profil du type ; quota vérifié |
| `GET /v1/collections/:collectionId/items` | idem | Projection légère dépendante du type ; filtres `provider[in]`, `nodeId` |
| `GET /v1/items/:id` | idem | |
| `GET /v1/items/:id/sources` | idem | |
| `POST /v1/items/:id/sources` | idem | Attacher une source (ex. ISBN/BnF) |
| `PATCH /v1/items/:id` | idem | `unifiedData` / `userData` (schéma `dto/item-user-data.schema.ts`) |
| `DELETE /v1/items/:id` | idem | |

## Recherche externe

| Route | Controller | Notes |
|---|---|---|
| `GET /v1/search` | `search/search.controller.ts` | `type` obligatoire ; `q` XOR `barcode` ; routé vers l'adapter du type (Discogs pour vinyl, MAL/BnF pour manga) via `SOURCE_ADAPTERS` — voir [../transverse/sources-externes.md](../transverse/sources-externes.md) |
| `GET /v1/search/edition-mapping` | idem | `title` (+ `edition`) → énumération des tomes d'une édition via BnF (`common/sources/bnf/bnf.service.ts`) |

## OAuth externe (comptes user Discogs / MAL / TMDB)

| Route | Controller | Notes |
|---|---|---|
| `GET /v1/oauth/:provider/start` | `oauth/oauth.controller.ts` | Démarre le flow (OAuth1 Discogs, PKCE MAL, TMDB) |
| `GET /v1/oauth/:provider/callback` | idem | Callback ; tokens chiffrés AES-256-GCM en base |
| `GET /v1/oauth` | idem | Providers connectés |
| `DELETE /v1/oauth/:provider` | idem | Révocation |

## Premium & abonnements

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/subscriptions/verify` | `subscriptions/subscriptions.controller.ts` | Vérifie un achat Play (`google-play.service.ts`), upsert `Subscription` |
| `POST /webhooks/google-play/rtdn` **[public, hors JWT]** | idem | Webhook RTDN Pub/Sub ; signature vérifiée par `pubsub-verifier.service.ts` |

Cron de réconciliation : `subscriptions/subscriptions.cron.ts`. Statuts : `subscriptions/subscription-status.ts`. Gate premium : `premium/premium.guard.ts`.

## Invitations

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/admin/invitations` **[admin]** | `invitations/invitations.controller.ts` | Protégé par `admin-token.guard.ts` (token admin, pas JWT user). Code retourné une seule fois |
| `POST /v1/invitations/redeem` | idem | JWT requis ; code haché HMAC comparé à `Invitation.codeHash` ; peut créer un `PremiumGrant` |
