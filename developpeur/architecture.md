---
title: Architecture
layout: default
parent: Développeur
nav_order: 1
---

# Acervatim — Architecture macro

Vue d'ensemble **système** des composants Acervatim et de leurs interactions. Pour les détails, suivre
les liens vers chaque dépôt (cf. [Accueil](../index.md)).

## Vue système

```
            ┌─────────────────────────────┐
            │        Application           │   Kotlin · Compose · Room (offline-first)
            │            (mvc)             │   Source de vérité LOCALE par défaut
            └──────────────┬──────────────┘
                           │  HTTPS REST /v1/*  ·  Bearer JWT interne
                           ▼
            ┌─────────────────────────────┐
            │          Backend            │   NestJS · Prisma · MariaDB
            │      (acervus-backend)      │   Node 24 · déployé sur Raspberry Pi 4 / NAS
            └───────┬──────────────┬──────┘
                    │              │
   Google Identity  │              │  Proxy bases externes (clés côté serveur)
   (vérif. id_token)│              ▼
                    │     Discogs · Google Books · AniList · TMDB …
                    ▼
            ┌─────────────────────────────┐
            │   Infra AWS (FUTURE)        │   OpenTofu — non activée en v1
            │     (acervatim-infra)       │   prend le relais à la migration cloud
            └─────────────────────────────┘
```

**Principe directeur : offline-first.** L'app est 100 % utilisable sans compte ni réseau (toutes les
données en local via Room). Se connecter **ajoute** la synchronisation et l'enrichissement de
métadonnées ; ça ne conditionne jamais l'usage de base.

## Composants

### Application (`mvc`)
- Single-activity + Compose, architecture en couches `ui / domain / data`.
- Persistance locale **Room** (`acervatim.db`) : `records`, `series`, `artists`, `collections` + tables de jointure.
- Couche réseau **Retrofit** (`data/remote/`) : `AuthApi`, `AcervatimApi`, intercepteurs (Bearer + `problem+json` → `ApiException`).
- Détail : [`mvc/docs/contexte-global.md`](../../mvc/docs/contexte-global.md).

### Backend (`acervus-backend`)
- API REST versionnée `/v1/*`, NestJS + Prisma + MariaDB.
- Rôles : authentification (Google → JWT interne), CRUD des collections/items/nodes, **proxy de recherche externe** unifiée, quotas/premium, invitations.
- Contrat : [Référence API](api-reference.md) · [`openapi.json`](../../acervus-backend/openapi.json).

### Infra (`acervatim-infra`)
- IaC OpenTofu pour **AWS** — **cible de migration future**. La v1 ne l'utilise pas (backend sur Pi/NAS).

## Hébergement V1 — Raspberry Pi 4 + Podman

La V1 est **auto-hébergée**, sans cloud. Le plan AWS serverless (DynamoDB + Lambda) décrit dans
d'anciens docs n'a **jamais été déployé** : il reste une cible d'évolution (`acervatim-infra`).

> **Pourquoi pas le NAS Synology ?** Le NAS (armv7l, 32 bits) a été abandonné comme hôte applicatif :
> Prisma exige une plateforme 64 bits. Le NAS conserve un seul rôle : **héberger la base MariaDB**.

```
        Internet
           │  HTTPS (TLS)
           ▼
   [ Reverse proxy ]
           │  http://localhost:3000
           ▼
┌───────────────────────────┐        ┌─────────────────────────┐
│  Raspberry Pi 4 (arm64)   │        │   NAS Synology          │
│  Podman rootless          │  3306  │                         │
│  └─ conteneur acervatim-  │───────▶│   MariaDB 10.5+         │
│     api (NestJS + Prisma) │  mysql │   (base `acervatim`)    │
│     géré par Quadlet      │        │                         │
└───────────────────────────┘        └─────────────────────────┘
```

| Élément | Choix V1 |
|---|---|
| Hôte applicatif | Raspberry Pi 4, **arm64**, Node 24 LTS |
| Conteneurisation | **Podman rootless**, unité **Quadlet** systemd (`acervatim-api.container`) |
| Image | construite sur le Pi (`podman build --platform linux/arm64`) |
| Base de données | **MariaDB sur le NAS** (pas de conteneur DB en prod) ; `DATABASE_URL` pointe le NAS |
| Migrations Prisma | cible de build dédiée (`--target migrate`) lancée à la main avant un déploiement schéma |
| Healthcheck | `/health` défini dans l'image, planifié par un timer systemd transient |
| Persistance du service | `loginctl enable-linger` pour survivre à la fermeture de session SSH |
| Coût | quasi nul (matériel déjà possédé, pas de facture cloud) |

**Détail du déploiement** : voir [Déploiement Raspberry Pi / Podman](deploiement-raspberry-pi.md).
**Source de vérité** : le dépôt `acervus-backend` —
[`deploy/acervatim-api.container`](../../acervus-backend/deploy/acervatim-api.container) (Quadlet
commenté pas à pas) et l'[index des sprints backend](../../acervus-backend/docs/interne/sprints/).

## Flux clés

### 1. Authentification
1. App : Google Sign-In (Credential Manager) → `id_token`.
2. App → Backend : échange `id_token` contre un **JWT interne** (le backend vérifie le token Google, hash du `sub` avec pepper).
3. App : JWT stocké chiffré (`TokenStore` / EncryptedSharedPreferences), injecté en `Authorization: Bearer` par l'`AuthInterceptor`.

### 2. Synchronisation (mode connecté)
- Modèle **miroir** : chaque entité locale porte `remoteId`, `syncState`, `serverUpdatedAt`.
- Push/pull *last-write-wins* (`SyncManager`), via WorkManager + déclenchement manuel depuis l'écran Compte.
- Quota dépassé (`402`) → **sync partielle** (pas un échec), jauges affichées côté app.
- Limites actuelles : push manga bloqué (source provider requise), champs *app-only* non synchronisés (cf. sprint 11).

### 3. Enrichissement au scan
1. App : scan EAN/UPC (vinyle) ou ISBN (manga) via ML Kit.
2. Dédup **locale** d'abord ; si absent → `GET /v1/search?type=<type>&barcode=<code>` (param `barcode`, **XOR** avec `q`).
3. Backend : route vers la bonne source (Discogs pour le vinyle ; **MyAnimeList + BnF** pour le manga, via un pivot ISBN → titre original ; TMDB pour les films), cache + quotas, renvoie un **`UnifiedItem`**.
4. App : pré-remplit le formulaire d'ajout ; le code-barres scanné est persisté sur l'item.
- Détail dev (provenance des champs, flux scan, pivot ISBN manga) : [Sources de données & pivot ISBN](sources-et-pivot-isbn.md).
- Détail app (wiring Android) : [`mvc/docs/travail/guide-scan-enrichissement-app.md`](../../mvc/docs/travail/guide-scan-enrichissement-app.md).

## Modèle de données — façade

Pour ne pas réécrire ~15 écrans, l'app garde son vocabulaire historique et **mappe** vers le modèle backend :

| App (local) | Backend (contrat) |
|-------------|-------------------|
| `Record`    | `Item`            |
| `Series`    | `Node`            |
| (collection active) | `Collection` |

Le mapping vit dans `data/remote/mapper/FacadeMapper` côté app. Les champs *app-only* (note, prix,
vitesse, dernière écoute, démographie manga…) n'ont pas de place dans le `unifiedData` strict du
contrat → conservés en local, non synchronisés (arbitrage produit à venir).

## Glossaire

- **Collection / Item / Node** : entités du contrat backend (collection conteneuse / élément / nœud-série).
- **Record / Series** : équivalents côté app (façade).
- **UnifiedItem** : résultat normalisé de `/v1/search`, indépendant de la source externe.
- **SyncState** : `LOCAL_ONLY` · `DIRTY` · `SYNCED` — état de synchro d'une ligne locale.
- **Mode connecté** : session active (JWT) → sync + enrichissement. Distinct du **premium** (quotas serveur, pas un palier d'UI).
- **Track A / Track B** : phase offline-first publiable (A, terminée) puis backend & migration des providers (B, en cours).

## Voir aussi

- Statut détaillé par sprint (app) : [`sprint-realises.md`](../../mvc/docs/interne/sprint-realises.md).
- Accueil de la documentation : [index](../index.md).
