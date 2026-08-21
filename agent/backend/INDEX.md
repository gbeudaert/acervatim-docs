# acervatim-backend — Carte du dépôt NestJS

> Remonter : [../INDEX.md](../INDEX.md) · Règles dures : `acervatim-backend/CLAUDE.md`

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
├── search/         — recherche externe unifiée + edition-mapping BnF
├── oauth/          — connexion comptes externes user (Discogs OAuth1, MAL PKCE, TMDB)
├── premium/        — PremiumGuard + résolution du statut premium
├── subscriptions/  — Google Play : verify, webhook RTDN, cron de réconciliation
├── invitations/    — codes d'invitation anonymes (admin + redeem)
├── prisma/         — PrismaService/module
├── config/         — validation des variables d'env
└── common/         — audit, cache, crypto, decorators, filters, guards, http,
                      interceptors, limits, logging, pagination, rate-limit, sources/bnf
```

## Fiches détaillées

| Fiche | Contenu |
|---|---|
| [api-endpoints.md](api-endpoints.md) | Toutes les routes HTTP → controller/service/guards |
| [modele-de-donnees.md](modele-de-donnees.md) | Tables Prisma, relations, invariants |
| [modules.md](modules.md) | Détail de chaque module feature et de `common/` |
| [contrat-app-serveur.md](contrat-app-serveur.md) | **Tableau de correspondance champ app ↔ champ serveur** (item et nœud), avec la raison de chaque champ non synchronisé. À mettre à jour dès qu'un champ bouge d'un côté ou de l'autre |
| [securite-et-privacy.md](securite-et-privacy.md) | Auth JWT, hachage identité, AES, guards, secrets |
| [operations-dev.md](operations-dev.md) | Wrapper dev.ps1, workflow Prisma, tests, déploiement |
| [scan-asynchrone-bullmq.md](scan-asynchrone-bullmq.md) | **Chantier en cours** : passerelle sources BullMQ+Redis (throttle global, single-flight). P0 gbooks implémenté (branche `feat/p0-bullmq-gbooks`) ; état + suite |
| [pivot-vs-pipeline-enrichissement.md](pivot-vs-pipeline-enrichissement.md) | **Réflexion non tranchée** : *Kingdom* introuvable côté jaquettes MangaDex (matching trop strict) + Sakamoto Days/Hellsing introuvables (BnF pas encore à jour, aucun repli) → limites structurelles du pivot ; pistes de correction |
| [partage-de-collection.md](partage-de-collection.md) | **Spécifié, non implémenté** : partage d'une collection entre membres (tout/possédés/wantlist), lecture seule, fonctionnalité Premium — modèle `CollectionShare`, prérequis bloquant sur `Item.userData.status` |

## Concepts clés (à connaître avant de toucher au code)

- **`unifiedData`** (JSON) = vérité curée par l'utilisateur sur un item/nœud ; **`sources`** (JSON) = snapshots bruts des providers externes ; **`userData`** (JSON) = subjectif (rating, dernière écoute). Séparation stricte des trois.
- **Profils de type** (`src/collections/types/`) : le comportement (schéma `unifiedData`, projection liste, hiérarchie) dépend du `typeCode` de la collection — `vinyl` (plat), `manga` (hiérarchie série→tome), fallback permissif pour le reste.
- **Nodes vs items** : une série manga = `CollectionNode` (porte ses propres sources, ex. MAL) ; un tome = `Item` (sources ISBN/BnF). Le vinyle n'utilise pas de nodes.
- **Identité anonyme** : `users.googleSubHash` (HMAC du `sub` Google), aucun email/nom en base. Voir [securite-et-privacy.md](securite-et-privacy.md).
