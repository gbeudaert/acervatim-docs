# Cible — Découpage micro-services (un service par type de collection + partages)

> Remonter : [INDEX.md](INDEX.md) · **Cible d'architecture engagée le 2026-11-16** — document de
> travail, pas une page du site public.
> Dernière mise à jour : 2026-11-16.
>
> **Décision du 2026-11-16 : bascule complète de la production vers la constellation**, sans arrêt
> à un palier intermédiaire. Le monolithe `acervatim-backend` (déployé le 2026-08-24) reste
> ce qui tourne jusqu'à la bascule ; l'implémentation de cette cible est dans le dépôt
> `acervatim/backend-services`, pas encore déployée. La séparation des comptes SQL par service est
> reportée : un seul compte, une seule instance MariaDB, un schéma par service, à revoir selon
> l'évolution de la charge. Ce document décrit la cible, pas l'état de ce dépôt.

## TL;DR

Découper `acervatim-backend` en **8 services** : un **catalog par type de collection**
(`catalog-manga`, `catalog-vinyl`, `catalog-generic` pour movie/book/game), un service
**`sharing`**, et les transverses **`search`**, **`connections`**, **`identity`**, **`billing`**,
le tout derrière un **`gateway`** qui conserve le contrat `/v1/*` — l'app Android n'est pas touchée.
Une **seule instance MariaDB**, un **schéma logique par service**, **aucune FK inter-schémas**.

Le chantier n'est pas le découpage : c'est la **rupture de sept coutures** (§3) que le monolithe
tient aujourd'hui gratuitement — au premier rang desquelles le couplage bidirectionnel
`sharing ↔ collections/items/nodes`, le routage `collectionId → typeCode`, et la cascade RGPD.
Le phasage (§4) place **80 % du risque en M0–M2, sans un seul processus supplémentaire**.

**Réserve à peser avant d'engager quoi que ce soit après M2.** Le domaine tient en ~13 000 LOC
(hors tests, 226 fichiers TS) et 15 modèles Prisma, servis à un unique client Android. Huit
processus Node/Nest, chacun avec son client Prisma, représentent de l'ordre de **1 à 1,4 Go de RSS
au repos** sur l'hôte applicatif, en plus de MariaDB et Redis — pour un volume de données
aujourd'hui négligeable. Le phasage est donc conçu pour être **arrêtable à chaque palier** : M0–M2
livrent l'essentiel du découplage à coût opérationnel nul, et s'arrêter à M5 (6 processus) ne
condamne pas M6. Chaque passage de palier doit être justifié par une mesure (§4), pas par la cible.

**Mesure du 2026-11-14, qui lève cette réserve pour la mémoire.** Les deux solutions ont tourné en
images de production sur un poste de développement (amd64, Docker), avec le même parcours
authentifié. Le monolithe occupe 121 Mo au repos et 130 Mo après le parcours. Les neuf services de
la constellation totalisent 941 à 1 080 Mo de RSS additionné, ce qui confirme l'ordre de grandeur
ci-dessus, mais 481 à 603 Mo de mémoire réelle (PSS) : leurs conteneurs partagent les pages du
binaire Node de l'image de base commune. MariaDB (100 à 107 Mo) et Redis (10 Mo) pèsent autant dans
les deux cas. L'hôte applicatif avait 7,1 Go disponibles le 2026-11-16 : la constellation en
demanderait 0,6 à 1,1 Go de plus. La mesure n'a pas été refaite sur l'hôte ni sous Podman, et la
latence n'a pas été mesurée.

---

## 1. Architecture cible

```
App Android ──HTTPS──> gateway  (/v1/*, JWT, routage, agrégation OpenAPI, throttler)
                          │
                          │  HTTP interne + principal signé X-Acervatim-Principal
     ┌─────────────┬──────┴──────┬─────────────────┬──────────┬──────────┐
 catalog-manga  catalog-vinyl  catalog-generic  sharing    search   connections
     └───────────── identity (auth/users/RGPD/audit) ── billing (premium/subs/invit.)
```

`catalog-*` : `Collection` du type + `CollectionNode` + `Item` + le profil de type.

| Service | Reprend (`src/`) | Schéma SQL | Tables |
|---|---|---|---|
| `gateway` | `main.ts`, `common/guards/jwt-auth.guard.ts`, `common/filters/problem-details.filter.ts`, `ThrottlerModule` | `acv_gateway` | registre de routage (§3.3) |
| `catalog-manga` · `catalog-vinyl` · `catalog-generic` | `collections/`, `items/`, `nodes/` + un profil de `collections/types/` | `acv_cat_<type>` | `collections`, `collection_nodes`, `items` |
| `sharing` | `sharing/` + `premium/collection-access.service.ts` | `acv_sharing` | `collection_shares`, `collection_share_entries`, `collection_share_members` |
| `search` | `search/`, `common/sources/{bnf,googlebooks,mangadex}/`, files BullMQ | `acv_search` | `api_cache` |
| `connections` | `oauth/`, `sources/`, `TokenResolverService` | `acv_conn` | `oauth_credentials` |
| `identity` | `auth/`, `users/`, `common/audit/` | `acv_core` | `users`, `audit_logs`, `collection_types` |
| `billing` | `premium/`, `subscriptions/`, `invitations/` | `acv_billing` | `subscriptions`, `premium_grants`, `invitations`, `invitation_redemptions` |

**Monorepo Nest** dans `acervatim-backend` : `apps/<service>/` + `libs/`. Aucune dépendance directe
entre deux `apps/` — c'est l'invariant qui rend le découpage réel plutôt que cosmétique.

| Lib | Contenu |
|---|---|
| `libs/common` | `common/pagination/`, `common/filters/parse-filters.ts`, Problem Details, `common/validation/bounded-json.ts`, `common/crypto/`, `common/http/`, `common/logging/` |
| `libs/catalog-core` | Le CRUD collections/nodes/items mutualisé par les trois catalogs (§2) |
| `libs/contracts` | Types des routes `/internal/*` + un client HTTP typé par service (§6) |

**Prisma** : un `schema.prisma` par service (`prisma/<service>/schema.prisma`), un client généré par
service, une `DATABASE_URL` par service pointant la **même instance** sur un schéma différent.
**Aucune `@relation` ne franchit un schéma** : c'est l'invariant central, et la source de la
quasi-totalité du travail décrit en §3.

---

## 2. Ce que le découpage par type gagne, et ce qu'il coûte

À dire sans détour : `catalog-manga` et `catalog-vinyl` partagent l'essentiel de leur code. Le CRUD
`collections/`/`items/`/`nodes/` est identique ; ce qui diffère tient dans
`collections/types/manga.ts` vs `collections/types/vinyl.ts` — un `itemSchema`, une
`toListProjection`, une `hierarchy` (vide pour le vinyle), un `reconcileOnUpdate` optionnel. C'est
déjà un point d'extension propre (`collections/types/registry.ts`), et c'est précisément ce qui rend
le découpage par type **possible** : la frontière existe, il s'agit de la faire passer du `Record`
en mémoire au réseau.

| | Gain | Coût |
|---|---|---|
| **Panne** | Un adapter MangaDex qui boucle ne fait plus tomber le vinyle | — |
| **Déploiement** | Faire évoluer le profil manga sans redéployer le reste | 3 pipelines, 3 images |
| **Charge** | Dimensionner par type | 3 × l'empreinte mémoire d'un Nest+Prisma |
| **Routage** | — | **Registre `collectionId → typeCode`** (§3.3), sans lequel rien ne route |
| **Lecture multi-types** | — | **Scatter-gather + cursor composite** (§3.7) |

Les deux lignes du bas sont le vrai prix. Elles n'apparaissent **qu'en M6** : c'est pourquoi M6 est
le seul jalon non réversible du phasage.

---

## 3. Les coutures dures

### 3.1 — `ShareFilterService` est injecté dans le domaine collection

`sharing/share-filter.service.ts` est injecté dans **trois services du domaine collection** —
`collections.service.ts`, `items.service.ts`, `nodes.service.ts` — et y fait du SQL brut qui joint
`collection_share_entries` à `items` pour filtrer par statut. La dépendance est **bidirectionnelle**
(`sharing` lit `items` ; `items` appelle `sharing`) et ne survit pas au découpage.

**Résolution — inversion de la dépendance.** `sharing` expose
`GET /internal/access?userId=&collectionId=` → `{ role: 'owner' | 'shared', statuses[] }` — c'est
déjà littéralement le contrat de `CollectionAccessService.resolveAccess`. Le catalog reçoit les
statuts autorisés et applique le filtre **localement** : le `status` d'un item vit dans
`Item.userData`, donc chez lui. Le `IN`/`NOT IN` SQL de `share-filter` devient un prédicat sur ses
propres tables, et `sharing` n'a plus besoin de connaître `items`.

`CollectionAccessService` (aujourd'hui dans `premium/`, consommé par `collection-premium.guard.ts`,
`collection-write.guard.ts` et `collection-ref.resolve.ts`) déménage dans `sharing`. Son cas
« propriétaire » interroge `GET /internal/collections/:id/owner` du catalog.

**Conséquence sur les guards.** Les deux guards à `@CollectionRef` deviennent des guards **du
gateway** : il résout l'accès une fois, et propage `{ role, statuses }` dans le principal signé
(§3.6). Les services ne re-résolvent rien — ce qui préserve l'invariant actuel, énoncé dans
[api-endpoints.md](api-endpoints.md) : *les statuts n'ont aucun chemin par lequel arriver du client*.

### 3.2 — `itemCount` des partages reçus

`ReceivedEntryView.itemCount` (`sharing/sharing.service.ts`) fait un `COUNT(*)` sur `items` que
`sharing` ne peut plus faire. Le code note déjà que le **propriétaire** n'a pas ce compte, justement
parce qu'il coûterait une requête par entrée et par partage sur une liste non plafonnée : la
remarque devient un argument de conception.

**Résolution.** `POST /internal/items/count` sur le catalog : `[{ collectionId, statuses }]` →
`[{ collectionId, count }]`. Un appel par catalog concerné, en parallèle, jamais un par entrée.
Le compte reste réservé au **membre**, comme aujourd'hui.

### 3.3 — Routage `collectionId → typeCode` : le point structurant

`GET /v1/items/:id`, `PATCH /v1/nodes/:id`, `GET /v1/collections/:id` ne portent **pas** le type.
Le gateway ne peut pas router sans le connaître, et un broadcast à tous les catalogs sur chaque
lecture est exclu.

**Résolution — registre de routage** dans `acv_gateway` :

```
collection_registry(collection_id PK, user_id, type_code)
resource_registry(resource_id PK, kind ENUM('item','node'), collection_id)
```

Écrit **en synchrone** par le catalog à la création et à la suppression, via `PUT|DELETE
/internal/registry` du gateway, idempotent. Reconstructible intégralement par re-scan des catalogs
— **prévoir le job de réparation dès M6**, pas après : un registre divergent produit des `404` sur
des ressources existantes, c'est-à-dire le symptôme le plus difficile à diagnostiquer de toute la
cible.

**Alternative écartée** : préfixer les routes par le type
(`/v1/manga/collections/:id/items`). Supprime le registre, mais casse le contrat V1 et donc l'app —
hors périmètre, la contrainte étant précisément que l'app ne bouge pas.

### 3.4 — RGPD : `DELETE /v1/me` et `GET /v1/me/export`

`users/users.service.ts` s'appuie sur `onDelete: Cascade` de Prisma à travers `collections`,
`collection_nodes`, `items`, `oauth_credentials`, `subscriptions`, `premium_grants`,
`invitation_redemptions`, `collection_shares`, `collection_share_members`. **La cascade disparaît
entre schémas.**

**Résolution — saga orchestrée par `identity`** : table
`user_deletion_jobs(user_id, service, state, attempts, last_error)`, un `DELETE
/internal/users/:id/data` **idempotent** par service, retry avec backoff. L'export devient un
fan-out qui agrège N réponses dans la forme actuelle du payload.

**C'est le point à risque juridique du chantier.** Une saga partiellement échouée laisse des données
d'un compte supprimé. La cible **exige** : un job de réconciliation périodique qui rejoue les
`user_deletion_jobs` non terminés, et une alerte au-delà d'un seuil d'ancienneté — pas seulement un
retry en ligne. Tant que ce mécanisme n'est pas en place et testé, M2 n'est pas terminé.

### 3.5 — Plafonds techniques

`common/limits/limits.service.ts` compte par utilisateur à travers toutes ses collections.

**Résolution — plafond par catalog.** Le plafond est un garde-fou anti-emballement, pas une limite
d'offre : la sémantique survit au découpage, seul le chiffre effectif change (jusqu'à N × plafond
pour un utilisateur qui possède des collections de N types). À arbitrer explicitement au moment de
M6 : soit on assume le nouveau total, soit on divise les plafonds unitaires.

### 3.6 — Statut premium sur le chemin chaud

`PremiumGuard`, `CollectionPremiumGuard` et `CollectionWriteGuard` résolvent le premium **en base à
chaque requête** — y compris celui du **propriétaire** lorsqu'un membre lit une collection partagée.
En micro-services, c'est un aller-retour réseau par requête.

**Résolution — principal signé au gateway.** Le gateway vérifie le JWT une fois, résout le premium
auprès de `billing` (cache Redis, TTL 60 s), résout l'accès auprès de `sharing` (§3.1), et propage
un en-tête `X-Acervatim-Principal` — JSON compact signé HMAC avec un secret interne, portant
`{ userId, premium, role, statuses, exp }`. Les services aval ne vérifient plus que cette signature.

Conséquences à documenter au moment de l'implémentation : le TTL de 60 s introduit une fenêtre
pendant laquelle une perte de premium n'est pas encore effective (acceptable — la perte de premium
*suspend* les partages, elle ne détruit rien), et le secret interne devient un secret de production
au même titre que `SUB_HASH_PEPPER`.

### 3.7 — `GET /v1/collections` multi-types et la pagination cursor

Les filtres `type[in]` et `items[in]`/`items[all]` (`CollectionsService.matchingCollectionIds`, SQL
brut avec `COLLATE utf8mb4_general_ci`) traversent **tous** les types.

**Résolution — scatter-gather au gateway** : appel parallèle aux catalogs du `type[in]` (tous s'il
est absent), merge et tri.

**Le piège est le cursor.** Un cursor opaque unique ne décrit plus une position quand la liste est
la fusion de N listes indépendantes. Retenu : **cursor composite** — un curseur par service, encodés
ensemble et opaques vus du client —, page size borné, ordre stable `(createdAt, id)`. À spécifier
avant d'écrire la première ligne : c'est le point le plus facile à casser **en silence**, une
pagination qui saute ou duplique des lignes ne lève aucune erreur.

Même traitement pour toute future route de lecture inter-types.

### 3.8 — Ce qui ne casse pas

À dire aussi, pour éviter le faux travail :

- `Item.@@unique([userId, nodeId, volume])` **tient tel quel** : `nodeId` n'existe que pour les types
  hiérarchiques et ne franchit jamais une frontière de catalog.
- `Collection.itemCount` (dénormalisé) reste local au catalog, mis à jour dans la même transaction.
- `search` est **déjà** un sous-système autonome : files BullMQ, `api_cache`, adapters. Il ne lit
  aucune table du domaine collection. Son extraction (M4) est quasi sans couture — c'est pourquoi
  elle sert de validation à faible risque du transport interne.
  **Exception depuis le rattrapage manga (SD3, 2026-11)** : le passage hebdomadaire et
  `GET /v1/series-updates` lisent les séries et les tomes possédés. Dans le monolithe, c'est une
  lecture Prisma directe de `collection_nodes`/`items` ; découpé, c'est
  `GET /internal/nodes` et `GET /internal/users/:id/nodes` sur `catalog-manga`, plus
  `GET /internal/premium/:id` sur `billing` (§6). L'état conservé par `search` reste indexé par
  édition, jamais par compte.
- `connections` (oauth) idem : `oauth_credentials` n'est lue que par lui et par `TokenResolverService`.

---

## 4. Phasage

| Jalon | Contenu | Processus | Réversible |
|---|---|---|---|
| **M0** | Monorepo Nest : `apps/api` + `libs/{common,catalog-core,contracts}`. Rien d'autre que la réorganisation des imports croisés. | 1 | oui |
| **M1** | **Couper les coutures en in-process** : ports `AccessResolverPort`, `ItemCounterPort`, `CollectionRegistryPort`, `PremiumPort`, implémentations locales. Le code devient réseau-ready **sans réseau**. | 1 | oui |
| **M2** | **Séparer les schémas SQL** : `acv_core`, `acv_sharing`, `acv_catalog`, `acv_search`, `acv_conn`, `acv_billing`. Suppression des FK inter-schémas, un `schema.prisma` par schéma. **Saga RGPD (§3.4) en place et testée**, toujours en un seul processus. | 1 (N clients Prisma) | coûteux |
| **M3** | Extraire **`sharing`** + **`gateway`**. Premier multi-processus : valide le principal signé, le client HTTP interne, la saga à deux participants. | 2 | oui |
| **M4** | Extraire **`search`** et **`connections`** (transverses, aucun routage par `collectionId`). | 4 | oui |
| **M5** | Extraire **`identity`** et **`billing`**. | 6 | oui |
| **M6** | Éclater le catalog par type : `catalog-manga`, puis `catalog-vinyl`, puis `catalog-generic`. Le **registre de routage** (§3.3) et le **scatter-gather** (§3.7) entrent en service ici. | 8 | **non** |

**M1 est le jalon qui compte.** L'essentiel du risque architectural — l'inversion sharing↔catalog,
la disparition des jointures, la saga RGPD — y tombe pour un coût opérationnel nul. Un projet qui
s'arrêterait à M2 aurait déjà gagné la partie difficile.

**Reprise des données (2026-11-03).** Dans ce phasage, les données suivent les schémas : M2
sépare les tables sur place, M6 déplace les collections vers leur catalog. L'implémentation de
`acervatim/backend-services`, passée directement de M0 à M6, démarre au contraire sur neuf
schémas vides : la bascule exige une reprise ponctuelle du schéma du monolithe, écritures
gelées, avec les mêmes secrets — `acervatim/backend-services/db/reprise/` (procédure, contrôles,
retour arrière ; répétée en développement, jamais sur un vidage de production). Son point de
non-retour est la première écriture sur la constellation : il n'existe pas de reprise en sens
inverse.

**Critères de passage**, à mesurer et non à supposer, avant chaque palier à partir de M3 :

| Critère | Seuil |
|---|---|
| RSS total des processus au repos | doit laisser une marge sur l'hôte applicatif, MariaDB et Redis inclus |
| Latence p95 de `GET /v1/collections` et `GET /v1/collections/:id/items` | pas de régression au-delà d'un budget fixé avant le palier |
| Saga RGPD | 0 job non terminé au-delà du seuil d'alerte, sur une fenêtre d'observation réelle |
| Registre de routage (M6 seulement) | 0 divergence après re-scan complet |

---

## 5. Ce qui change côté app Android

**Rien, par construction.** Le gateway conserve `/v1/*`, les Problem Details RFC 9457, la pagination
cursor et les codes de refus documentés dans [api-endpoints.md](api-endpoints.md) — `404` pour un
non-membre, `403` pour un membre en écriture, `402` pour un propriétaire non premium.
`API_BASE_URL` et `NetworkModule.kt` ne bougent pas.

**Deux points de non-régression à surveiller**, tous deux introduits en M6 et M3 respectivement :

- la latence de `GET /v1/collections` (scatter-gather), et l'exactitude de sa pagination ;
- la latence des écritures, le premium étant résolu en amont par le gateway.

---

## 6. Contrat des routes internes

C'est ce tableau, et non les imports TypeScript, qui définit la frontière entre services. Toute
route ajoutée ici est un engagement ; toute donnée qui n'y passe pas n'est pas partagée.

| Route | Servie par | Appelée par | Objet |
|---|---|---|---|
| `GET /internal/access?userId&collectionId` | `sharing` | `gateway` | `{ role: 'owner'\|'shared', statuses[] }` — §3.1 |
| `GET /internal/collections/:id/owner` | `catalog-*` | `sharing` | `{ ownerUserId, typeCode }` |
| `POST /internal/items/count` | `catalog-*` | `sharing` | `[{collectionId, statuses}]` → `[{collectionId, count}]` — §3.2 |
| `PUT /internal/registry` · `DELETE /internal/registry/:id` | `gateway` | `catalog-*` | Registre de routage — §3.3 |
| `GET /internal/registry/rebuild` | `gateway` | job de réparation | Re-scan des catalogs — §3.3 |
| `DELETE /internal/users/:id/data` | tous | `identity` | Participant de la saga RGPD, idempotent — §3.4 |
| `GET /internal/users/:id/export` | tous | `identity` | Fragment d'export RGPD — §3.4 |
| `GET /internal/premium/:userId` | `billing` | `gateway`, `sharing`, `search` | `{ premium: bool, source }` — §3.6. `search` : rattrapage manga, une fois par compte et par passage |
| `GET /internal/token/:userId/:provider` | `connections` | `search` | Résolution BYOT → repli premium → dégradé |
| `GET /internal/nodes?typeCode&level&cursor&limit` | `catalog-*` | `search` | Re-scan paginé des nœuds d'un type et d'un niveau, tous comptes — rattrapage hebdomadaire des tomes manga (ajout 2026-11-02) |
| `GET /internal/users/:id/nodes?typeCode&level` | `catalog-*` | `search` | Nœuds d'un compte + volumes déjà présents — `GET /v1/series-updates` (ajout 2026-11-02) |

Toutes protégées par le **principal signé** (§3.6) ou un secret de service à service ; **aucune
n'est exposée par le gateway** vers l'extérieur.

---

## 7. Décisions actées

| Sujet | Décision |
|---|---|
| Découpage | **Un service par type de collection** (`manga`, `vinyl`, `generic` pour movie/book/game) + `sharing` + transverses `search`/`connections`/`identity`/`billing` + `gateway` |
| Données | **Une instance MariaDB, un schéma logique par service.** Aucune FK inter-schémas, cohérence applicative |
| Transport | **HTTP interne** (pas de transport message Nest) ; `gateway` HTTP conserve `/v1/*` |
| Contrat public | **Inchangé.** L'app Android n'est pas modifiée — c'est une contrainte, pas un objectif secondaire |
| Auth interne | Principal signé HMAC `X-Acervatim-Principal` posé par le gateway ; JWT Google vérifié **une seule fois** |
| Code partagé | Monorepo Nest `apps/` + `libs/` ; **aucune dépendance directe entre deux `apps/`** |
| Réversibilité | M0→M5 réversibles ; **M6 ne l'est pas** — il ne s'engage que sur les mesures du §4 |
