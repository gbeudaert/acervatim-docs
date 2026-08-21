# acervatim-backend — Modèle de données (Prisma / MariaDB)

> Remonter : [INDEX.md](INDEX.md) · Source de vérité : `acervatim-backend/prisma/schema.prisma` · Migrations : `prisma/migrations/`
> Conventions : IDs UUID v4 `Char(36)`, tables/colonnes snake_case via `@map`, workflow migration → [operations-dev.md](operations-dev.md).

## Vue d'ensemble

```
User ─┬─ Collection ─┬─ Item            (albums / tomes)
      │              └─ CollectionNode  (séries manga, arbre via parentId)
      │                      └─ Item    (tomes rattachés au nœud)
      ├─ OauthCredential                (tokens externes chiffrés)
      ├─ Subscription / PremiumGrant    (premium)
      └─ InvitationRedemption ── Invitation
AuditLog · ApiCache                     (techniques, sans FK user obligatoire)
```

## Tables

| Modèle (table) | Rôle | Points clés |
|---|---|---|
| `User` (`users`) | Identité anonyme | `googleSubHash` unique (HMAC-SHA-256 du sub Google) ; **aucune** donnée perso |
| `Collection` (`collections`) | Collection d'un user | FK `typeId` → `CollectionType` (Restrict) ; `itemCount` dénormalisé |
| `CollectionType` (`collection_types`) | Constantes de types | `code` logique stable (`vinyl`, `manga`, …) ; seedé (`prisma/seed.ts`) |
| `Item` (`items`) | Album / tome | `unifiedData` (vérité curée) · `sources[]` (snapshots providers) · `userData` (rating, `status` possédé/désiré…) ; `nodeId`+`volume` pour manga (unique `[userId, nodeId, volume]`), null pour types plats |
| `CollectionNode` (`collection_nodes`) | Série manga (arbre) | `level` (`serie`/`set`), `parentId` auto-référent, mêmes triplets JSON qu'`Item`, `isWishlist` survit à 0 tome possédé |
| `OauthCredential` (`oauth_credentials`) | Tokens Discogs/MAL/TMDB | PK `[userId, provider]` ; tokens chiffrés AES-256-GCM + `encryptedWithKeyVersion` |
| `Subscription` (`subscriptions`) | Abonnement Google Play | Statuts `active/grace_period/on_hold/paused/cancelled/expired` ; `purchaseToken` unique ; index `[status, expiresAt]` pour le cron |
| `PremiumGrant` (`premium_grants`) | Premium hors billing | `reason` (`self/beta_tester/comp/support/admin`), `expiresAt` null = permanent |
| `Invitation` (`invitations`) | Codes d'invitation | PK = `codeHash` (HMAC, jamais le code en clair) ; `maxUses`/`usedCount` ; peut porter un grant premium |
| `InvitationRedemption` (`invitation_redemptions`) | Trace de claim | PK `[invitationCodeHash, userId]` : un user ne re-claim pas le même code |
| `AuditLog` (`audit_logs`) | Journal append-only | Aucun update/delete exposé (`common/audit/audit-log.service.ts`) ; actions `auth.login`, `user.delete`, `premium.grant`, … |
| `ApiCache` (`api_cache`) | Cache réponses providers | TTL applicatif : rows expirées purgées par cron |

> Le modèle `RateLimitBucket` (token bucket SQL) a été **retiré** : le throttle des appels sortants passe désormais par les **limiters BullMQ** (une file par source — cf. [scan-asynchrone-bullmq.md](scan-asynchrone-bullmq.md)).

## Invariants à respecter

- Tout nouveau champ sur `users` doit passer le check « est-ce que ça réidentifie un humain ? » (cf. [securite-et-privacy.md](securite-et-privacy.md)).
- `unifiedData` / `sources` / `userData` : ne jamais mélanger les trois rôles (curé / brut externe / subjectif).
- Suppressions : cascades Prisma en place (`onDelete: Cascade`) depuis `User` — la suppression de compte s'appuie dessus.
- `Item.userId` est dénormalisé volontairement (requêtes par user sans join).
- Schéma des `userData` d'item : `src/items/dto/item-user-data.schema.ts` ; des `userData` de nœud : `src/nodes/dto/node-user-data.schema.ts` (`note` 0–10, `comment`, voyageant **à plat** dans le PATCH) ; schémas `unifiedData` par type : `src/collections/types/{vinyl,manga,fallback}.ts`.
- Le **code-barres** n'a qu'une source de vérité : `sources[]` avec `provider: 'barcode'`. Il n'est plus un champ de `unifiedData` (retiré silencieusement à la validation, cf. [contrat-app-serveur.md](contrat-app-serveur.md)).
- `userData.status` (`OWNED` / `WISHLIST` / `IGNORED`) : statut de possession **par item**, poussé par l'app. Absent ou `null` ⇒ `OWNED` (`DEFAULT_ITEM_STATUS` / `resolveItemStatus()` dans le même fichier) — les items synchronisés avant son introduction restent valides, aucune migration. Distinct de `CollectionNode.isWishlist`, qui reste le marqueur au niveau **série** : les deux coexistent sans fusionner.
