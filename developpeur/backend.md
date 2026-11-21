---
title: Le serveur Acervatim
layout: default
parent: Développeur
nav_order: 2
---

# Le serveur Acervatim (backend)
{: .no_toc }

Le backend Acervatim est le composant serveur **optionnel** de l'écosystème. L'application Android fonctionne intégralement sans lui ; il n'entre en jeu que si vous activez le mode connecté.

Il existe sous deux formes qui exposent la même API `/v1` : le **monolithe** (dépôt
`acervatim-backend`), qui tourne en production, et la **constellation de micro-services** (dépôt
`acervatim-services`), vers laquelle la production basculera — décision du 16 novembre 2026, date de
bascule non fixée. Cette page décrit ce que l'API fait, quelle que soit la forme ; le découpage est
présenté dans [Architecture](architecture.md).

## Sommaire
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Rôle

Le serveur remplit cinq rôles, par ordre de priorité :

1. **Proxy des bases de données externes.** L'application ne parle jamais directement à une source (Discogs, BnF, MangaDex, MyAnimeList, Google Books, TMDB…) : elle interroge un unique endpoint de recherche (`/v1/search`) qui route la requête vers la bonne source selon le type de collection. Les clés et secrets d'API restent côté serveur, jamais embarqués dans l'application. Liste complète : [Sources de données](sources-et-pivot-isbn.md).
2. **Cache partagé.** Une recherche effectuée par un utilisateur bénéficie aux suivants : réponses plus rapides et charge réduite sur les API tierces.
3. **Throttling centralisé.** Une passerelle d'appels sortants (files **BullMQ**, une par source, adossées à Redis) plafonne le débit vers les API externes et déduplique les requêtes concurrentes identiques (*single-flight*), protégeant leurs quotas (Discogs limite par exemple à 60 requêtes/minute).
4. **OAuth complexes côté serveur.** Certains protocoles (Discogs OAuth 1.0a, avec signatures HMAC) ne peuvent pas être implémentés proprement sur un appareil sans y embarquer des secrets. Le serveur gère ces flux et stocke les jetons **chiffrés** (AES-256-GCM).
5. **Services de compte.** Synchronisation et partage des collections (premium), statut premium, export et suppression des données.

## Architecture

```
[Application Android]
        │  HTTPS + JWT (RS256)
        ▼
[Reverse proxy (TLS Let's Encrypt)]
        │
        ▼
[API NestJS (Node.js)]  — monolithe, ou gateway + services de la constellation
        ├──► MariaDB (Prisma)
        ├──► Redis (files BullMQ, une par source)
        └──► APIs externes : Discogs · BnF · MangaDex · MyAnimeList · Google Books · TMDB
             (+ Rebrickable · Brickset · GameUPC · BoardGameGeek dans la constellation)
```

| Couche | Choix |
|---|---|
| Framework | NestJS (Node.js, TypeScript strict) — versions : `package.json` de chaque dépôt |
| Base de données | MariaDB, via l'ORM Prisma |
| Authentification | Google Sign-In côté app → JWT interne RS256 (validité 7 jours) |
| Chiffrement | AES-256-GCM pour les jetons OAuth tiers |
| Files d'appels sortants | **BullMQ** sur Redis : une file par source, débit borné, requêtes identiques dédoublonnées |
| Déploiement | Conteneurs **Podman** derrière un reverse proxy TLS ; base MariaDB à part — voir [Déploiement du backend](deploiement.md) |

## Confidentialité : privacy by design

La protection des données n'est pas une couche ajoutée, c'est une contrainte de conception :

- **Aucune donnée personnelle directe en base** : pas d'email, pas de nom, pas de photo de profil.
- L'identifiant Google (`sub`) n'est **jamais stocké en clair** : seul un hachage HMAC-SHA-256 avec un secret serveur (*pepper*) sert d'identifiant de connexion. Sans ce secret, impossible de relier un compte Acervatim à un compte Google.
- L'identifiant utilisateur interne est un UUID opaque, non corrélable.
- Les jetons OAuth des services externes sont chiffrés en AES-256-GCM.
- Aucune image n'est stockée côté serveur : les jaquettes restent sur votre appareil ou sont référencées par URL.
- Les journaux ne contiennent jamais d'identifiant Google, d'email ni de jeton en clair.
- **Export RGPD** (`GET /v1/me/export`) et **suppression de compte en cascade** (`DELETE /v1/me`) intégrés à l'API.

## Fonctionnalités de l'API

### Authentification et compte

- `POST /v1/auth/google` — échange un jeton Google Sign-In contre un JWT applicatif ; crée le compte au premier login.
- `GET /v1/me` — profil minimal et statut premium.
- `GET /v1/me/export` — export complet des données (portabilité, RGPD art. 20).
- `DELETE /v1/me` — suppression du compte et de toutes les données associées.

### Collections, items et nœuds

**Ces routes sont réservées au premium** : le stockage serveur a un coût réel, il n'existe donc pas
de palier gratuit synchronisé. Un compte non premium reçoit `402` aussi bien en écriture qu'en
lecture ; en lecture, c'est le statut du **propriétaire** de la collection qui compte, afin qu'un
membre non premium puisse consulter une collection partagée avec lui.

Le modèle serveur est le miroir du modèle local de l'application :

- **Collections** (`/v1/collections`) — CRUD complet, typées (`vinyl`, `manga`, `movie`, `book`, `game`), avec recherche plein texte sur le contenu des items. L'application ne gère encore que le vinyle et le manga.
- **Items** (`/v1/collections/:id/items`, `/v1/items/:id`) — les fiches individuelles (un disque, un tome). Chaque item porte une « vérité curée » éditable, et peut être lié à des **snapshots de sources externes** (la fiche Discogs brute, par exemple) pour ré-enrichissement ultérieur.
- **Nœuds** (`/v1/collections/:id/nodes`, `/v1/nodes/:id`) — les regroupements hiérarchiques des types sériels : une série manga, un coffret. Les tomes s'y rattachent, ce qui permet la vue par séries multi-appareils.
- Données typées par collection (`unifiedData`) : champs structurés propres au vinyle (label, format, pays…) ou au manga (ISBN, éditeur, numéro de tome…).

### Recherche externe unifiée

- `GET /v1/search?type=...&q=...` (recherche texte) ou `GET /v1/search?type=...&barcode=...` (recherche par code EAN/UPC ou ISBN) — un seul endpoint quel que soit le type de collection. `q` et `barcode` sont **mutuellement exclusifs** (XOR : exactement l'un des deux). Le serveur choisit la ou les sources (Discogs pour le vinyle — `barcode` y route vers `database/search?barcode=…` ; BnF puis MangaDex pour le manga, MyAnimeList en repli ; TMDB pour les films), gère le cache et les débits, et renvoie des résultats au format unifié.
- Manga : `GET /v1/search/edition-mapping` (tous les tomes d'une édition, avec jaquettes), `GET /v1/search/cover`, import de série asynchrone (`POST /v1/search/import-jobs`). Fusionnés dans `main`, pas encore en production : `GET /v1/search/editions` (éditions françaises d'une série) et `GET /v1/series-updates` (tomes parus).
- Jeux de société, **constellation seulement** : `GET /v1/search/boardgame/barcode/:code` et `POST /v1/search/boardgame/barcode/:code/choice` — GameUPC puis BoardGameGeek ; la clé GameUPC est accordée, la licence BoardGameGeek reste **pending**.

### Connexion de comptes externes

- `GET /v1/oauth/:provider/start` / `callback` — flux de connexion d'un compte externe (Discogs, MyAnimeList), entièrement géré côté serveur.
- `GET /v1/oauth` — liste des comptes connectés ; `DELETE /v1/oauth/:provider` — révocation.
- `PUT` / `DELETE /v1/sources/tmdb/token` — clé personnelle TMDB, qui n'a pas d'OAuth.

### Premium et invitations

- `POST /v1/subscriptions/verify` et `POST /v1/invitations/redeem` — vérification d'un abonnement Google Play et activation par code d'invitation. Aujourd'hui, le premium s'obtient par invitation : l'achat n'est pas encore proposé dans l'application.
- Le statut premium conditionne l'accès à la synchronisation et aux fonctionnalités qui lui sont réservées.

### Qualité de l'API

- Tous les endpoints sont versionnés sous `/v1`.
- Erreurs normalisées **RFC 9457** (*Problem Details*), avec identifiant de requête (`X-Request-Id`) sur chaque réponse pour faciliter le support.
- Pagination par curseur sur toutes les listes ; filtres expressifs (`?type[in]=vinyl,manga`, `?items[all]=miles,davis`).
- Validation stricte des entrées (schémas Zod) : toute donnée inattendue est rejetée.
- Rate limiting global par IP en plus du throttling par source.
- Documentation **OpenAPI** générée automatiquement depuis le code (interface Swagger disponible hors production).
- `GET /health` — supervision (état du service). Une sonde de disponibilité `GET /health/ready`, qui vérifie la base, est en cours d'ajout dans la constellation.

## Ce que le serveur ne fait pas

- Il n'est **jamais sollicité en arrière-plan** par l'application : chaque appel correspond à une action de l'utilisateur (recherche, scan, synchronisation, ouverture d'une série). Côté serveur, seules des tâches internes planifiées tournent : réconciliation des abonnements, rattrapage hebdomadaire des tomes des séries suivies (premium), import quotidien du catalogue LEGO dans la constellation.
- Il ne stocke **ni images, ni données personnelles directes**.
- Il ne conditionne pas l'usage local : sans réseau ou sans compte, toutes les fonctionnalités locales de l'application (liste, fiches, wishlist, import/export) restent disponibles.

## Voir aussi

- [Architecture](architecture.md) — vue système et flux.
- [Référence API](api-reference.md) — contrat `/v1` détaillé.
- [Authentification Google](auth-google.md) · [Déploiement du backend](deploiement.md).
- Vue utilisateur : [Mode connecté](../utilisateur/mode-connecte.md).
