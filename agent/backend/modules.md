# acervatim-backend — Modules (feature + common)

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-backend/src/`
> Chaque module feature = dossier avec `*.module.ts`, `*.controller.ts`, `*.service.ts`, `dto/`. Tests co-localisés (`*.spec.ts`, `*.e2e.spec.ts`).

## Modules feature

| Module | Fichiers clés | Responsabilité |
|---|---|---|
| `auth/` | `auth.service.ts`, `identity-resolver.service.ts`, `providers/google.provider.ts`, `providers/identity-provider.registry.ts` | Vérifie l'idToken Google, hache le `sub` (HMAC), trouve/crée le `User`, émet le JWT interne. Le registry permet d'ajouter d'autres identity providers |
| `users/` | `users.service.ts` | `/me` : profil, statut premium, export RGPD (user + collections + **nœuds** + items + oauth + sub/grant + audit), suppression de compte (+ audit log) |
| `collections/` | `collections.service.ts`, `collection-type-codes.ts`, `types/` | CRUD collections. `types/registry.ts` → profils `vinylProfile` / `mangaProfile` / `makeFallbackProfile` ; `types/common.ts` (interface `CollectionTypeProfile`), `types/hierarchy.ts` (règles hiérarchie). Recherche dans le contenu : `matchingCollectionIds` (SQL collate CI) |
| `items/` | `items.service.ts`, `item-source-codes.ts`, `dto/item-user-data.schema.ts` | CRUD items, validation `unifiedData` par profil, attach de sources, plafond technique, maintenance `itemCount` |
| `nodes/` | `nodes.service.ts` | CRUD nœuds (séries), attach de sources, purge des nœuds vides non-wishlist |
| `search/` | `search.service.ts` | Route `q`/`barcode` vers l'adapter du type (`Map<CollectionTypeCode, SourceAdapter>` injectée via `SOURCE_ADAPTERS`) ; `editionMapping` → BnF |
| `oauth/` | `oauth.service.ts`, `oauth-flow.registry.ts`, `providers/` | Flows OAuth user : `discogs.adapter.ts` (OAuth 1.0a, cf. `providers/oauth1.ts`), `mal.adapter.ts` (OAuth2 PKCE, cf. `providers/pkce.ts`), `tmdb.adapter.ts`. Tokens chiffrés via `AesService`. `providers/types.ts` définit `SourceAdapter`/`UnifiedItem` (aussi utilisés par search) |
| `premium/` | `premium.service.ts`, `premium.guard.ts`, `payment-required.exception.ts` | Statut premium = `Subscription` active OU `PremiumGrant` valide ; guard 402 sur les features payantes |
| `subscriptions/` | `subscriptions.service.ts`, `google-play.service.ts`, `pubsub-verifier.service.ts`, `subscriptions.cron.ts`, `subscription-status.ts` | Vérification d'achat Play, webhook RTDN (Pub/Sub), machine à états des statuts, cron de réconciliation des subs qui expirent |
| `sharing/` | `sharing.service.ts`, `sharing.controller.ts`, `share-scope.ts`, `share-code-invalid.exception.ts` | Partage d'une collection en lecture (S3) : modèle `CollectionShare`, code hors-bande HMAC (`SHARE_CODE_PEPPER`), création/révocation/redeem. **Aucune route de lecture** — le filtrage par portée est le travail de S4 |
| `invitations/` | `invitations.service.ts`, `admin-token.guard.ts`, exceptions dédiées | Création admin (code HMAC-hashé) + redeem user (grant premium éventuel). DTOs encore en class-validator (migration prévue) |
| `prisma/` | `prisma.service.ts` | Client Prisma injectable |
| `config/` | `env.validation.ts` | Validation stricte des variables d'environnement au boot |

## `common/` — infrastructure partagée

| Sous-dossier | Fichiers clés | Responsabilité |
|---|---|---|
| `common/audit/` | `audit-log.service.ts` | Journal append-only (aucune méthode update/delete) |
| `common/cache/` | `api-cache.service.ts` | Cache DB des réponses providers (TTL applicatif) |
| `common/crypto/` | `aes.service.ts`, `hash.service.ts` | AES-256-GCM (tokens OAuth) ; HMAC-SHA-256 (subHash, codes d'invitation) |
| `common/decorators/` | `current-user-id.decorator.ts` | `@CurrentUserId()` dans les controllers |
| `common/filters/` | `parse-filters.ts`, `problem-details.filter.ts` | Parsing `field[in]`/`field[all]` → `where` Prisma (whitelist Zod) ; filtre global d'erreurs RFC 9457 |
| `common/guards/` | `jwt-auth.guard.ts` | Vérification JWT Bearer (guard par défaut des routes /v1) |
| `common/http/` | `http-client.service.ts` | Client HTTP sortant (providers externes) |
| `common/interceptors/` | `correlation-id.interceptor.ts` | `requestId` propagé dans réponses et erreurs |
| `common/logging/` | `access-log.middleware.ts` | Log d'accès (sans données perso) |
| `common/pagination/` | `paginate.ts` | Helper pagination cursor (`CursorPage`, wrapper `{ data, meta }`) |
| `common/limits/` | `limits.service.ts`, `technical-limit.exception.ts` | **Plafonds techniques** (`TECHNICAL_LIMITS`), identiques pour tous, non exposés, décorrélés du palier — anti-emballement, pas une limite d'offre. Dépassement → **409** `/probs/technical-limit`. Ex-`common/quota/`, requalifié en S2 |
| `common/sources/` | `source-snapshot.service.ts` (token `SOURCE_ADAPTERS`), `bnf/`, `mangadex/`, `googlebooks/`, `manga-matching.ts` | Registre des adapters ; `bnf/` (SRU BnF, parsing UNIMARC, énumération d'éditions) ; `mangadex/` (identité manga **source principale** + jaquettes par tome, file BullMQ, sans adapter) ; `googlebooks/` (jaquette par ISBN, repli) ; `manga-matching.ts` (rapprochement titre/auteur partagé MAL+MangaDex + `isSpecialArtEdition`) |

> Le throttle des appels sortants (ex-`common/rate-limit/token-bucket.service.ts`, **retiré**) passe désormais par les **limiters BullMQ** — une file par source (`gbooks`, `bnf`, `mal`, `discogs`, `tmdb`, `mangadex`) ; cf. [scan-asynchrone-bullmq.md](scan-asynchrone-bullmq.md).

## Ajouter un nouveau type de collection (recette)

1. Seed du `CollectionType` (`prisma/seed.ts`) + code dans `collections/collection-type-codes.ts`.
2. Profil dans `collections/types/<code>.ts` (schéma `unifiedData`, projection liste, hiérarchie éventuelle) + enregistrement dans `types/registry.ts`.
3. Adapter de recherche implémentant `SourceAdapter` (`oauth/providers/types.ts`), enregistré sous `SOURCE_ADAPTERS` (`common/sources/sources.module.ts`).
4. Côté app : visibilité du type + écrans (voir [../app/catalogue-fonctionnalites.md](../app/catalogue-fonctionnalites.md)).
