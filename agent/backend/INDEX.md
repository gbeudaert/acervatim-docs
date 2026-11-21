# acervatim-backend — Carte du dépôt NestJS (le monolithe)

> Remonter : [../INDEX.md](../INDEX.md) · Règles dures : `acervatim-backend/AGENTS.md`
> Le monolithe est le serveur **en production** jusqu'à la bascule sur la constellation (décision du
> 2026-11-16). La constellation (`acervatim/backend-services`) reprend ce code service par service :
> sa carte est dans [../INDEX.md](../INDEX.md) § Carte de la constellation. Mis à jour le 2026-11-16.

## Identité

- **Stack** : Node.js · NestJS · Prisma · MariaDB (provider `mysql`) · Redis (BullMQ) · TypeScript strict · Zod (nestjs-zod). Versions : `package.json` et `Dockerfile`.
- **Déploiement** : conteneur Podman (Quadlet) ; MariaDB hébergée séparément. Dev local : Docker Compose piloté par `scripts/dev.ps1` (**obligatoire**, jamais npm direct — voir [operations-dev.md](operations-dev.md)).
- **API** : REST `/v1/*`, JWT Bearer, erreurs RFC 9457, pagination cursor. Swagger sur `/docs` (hors prod).
- **Tests** : co-localisés `*.spec.ts` (unit) et `*.e2e.spec.ts` (e2e) à côté des sources.

## Arborescence (`acervatim-backend/src/`)

```
src/
├── main.ts / app.module.ts / app.controller.ts   — bootstrap, prefix /v1, pipes globaux, /health
├── auth/           — POST /v1/auth/google : idToken Google → JWT interne
├── users/          — /v1/me : profil, premium, export RGPD, suppression
├── collections/    — CRUD collections + profils de type (vinyl, manga, fallback)
├── nodes/          — hiérarchie (séries manga) : nœuds + sources externes
├── items/          — items (albums/tomes) : CRUD, sources, userData
├── search/         — recherche externe unifiée + edition-mapping / éditions BnF + import (import/) + rattrapage (watch/)
├── sources/        — PUT/DELETE /v1/sources/tmdb/token (clé personnelle TMDB)
├── metrics/        — GET /v1/admin/queues : files BullMQ et taux de hit du cache
├── oauth/          — connexion comptes externes user (Discogs OAuth1, MAL PKCE, TMDB)
├── premium/        — PremiumGuard + résolution du statut premium
├── subscriptions/  — Google Play : verify, webhook RTDN, cron de réconciliation
├── invitations/    — codes d'invitation anonymes (admin + redeem)
├── sharing/        — partage de collections : CollectionShare + codes hors-bande (S3),
│                   filtrage par statuts et masquage du userData partage (S4),
│                   N collections par code + libelles des deux cotes (S5)
├── prisma/         — PrismaService/module
├── config/         — validation des variables d'env
└── common/         — audit, cache, crypto, decorators, filters, guards, http,
                      interceptors, limits, logging, pagination, redis, validation,
                      sources/ (bnf, googlebooks, mangadex, manga-matching, title-ladder)
```

## Fiches détaillées

| Fiche | Contenu |
|---|---|
| [api-endpoints.md](api-endpoints.md) | Toutes les routes HTTP → controller/service/guards |
| [contrats-api-v1.md](contrats-api-v1.md) | **Contrats API V1** : versioning, Zod/nestjs-zod, erreurs RFC 9457, pagination cursor, filtres `[in]`/`[all]`, OpenAPI. À lire avant d'écrire un controller ou un DTO |
| [modele-de-donnees.md](modele-de-donnees.md) | Tables Prisma, relations, invariants |
| [modules.md](modules.md) | Détail de chaque module feature et de `common/` |
| [contrat-app-serveur.md](contrat-app-serveur.md) | **Tableau de correspondance champ app ↔ champ serveur** (item et nœud), avec la raison de chaque champ non synchronisé. À mettre à jour dès qu'un champ bouge d'un côté ou de l'autre |
| [securite-et-privacy.md](securite-et-privacy.md) | Auth JWT, hachage identité, AES, guards, secrets |
| [operations-dev.md](operations-dev.md) | Wrapper dev.ps1, workflow Prisma, tests, déploiement |
| [scan-asynchrone-bullmq.md](scan-asynchrone-bullmq.md) | Passerelle sources BullMQ+Redis (throttle global, single-flight) : P0 → P2 et P3 partiel **en production** ; reste P3 (Bull Board, calibrage des limiteurs). § 4ter : file `edition-watch`, rattrapage hebdomadaire des éditions suivies (SD3), dans `main`, pas en production |
| [recherche-textuelle-discogs.md](recherche-textuelle-discogs.md) | Recherche texte `?q=` vers Discogs : de quoi porte un résultat pour **choisir le bon pressage** (`catno`, `barcodes`, `masterId`), découpage artiste/titre, pagination par numéro de page, et le **relevé mesuré** du taux de hit du cache (texte : ~1 requête = 1 appel Discogs). Backend SD1 ; l'UI est le sprint app AD1 |
| [../../spec-macro-import-serie-manga.md](../../spec-macro-import-serie-manga.md) | **Recherche d'une série manga par titre** (SD1/SD2/SD3, dans `main` depuis le 2026-11-14, pas en production) : recherche MangaDex sans jeton, `GET /v1/search/editions`, édition = (éditeur, `205$a`), `edition-mapping?publisher=&since=`, rattrapage `GET /v1/series-updates` ; mesures dans [../../etude-leviers-recherche-manga.md](../../etude-leviers-recherche-manga.md) |
| [gameupc-bgg.md](gameupc-bgg.md) | **Jeux de société, code-barres → BoardGameGeek** (2026-11-04, constellation seulement, service `search`, dans `main`, non déployé ; clé GameUPC accordée le 2026-11-18, licence BGG **pending**) : étude des sources francophones (aucune ne remplace GameUPC), synthèse du Swagger GameUPC et **écarts mesurés** (pas de validation des codes, indexation par chaîne), flux, files et cache, contraintes BGG, **questions ouvertes** (clé de production, quotas, vote de version) |
| [pivot-vs-pipeline-enrichissement.md](pivot-vs-pipeline-enrichissement.md) | **Réflexion non tranchée** : *Kingdom* introuvable côté jaquettes MangaDex (matching trop strict) + Sakamoto Days/Hellsing introuvables (BnF pas encore à jour, aucun repli) → limites structurelles du pivot ; pistes de correction |
| [partage-de-collection.md](partage-de-collection.md) | Partage de collections entre membres : un code porte N collections, chacune restreinte aux statuts choisis (`OWNED`/`WISHLIST`/`IGNORED`), lecture seule, Premium côté propriétaire, libellé privé de chaque côté. **Chantier terminé** : serveur (S3, S4, S5) et app (A3bis, A4, août 2026) |
| [architecture-microservices.md](architecture-microservices.md) | **Cible engagée le 2026-11-16** (bascule complète décidée) : découpage en micro-services — un service par type de collection + `sharing` + transverses, derrière un gateway qui préserve `/v1/*`. Frontières, coutures dures (couplage `sharing`↔`collections/items/nodes`, routage `collectionId → typeCode`, cascade RGPD), phasage M0→M6. Implémentation : `acervatim/backend-services` (GitHub `acervatim-services`), non déployée |

## Concepts clés (à connaître avant de toucher au code)

- **`unifiedData`** (JSON) = vérité curée par l'utilisateur sur un item/nœud ; **`sources`** (JSON) = snapshots bruts des providers externes ; **`userData`** (JSON) = subjectif (rating, dernière écoute). Séparation stricte des trois.
- **Profils de type** (`src/collections/types/`) : le comportement (schéma `unifiedData`, projection liste, hiérarchie) dépend du `typeCode` de la collection — `vinyl` (plat), `manga` (hiérarchie série→tome), fallback permissif pour le reste (`movie`, `book`, `game`, semés par `prisma/seed.ts`).
- **Nodes vs items** : une série manga = `CollectionNode` (porte ses propres sources, ex. MAL) ; un tome = `Item` (sources ISBN/BnF). Le vinyle n'utilise pas de nodes.
- **Identité anonyme** : `users.googleSubHash` (HMAC du `sub` Google), aucun email/nom en base. Voir [securite-et-privacy.md](securite-et-privacy.md).
