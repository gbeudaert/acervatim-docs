# Acervatim — Carte de la base de code (doc agent IA)

> **But** : permettre à un agent IA de localiser le bon fichier/module **sans explorer le code**.
> **Règle** : le code est la source de vérité. Cette doc est un index dérivé — si elle contredit le code, le code gagne (et cette doc doit être mise à jour).
> **Mise à jour le** : 2026-11-16 (créée le 2026-07-02), à partir de l'état réel des dépôts.
> **Convention de chemins** : tous les chemins sont relatifs à `C:\Developpement\` (ex. `acervatim-app/app/src/main/...`).
> **Versions** : aucune version d'outil ou de dépendance n'est recopiée ici ; les fichiers de build font foi (`package.json`, `app/build.gradle.kts`, `gradle/libs.versions.toml`, `Dockerfile`).

## Qu'est-ce qu'Acervatim ?

Application de gestion de collections personnelles, offline-first, avec un serveur optionnel
(enrichissement, synchronisation et partage en premium). Quatre dépôts, tous publics sur GitHub
(`github.com/gbeudaert/…`) :

| Dépôt local | Dépôt GitHub | Rôle | État au 2026-11-16 |
|---|---|---|---|
| `acervatim-app/` | `acervatim-app` | App Android (Kotlin, Jetpack Compose ; Room = vérité locale, sync optionnelle) | `main` en avance sur le serveur de production (recherche de série manga par titre, tomes parus) |
| `acervatim-backend/` | `acervatim-backend` | **Monolithe** NestJS + Prisma + MariaDB + Redis : auth Google anonymisée, collections, recherche externe, premium, invitations, partage | **En production**, construit fin août 2026 ; `main` porte six commits de plus (manga) non déployés |
| `acervatim/backend-services/` | `acervatim-services` | **Constellation** : le même `/v1/*` découpé en services derrière un `gateway` ; LEGO et jeux de société n'existent qu'ici | **Cible engagée le 2026-11-16** (bascule complète décidée) ; tourne en dev, pas encore sous Podman ni en production |
| `acervatim-docs/` | `acervatim-docs` | Ce dépôt : site Jekyll public (utilisateur / développeur / légal) + cette doc agent sous `agent/` (exclue du site) + documents de travail à la racine | Site publié sur `gbeudaert.github.io/acervatim-docs` |

## Types de collection

| `typeCode` | App | Serveur | Sources (détail : [transverse/sources-externes.md](transverse/sources-externes.md)) |
|---|---|---|---|
| `vinyl` | ✅ vue à plat | monolithe + `catalog-vinyl` | Discogs |
| `manga` | ✅ vue par séries | monolithe + `catalog-manga` | BnF, MangaDex, MyAnimeList (repli), Google Books (repli) |
| `movie` | ❌ (F11, prochain) | monolithe + `catalog-generic` | TMDB |
| `book`, `game` (jeu vidéo) | ❌ | monolithe + `catalog-generic` | aucune |
| `lego` | ❌ dans l'app (F12) ; serveur prêt en constellation (C7, 2026-11-17) | `catalog-lego` (collections) + `rebrickable-data` (catalogue de référence, lu par `search`) | Rebrickable (CSV), Brickset (pending) |
| jeu de société (pas de `typeCode`) | ❌ (F13) | `search` (code-barres) ; `catalog-boardgame` + `bgg-data` en spec brouillon | GameUPC (pending), BoardGameGeek (pending) |

## Comment naviguer (routage par intention)

| Tu cherches… | Va lire |
|---|---|
| N'importe quoi, point de départ rapide (« où est X ? ») | [transverse/ou-trouver.md](transverse/ou-trouver.md) |
| Règles dures (privacy, secrets, wrapper dev.ps1, style, où documenter) | [conventions.md](conventions.md) |
| Un écran, un ViewModel, une route de navigation Android | [app/ecrans-et-navigation.md](app/ecrans-et-navigation.md) |
| L'architecture Android (couches, DI, use cases) | [app/architecture.md](app/architecture.md) |
| Room / DAO / migrations / DataStore / stockage covers / backup ZIP | [app/donnees-locales.md](app/donnees-locales.md) |
| Client HTTP, auth Google côté app, sync, compte | [app/reseau-sync-compte.md](app/reseau-sync-compte.md) |
| Une fonctionnalité app → fichiers impliqués | [app/catalogue-fonctionnalites.md](app/catalogue-fonctionnalites.md) |
| Un endpoint HTTP du backend (`/v1/...`) | [backend/api-endpoints.md](backend/api-endpoints.md) |
| Les conventions de contrat (Zod, RFC 9457, cursor, filtres) | [backend/contrats-api-v1.md](backend/contrats-api-v1.md) |
| « Le serveur connaît-il ce champ ? » (app ↔ serveur) | [backend/contrat-app-serveur.md](backend/contrat-app-serveur.md) |
| Un modèle Prisma / une table | [backend/modele-de-donnees.md](backend/modele-de-donnees.md) |
| Un module NestJS (feature ou `common/`) | [backend/modules.md](backend/modules.md) |
| Auth JWT, chiffrement, privacy by design, guards | [backend/securite-et-privacy.md](backend/securite-et-privacy.md) |
| Lancer/déployer le backend, migrations Prisma, tests | [backend/operations-dev.md](backend/operations-dev.md) |
| Files BullMQ, débits sortants, import asynchrone, rattrapage `edition-watch` | [backend/scan-asynchrone-bullmq.md](backend/scan-asynchrone-bullmq.md) |
| Recherche texte Discogs, choix du pressage | [backend/recherche-textuelle-discogs.md](backend/recherche-textuelle-discogs.md) |
| Partage de collections (codes, statuts, lecture partagée) | [backend/partage-de-collection.md](backend/partage-de-collection.md) |
| Jeux de société : code-barres → BoardGameGeek | [backend/gameupc-bgg.md](backend/gameupc-bgg.md) |
| Le découpage en micro-services, la bascule | [backend/architecture-microservices.md](backend/architecture-microservices.md) puis la carte de la constellation ci-dessous |
| Un flux bout-en-bout (scan → import, login Google, achat premium, jeu de société) | [transverse/flux-metier.md](transverse/flux-metier.md) |
| Les sources de données externes, toutes collections | [transverse/sources-externes.md](transverse/sources-externes.md) |
| Les conditions d'usage des sources, ce qui change au premium | [../developpeur/sources-et-pivot-isbn.md](../developpeur/sources-et-pivot-isbn.md) § Conditions d'usage |

## Carte de la constellation (`acervatim/backend-services/`)

Chaque service a un `README.md` (conception, pièges) et un `AGENTS.md` (règles dures) : les lire
avant d'y toucher. Vue d'ensemble, coutures et état : `acervatim/backend-services/README.md`.

| Service | Port | Schéma | Périmètre |
|---|---|---|---|
| `gateway/` | 3100 | `acv_gateway` | `/v1/*`, JWT → principal signé, table de routage (`src/routing/route-table.ts`), premium et débits, scatter-gather de `GET /v1/collections` |
| `identity/` | 3110 | `acv_core` | Auth, `/v1/me`, saga de suppression de compte, journal d'audit |
| `billing/` | 3120 | `acv_billing` | Premium, Google Play, invitations |
| `connections/` | 3130 | `acv_conn` | OAuth externe, clés personnelles, résolution des jetons |
| `sharing/` | 3140 | `acv_sharing` | Partages, membres, résolution d'accès |
| `search/` | 3150 | `acv_search` | Recherche externe, files BullMQ, cache ; jeux de société (`src/search/boardgame/`, `src/sources/{gameupc,bgg}/`) |
| `catalog-generic/` | 3160 | `acv_cat_generic` | movie, book, game |
| `catalog-manga/` | 3170 | `acv_cat_manga` | manga |
| `catalog-vinyl/` | 3180 | `acv_cat_vinyl` | vinyle |
| `rebrickable-data/` | 3190 | `acv_rebrickable` | Catalogue LEGO de référence, `/internal/*` et `/admin/*` seulement |
| `catalog-lego/` | 3220 | `acv_cat_lego` | lego (un item est une boîte) ; clone de `catalog-vinyl`, refait le 2026-11-17 |
| `catalog-core/` | — | — | Bibliothèque : CRUD commun compilé dans les quatre catalogs |
| `deploy/` | — | — | Quadlet Podman, manifeste du pod, image de migration (`deploy/README.md`) |
| `db/reprise/` | — | — | Reprise des données du monolithe vers les schémas, procédure de bascule (`db/reprise/README.md`) |

Règles propres à la constellation : `npm`/`prisma`/`nest` jamais lancés depuis Windows (passer par
`scripts/dev.ps1`) ; `src/common/internal/contracts.ts`, le filtre Problem Details et
`common/errors` sont dupliqués à l'identique dans chaque service et se modifient partout du même
coup ; aucune garde premium hors du gateway.

## Structure de cette doc (graphe)

```
INDEX.md (toi ici)
├── conventions.md                     — règles transverses non négociables
├── app/
│   ├── INDEX.md                       — carte du dépôt Android
│   ├── architecture.md                — couches ui/domain/data, Hilt, patterns
│   ├── ecrans-et-navigation.md        — routes ↔ écrans ↔ ViewModels
│   ├── donnees-locales.md             — Room, DataStore, covers, backup
│   ├── reseau-sync-compte.md          — Retrofit, auth, sync, compte
│   └── catalogue-fonctionnalites.md   — feature → fichiers
├── backend/
│   ├── INDEX.md                       — carte du monolithe NestJS
│   ├── api-endpoints.md               — toutes les routes → controller/service
│   ├── contrats-api-v1.md             — conventions de contrat
│   ├── contrat-app-serveur.md         — champ app ↔ champ serveur
│   ├── modele-de-donnees.md           — schéma Prisma commenté
│   ├── modules.md                     — modules feature + common/
│   ├── securite-et-privacy.md         — auth, crypto, guards, invariants
│   ├── operations-dev.md              — dev.ps1, Prisma, déploiement, tests
│   ├── scan-asynchrone-bullmq.md      — files BullMQ, débits, edition-watch
│   ├── recherche-textuelle-discogs.md — recherche texte vinyle
│   ├── partage-de-collection.md       — partage entre membres
│   ├── pivot-vs-pipeline-enrichissement.md — réflexion non tranchée sur le pivot manga
│   ├── gameupc-bgg.md                 — jeux de société (constellation)
│   └── architecture-microservices.md  — cible micro-services, coutures, phasage
└── transverse/
    ├── flux-metier.md                 — flux end-to-end app ↔ serveur
    ├── sources-externes.md            — toutes les sources, par type de collection
    └── ou-trouver.md                  — table de lookup « intention → fichier »
```

## Documents de travail (racine de `acervatim-docs`, exclus du site)

| Document | Rôle | État au 2026-11-16 |
|---|---|---|
| `audit-etat-avancement.md` | État d'avancement du projet, tous dépôts | Local (ignoré par git), refait le 2026-11-16 |
| `dossier-jeux-de-societe-bgg-gameupc.md` | Sources, licences et conception des jeux de société | En cours (lot F) : licence BGG et clé GameUPC pending |
| `spec-macro-import-serie-manga.md` | Ajout d'une série manga par titre (SD1-SD3, AD1-AD3) | Fusionné dans `main` le 2026-11-14, pas en production |
| `audit-limitations-recherche-manga.md`, `etude-leviers-recherche-manga.md` | Constat et leviers de la recherche manga | Clos, décisions reportées dans la spec ci-dessus |
| `spec-macro-recherche-tris-galerie.md` | Recherche texte, tris, galerie | Livré (août 2026) |
| `spec-macro-partage-collection.md` | Partage de collections | Serveur livré ; app : consultation et gestion livrées |
| `spec-repli-google-books-pivot.md` | Repli Google Books quand la BnF ignore l'ISBN | Livré (août 2026) |
| `spec-macro-export-sources-externes.md` | Export de la collection vers Discogs et autres | Non commencé, questions de sens à trancher avant de coder |
| `sprints-conformite-offre.md` | Offre Offline / Connecté / Premium | Local (ignoré par git), historique |
| `plan-consolidation-documentation.md` | Consolidation de la doc dans ce dépôt | Historique ; reste la suppression de l'ancien site `acervatim-app/docs/` (pending) |
| `_bmad-output/implementation-artifacts/` | Specs BMad : `spec-rebrickable-data-service.md` (fait), `spec-catalog-boardgame-bgg-data.md` (brouillon), `deferred-work.md` | Ignoré par git |
| `private/courriers/` | Brouillons de courriers (GameUPC, Brickset) | Ignoré par git |

## Autres documentations existantes (ne pas dupliquer)

| Doc | Rôle | Emplacement |
|---|---|---|
| `AGENTS.md` (app) | Règles dures Android (chargé auto en session ; `CLAUDE.md` n'est qu'un pointeur) | `acervatim-app/AGENTS.md` |
| `AGENTS.md` (backend) | Règles dures du monolithe | `acervatim-backend/AGENTS.md` |
| `AGENTS.md` par service | Règles dures de chaque service de la constellation | `acervatim/backend-services/<service>/AGENTS.md` |
| `CHANGELOG.md` (backend) | Notes de version du monolithe : variables d'env, migrations, ruptures | `acervatim-backend/CHANGELOG.md` |
| Site public | Doc humaine (utilisateur / développeur / légal) | racine de ce dépôt : [`../utilisateur/`](../utilisateur/) · [`../developpeur/`](../developpeur/) · [`../legal/`](../legal/) |

## Maintenance de cette doc

- Toute évolution structurante (nouveau module, nouvel endpoint, nouvel écran, nouvelle table, nouveau service) doit être répercutée dans la fiche concernée **et** dans `transverse/ou-trouver.md`.
- Ne pas ajouter de contenu narratif long : cette doc est un **index**, pas un tutoriel. Une ligne par élément, un lien par détail.
- Ne recopier aucun numéro de version d'outil ou de dépendance ; ce qui attend un tiers porte le statut **pending**.
