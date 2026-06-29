---
title: Déploiement Raspberry Pi / Podman
layout: default
parent: Développeur
nav_order: 5
---

# Déploiement — Raspberry Pi 4 + Podman

La V1 est **auto-hébergée**, sans cloud. Le plan AWS serverless (DynamoDB + Lambda) décrit dans
d'anciens documents n'a **jamais été déployé** : il reste une cible d'évolution (`acervatim-infra`).

> **Pourquoi pas le NAS Synology ?** Le NAS (armv7l, 32 bits) a été abandonné comme hôte applicatif :
> Prisma exige une plateforme 64 bits. Le NAS conserve un seul rôle : **héberger la base MariaDB**.

## Topologie

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

## Installation (rootless, recommandé)

1. **Builder l'image** sur le Pi :
   ```sh
   podman build -t acervatim-api:latest .
   ```
2. **Déposer l'unité Quadlet** dans `~/.config/containers/systemd/acervatim-api.container`.
3. **Déposer le `.env`** (chmod 600) dans `~/acervatim/.env`, avec au minimum :
   ```
   DATABASE_URL=mysql://user:pass@<ip-nas>:3306/acervatim
   SUB_HASH_PEPPER=…
   ENCRYPTION_KEY=…
   JWT_PRIVATE_KEY=…
   ```
   Ces secrets sont **non rotables sans coût** (perdre le pepper = perdre la corrélation `sub` Google
   → `userId`). Ils ne sont jamais commités.
4. **Recharger et démarrer** :
   ```sh
   systemctl --user daemon-reload
   systemctl --user start acervatim-api
   ```
5. **Survie hors session SSH** (sinon le service et le timer de healthcheck s'arrêtent à la
   déconnexion) :
   ```sh
   loginctl enable-linger $USER
   ```

> Variante root : déposer l'unité dans `/etc/containers/systemd/`, remplacer `%h` par un chemin
> absolu (ex. `/opt/acervatim/.env`) et `WantedBy=multi-user.target`.

## Migrations Prisma

Le conteneur runtime **n'exécute pas** les migrations (le CLI `prisma` est une devDep, absente de
l'image runtime). Avant un déploiement qui change le schéma, lancer une fois la cible dédiée contre la
base du NAS :

```sh
podman build --target migrate -t acervatim-migrate .
podman run --rm --env-file ~/acervatim/.env acervatim-migrate
```

## Points d'attention

- `RunInit=true` (Podman `--init`) : `main.ts` n'a pas de handler `SIGTERM` ; sans init, Node en PID 1
  ignore le signal et l'arrêt attend le `SIGKILL`.
- `After=network-online.target` + `Restart=always` : la base est distante (NAS), le réseau doit être
  prêt ; `Restart=always` couvre le cas où le NAS n'est pas encore joignable au démarrage du Pi.
- `TimeoutStartSec=90` : laisse le temps au premier `prisma connect` (NAS) + au bootstrap Nest.

## Source de vérité

Le dépôt `acervus-backend` —
[`deploy/acervatim-api.container`](../../acervus-backend/deploy/acervatim-api.container) (Quadlet
commenté pas à pas) et l'[index des sprints backend](../../acervus-backend/docs/interne/sprints/)
(sprint « Déploiement Pi + CI/CD »).

## Voir aussi

- [Architecture § Hébergement V1](architecture.md#hébergement-v1--raspberry-pi-4--podman).
