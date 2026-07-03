# Acervatim — Carte de la base de code (doc agent IA)

> **But** : permettre à un agent IA de localiser le bon fichier/module **sans explorer le code**.
> **Règle** : le code est la source de vérité. Cette doc est un index dérivé — si elle contredit le code, le code gagne (et cette doc doit être mise à jour).
> **Générée le** : 2026-07-02, à partir de l'état réel des dépôts.
> **Convention de chemins** : tous les chemins sont relatifs à `C:\Developpement\` (ex. `acervatim-app/app/src/main/...`).

## Qu'est-ce qu'Acervatim ?

Application de gestion de collections personnelles (vinyles, mangas, extensible à d'autres types) :

- **`acervatim-app/`** — app Android (Kotlin, Jetpack Compose, offline-first : Room = vérité locale, sync optionnelle vers le backend).
- **`acervatim-backend/`** — API REST (NestJS 10, Prisma 6, MariaDB) : compte Google anonymisé, sync des collections, recherche externe (Discogs/MAL/BnF), premium/abonnements Google Play, invitations.
- **`acervatim-docs/`** — ce dépôt : site Jekyll public (doc utilisateur/développeur/légal, lisible par humains) **+ la présente doc agent** sous `agent/` (exclue du site publié).

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
| Un modèle Prisma / une table | [backend/modele-de-donnees.md](backend/modele-de-donnees.md) |
| Un module NestJS (feature ou `common/`) | [backend/modules.md](backend/modules.md) |
| Auth JWT, chiffrement, privacy by design, guards | [backend/securite-et-privacy.md](backend/securite-et-privacy.md) |
| Lancer/déployer le backend, migrations Prisma, tests | [backend/operations-dev.md](backend/operations-dev.md) |
| Un flux bout-en-bout (scan → import, login Google, achat premium) | [transverse/flux-metier.md](transverse/flux-metier.md) |
| Les sources de données externes (Discogs, MAL, TMDB, BnF) | [transverse/sources-externes.md](transverse/sources-externes.md) |

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
│   ├── INDEX.md                       — carte du dépôt NestJS
│   ├── api-endpoints.md               — toutes les routes → controller/service
│   ├── modele-de-donnees.md           — schéma Prisma commenté
│   ├── modules.md                     — modules feature + common/
│   ├── securite-et-privacy.md         — auth, crypto, guards, invariants
│   └── operations-dev.md              — dev.ps1, Prisma, déploiement, tests
└── transverse/
    ├── flux-metier.md                 — flux end-to-end app ↔ backend
    ├── sources-externes.md            — Discogs / MAL / TMDB / BnF
    └── ou-trouver.md                  — table de lookup « intention → fichier »
```

## Autres documentations existantes (ne pas dupliquer)

| Doc | Rôle | Emplacement |
|---|---|---|
| `CLAUDE.md` (app) | Règles de code Android (chargé auto en session) | `acervatim-app/CLAUDE.md` |
| `CLAUDE.md` (backend) | Règles dures backend (chargé auto en session) | `acervatim-backend/CLAUDE.md` |
| Site public | Doc humaine (utilisateur / développeur / légal) | racine de ce dépôt : [`../utilisateur/`](../utilisateur/) · [`../developpeur/`](../developpeur/) · [`../legal/`](../legal/) |

## Maintenance de cette doc

- Toute évolution structurante (nouveau module, nouvel endpoint, nouvel écran, nouvelle table) doit être répercutée dans la fiche concernée **et** dans `transverse/ou-trouver.md`.
- Ne pas ajouter de contenu narratif long : cette doc est un **index**, pas un tutoriel. Une ligne par élément, un lien par détail.
