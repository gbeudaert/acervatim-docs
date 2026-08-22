---
title: Déploiement du backend
layout: default
parent: Développeur
nav_order: 5
---

# Déploiement du backend — conteneur Podman

Le backend est déployé en **conteneur** (Podman rootless, unité **Quadlet** systemd), derrière un
reverse proxy TLS. La base MariaDB est hébergée **séparément** du conteneur applicatif (pas de
conteneur DB en production). Un conteneur **Redis** local héberge les files **BullMQ** (throttle des
appels sources + single-flight). Cette page décrit le *comment* du déploiement, indépendamment de
l'infrastructure qui l'héberge.

## Topologie

```
        Internet
           │  HTTPS (TLS)
           ▼
   [ Reverse proxy ]
           │  http://localhost:3000
           ▼
┌────────────────────────────────────┐        ┌─────────────────────────┐
│  Hôte applicatif (Podman rootless) │        │  Serveur de base de     │
│  réseau podman `acervatim`         │  3306  │  données                │
│  ├─ conteneur acervatim-api ───────┼───────▶│  MariaDB 10.5+          │
│  │    (NestJS + Prisma)            │  mysql │  (base `acervatim`)     │
│  │        │ 6379 (réseau interne)  │        └─────────────────────────┘
│  └─ conteneur acervatim-redis ◀────┘
│       (BullMQ backing store)       │
└────────────────────────────────────┘
```

| Élément | Choix |
|---|---|
| Runtime | Node 24 LTS |
| Conteneurisation | **Podman rootless**, unités **Quadlet** systemd (`acervatim-api.container`, `acervatim-redis.container`) |
| Réseau | réseau Podman partagé `acervatim` (`acervatim.network`) → l'API joint Redis par son nom |
| Image | construite sur l'hôte de déploiement (`podman build`) |
| Base de données | **MariaDB distante** (pas de conteneur DB en prod) ; `DATABASE_URL` pointe le serveur de base |
| File d'attente | **Redis** local (conteneur), non exposé sur l'hôte ; `REDIS_HOST`/`REDIS_PORT` pointent le conteneur |
| Migrations Prisma | cible de build dédiée (`--target migrate`) lancée à la main avant un déploiement schéma |
| Healthcheck | `/health` défini dans l'image, planifié par un timer systemd transient |
| Persistance du service | `loginctl enable-linger` pour survivre à la fermeture de session SSH |

## Installation (rootless, recommandé)

1. **Builder l'image** sur l'hôte :
   ```sh
   podman build -t acervatim-api:latest .
   ```
2. **Déposer l'unité Quadlet** dans `~/.config/containers/systemd/acervatim-api.container`.
3. **Déposer le `.env`** (chmod 600) dans `~/acervatim/.env`, avec au minimum :
   ```
   DATABASE_URL=mysql://user:pass@<hôte-db>:3306/acervatim
   REDIS_HOST=acervatim-redis
   REDIS_PORT=6379
   SUB_HASH_PEPPER=…
   ENCRYPTION_KEY=…
   JWT_PRIVATE_KEY=…
   INVITE_CODE_PEPPER=…
   SHARE_CODE_PEPPER=…
   ```
   Ces secrets sont **non rotables sans coût** (perdre le pepper = perdre la corrélation `sub` Google
   → `userId` ; perdre `SHARE_CODE_PEPPER` = invalider tous les codes de partage en circulation). Ils
   ne sont jamais commités. La liste **exhaustive et faisant foi** des variables exigées au démarrage
   est le schéma de validation d'env du backend (`src/config/env.validation.ts`) : le service refuse
   de démarrer s'il en manque une.
4. **Déposer les unités Redis + réseau** dans `~/.config/containers/systemd/` :
   `acervatim.network` (réseau partagé) et `acervatim-redis.container`.
5. **Recharger et démarrer** (Redis d'abord — l'API se reconnecte seule mais autant l'avoir prêt) :
   ```sh
   systemctl --user daemon-reload
   systemctl --user start acervatim-redis
   systemctl --user start acervatim-api
   ```
6. **Survie hors session SSH** (sinon les services et le timer de healthcheck s'arrêtent à la
   déconnexion) :
   ```sh
   loginctl enable-linger $USER
   ```

> Variante root : déposer l'unité dans `/etc/containers/systemd/`, remplacer `%h` par un chemin
> absolu (ex. `/opt/acervatim/.env`) et `WantedBy=multi-user.target`.

## Migrations Prisma

Le conteneur runtime **n'exécute pas** les migrations (le CLI `prisma` est une devDep, absente de
l'image runtime). Avant un déploiement qui change le schéma, lancer une fois la cible dédiée contre la
base distante :

```sh
podman build --target migrate -t acervatim-migrate .
podman run --rm --env-file ~/acervatim/.env acervatim-migrate
```

## Redis (files BullMQ)

Redis héberge les files **BullMQ** qui régulent les appels aux sources externes (throttle global +
single-flight + cache mutualisé). Il tourne en **conteneur local** sur l'hôte applicatif, sur le même
réseau Podman que l'API.

| Aspect | Choix | Pourquoi |
|---|---|---|
| Exposition | **aucun port publié** sur l'hôte | seul l'API, sur le réseau `acervatim`, y accède |
| Résolution | par nom de conteneur (`acervatim-redis`) via le DNS Podman | d'où `REDIS_HOST=acervatim-redis` |
| Éviction | `--maxmemory-policy noeviction` | ne jamais évincer un job en attente sous pression mémoire |
| Persistance | `--appendonly yes` (volume `acervatim-redis-data`) | confort : les jobs sont **idempotents/rejouables** (réchauffeurs de cache), une perte ne fait que re-déclencher une résolution au prochain scan |

Vérifier après démarrage :

```sh
systemctl --user status acervatim-redis
podman exec acervatim-redis redis-cli ping           # → PONG
podman exec acervatim-api sh -c 'nc -z acervatim-redis 6379 && echo reachable'
```

> Panne Redis : l'API se reconnecte automatiquement. Une sonde de disponibilité dédiée court-circuite
> le chemin interactif — Redis down → **`503` propre immédiat** (pas d'attente du timeout de file) ;
> sur le chemin best-effort (jaquettes), l'indisponibilité dégrade en résultat vide. L'endpoint
> `/health` expose `redis: up|down` (le `status` reste `ok` : Redis n'est pas vital au boot).

## Points d'attention

- `RunInit=true` (Podman `--init`) : `main.ts` n'a pas de handler `SIGTERM` ; sans init, Node en PID 1
  ignore le signal et l'arrêt attend le `SIGKILL`.
- `After=network-online.target` + `Restart=always` : la base est **distante**, le réseau doit être
  prêt ; `Restart=always` couvre le cas où la base n'est pas encore joignable au démarrage de l'hôte.
- `TimeoutStartSec=90` : laisse le temps au premier `prisma connect` (base distante) + au bootstrap Nest.

## Source de vérité

Dépôt `acervatim-backend`, dossier `deploy/` (unités Quadlet commentées pas à pas) :
`acervatim-api.container`, `acervatim-redis.container`, `acervatim.network`.

## Voir aussi

- [Architecture § Déploiement](architecture.md#déploiement).
