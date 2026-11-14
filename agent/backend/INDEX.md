# acervatim-backend — Carte du dépôt NestJS

> Remonter : [../INDEX.md](../INDEX.md) · Règles dures : `acervatim-backend/AGENTS.md`

## Identité

- **Stack** : Node.js 24 · NestJS 10 · Prisma 6 · MariaDB 10.5+ (provider `mysql`) · TypeScript strict · Zod (nestjs-zod).
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
                      interceptors, limits, logging, pagination, rate-limit, sources/bnf
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
| [scan-asynchrone-bullmq.md](scan-asynchrone-bullmq.md) | **Chantier en cours** : passerelle sources BullMQ+Redis (throttle global, single-flight). P0 gbooks implémenté (branche `feat/p0-bullmq-gbooks`) ; état + suite. § 4ter : file `edition-watch`, rattrapage hebdomadaire des éditions suivies (SD3) |
| [recherche-textuelle-discogs.md](recherche-textuelle-discogs.md) | Recherche texte `?q=` vers Discogs : de quoi porte un résultat pour **choisir le bon pressage** (`catno`, `barcodes`, `masterId`), découpage artiste/titre, pagination par numéro de page, et le **relevé mesuré** du taux de hit du cache (texte : ~1 requête = 1 appel Discogs). Backend SD1 ; l'UI est le sprint app AD1 |
| [../../spec-macro-import-serie-manga.md](../../spec-macro-import-serie-manga.md) | **Recherche d'une série manga par titre** (SD1/SD2/SD3, branche `feat/manga-recherche-serie`) : recherche MangaDex sans jeton, `GET /v1/search/editions`, édition = (éditeur, `205$a`), `edition-mapping?publisher=&since=`, rattrapage `GET /v1/series-updates` ; mesures dans [../../etude-leviers-recherche-manga.md](../../etude-leviers-recherche-manga.md) |
| [gameupc-bgg.md](gameupc-bgg.md) | **Jeux de société, code-barres → BoardGameGeek** (2026-11-04, dépôt local `backend-services/search`, non déployé) : étude des sources francophones (aucune ne remplace GameUPC), synthèse du Swagger GameUPC et **écarts mesurés** (pas de validation des codes, indexation par chaîne), flux, files et cache, contraintes BGG, **questions ouvertes** (clé de production, quotas, vote de version) |
| [pivot-vs-pipeline-enrichissement.md](pivot-vs-pipeline-enrichissement.md) | **Réflexion non tranchée** : *Kingdom* introuvable côté jaquettes MangaDex (matching trop strict) + Sakamoto Days/Hellsing introuvables (BnF pas encore à jour, aucun repli) → limites structurelles du pivot ; pistes de correction |
| [partage-de-collection.md](partage-de-collection.md) | Partage de collections entre membres : un code porte N collections, chacune restreinte aux statuts choisis (`OWNED`/`WISHLIST`/`IGNORED`), lecture seule, Premium côté propriétaire, libellé privé de chaque côté. **Chantier backend terminé** (S3, S4, S5). Côté app : A3 est à refaire (rupture d'API S5), puis A4 |
| [architecture-microservices.md](architecture-microservices.md) | **Cible non engagée** : découpage en micro-services — un service par type de collection + `sharing` + transverses, derrière un gateway qui préserve `/v1/*`. Frontières, coutures dures (couplage `sharing`↔`collections/items/nodes`, routage `collectionId → typeCode`, cascade RGPD), phasage M0→M6. Une implémentation existe dans le dépôt local `acervatim/backend-services`, non déployée |

## Concepts clés (à connaître avant de toucher au code)

- **`unifiedData`** (JSON) = vérité curée par l'utilisateur sur un item/nœud ; **`sources`** (JSON) = snapshots bruts des providers externes ; **`userData`** (JSON) = subjectif (rating, dernière écoute). Séparation stricte des trois.
- **Profils de type** (`src/collections/types/`) : le comportement (schéma `unifiedData`, projection liste, hiérarchie) dépend du `typeCode` de la collection — `vinyl` (plat), `manga` (hiérarchie série→tome), fallback permissif pour le reste.
- **Nodes vs items** : une série manga = `CollectionNode` (porte ses propres sources, ex. MAL) ; un tome = `Item` (sources ISBN/BnF). Le vinyle n'utilise pas de nodes.
- **Identité anonyme** : `users.googleSubHash` (HMAC du `sub` Google), aucun email/nom en base. Voir [securite-et-privacy.md](securite-et-privacy.md).
