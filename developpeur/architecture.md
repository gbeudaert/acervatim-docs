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
            │       (acervatim-app)        │   Source de vérité LOCALE par défaut
            └──────────────┬──────────────┘
                           │  HTTPS REST /v1/*  ·  Bearer JWT interne
                           ▼
            ┌─────────────────────────────┐
            │          Backend            │   NestJS · Prisma · MariaDB
            │    (acervatim-backend)      │   Node 24 · conteneurisé
            └───────┬──────────────┬──────┘
                    │              │
   Google Identity  │              │  Proxy bases externes
   (vérif. id_token)│              ▼
                    │     Discogs · Google Books · AniList · TMDB …
```

**Principe directeur : offline-first.** L'app est 100 % utilisable sans compte ni réseau (toutes les
données en local via Room). Se connecter **ajoute** la synchronisation et l'enrichissement de
métadonnées ; ça ne conditionne jamais l'usage de base.

## Composants

### Application (`acervatim-app`)
- Single-activity + Compose, architecture en couches `ui / domain / data`.
- Persistance locale **Room** (`acervatim.db`) : `records`, `series`, `artists`, `collections` + tables de jointure.
- Couche réseau **Retrofit** (`data/remote/`) : `AuthApi`, `AcervatimApi`, intercepteurs (Bearer + `problem+json` → `ApiException`).

### Backend (`acervatim-backend`)
- API REST versionnée `/v1/*`, NestJS + Prisma + MariaDB.
- Rôles : authentification (Google → JWT interne), CRUD des collections/items/nodes, **proxy de recherche externe** unifiée, quotas/premium, invitations.
- Contrat : [Référence API](api-reference.md) · `openapi.json` (dépôt `acervatim-backend`).

## Déploiement

Le backend est déployé en **conteneur** (Podman rootless, unité **Quadlet** systemd), derrière un
reverse proxy TLS ; la base MariaDB est hébergée **séparément** du conteneur applicatif.

**Détail du déploiement** (build de l'image, unité, secrets, migrations) : voir
[Déploiement du backend](deploiement.md).
**Source de vérité** : `deploy/acervatim-api.container` (unité Quadlet commentée pas à pas) dans le
dépôt `acervatim-backend`.

## Flux clés

### 1. Authentification
1. App : Google Sign-In (Credential Manager) → `id_token`.
2. App → Backend : échange `id_token` contre un **JWT interne** (le backend vérifie le token Google, hash du `sub` avec pepper).
3. App : JWT stocké chiffré (`TokenStore` / EncryptedSharedPreferences), injecté en `Authorization: Bearer` par l'`AuthInterceptor`.

### 2. Synchronisation (palier premium à terme)
- Modèle **miroir** : chaque entité locale porte `remoteId`, `syncState`, `serverUpdatedAt`.
- Push/pull *last-write-wins* (`SyncManager`), via WorkManager + déclenchement manuel depuis l'écran Compte.
- Quota dépassé (`402`) → **sync partielle** (pas un échec), jauges affichées côté app.
- Limites actuelles : push manga bloqué (source provider requise), champs *app-only* non synchronisés.

### 3. Enrichissement au scan
1. App : scan EAN/UPC (vinyle) ou ISBN (manga) via ML Kit.
2. Dédup **locale** d'abord ; si absent → `GET /v1/search?type=<type>&barcode=<code>` (param `barcode`, **XOR** avec `q`).
3. Backend : route vers la bonne source (Discogs pour le vinyle ; **BnF → MangaDex** pour le manga, via un pivot ISBN → titre original, MAL en repli ; TMDB pour les films), cache + quotas, renvoie un **`UnifiedItem`**.
4. App : pré-remplit le formulaire d'ajout ; le code-barres scanné est persisté sur l'item.
- Détail dev (provenance des champs, flux scan, pivot ISBN manga) : [Sources de données & pivot ISBN](sources-et-pivot-isbn.md).

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
- **Mode connecté** : session active (JWT) → enrichissement avec les jetons de l'utilisateur (*bring your own token*). Le **premium** ajoute l'usage des jetons Acervatim et, à terme, la synchronisation cloud.
- **Track A / Track B** : phase offline-first publiable (A, terminée) puis backend & migration des providers (B, en cours).

## Voir aussi

- Accueil de la documentation : [index](../index.md).
