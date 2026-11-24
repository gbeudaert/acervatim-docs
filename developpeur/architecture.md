---
title: Architecture
layout: default
parent: Développeur
nav_order: 1
---

# Acervatim — Architecture macro

Vue d'ensemble **système** des composants Acervatim et de leurs interactions. Depuis le 23 novembre
2026, la production tourne sur la **constellation de micro-services** ; le **monolithe**, qui la
servait jusque-là, est arrêté et gardé pour un retour. L'application ne voit aucune différence : les
deux exposent le même contrat `/v1/*`.

## Vue système

```
            ┌─────────────────────────────┐
            │        Application          │   Kotlin · Compose · Room (offline-first)
            │       (acervatim-app)       │   Source de vérité LOCALE par défaut
            └──────────────┬──────────────┘
                           │  HTTPS REST /v1/*  ·  Bearer JWT interne
                           ▼
   ┌───────────────────────────────────────────────────────────┐
   │ Serveur Acervatim : la constellation (acervatim-services),│
   │ en production depuis le 23 novembre 2026, à la place du   │
   │ monolithe (acervatim-backend). NestJS · Prisma · MariaDB  │
   └───────┬──────────────────────────────────┬────────────────┘
           │                                  │
  Google Identity                    Proxy des bases externes
  (vérif. id_token)          Discogs · BnF · MangaDex · MyAnimeList ·
                             Google Books · TMDB — et, dans la constellation :
                             Rebrickable · Brickset · GameUPC · BoardGameGeek
```

**Principe directeur : offline-first.** L'app est 100 % utilisable sans compte ni réseau (toutes les
données en local via Room). Se connecter **ajoute** l'enrichissement des fiches et, en premium, la
synchronisation et le partage ; ça ne conditionne jamais l'usage de base.

## Composants

### Application (`acervatim-app`)
- Single-activity + Compose, architecture en couches `ui / domain / data`.
- Persistance locale **Room** (`acervatim.db`) : `records`, `series`, `artists`, `collections` + tables de jointure.
- Couche réseau **Retrofit** (`data/remote/`) : `AuthApi`, `AcervatimApi`, intercepteurs (Bearer + `problem+json` → `ApiException`).
- Types de collection gérés : **vinyle** et **manga**. Les films et le LEGO sont les prochains.

### Ancien serveur : le monolithe (`acervatim-backend`)
- Un seul processus NestJS, une base MariaDB, un Redis pour les files BullMQ.
- Rôles : authentification (Google → JWT interne), collections / items / nœuds, **proxy de recherche
  externe** unifiée, premium, invitations, partage de collections.
- Contrat : [Référence API](api-reference.md) · `openapi.json` (dépôt `acervatim-backend`).
- Il a servi la production jusqu'au 23 novembre 2026, dans une version construite fin août 2026, sans
  la recherche de série manga par titre ni le rattrapage des tomes parus. Il est arrêté depuis, et
  gardé pour un retour pendant la fenêtre de retour.

### Serveur en production : la constellation (`acervatim-services`)
Le même contrat `/v1/*`, découpé en services qui ont chacun leur schéma dans une même instance
MariaDB, derrière un **gateway** :

| Service | Rôle |
|---|---|
| `gateway` | Point d'entrée `/v1/*` : vérifie le JWT, émet un principal signé, route vers le bon service, applique les droits premium et les débits |
| `identity` | Authentification, `/v1/me`, suppression de compte orchestrée entre services, journal d'audit |
| `billing` | Premium, abonnements Google Play, invitations |
| `connections` | Comptes externes reliés (OAuth), clés personnelles, résolution des jetons |
| `sharing` | Partages de collections et droits d'accès |
| `search` | Recherche externe, files BullMQ, cache des réponses ; jeux de société (GameUPC, BoardGameGeek) |
| `catalog-vinyl`, `catalog-manga`, `catalog-lego`, `catalog-generic` | Collections, items et nœuds, par type (`catalog-generic` porte film, livre, jeu vidéo) |
| `rebrickable-data` | Catalogue LEGO de référence (Rebrickable, Brickset), interne, sans route `/v1` |

En production depuis le 23 novembre 2026 : les onze services tournent dans un pod Podman, leurs
schémas sur l'instance MariaDB qui porte aussi la base du monolithe, et les données du monolithe ont
été reprises à la bascule. Le LEGO (`catalog-lego`, refait le 17 novembre 2026 sur le code commun des
catalogues) en fait partie. La mémoire a été mesurée sur un poste de développement : 481 à 603 Mo
de mémoire réelle pour neuf services, contre 121 à 130 Mo pour le monolithe. Elle reste à mesurer en
production.

## Déploiement

Les deux serveurs se déploient en **conteneurs** (Podman rootless, unités **Quadlet** systemd),
derrière un reverse proxy TLS.

- **Constellation**, en production : un pod qui regroupe les services, un seul port publié (celui du
  gateway), les migrations des schémas jouées par un initContainer avant le démarrage, MariaDB et
  Redis hors du pod — voir [Déploiement du backend](deploiement.md). Source de vérité :
  `deploy/README.md` dans `acervatim-services`.
- **Monolithe**, arrêté et gardé pour un retour : un conteneur applicatif, un conteneur Redis, la
  base MariaDB hébergée à part — voir [Mise à jour du backend](mise-a-jour.md). Source de vérité :
  `deploy/acervatim-api.container` dans `acervatim-backend`.

## Flux clés

### 1. Authentification
1. App : Google Sign-In (Credential Manager) → `id_token`.
2. App → Serveur : échange `id_token` contre un **JWT interne** (le serveur vérifie le token Google, hache le `sub` avec un pepper).
3. App : JWT stocké chiffré (`TokenStore` / EncryptedSharedPreferences), injecté en `Authorization: Bearer` par l'`AuthInterceptor`.

### 2. Synchronisation (premium)
- Modèle **miroir** : chaque entité locale porte `remoteId`, `syncState`, `serverUpdatedAt`.
- Push/pull *last-write-wins* (`SyncManager`), déclenché depuis l'écran Compte.
- Ordre de poussée : collections → nœuds (séries) → items (tomes, disques). Vinyles et mangas sont synchronisés.
- Réservée au premium : un compte non premium reçoit `402`. Les champs propres à l'app qui n'ont pas
  de place côté serveur sont listés, avec leur raison, dans le tableau de correspondance app ↔ serveur
  (dépôt `acervatim-docs`, `agent/backend/contrat-app-serveur.md`).

### 3. Enrichissement au scan
1. App : scan EAN/UPC (vinyle) ou ISBN (manga) via ML Kit, reconnaissance embarquée.
2. Dédup **locale** d'abord ; si absent → `GET /v1/search?type=<type>&barcode=<code>` (param `barcode`, **XOR** avec `q`).
3. Serveur : route vers la bonne source (Discogs pour le vinyle ; **BnF → MangaDex** pour le manga, MyAnimeList en repli), cache et débits, renvoie un **`UnifiedItem`**.
4. App : pré-remplit le formulaire d'ajout ; le code-barres scanné est persisté sur l'item.
- Toutes les sources, type par type : [Sources de données](sources-et-pivot-isbn.md).

## Modèle de données — façade

Pour ne pas réécrire ~15 écrans, l'app garde son vocabulaire historique et **mappe** vers le modèle serveur :

| App (local) | Serveur (contrat) |
|-------------|-------------------|
| `Record`    | `Item`            |
| `Series`    | `Node`            |
| (collection active) | `Collection` |

Le mapping vit dans `data/remote/mapper/FacadeMapper` côté app.

## Glossaire

- **Collection / Item / Node** : entités du contrat serveur (collection conteneuse / élément / nœud-série).
- **Record / Series** : équivalents côté app (façade).
- **UnifiedItem** : résultat normalisé de `/v1/search`, indépendant de la source externe.
- **SyncState** : `LOCAL_ONLY` · `DIRTY` · `SYNCED` — état de synchro d'une ligne locale.
- **Mode connecté** : session active (JWT) → enrichissement avec les jetons de l'utilisateur (*bring your own token*). Le **premium** ajoute l'usage des jetons Acervatim, la synchronisation et le partage.
- **Monolithe / constellation** : le serveur d'origine, en un seul programme, arrêté depuis le 23 novembre 2026, et le serveur en production, découpé en micro-services.

## Voir aussi

- Accueil de la documentation : [index](../index.md).
