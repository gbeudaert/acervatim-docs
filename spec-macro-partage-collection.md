# Spec macro — Partage de collection entre membres

> **Document interne** (exclu du site publié, comme `sprints-conformite-offre.md`).
> Créé le 2026-08-20. Vue d'ensemble transverse : *quoi*, *pourquoi*, *dans quel ordre*.
> Détail technique backend : [agent/backend/partage-de-collection.md](agent/backend/partage-de-collection.md).
> Sprints d'exécution : `acervatim-backend/docs/interne/sprints/sharing/` et
> `acervatim-app/docs/interne/sprints/sharing/` (répertoires **gitignorés** dans les deux dépôts).

## 1. Objectif produit

Un utilisateur premium partage une de ses collections avec d'autres membres (usage typique : le
foyer), en choisissant ce qui est exposé :

Un code de partage expose **une ou plusieurs collections**, chacune restreinte aux **statuts** de
son choix :

| Statuts cochés pour une collection | Ce que le membre y voit |
|---|---|
| `OWNED` | Les éléments possédés (statut absent compris : c'est le défaut) |
| `WISHLIST` | Les éléments désirés, et les séries désirées sans aucun tome |
| `IGNORED` | Les éléments écartés |

Les trois portées d'origine (`ALL`/`OWNED`/`WANTLIST`) ne sont plus qu'un raccourci d'interface :
`ALL` = les trois cases, `OWNED` = une seule, `WANTLIST` = une seule. L'ensemble libre rend
exprimable ce qu'elles interdisaient — « possédés **et** désirés, sans les ignorés ».

**Décisions cadrantes** (2026-08-20, complétées le 2026-08-22) :
- Partage **par collection**, pas par compte entier — on peut partager ses vinyles sans ses mangas.
- Un partage peut réunir **plusieurs collections** sous un seul code, avec des statuts différents
  pour chacune.
- **Autant de partages qu'on veut** : aucun plafond.
- Chaque partage porte un **libellé libre de chaque côté** — « Wantlist manga avec Alice » chez le
  propriétaire, « Wantlist manga de Bob » chez le membre. Les deux sont privés : aucun ne franchit
  la frontière.
- Accès **lecture seule** pour le membre : pas d'ajout/édition/suppression. Garanti côté **guard**,
  pas seulement côté UI.
- Fonctionnalité **Premium**, côté propriétaire. Un membre non-premium peut rejoindre et consulter.

## 2. Redéfinition du Premium (préalable)

Le partage a rouvert la définition de l'offre. Modèle acté le 2026-08-20, détaillé dans
[sprints-conformite-offre.md](sprints-conformite-offre.md) :

> Le palier payant existe pour **partager le coût réel du serveur**, pas pour maximiser le revenu.
> **Tout ce qui a un coût serveur réel** (stockage de la synchronisation, appels API en repli sur
> les jetons Acervatim) **est premium** ; **tout ce qui n'en a pas** (enrichissement en BYOT, usage
> local) **reste gratuit, sans quota artificiel.** Un seul palier payant, à prix bas.

Conséquences sur le code — **toutes livrées le 2026-08-21** (sprints S2 backend et A2 app) :
- La sync (`collections`/`items`/`nodes`) est **premium-only** : `PremiumGuard` en écriture (statut
  du **requérant**), `CollectionPremiumGuard` en lecture (statut du **propriétaire** — condition
  pour que le partage reste lisible par un membre gratuit, cf. §2 de la brique 2).
- `common/quota/` a été **requalifié** en `common/limits/` : plafond **technique** anti-emballement,
  identique pour tous, non communiqué, que payer ne lève pas → `409 technical-limit` et non `402`.
  `GET /v1/me/quota` a été supprimé, et le canal quota retiré de l'app.
- Le partage s'ajoute comme fonctionnalité premium : il dépend de la sync cloud.

## 3. Les trois briques à construire

### Brique 1 — Le statut des éléments doit remonter au serveur *(prérequis bloquant)* ✅

Le statut possédé/wishlist/ignoré existait **uniquement sur l'appareil**. Deux trous distincts, tous
deux comblés (sprints **S1** et **A1**) :

- **App** : `FacadeMapper.toLocalExtras()` rangeait `status` dans les champs *app-only*, commentés
  « conservé localement et **jamais poussé** ». Plus profond : `ItemDto` n'avait **aucun** champ
  `userData` et `PatchItemRequest` ne transportait que `unifiedData` — le canal `userData` des items
  était entièrement absent côté client, pas seulement `status`. Le canal est désormais ouvert, avec
  un repli sur le cliché local (`resolveStatus`) pour ne rien perdre des items déjà synchronisés.
- **Backend** : `ItemUserDataSchema` porte maintenant `status`, avec `DEFAULT_ITEM_STATUS = 'OWNED'`
  et le helper `resolveItemStatus` sur lequel s'appuie le filtrage des partages. Les trois profils
  (`vinyl`, `manga`, `fallback`) projettent le statut en vue liste.

Sans cette brique, `OWNED`/`WANTLIST` n'auraient pu filtrer que des séries manga
(`CollectionNode.isWishlist`), pas des éléments.

### Brique 1bis — Synchroniser *tout* le reste *(pour le web, pas pour le partage)*

La brique 1 n'a poussé qu'**un seul champ**, volontairement : un changement de contrat à la fois.
Tout le reste des données perso demeure dans `localExtras` — note, prix d'achat, vitesse de gravure,
pays de pressage, dernière écoute, extras manga, note et commentaire de série.

C'est tenable tant que l'unique client est l'app Android, qui a sa base Room. **Ça ne l'est plus dès
qu'on vise une version web** : un client web n'a aucun stockage local et ne peut afficher que ce que
le serveur détient. Il afficherait aujourd'hui une collection amputée.

Bonne nouvelle relevée à l'audit : la quasi-totalité de ces champs **ont déjà leur place** dans les
schémas serveur (`ItemUserDataSchema.rating`/`purchasePrice`/`lastPlayedAt`,
`VinylItemSchema.recordingSpeed`/`country`, `MangaItemSchema.publisherFr`/`pageCount`,
`MangaSerieSchema.demographic`/`author`, `CollectionNode.userData`). C'est l'app qui ne les envoie
pas. Le travail est donc très majoritairement côté app (**A1bis**), le backend se limitant à combler
les trous restants et à verrouiller le contrat par des tests de round-trip (**S1bis**).

Cette brique **ne sert pas au partage** — le filtrage n'a besoin que du `status`. Elle peut donc
être menée en parallèle ou différée, mais elle conditionne la faisabilité du web.

> **Limite structurelle à trancher avant le web** : `coverPath` est une photo prise par
> l'utilisateur, stockée sur l'appareil, et le backend ne stocke **aucune image** — invariant privacy
> assumé et publié (`developpeur/backend.md`). Un client web n'affichera donc jamais ces
> jaquettes-là ; seules les `coverUrl` des providers passent. C'est un arbitrage produit (stocker des
> images côté serveur, accepter un web dégradé, ou autre), pas un bug à corriger dans un sprint.

### Brique 2 — Le gate Premium sur la synchronisation ✅

*(Livrée le 2026-08-21, sprint S2 — le constat ci-dessous décrit l'état d'avant.)*

`collections.controller.ts`, `items.controller.ts` et `nodes.controller.ts` ne portaient
que `@UseGuards(JwtAuthGuard)` : **aucun** `PremiumGuard`. Le seul frein était `QuotaService`, appelé
à la création uniquement, qui laissait passer sous le seuil gratuit.

Subtilité à ne pas rater — le `PremiumGuard` actuel vérifie `req.userId`, c'est-à-dire **le
requérant** :

| Opération | Premium de qui ? |
|---|---|
| Écriture (POST/PATCH/DELETE) | Du **requérant** — seul le propriétaire écrit. |
| Lecture (GET) | Du **propriétaire de la collection**, pas du requérant. Sinon un membre gratuit ne pourrait pas lire ce qu'on lui partage — ce qui viderait la fonctionnalité de son sens. |

S4 a complété le tableau côté écriture : `CollectionWriteGuard` résout d'abord le rôle, refuse un
membre en `403`, puis vérifie le premium. Un `PremiumGuard` seul aurait laissé un membre premium
écrire dans la collection d'autrui.

### Brique 3 — Le partage lui-même

**Modèle et CRUD livrés le 2026-08-21** (sprint S3), **UI propriétaire livrée le 2026-08-22**
(sprint A3 : créer un partage, transmettre le code hors-bande, révoquer un partage ou un membre) :
`CollectionShare` / `CollectionShareMember`,
calqués sur le patron `Invitation` existant — code hors-bande, seul le HMAC stocké, pepper dédié
`SHARE_CODE_PEPPER`. Six routes dans le module `sharing/` (créer, lister, révoquer, éjecter un
membre, rejoindre, lister ses partages reçus), audit `share.*`, export RGPD des deux sens.

**Lecture partagée livrée le 2026-08-22** (sprint S4) : `resolveAccess` rend désormais
`owner | shared`, les routes de lecture de `collections`/`items`/`nodes` filtrent, et toute mutation
d'un membre est refusée **au guard** (`403`).

**Partage multi-collections livré le 2026-08-22** (sprint S5) : `CollectionShareEntry` (N
collections par code, statuts au choix pour chacune), libellé privé de chaque côté, routes ancrées
sur `/v1/shares`. Le chantier est complet côté serveur.

**UI refondue et vue membre livrées le 2026-08-22** (sprints **A3bis** et **A4**) : S5 avait cassé
le contrat de A3 (`POST`/`GET /v1/collections/:id/shares`, champ `scope`), l'écran propriétaire a été
reconstruit sur `/v1/shares` et remonté au niveau du compte — un partage n'appartenant plus à une
collection. Le membre rejoint par code sans être premium, nomme le partage pour lui-même, et consulte
en lecture seule. **Le chantier est complet des deux côtés.**

> **Décision structurante côté membre** : les collections reçues en partage ne sont **jamais
> persistées** — lecture en ligne uniquement, par un chemin séparé de Room. Ce ne sont pas les
> données de l'utilisateur : les mettre en cache les ferait entrer dans son export et sa sauvegarde
> ZIP, et laisserait une copie fantôme après révocation. Le prix assumé est que les écrans de
> liste/détail existants, qui lisent Room, ne sont pas réutilisables tels quels.

**Pourquoi un code et pas « inviter par email »** : les comptes sont anonymes (`googleSubHash`,
aucun email ni nom en base). Chercher un membre par identifiant est structurellement impossible et
doit le rester — le code voyage hors-bande, comme les codes d'invitation premium.

Le filtrage n'ajoute pas de routes de lecture : il étend l'autorisation existante de
`collections/items/nodes` (`resolveAccess(userId, collectionId) → owner | shared+statuses`), les
statuts étant injectés serveur-side dans le `where` et **jamais** acceptés depuis le client.

## 4. Ordre d'exécution

```
Backend                                  App
───────                                  ───
S1 statut item (userData.status) ◄──────► A1 canal userData + push du statut   ✅ FAIT
        │
        ├───► S1bis contrat complet ◄────► A1bis sync complète ──► (version web, différée)
        │
        ▼                                        ▼
S2 gate premium sync (owner-aware)       A2 écran Compte : sync premium-only
        │                                        │
        ▼                                        │
S3 modèle + CRUD partages  ✅ FAIT                │
        │                                        ▼
        │                                A3 UI propriétaire (créer/révoquer)  🗄️ HISTORIQUE
        ▼                                        │      (contrat cassé par S5)
S4 lecture partagée ✅ FAIT                       │
        │                                        ▼
        ▼                                A3bis UI propriétaire, nouveau contrat  ✅ FAIT
S5 N collections + statuts + libellés ✅ FAIT ────┴──► A4 UI membre (lecture seule)  ✅ FAIT
```

- **S1/A1 sont jumeaux** et **livrés** : le contrat a changé des deux côtés (backend tolérant
  d'abord — champ optionnel — puis l'app pousse).
- **S1bis/A1bis sont jumeaux** et servent la **version web**, pas le partage. Parallélisables ou
  différables, mais prérequis de tout chantier web.
- **S2** peut se faire en parallèle de S3, mais doit précéder la mise en service réelle du partage.
  Les deux sont livrés.
- **S3/A3 sont livrés** : un propriétaire premium peut émettre un code, voir qui a rejoint et
  révoquer.
- **S4 est livré** : le membre qui rejoint lit effectivement la collection, filtrée.
- **S5 est livré** : un code porte N collections avec leurs statuts, et chacun nomme le partage pour
  soi.
- **A3bis et A4 sont livrés** (2026-08-22) et closent le chantier : ils partagent la couche
  data/domaine du partage, les séparer aurait produit deux contrats successifs pour un seul travail.
  Ne reste hors partage que **S1bis/A1bis**, qui servent la version web.

## 5. Rétro-compatibilité

- `userData.status` absent = `OWNED` (défaut déjà utilisé côté app : `?: ItemStatus.OWNED`).
  Aucune migration de données : `userData` est une colonne JSON déjà présente.
- Les items synchronisés avant S1 se re-poussent naturellement au push suivant.
- Le passage de la sync en premium-only **cassera l'usage des comptes gratuits qui synchronisent
  aujourd'hui** — prévoir la communication et une fenêtre de transition (à trancher au moment
  d'activer S2, hors périmètre technique).

## 6. Points ouverts

- Sort de `common/quota/` : retrait sec ou requalification en plafond technique anti-abus
  (non communiqué, non lié au palier) — à trancher au Sprint C0 de
  [sprints-conformite-offre.md](sprints-conformite-offre.md).
- ~~Que devient un partage émis quand le propriétaire perd son statut premium~~ — **tranché en S4** :
  **suspension** (`402` pour le propriétaire comme pour ses membres), jamais de suppression. Rendre
  le premium rend l'accès.
- ~~Anti-abus sur le nombre de partages~~ — **tranché le 2026-08-22** : aucun plafond. Seul le
  `maxUses` d'un code reste borné (≤ 100, validation de DTO), et il borne les membres de ce code.
- Notification du propriétaire quand un membre rejoint : hors périmètre v1 (le pull sur
  `GET /v1/shares` suffit).
- **Vue hiérarchique d'une collection manga partagée** : la vue membre est plate, la projection
  légère portant `serie` et `volume` en sous-ligne. Reconstruire l'arbre demanderait
  `GET /v1/collections/:id/nodes` et une navigation à deux niveaux, pour un écran de consultation —
  à reprendre si l'usage le demande.
- **Jaquettes photographiées invisibles pour un membre** : `coverPath` est local, le backend ne
  stocke aucune image (invariant privacy). C'est le même arbitrage produit que pour le web (§3,
  brique 1bis), pas un bug de A4.

## 7. Documents liés

| Document | Contenu |
|---|---|
| [agent/backend/partage-de-collection.md](agent/backend/partage-de-collection.md) | Spec technique backend : modèle Prisma, endpoints, filtrage, sécurité |
| [sprints-conformite-offre.md](sprints-conformite-offre.md) | Modèle d'offre, écarts code↔cible, sprints C0→C5 |
| `acervatim-backend/docs/interne/sprints/sharing/` | Sprints backend S1, S1bis, S2→S4 (gitignoré) |
| `acervatim-app/docs/interne/sprints/sharing/` | Sprints app A1, A1bis, A2, A3 (historique), A3bis, A4 (gitignoré) |
