# acervatim-backend — Partage de collection entre membres (spec)

> Remonter : [INDEX.md](INDEX.md) · Statut : **modèle et CRUD livrés** (S3, 2026-08-21) ; **lecture partagée à faire** (S4)
> Fonctionnalité **Premium** (cf. [../../sprints-conformite-offre.md](../../sprints-conformite-offre.md) — Sprint C4bis).
> Décisions produit : partage **par collection** (pas par compte entier), accès **lecture seule**
> pour le membre (pas d'édition).

## Objectif

Un utilisateur premium partage une de ses collections avec d'autres membres (usage typique :
foyer), au choix :
- **Tout** (`ALL`) — miroir complet de la collection.
- **Uniquement les possédés** (`OWNED`) — masque ce qui est en wishlist/ignoré.
- **Uniquement la wantlist** (`WANTLIST`) — n'expose que ce qui est désiré.

## Prérequis : faire remonter le statut item côté serveur — **livré**

**Décidé le 2026-08-20, implémenté des deux côtés** (sprints S1 backend et A1 app). Avant ça, le
statut possédé/wishlist/ignoré par item (`ItemStatus.OWNED/WISHLIST/IGNORED`, app
`domain/model/ItemStatus.kt`) était **délibérément exclu** de la synchronisation, et côté serveur
seul `CollectionNode.isWishlist` existait — au niveau **série** manga uniquement, absent pour le
vinyle. `OWNED`/`WANTLIST` n'auraient donc pu filtrer que des séries manga, pas des items.

Ce qui est en place :

1. **Backend** — `ItemUserDataSchema` (`item-user-data.schema.ts`) porte
   `status: z.enum(ITEM_STATUSES).nullable()`. Le `.strict().partial()` existant couvre le PATCH
   partiel ; aucune migration (colonne JSON existante). Le défaut est **écrit dans le code** :
   `DEFAULT_ITEM_STATUS` + `resolveItemStatus(userData)`, sur lesquels s'appuie le `where` du
   filtrage par scope.
2. **App** — le canal `userData` des items, jusque-là entièrement absent, est ouvert :
   `ItemDto`/`CreateItemRequest`/`PatchItemRequest` et `RemoteItem` le transportent,
   `RemoteDtoMapper` le mappe dans les deux sens, `FacadeMapper.recordToRemote()` y écrit
   `status`, et le push passe par `toPatchRequest()`.
   - **Seul `status` y transite.** `rating`/`purchasePrice`/`lastPlayedAt`/`note` ont un
     équivalent backend mais restent app-only : le PATCH étant un merge partiel, envoyer une clé
     non modélisée par l'app écraserait la valeur distante. Migration prévue aux sprints S1bis/A1bis.
   - **`status` reste malgré tout écrit dans `toLocalExtras()`**, contrairement au plan initial.
     Ce holder n'est pas qu'un fourre-tout app-only : c'est aussi le cliché du local que
     `SyncManagerImpl` réinjecte au pull (`preservedExtras`, reconstruit depuis le `Record` en
     base). L'en retirer ferait retomber sur `OWNED` tout item synchronisé avant l'ouverture du
     canal — le serveur ignore son statut, le local non. La lecture applique donc
     `userData.status` → repli sur le cliché local → `OWNED`.
3. Les items existants côté serveur ont `userData.status` absent — traités comme `OWNED` par le
   filtre (comportement voulu, pas une migration à écrire : la majorité des items d'une collection
   sont possédés). Ils se re-poussent naturellement au push suivant.

## Où en est le chantier

| Brique | État |
|---|---|
| Statut d'item synchronisé (`userData.status`) | ✅ livré (S1/A1) |
| Gate premium sur la sync, owner-aware en lecture | ✅ livré (S2/A2) |
| Modèle `CollectionShare` + CRUD des partages | ✅ livré (S3) — module `src/sharing/`, six routes, audit, export RGPD |
| Lecture partagée + filtrage par `scope` | ⬜ à faire (S4) — `resolveAccess` ne connaît encore que le rôle `owner` |

Conséquence de cet état : un membre peut rejoindre un partage, il apparaît côté propriétaire, mais
**il ne voit rien de la collection** tant que S4 n'a pas étendu l'autorisation de lecture.

## Modèle de données

```
Collection ── CollectionShare ──┬── CollectionShareMember (1 par membre ayant redeem)
  (propriétaire)                │        FK memberUserId → User (cascade)
                                 └── scope: ALL | OWNED | WANTLIST
```

| Modèle | Champs clés | Notes |
|---|---|---|
| `CollectionShare` | `id`, `collectionId` (FK `Collection`, cascade), `ownerUserId` (dénormalisé), `codeHash` (HMAC-SHA-256, pepper **dédié** `SHARE_CODE_PEPPER`), `scope` (`all\|owned\|wantlist`), `maxUses`, `usedCount`, `expiresAt?`, `revokedAt?`, `createdAt` | Même patron que `Invitation` : le code n'est retourné en clair qu'à la création, seul le hash est stocké |
| `CollectionShareMember` | `shareId` (FK, cascade), `memberUserId` (FK `User`, cascade), `redeemedAt`, `revokedAt?` | PK `[shareId, memberUserId]` : un membre ne rejoint qu'une fois (même logique que `InvitationRedemption`) |

**Pourquoi un code et pas une recherche par identifiant** : les comptes sont anonymes
(`googleSubHash`, aucun email/nom — [securite-et-privacy.md](securite-et-privacy.md)). Chercher
« un membre » par identifiant est structurellement impossible et resterait impossible même avec
cette fonctionnalité — le code voyage hors-bande, comme les invitations premium.

**Pourquoi un pepper dédié** (`SHARE_CODE_PEPPER` ≠ `INVITE_CODE_PEPPER`) : les codes de partage
sont générés par n'importe quel utilisateur (pas seulement l'admin) — surface et cadence de
rotation différentes de celles des invitations.

**Cascades** : suppression de compte du propriétaire → `Collection` cascade → `CollectionShare`/
`CollectionShareMember` cascadent (comportement Prisma standard déjà en place depuis `User`).
Suppression de compte d'un membre → ses `CollectionShareMember` cascadent via `User`.

## Endpoints

| Route | Qui | Effet |
|---|---|---|
| `POST /v1/collections/:id/shares` | Propriétaire premium | `{ scope, maxUses?, expiresAt? }` → `{ id, code, scope }` (code en clair une seule fois, comme `POST /v1/admin/invitations`) |
| `GET /v1/collections/:id/shares` | Propriétaire | Liste des partages actifs + membres (`memberUserId` opaque, `redeemedAt`) |
| `DELETE /v1/shares/:id` | Propriétaire | Révoque le partage entier (code + tous les membres) |
| `DELETE /v1/shares/:id/members/:memberUserId` | Propriétaire | Révoque un seul membre |
| `POST /v1/shares/redeem` | Membre (JWT) | `{ code }` → crée `CollectionShareMember`, retourne un résumé de la collection |
| `GET /v1/shares/received` | Membre | Collections partagées avec moi (`shareId`, `scope`, `redeemedAt`, `collectionId`) |

Livré tel quel en S3 (`src/sharing/sharing.controller.ts`), avec trois choix d'implémentation qui
n'étaient pas dans la spec initiale :

- **Aucun oracle d'existence** : code inconnu, révoqué, expiré, épuisé, ou membre déjà éjecté →
  un seul `404` `/probs/share-code-invalid`, même `detail`. Un statut distinct par état *serait*
  l'oracle qu'on refuse. Seul le propriétaire qui tente de rejoindre son propre partage obtient un
  `400` explicite — il connaît déjà son code.
- **Redeem idempotent** (`alreadyMember: true`) : re-jouer ne consomme pas de place. L'incrément de
  `usedCount` est gardé par un `updateMany where usedCount < maxUses` dans la transaction, comme
  pour les invitations.
- **Révocation = marquage** (`revokedAt`), jamais de suppression : révoquer un partage marque aussi
  ses membres, et un membre éjecté ne revient pas avec le même code — sinon l'éjection ne servirait
  à rien tant que le partage vit.

Pas de routes de lecture dédiées : **étendre l'autorisation existante** de
`collections/items/nodes`. Aujourd'hui l'accès est implicitement `userId = @CurrentUserId()` ; le
service de résolution devient `resolveAccess(userId, collectionId) → { role: 'owner' } |
{ role: 'shared', scope }`. Lecture autorisée pour `owner` et `shared` (le `scope` est injecté
serveur-side dans le `where`, **jamais fourni par le client** — un membre ne peut pas demander
`ALL` sur un partage `OWNED`). Toute mutation (`PATCH`/`POST`/`DELETE`) reste réservée à `owner` :
la lecture seule est garantie au niveau guard, pas seulement côté UI app.

**Interaction avec le gate Premium (Sprint C3, cf. [../../sprints-conformite-offre.md](../../sprints-conformite-offre.md))** :
la sync `collections/items/nodes` devient premium-only, mais côté **propriétaire** — un membre
non-premium doit pouvoir continuer à lire une collection partagée tant que le propriétaire est
premium (sinon un partage serait inutilisable par des membres gratuits, ce qui contredit l'objectif
de la fonctionnalité). `resolveAccess` doit donc checker le statut premium du **propriétaire de la
collection**, pas celui du requérant, avant de trancher `owner`/`shared`/refusé. Si le propriétaire
perd son statut premium, les partages qu'il a émis devraient être suspendus (401/402) plutôt que
supprimés — décision d'implémentation à confirmer en C0.

## Filtrage par `scope`

Réutiliser/étendre le pattern déjà en place pour `matchingCollectionIds`
(`collections.service.ts`, filtre JSON `COLLATE utf8mb4_general_ci`) sur `userData->status` :

| `scope` | Items retournés | Nœuds manga retournés |
|---|---|---|
| `ALL` | Tous | Tous |
| `OWNED` | `status IS NULL OR status = 'OWNED'` | Nœuds ayant ≥ 1 item retenu |
| `WANTLIST` | `status = 'WISHLIST'` | + nœuds `isWishlist = true` même sans item (série désirée sans tome) |

`IGNORED` est exclu de `OWNED` et de `WANTLIST` (visible seulement en `ALL`).

## Sécurité, vie privée, plafonds

- **Rate-limit** sur `POST /v1/shares/redeem` (par IP/JWT) pour empêcher le brute-force de code —
  même besoin que pour `POST /v1/invitations/redeem`.
- **Plafond technique** (`common/limits/limits.service.ts`, ex-`common/quota/` requalifié en S2) :
  une collection reçue en partage ne doit **pas** compter dans le plafond du membre, seules les
  collections dont il est propriétaire comptent.
- **Audit log** : `share.create` / `share.revoke` / `share.redeem`, `target = shareId`, conforme à
  l'invariant « aucune donnée perso » ([securite-et-privacy.md](securite-et-privacy.md)).
- **Export RGPD** (`GET /v1/me/export`) : inclure les deux sens — partages créés et partages
  rejoints.
- **Gate Premium** : `POST /v1/collections/:id/shares` exige `PremiumGuard` côté propriétaire
  (créer un partage nécessite la sync cloud, donc le premium). `POST /v1/shares/redeem` reste
  accessible à tout compte connecté non-premium — sinon un partage serait inutilisable par des
  membres non premium.

## Ouvert / non tranché

- Anti-abus technique sur le nombre de partages/membres par collection (pas de limite produit
  décidée — à trancher si besoin apparaît en usage réel).
- Notification du propriétaire quand un membre rejoint (hors périmètre v1, l'endpoint `GET
  /v1/collections/:id/shares` suffit en pull).
