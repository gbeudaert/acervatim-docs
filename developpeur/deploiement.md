---
title: Déploiement du backend
layout: default
parent: Développeur
nav_order: 5
---

# Déploiement du backend — conteneur Podman

Le backend est déployé en **conteneur** (Podman rootless, unité **Quadlet** systemd), derrière un
reverse proxy TLS. La base MariaDB est hébergée **séparément** du conteneur applicatif (pas de
conteneur DB en production). Cette page décrit le *comment* du déploiement, indépendamment de
l'infrastructure qui l'héberge.

## Topologie

```
        Internet
           │  HTTPS (TLS)
           ▼
   [ Reverse proxy ]
           │  http://localhost:3000
           ▼
┌───────────────────────────┐        ┌─────────────────────────┐
│  Hôte applicatif          │        │  Serveur de base de     │
│  Podman rootless          │  3306  │  données                │
│  └─ conteneur acervatim-  │───────▶│  MariaDB 10.5+          │
│     api (NestJS + Prisma) │  mysql │  (base `acervatim`)     │
│     géré par Quadlet      │        │                         │
└───────────────────────────┘        └─────────────────────────┘
```

| Élément | Choix |
|---|---|
| Runtime | Node 24 LTS |
| Conteneurisation | **Podman rootless**, unité **Quadlet** systemd (`acervatim-api.container`) |
| Image | construite sur l'hôte de déploiement (`podman build`) |
| Base de données | **MariaDB distante** (pas de conteneur DB en prod) ; `DATABASE_URL` pointe le serveur de base |
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
base distante :

```sh
podman build --target migrate -t acervatim-migrate .
podman run --rm --env-file ~/acervatim/.env acervatim-migrate
```

## Points d'attention

- `RunInit=true` (Podman `--init`) : `main.ts` n'a pas de handler `SIGTERM` ; sans init, Node en PID 1
  ignore le signal et l'arrêt attend le `SIGKILL`.
- `After=network-online.target` + `Restart=always` : la base est **distante**, le réseau doit être
  prêt ; `Restart=always` couvre le cas où la base n'est pas encore joignable au démarrage de l'hôte.
- `TimeoutStartSec=90` : laisse le temps au premier `prisma connect` (base distante) + au bootstrap Nest.

## Source de vérité

`deploy/acervatim-api.container` (unité Quadlet commentée pas à pas) dans le dépôt
`acervatim-backend`.

## Voir aussi

- [Architecture § Déploiement](architecture.md#déploiement).
