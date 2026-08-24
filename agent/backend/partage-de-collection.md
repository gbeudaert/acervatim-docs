# acervatim-backend — Partage de collection entre membres (spec)

> Remonter : [INDEX.md](INDEX.md) · Statut : **chantier backend terminé** — modèle et CRUD (S3, 2026-08-21),
> lecture partagée (S4, 2026-08-22), partage multi-collections à statuts et libellés (S5, 2026-08-22).
> Côté app : **A3bis** (UI propriétaire refondue sur ce contrat) et **A4** (UI membre) sont livrés
> le 2026-08-22 — le chantier est complet des deux côtés.
> Fonctionnalité **Premium** (cf. [../../sprints-conformite-offre.md](../../sprints-conformite-offre.md) — Sprint C4bis).
> Décisions produit : partage **par collection** (pas par compte entier), accès **lecture seule**
> pour le membre (pas d'édition).

## Objectif

Un utilisateur premium émet un **code de partage** qui expose, en lecture seule, **une ou plusieurs
de ses collections**, chacune restreinte aux **statuts d'item** de son choix
(`OWNED` / `WISHLIST` / `IGNORED`).

Les trois portées historiques (`ALL` / `OWNED` / `WANTLIST`) ne sont plus un vocabulaire du serveur
depuis S5 : ce ne sont que des raccourcis d'interface au-dessus d'un jeu de statuts —
`ALL` = les trois, `OWNED` = `['OWNED']`, `WANTLIST` = `['WISHLIST']`. L'ensemble libre rend
exprimables les combinaisons qu'elles interdisaient, à commencer par « possédés **et** désirés,
sans les ignorés ».

Un utilisateur crée **autant de partages qu'il veut** : aucun plafond, ni produit ni technique.
`LimitsService` ne compte que les collections (500), les nœuds (20 000) et les items (100 000) ; le `maxUses ≤ 100` d'un
code borne le nombre de membres de **ce** code, pas le nombre de codes.

Chaque partage porte deux libellés libres et **indépendants** : celui du propriétaire (« Wantlist
manga avec Alice ») et celui du membre (« Wantlist manga de Bob »). Aucun des deux ne franchit la
frontière — voir *Libellés*.

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
| Modèle `CollectionShare` + CRUD des partages | ✅ livré (S3) — module `src/sharing/`, audit, export RGPD |
| Lecture partagée + filtrage | ✅ livré (S4) — `resolveAccess` rend `owner \| shared`, lecture seule garantie au guard |
| Partage multi-collections, statuts au choix, libellés des deux côtés | ✅ livré (S5) — `CollectionShareEntry`, routes ancrées sur `/v1/shares` |

Le chantier est utilisable de bout en bout côté serveur : un propriétaire premium émet un code
portant les collections et les statuts qu'il choisit, le membre le consomme et **lit effectivement**
ce qui lui est exposé.

> **Rupture d'API en S5.** `POST`/`GET /v1/collections/:id/shares` n'existent plus, et `scope` a
> disparu du contrat. L'UI propriétaire de **A3** parlait à ces routes mortes ; **A3bis** l'a refondue
> le 2026-08-22, et **A4** a livré la vue membre. Côté base, la migration `purge_pre_s5_shares`
> supprime les partages émis sous l'ancien modèle — décision de rupture nette, la fonctionnalité
> n'ayant pas encore été mise entre les mains d'utilisateurs.
>
> **Ce que l'app en a fait** : les portées `all`/`owned`/`wantlist` ne survivent côté client que
> comme raccourcis d'*affichage* (« Toute la collection » se lit mieux que trois statuts énumérés) ;
> elles ne contraignent plus le choix. Et pour **effacer** un libellé, l'app envoie la chaîne vide
> plutôt que `null` — son `Json` est en `explicitNulls = false`, un `null` ne serait pas sérialisé et
> le schéma strict refuserait le corps vide. Le `preprocess` de `shareLabel()` la normalise.

## Modèle de données

```
User ── CollectionShare ──┬── CollectionShareEntry (1 par collection exposée)
 (propriétaire)           │        FK collectionId → Collection (cascade)
                          │        statuses: ['OWNED','WISHLIST',…]
                          └── CollectionShareMember (1 par membre ayant redeem)
                                   FK memberUserId → User (cascade)
```

| Modèle | Champs clés | Notes |
|---|---|---|
| `CollectionShare` | `id`, `ownerUserId` (FK `User`, cascade), `codeHash` (HMAC-SHA-256, pepper **dédié** `SHARE_CODE_PEPPER`), `label?` (200), `maxUses`, `usedCount`, `expiresAt?`, `revokedAt?`, `createdAt` | Même patron que `Invitation` : le code n'est retourné en clair qu'à la création, seul le hash est stocké |
| `CollectionShareEntry` | `shareId` (FK, cascade), `collectionId` (FK `Collection`, cascade), `statuses` (JSON, non vide) | PK `[shareId, collectionId]` : une collection n'apparaît qu'une fois par partage |
| `CollectionShareMember` | `shareId` (FK, cascade), `memberUserId` (FK `User`, cascade), `label?` (200), `redeemedAt`, `revokedAt?` | PK `[shareId, memberUserId]` : un membre ne rejoint qu'une fois (même logique que `InvitationRedemption`) |

**Pourquoi une FK `ownerUserId → User`** (nouvelle en S5) : avant, la cascade de suppression de
compte passait par `collection_id`. Un partage n'appartenant plus à une collection, sans cette FK
la suppression d'un compte laisserait ses partages orphelins.

**Pourquoi un code et pas une recherche par identifiant** : les comptes sont anonymes
(`googleSubHash`, aucun email/nom — [securite-et-privacy.md](securite-et-privacy.md)). Chercher
« un membre » par identifiant est structurellement impossible et resterait impossible même avec
cette fonctionnalité — le code voyage hors-bande, comme les invitations premium.

**Pourquoi un pepper dédié** (`SHARE_CODE_PEPPER` ≠ `INVITE_CODE_PEPPER`) : les codes de partage
sont générés par n'importe quel utilisateur (pas seulement l'admin) — surface et cadence de
rotation différentes de celles des invitations.

**Cascades** : suppression de compte du propriétaire → ses `CollectionShare` cascadent (FK directe),
et avec eux entrées et membres. Suppression d'**une** collection → seules ses `CollectionShareEntry`
tombent : un partage qui portait aussi d'autres collections survit, amputé de celle-là. Suppression
de compte d'un membre → ses `CollectionShareMember` cascadent via `User`.

## Endpoints

Toutes les routes sont ancrées sur `/v1/shares` : un partage porte N collections, il n'appartient
donc à aucune. La collection n'est plus qu'un **filtre de liste**.

| Route | Qui | Effet |
|---|---|---|
| `POST /v1/shares` | Propriétaire premium | `{ label?, collections: [{ collectionId, statuses[] }], maxUses?, expiresAt? }` → `{ id, code, label, collections[], … }` (code en clair une seule fois, comme `POST /v1/admin/invitations`) |
| `GET /v1/shares[?collectionId=]` | Propriétaire | Partages actifs qu'il a émis : collections exposées + statuts + membres (`memberUserId` opaque, `redeemedAt`). Le filtre sert un écran ancré sur une collection |
| `PATCH /v1/shares/:id` | Propriétaire | `{ label }` — **le libellé et rien d'autre** |
| `DELETE /v1/shares/:id` | Propriétaire | Révoque le partage entier (code + tous les membres) |
| `DELETE /v1/shares/:id/members/:memberUserId` | Propriétaire | Révoque un seul membre |
| `POST /v1/shares/redeem` | Membre (JWT) | `{ code, label? }` → crée `CollectionShareMember`, retourne les collections exposées avec leur `itemCount` visible |
| `GET /v1/shares/received` | Membre | Partages rejoints (`shareId`, son `label`, `redeemedAt`, `collections[]`) |
| `PATCH /v1/shares/received/:shareId` | Membre | `{ label }` — son libellé à lui |

**Pourquoi le libellé est le seul champ modifiable** : re-scoper un partage vivant (changer ses
collections ou ses statuts) modifierait sans le dire ce que ses membres voient déjà. Un changement
de portée passe par une révocation et un nouveau code — visible des deux côtés.

**Ordre des entrées** : trié par nom de collection. Sans `orderBy`, Prisma les rend dans l'ordre de
la clé composite, c'est-à-dire par UUID — arbitraire, et différent d'un partage à l'autre.

**`GET`, `PATCH` et `DELETE` sont hors gate premium** : un propriétaire dont le premium a expiré doit
pouvoir constater et retirer ce qu'il a partagé. Seule l'émission (`POST`) est payante.

## Libellés

Deux champs libres, symétriques et **cloisonnés** : `CollectionShare.label` appartient au
propriétaire, `CollectionShareMember.label` au membre. Aucun des deux n'est transmis à l'autre
partie — un partage à plusieurs membres ne doit pas révéler à Bob que le propriétaire l'a nommé
« … avec Alice », et le propriétaire n'a pas à savoir comment Bob le range chez lui.

C'est le premier champ dont l'**objet** est de nommer une personne. Il ne s'agit pas pour autant
d'un précédent en base : du texte libre saisi par l'utilisateur y existe déjà (`Collection.name`,
`userData.note` d'item, `comment` de nœud). L'invariant privacy porte sur l'identité — aucun email,
aucun nom, aucun `sub` Google — et il reste tenu : un libellé est du contenu utilisateur, pas un
identifiant, et il ne réidentifie personne côté serveur.

Les garde-fous appliqués :

- **Jamais journalisé** — les lignes `share.*` ne portent que des préfixes d'id et un décompte.
- **Jamais en métadonnée d'audit** — `share.create` enregistre `{ collections: n }`, ni le libellé
  ni les identifiants de collection.
- **Jamais vers l'autre partie** — vérifié par des tests e2e des deux sens.
- **Dans l'export RGPD de son auteur**, et de lui seul : c'est sa donnée.
- Chaîne vide ou blancs seuls ⇒ `null` : « pas de libellé » a une seule représentation.

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

Pas de routes de lecture dédiées : S4 a **étendu l'autorisation existante** de
`collections/items/nodes`. L'accès n'est plus implicitement `userId = @CurrentUserId()` : il passe
par `CollectionAccessService.resolveAccess(userId, collectionId)`, qui rend `{ role: 'owner' }` ou
`{ role: 'shared', statuses }`, ou `null` (→ `404`). Le résultat est posé sur la requête par le
guard et lu par le service via `@CurrentCollectionAccess` : une seule résolution par requête, et
**aucun chemin par lequel les statuts pourraient venir du client**. Un membre ne peut pas s'accorder
un statut de plus — les query params sont validés en `.strict()`, tout paramètre inventé est un
`400`, et rien d'autre n'est lu.

Ce sont les **entrées** qui portent les statuts, pas le partage : `resolveAccess` cherche les
`CollectionShareEntry` de cette collection parmi les partages vivants que le requérant a rejoints.

Sur la résolution elle-même :

- `expiresAt` n'entre **pas** dans le filtre : il borne l'usage du *code*, pas l'adhésion déjà
  acquise (invariant posé en S3, écrit dans le modèle Prisma et appliqué par `listReceived`). Un
  partage expiré cesse d'être rejoignable ; ceux qui l'ont rejoint lisent jusqu'à révocation.
  L'inverse ferait mentir `GET /v1/shares/received`, qui les liste encore.
- Un membre détenant **plusieurs partages actifs** de la même collection voit **l'union** des statuts
  que chacun lui accorde — ni plus, ni moins. C'est ce que le modèle en ensembles rend enfin
  exprimable : avec les portées de S4, `owned` et `wantlist` n'étant pas comparables, il fallait en
  choisir une arbitrairement.
- Une entrée dont les `statuses` seraient illisibles (ligne abîmée) donne l'ensemble vide, et
  l'ensemble vide vaut **pas d'accès** — pas un accès qui ne montre rien.

**Écriture** : `collection-write.guard.ts` remplace `PremiumGuard` sur toute mutation visant une
collection identifiable. Un membre y reçoit `403` — et non `404` comme un inconnu : il *voit* déjà
la ressource, un 404 mentirait sur ce qu'il lit. La lecture seule est donc garantie **au niveau
guard**, avant tout service, indépendamment de ce que l'UI de l'app affiche. Les routes de *gestion*
des partages (`POST`/`GET /v1/shares`) ne rendent jamais qu'un partage dont on est propriétaire :
un membre n'a pas à connaître cette surface.

**Interaction avec le gate Premium** : la sync `collections/items/nodes` est premium-only, mais côté
**propriétaire** — un membre non-premium lit une collection partagée tant que le propriétaire est
premium (sinon un partage serait inutilisable par des membres gratuits, ce qui contredit l'objectif
de la fonctionnalité). `CollectionPremiumGuard` vérifie donc le statut premium de
`access.ownerUserId`. **Tranché en S4** : un propriétaire qui perd son premium **suspend** les
partages qu'il a émis (`402` pour lui comme pour ses membres) ; rien n'est supprimé, et rendre le
premium rend l'accès.

## Filtrage par statuts

`sharing/share-filter.service.ts` traduit un jeu de statuts en `where` Prisma :

| Statuts exposés | Items retournés | Nœuds manga retournés |
|---|---|---|
| Les trois | Tous | Tous, y compris un nœud encore vide |
| Contenant `OWNED` | Statut absent/inconnu compris (c'est le défaut) | Nœuds gardant ≥ 1 item retenu |
| Contenant `WISHLIST` | Les désirés | + nœuds `isWishlist = true` **même sans item** (série désirée sans tome) |
| Sans `IGNORED` | Les ignorés disparaissent | — |

**Le défaut « statut absent = `OWNED` » n'est pas redéfini ici** : il appartient à
`resolveItemStatus` (S1), et le SQL le reproduit littéralement
(`CASE WHEN status IN (…) THEN status ELSE 'OWNED' END`), en tirant la liste et le défaut du module
d'origine plutôt qu'en les recopiant. Deux formes doivent dire la même chose — le prédicat SQL pour
les listes, `itemVisibleUnderStatuses` pour les lectures unitaires (sur un item déjà chargé, un
aller-retour SQL pour trancher sa visibilité ne se justifie pas). Un test unitaire couvre la seconde
sur les cas de bord (clé absente, `null` JSON, casse différente, valeur inconnue) ; les e2e couvrent
l'accord des deux.

> La visibilité d'un **nœud** se décide elle aussi à deux endroits — le `where` des listes et le
> prédicat des lectures unitaires. S5 y a introduit une régression (un nœud vide non désiré devenait
> `404` **pour son propre propriétaire**), attrapée par les e2e items ; les deux formes sont
> désormais couvertes des deux côtés.

**Forme du filtre** : un ensemble d'ids calculé en SQL brut, injecté en `id in/notIn` — même patron
que `matchingCollectionIds` et `itemIdsByProvider`. Un filtre JSON ne s'exprime pas de façon fiable
en Prisma sur MariaDB (la clé *absente* casse toute négation). On calcule systématiquement
l'ensemble **minoritaire** attendu — les items masqués quand le statut par défaut est exposé (cas
courant, la majorité d'une collection), les items visibles sinon — pour que la liste reste courte.
Limite assumée et non mesurée : sur une collection où cet ensemble est en fait volumineux, la liste
d'ids grossit, et le filtre JSON n'est pas indexé. La
sortie, le jour où ça se mesure, est une colonne générée indexée sur `user_data->'$.status'`.

**Compteurs** : `Collection.itemCount` est dénormalisé sur la collection entière, le résumé
`hierarchy` compte tous les nœuds, `ownedCount` tous les tomes d'une série, et
`GET /v1/shares/received` sert `itemCount` depuis la même colonne. Servis tels quels à un membre,
ils annonceraient « 240 éléments » et « 30 séries » sur une vue qui en montre 12 et 3. **Les quatre
sont recalculés sous les statuts exposés** — le compte reste utile au membre, il doit juste être
celui de ce qu'il voit. Le propriétaire, lui, n'a pas ce compte dans `GET /v1/shares` : sur un écran
de gestion ce qui compte est *quelles* collections et *quels* statuts, et le calculer coûterait une
requête par entrée sur une liste de partages non plafonnée.

**`userData` du propriétaire** : réduit à une **liste blanche** pour un membre
(`sharing/shared-user-data.ts`). Passent l'appréciation et le fait de collection — `status`
(indispensable au filtrage), `rating`, `lastPlayedAt`, `playCount` pour un item, `note` (0-10) pour
un nœud. Ne
passent ni le texte libre (`note` d'item, `comment` de nœud, qui peuvent contenir n'importe quoi de
personnel) ni le financier (`purchasePrice`). Liste blanche et non liste noire : les champs que
S1bis fera remonter resteront invisibles tant qu'ils n'auront pas été jugés partageables, si bien
que le mode de défaillance est « un champ manque dans une vue partagée », jamais « un champ privé a
fuité ». Le masquage dépend du **rôle**, pas des statuts : un partage qui expose tout masque autant
qu'un partage restreint.

**`sources[]`** sortent tels quels, `rawData` compris — vérifié plutôt que supposé : les adapters
(BnF, Google Books, MangaDex, Discogs, MAL, TMDB) ne récupèrent que des fiches d'œuvres
(`fetchDetails(id)`), jamais de donnée de compte.

## Sécurité, vie privée, plafonds

- **Rate-limit** sur `POST /v1/shares/redeem` (par IP/JWT) pour empêcher le brute-force de code —
  même besoin que pour `POST /v1/invitations/redeem`.
- **Plafond technique** (`common/limits/limits.service.ts`, ex-`common/quota/` requalifié en S2) :
  une collection reçue en partage ne doit **pas** compter dans le plafond du membre, seules les
  collections dont il est propriétaire comptent.
- **Audit log** : `share.create` / `share.revoke` / `share.member.revoke` / `share.redeem`,
  `target = shareId`. Aucune métadonnée nominative : `share.create` porte `{ collections: n }`, ni
  le libellé ni les identifiants de collection ([securite-et-privacy.md](securite-et-privacy.md)).
- **Export RGPD** (`GET /v1/me/export`) : les deux sens — partages créés (entrées + libellé du
  propriétaire) et partages rejoints (entrées + libellé du membre), jamais le `codeHash`, jamais le
  libellé de l'autre partie.
- **Gate Premium** : `POST /v1/shares` exige `PremiumGuard` côté propriétaire (créer un partage
  nécessite la sync cloud, donc le premium). `POST /v1/shares/redeem` reste accessible à tout compte
  connecté non-premium — sinon un partage serait inutilisable par des membres non premium. Lister,
  renommer et révoquer restent accessibles sans premium.
- **Lecture seule au guard** : `CollectionWriteGuard` refuse toute mutation d'un membre en `403`,
  avant d'atteindre le service. Les services gardent par ailleurs leur `where` scopé sur le
  propriétaire — défense en profondeur, un guard oublié sur une route future ne suffit pas à ouvrir
  l'écriture.
- **Pas d'oracle par balayage d'ids** : un item ou un nœud hors des statuts exposés répond `404`,
  comme un id inexistant. Un membre à qui l'on ne partage que les possédés ne peut pas déduire
  l'existence d'un désiré. Même logique à la création d'un partage : si l'une des collections
  demandées ne lui appartient pas, le `404` ne dit pas **laquelle**.

## Ouvert / non tranché

- ~~Anti-abus sur le nombre de partages~~ — **tranché en S5** : aucun plafond, ni produit ni
  technique. `LimitsService` ne compte que collections, nœuds et items ; `maxUses ≤ 100` borne les membres
  d'un code, pas le nombre de codes. À rouvrir si un usage réel le demande.
- Notification du propriétaire quand un membre rejoint (hors périmètre v1, `GET /v1/shares` suffit
  en pull).
- Coût du filtre JSON sur une collection volumineuse : non mesuré. La sortie est une colonne
  générée indexée, pas un contournement dans `share-filter.service.ts` (cf. *Filtrage par statuts*).
- Modifier les collections ou les statuts d'un partage vivant : volontairement impossible
  (révocation + nouveau code). À rouvrir si l'usage montre que la révocation est trop brutale.
