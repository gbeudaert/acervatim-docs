# acervatim-backend — Catalogue des endpoints HTTP

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-backend/src/`
> Préfixe global `/v1` (posé dans `main.ts`) sauf mention contraire. Toutes les routes sont JWT-protected (`JwtAuthGuard`) sauf mention **[public]** ou **[admin]**.
> Conventions (pagination cursor, filtres `[in]`/`[all]`, Problem Details) : [contrats-api-v1.md](contrats-api-v1.md).

## Infra

| Route | Fichier | Notes |
|---|---|---|
| `GET /health` **[public, hors /v1]** | `app.controller.ts` | Healthcheck infra ; expose aussi `redis: up\|down` (le `status` reste `ok` : Redis non vital au boot) |
| `GET /docs` **[hors prod]** | `main.ts` | Swagger UI (OpenAPI généré au bootstrap) |
| `GET /v1/admin/queues` **[admin]** | `metrics/queue-metrics.controller.ts` | Snapshot des compteurs BullMQ par file (backlog/débit/échecs) pour surveiller la passerelle sortante et calibrer les limiters, **plus** `cache[]` : hit/miss du cache partagé par famille de clés (`discogs:search:q`, `discogs:search:barcode`, …) — la mesure de la pression quota réelle d'une recherche texte. Protégé par `admin-token.guard.ts` |

## Auth & utilisateur

| Route | Controller | Service | Notes |
|---|---|---|---|
| `POST /v1/auth/google` **[public]** | `auth/auth.controller.ts` | `auth/auth.service.ts` | idToken Google → JWT interne ; résolution/création user via `identity-resolver.service.ts` |
| `GET /v1/me` | `users/users.controller.ts` | `users/users.service.ts` | Profil (id opaque, dates, premium) |
| `GET /v1/me/export` | idem | idem | Export RGPD des données : `user`, `collections`, `collectionNodes`, `items`, `oauthCredentials` (chiffrés), `subscription`, `premiumGrant`, `invitationRedemptions`, `collectionShares` (partages émis : entrées + son libellé, sans `codeHash`), `shareMemberships` (partages rejoints : entrées + son libellé), `auditLogs`. Le libellé de l'autre partie n'y figure jamais |
| `DELETE /v1/me` | idem | idem | Suppression compte (cascade Prisma) |

## Collections / Nodes / Items — **premium-only depuis S2**

> Ces trois familles portent la synchronisation, donc un coût serveur : elles sont **réservées au
> premium**. Trois guards, et les confondre casserait le partage :
>
> | Opération | Guard | Statut vérifié |
> |---|---|---|
> | Lecture (GET d'une ressource) | `premium/collection-premium.guard.ts` + `@CollectionRef(via, param)` | Le **propriétaire de la collection**, pour qu'un membre non-premium puisse lire ce qu'on lui partage |
> | Écriture visant une collection identifiable | `premium/collection-write.guard.ts` + `@CollectionRef(...)` | Le **propriétaire**, qui doit aussi être le requérant |
> | Écriture sans collection en paramètre (`POST /v1/collections`, gestion des partages) | `premium/premium.guard.ts` | Le **requérant** |
>
> Les deux guards à `@CollectionRef` passent par `CollectionAccessService.resolveAccess`, qui rend
> `{ role: 'owner' }` ou `{ role: 'shared', statuses }` et pose le résultat sur la requête
> (`@CurrentCollectionAccess`) — le service n'a rien à re-résoudre, et les statuts n'ont aucun
> chemin par lequel arriver du client. Codes de refus :
>
> | Cas | Réponse |
> |---|---|
> | Ni propriétaire ni membre | `404` — on ne confirme pas l'existence d'une ressource d'autrui |
> | Membre d'un partage, en écriture | `403` — il **voit** déjà la ressource, un 404 mentirait sur ce qu'il lit |
> | Propriétaire non premium | `402` `/probs/payment-required` — y compris pour ses membres : perdre le premium **suspend** les partages émis, il ne les supprime pas |
> | Identifiant malformé | `400` (le guard passe la main à `ParseUUIDPipe`) |

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
| `POST /v1/collections/:collectionId/nodes` | `nodes/nodes.controller.ts` | Créer une série ; `dto/create-node.dto.ts`. `source` **et/ou** `unifiedData`, au moins l'un des deux ; plafond technique `nodes` vérifié |
| `GET /v1/collections/:collectionId/nodes` | idem | `?level=serie` |
| `GET /v1/nodes/:id` | idem | |
| `GET /v1/nodes/:id/sources` | idem | Sources avec `rawData` |
| `PATCH /v1/nodes/:id` | idem | `unifiedData` (re-validé par le niveau) + `note`/`comment` **à plat** (pas sous `userData` — forme historique du contrat V1, schéma `dto/node-user-data.schema.ts`) + `isWishlist`. Peut purger un nœud vide non-wishlist → `204` |
| `POST /v1/nodes/:id/sources` | idem | Attacher une source externe (ex. MAL) ; `dto/attach-source.dto.ts` |

**Deux façons de remplir un nœud, combinables.** `source` est une référence provider : si un
adapter existe (MAL), son snapshot pré-remplit la vérité curée ; sinon (`bnf`, `mangadex`, `isbn` —
pas d'adapter) la référence est simplement conservée dans `sources[]`. `unifiedData` est la vérité
curée envoyée par le client, re-validée par le `nodeSchema` du niveau — c'est le chemin de la
**saisie manuelle**, sans lequel une série créée à la main dans l'app ne serait pas synchronisable.
Quand les deux sont là, `unifiedData` l'emporte : c'est ce que l'utilisateur voit dans son app.
Une `source` vers un provider sans adapter **et** sans `unifiedData` reste un `400` : rien ne
décrirait le nœud. La dédup par `source` (idempotence des imports provider) est inchangée ; sans
`source` il n'y a rien sur quoi dédupliquer, le client garde l'`id` rendu et repasse par `PATCH`.

## Items

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/collections/:collectionId/items` | `items/items.controller.ts` | Validation `unifiedData` par le profil du type ; plafond technique vérifié. Série parente par `node` (réf. provider, upsert) **ou** `nodeId` (série déjà créée) — exclusifs |
| `GET /v1/collections/:collectionId/items` | idem | Projection légère dépendante du type ; filtres `provider[in]`, `nodeId`. Toutes les projections exposent `status` (statut de possession résolu, `OWNED` par défaut) et `playCount` (compteur d'écoutes résolu, `0` par défaut) — la galerie trie dessus sans re-fetch. N'y projeter que des champs de la liste blanche de partage : la liste sert aussi les membres |
| `GET /v1/items/:id` | idem | |
| `GET /v1/items/:id/sources` | idem | |
| `POST /v1/items/:id/sources` | idem | Attacher une source (ex. ISBN/BnF) |
| `PATCH /v1/items/:id` | idem | `unifiedData` (remplacé en bloc) / `userData` (mergé) — schéma `dto/item-user-data.schema.ts`, dont `status: OWNED\|WISHLIST\|IGNORED\|null` et `playCount` (entier ≥ 0, absent ⇒ 0, valeur **absolue** : pas d'incrément côté API) — plus `nodeId`/`volume` (identité structurelle du tome, `null` accepté) |
| `DELETE /v1/items/:id` | idem | |

**Tome et série (types hiérarchiques).** Rien n'est exigé : ni série, ni numéro de tome. Supprimer
une série côté app laisse ses tomes orphelins (le modèle app les conserve) et un hors-série n'a pas
de numéro — les refuser ferait de la synchronisation une perte de données, pas un miroir. Sur un
type **plat** (vinyl) `node`/`nodeId`/`volume` restent interdits (`400`).

- `nodeId` — série déjà créée par `POST /collections/:id/nodes`. C'est le chemin de la
  **synchronisation** : le client pousse ses nœuds, puis ses items, comme collections → items. Un
  `nodeId` hors de la collection visée ou d'un autre compte est un `404`, jamais un `400` (même
  règle qu'en lecture : pas d'oracle d'existence).
- `node` — référence provider, le nœud étant upserté s'il n'existe pas. Chemin des imports enrichis
  (MAL), qui ne connaissent pas encore l'`id` serveur du nœud. Sans adapter pour ce provider, un
  item ne peut pas inventer la vérité curée de son parent : `400` explicite qui renvoie vers
  `POST /collections/:id/nodes` puis `nodeId`.
- `PATCH /v1/items/:id` accepte `nodeId` et `volume` (tous deux `null`-ables) : sans eux, un
  changement de série ou de numéro resterait local et le miroir garderait l'ancienne valeur.
  Détacher le dernier tome d'une série non désirée la purge, comme le fait sa suppression.
  `(série, volume)` déjà pris → `409`.

## Recherche externe

| Route | Controller | Notes |
|---|---|---|
| `GET /v1/search` | `search/search.controller.ts` | `type` obligatoire ; `q` XOR `barcode` ; routé vers l'adapter du type (Discogs pour vinyl, MAL/BnF pour manga) via `SOURCE_ADAPTERS`. Jeton résolu par `TokenResolverService` : BYOT → repli premium → mode dégradé → `403 source-token-required` — voir [../transverse/sources-externes.md](../transverse/sources-externes.md#résolution-des-jetons-byot--repli-premium--mode-dégradé). **Pagination** : `cursor` opaque (`limit` 1-100, défaut 50) ; côté Discogs c'est le **numéro de page** (`meta.pagination.nextCursor = "2"`, absent en dernière page), un curseur illisible retombe page 1. La page fait partie de la clé de cache. **Désambiguïsation** (recherche texte) : chaque résultat porte `metadata.catno`, `metadata.barcodes` (digits, dédupliqués), `metadata.masterId` (commun à tous les pressages d'un même album → permet de regrouper) en plus de l'année, du label, du pays, du format et de la jaquette — cf. [recherche-textuelle-discogs.md](recherche-textuelle-discogs.md) |
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

## Partage de collections — **S3 (modèle + CRUD) · S4 (lecture partagée) · S5 (multi-collections, statuts, libellés)**

> Le partage n'ouvre **aucune route de lecture** : il étend l'autorisation des routes
> `collections`/`items`/`nodes` ci-dessus. Un membre lit donc la collection partagée par les mêmes
> URLs, avec les mêmes DTOs et la même pagination que le propriétaire — seuls le contenu retenu et
> la part visible de `userData` changent (cf. le tableau des statuts ci-dessous).
> Modèle et décisions : [partage-de-collection.md](partage-de-collection.md).

> **Rupture S5** : un partage porte **N collections**, chacune avec son jeu de statuts. Les routes
> sont ancrées sur `/v1/shares` — `POST`/`GET /v1/collections/:id/shares` **n'existent plus**, et
> `scope` a disparu du contrat.

| Route | Controller | Notes |
|---|---|---|
| `POST /v1/shares` | `sharing/sharing.controller.ts` | JWT + `PremiumGuard` (**requérant**). `{ label?, collections: [{ collectionId, statuses: [OWNED\|WISHLIST\|IGNORED] }], maxUses?, expiresAt? }` → `{ id, code, label, collections[], … }`. Code en clair **une seule fois** ; seul son HMAC (`SHARE_CODE_PEPPER`) est stocké. `404` si une collection n'est pas au requérant, sans dire laquelle |
| `GET /v1/shares[?collectionId=]` | idem | Propriétaire, **sans gate premium**. Partages **non révoqués** (les expirés restent listés, `expiresAt` le dit) + collections exposées + membres actifs (`memberUserId` opaque, `redeemedAt`). Le filtre sert un écran ancré sur une collection |
| `PATCH /v1/shares/:id` | idem | Propriétaire. `{ label }` **et rien d'autre** : re-scoper un partage vivant changerait sans le dire ce que ses membres voient |
| `DELETE /v1/shares/:id` | idem | Propriétaire. Révoque le partage **et** tous ses membres (marquage `revokedAt`, jamais de suppression physique). `204`, idempotent |
| `DELETE /v1/shares/:id/members/:memberUserId` | idem | Propriétaire. Éjecte un membre ; le code reste valide pour les autres et ne le fait pas revenir. `204`, idempotent |
| `POST /v1/shares/redeem` | idem | JWT, **pas** de gate premium (c'est le propriétaire qui paie le stockage). `{ code, label? }` → adhésion + collections exposées avec leur `itemCount` visible. Idempotent (`alreadyMember`) ; un `label` fourni est appliqué même sur un redeem répété. Bucket throttle dédié 5/min |
| `GET /v1/shares/received` | idem | Partages que j'ai rejoints et qui vivent encore (`shareId`, **mon** `label`, `redeemedAt`, `collections[]`) |
| `PATCH /v1/shares/received/:shareId` | idem | Membre. `{ label }` — le sien, que le propriétaire ne voit pas |

- **Deux libellés cloisonnés** : celui du propriétaire (« Wantlist manga avec Alice ») et celui du
  membre (« Wantlist manga de Bob ») ne franchissent jamais la frontière — un partage à plusieurs
  membres ne doit pas révéler à l'un comment le propriétaire a nommé l'autre. Ni log, ni métadonnée
  d'audit ; dans l'export RGPD de son auteur seulement. Vide ou blancs seuls ⇒ `null`.
- **Aucun plafond sur le nombre de partages** : `maxUses ≤ 100` borne les membres d'un code, pas le
  nombre de codes. `LimitsService` ne compte que collections, nœuds et items.
- Les entrées sortent **triées par nom de collection** : sans `orderBy`, Prisma les rend dans
  l'ordre de la clé composite, donc par UUID.

- **Aucun oracle d'existence** : code inconnu, révoqué, expiré, épuisé ou membre éjecté renvoient
  tous le même `404` `/probs/share-code-invalid`, même `detail`. Seul le propriétaire qui tente de
  rejoindre son propre partage obtient un `400` distinct — il connaît déjà son code.
- `usedCount` est incrémenté sous **garde optimiste** (`updateMany where usedCount < maxUses`) dans
  la transaction du redeem : deux redeems concurrents sur la dernière place ne passent pas tous deux.
- Audit : `share.create`, `share.revoke`, `share.member.revoke`, `share.redeem` (`target = shareId`).
  `share.create` porte `{ collections: n }` — ni le code, ni le libellé, ni les identifiants de
  collection.

### Ce qu'un membre voit (S4, revu en S5)

| Statuts exposés | Items | Nœuds (séries) |
|---|---|---|
| Les trois | Tous | Tous, y compris un nœud encore vide |
| Contenant `OWNED` | Statut absent/inconnu compris (c'est le défaut) | Ceux qui gardent au moins un tome retenu |
| Contenant `WISHLIST` | Les désirés | + les `isWishlist` **sans aucun tome** |
| Sans `IGNORED` | Les ignorés disparaissent | — |

- Le défaut « statut absent = `OWNED` » vient de `resolveItemStatus` (S1) et n'est pas redéfini :
  `sharing/share-filter.service.ts` en tient le miroir SQL (`CASE WHEN … ELSE 'OWNED' END`, liste et
  défaut importés du module d'origine), et `itemVisibleUnderStatuses` le pendant TS pour les
  lectures unitaires.
- Un membre détenant plusieurs partages actifs de la même collection voit **l'union** des statuts.
- **Hors des statuts exposés = `404`**, jamais `403` : sinon un membre à qui l'on ne partage que
  les possédés déduirait l'existence d'un désiré en balayant des ids.
- **Compteurs recalculés** sous les statuts exposés : `Collection.itemCount`, le résumé `hierarchy`,
  `ownedCount` d'un nœud, et l'`itemCount` par collection de `GET /v1/shares/received`. La colonne dénormalisée
  compterait la collection entière et afficherait « 240 éléments » sur une vue qui en montre 12.
- **`userData` réduit** à une liste blanche pour un membre : passent `status`, `rating`,
  `lastPlayedAt`, `playCount` (item) et `note` (nœud, appréciation 0-10) ; ne passent ni le texte libre
  (`note` d'item, `comment` de nœud) ni le financier (`purchasePrice`). Liste blanche et non noire :
  un champ ajouté plus tard reste invisible tant qu'il n'a pas été jugé partageable.
- `sources[]` (y compris `rawData`) sortent tels quels : ce sont des fiches de catalogue public
  (BnF, Google Books, MangaDex, Discogs, MAL, TMDB), les adapters ne récupèrent rien qui relève d'un
  compte.
- `GET /v1/collections` ne liste **que les siennes** : une collection reçue se lit par
  `GET /v1/shares/received`, pour ne pas mélanger « mes collections » et celles d'autrui dans les
  compteurs, les plafonds et les écrans.
