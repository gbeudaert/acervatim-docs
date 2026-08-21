# Spec macro — Partage de collection entre membres

> **Document interne** (exclu du site publié, comme `sprints-conformite-offre.md`).
> Créé le 2026-08-20. Vue d'ensemble transverse : *quoi*, *pourquoi*, *dans quel ordre*.
> Détail technique backend : [agent/backend/partage-de-collection.md](agent/backend/partage-de-collection.md).
> Sprints d'exécution : `acervatim-backend/docs/interne/sprints/sharing/` et
> `acervatim-app/docs/interne/sprints/sharing/` (répertoires **gitignorés** dans les deux dépôts).

## 1. Objectif produit

Un utilisateur premium partage une de ses collections avec d'autres membres (usage typique : le
foyer), en choisissant ce qui est exposé :

| Portée (`scope`) | Ce que voit le membre |
|---|---|
| `ALL` | Toute la collection |
| `OWNED` | Uniquement les éléments possédés |
| `WANTLIST` | Uniquement la wantlist (éléments désirés) |

**Décisions cadrantes** (2026-08-20) :
- Partage **par collection**, pas par compte entier — on peut partager ses vinyles sans ses mangas.
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

Conséquences directes sur le code existant :
- La sync (`collections`/`items`/`nodes`) devient **premium-only** — aujourd'hui elle est ouverte au
  gratuit sous quota (10 collections / 500 items).
- `common/quota/` perd sa raison d'être produit (à retirer ou requalifier en plafond technique).
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
  et le helper `resolveItemStatus` sur lequel s'appuiera le filtrage par portée. Les trois profils
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

Cette brique **ne sert pas au partage** — les portées n'ont besoin que du `status`. Elle peut donc
être menée en parallèle ou différée, mais elle conditionne la faisabilité du web.

> **Limite structurelle à trancher avant le web** : `coverPath` est une photo prise par
> l'utilisateur, stockée sur l'appareil, et le backend ne stocke **aucune image** — invariant privacy
> assumé et publié (`developpeur/backend.md`). Un client web n'affichera donc jamais ces
> jaquettes-là ; seules les `coverUrl` des providers passent. C'est un arbitrage produit (stocker des
> images côté serveur, accepter un web dégradé, ou autre), pas un bug à corriger dans un sprint.

### Brique 2 — Le gate Premium sur la synchronisation

Aujourd'hui `collections.controller.ts`, `items.controller.ts` et `nodes.controller.ts` ne portent
que `@UseGuards(JwtAuthGuard)` : **aucun** `PremiumGuard`. Le seul frein est `QuotaService`, appelé
à la création uniquement, qui laisse passer sous le seuil gratuit.

Subtilité à ne pas rater — le `PremiumGuard` actuel vérifie `req.userId`, c'est-à-dire **le
requérant** :

| Opération | Premium de qui ? |
|---|---|
| Écriture (POST/PATCH/DELETE) | Du **requérant** — seul le propriétaire écrit. `PremiumGuard` convient tel quel. |
| Lecture (GET) | Du **propriétaire de la collection**, pas du requérant. Sinon un membre gratuit ne pourrait pas lire ce qu'on lui partage — ce qui viderait la fonctionnalité de son sens. |

### Brique 3 — Le partage lui-même

Modèle `CollectionShare` / `CollectionShareMember`, calqué sur le patron `Invitation` existant :
code hors-bande, seul le HMAC stocké, pepper dédié `SHARE_CODE_PEPPER`.

**Pourquoi un code et pas « inviter par email »** : les comptes sont anonymes (`googleSubHash`,
aucun email ni nom en base). Chercher un membre par identifiant est structurellement impossible et
doit le rester — le code voyage hors-bande, comme les codes d'invitation premium.

Le filtrage par `scope` n'ajoute pas de routes de lecture : il étend l'autorisation existante de
`collections/items/nodes` (`resolveAccess(userId, collectionId) → owner | shared+scope`), le
`scope` étant injecté serveur-side dans le `where` et **jamais** accepté depuis le client.

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
S3 modèle + CRUD partages                        │
        │                                        ▼
        │                                A3 UI propriétaire (créer/révoquer)
        ▼                                        │
S4 lecture partagée + scopes ◄───────────────────┴──► A4 UI membre (lecture seule)
```

- **S1/A1 sont jumeaux** et **livrés** : le contrat a changé des deux côtés (backend tolérant
  d'abord — champ optionnel — puis l'app pousse).
- **S1bis/A1bis sont jumeaux** et servent la **version web**, pas le partage. Parallélisables ou
  différables, mais prérequis de tout chantier web.
- **S2** peut se faire en parallèle de S3, mais doit précéder la mise en service réelle du partage.
- **S4 dépend de S1 et S3** ; **A4 dépend de S4**.

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
- Que devient un partage émis quand le propriétaire perd son statut premium : suspension (402)
  plutôt que suppression, à confirmer.
- Anti-abus sur le nombre de partages/membres par collection : pas de limite produit décidée.
- Notification du propriétaire quand un membre rejoint : hors périmètre v1 (le pull sur
  `GET /v1/collections/:id/shares` suffit).

## 7. Documents liés

| Document | Contenu |
|---|---|
| [agent/backend/partage-de-collection.md](agent/backend/partage-de-collection.md) | Spec technique backend : modèle Prisma, endpoints, filtrage, sécurité |
| [sprints-conformite-offre.md](sprints-conformite-offre.md) | Modèle d'offre, écarts code↔cible, sprints C0→C5 |
| `acervatim-backend/docs/interne/sprints/sharing/` | Sprints backend S1, S1bis, S2→S4 (gitignoré) |
| `acervatim-app/docs/interne/sprints/sharing/` | Sprints app A1, A1bis, A2→A4 (gitignoré) |
