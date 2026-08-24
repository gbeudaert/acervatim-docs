# acervatim-backend — Opérations & développement

> Remonter : [INDEX.md](INDEX.md) · Règles dures : `acervatim-backend/AGENTS.md` (§ Running and verifying)

## Règle dure : le wrapper `scripts/dev.ps1`

`npm`/`npx`/`node`/`prisma`/`nest` ne se lancent **jamais** directement sous Windows (binaires Prisma/natifs incompatibles conteneur). Tout passe par `acervatim-backend/scripts/dev.ps1` :

```powershell
./scripts/dev.ps1 up | up-bg | down | reset      # stack Docker Compose (reset = drop db_data)
./scripts/dev.ps1 migrate <nom>                  # migration Prisma (crée + applique + regénère client)
./scripts/dev.ps1 generate                       # régénère le client Prisma
./scripts/dev.ps1 npm <args> | npx <args> | exec <cmd>   # passe-plats dans le conteneur app
./scripts/dev.ps1 db-shell | shell               # CLI MariaDB / shell conteneur
./scripts/dev.ps1 setup-hooks                    # pre-commit Prettier (une fois après clone)
```

URLs dev : app `http://localhost:3000` · Adminer `http://localhost:8080`.

## Workflow Prisma

1. Modifier `prisma/schema.prisma`.
2. `./scripts/dev.ps1 migrate <nom-court>`.
3. Client regénéré automatiquement.
4. Drift en dev : `./scripts/dev.ps1 npx prisma migrate reset --force --skip-seed` puis re-migrate.
5. Seed : `./scripts/dev.ps1 npx prisma db seed` (`prisma/seed.ts` — types de collection).

## Tests

- Unit : `*.spec.ts` co-localisés → `./scripts/dev.ps1 npm test` (ou passe-plat équivalent).
- E2E : `*.e2e.spec.ts` (auth, invitations, items, users, sharing).

> **Les e2e exigent `NODE_ENV=test`.** Le throttler global (100 req/min/IP) porte
> `skipIf: () => process.env.NODE_ENV === 'test'` ; le conteneur de dev, lui, tourne en
> `NODE_ENV=development`. Lancer jest dedans sans surcharger la variable fait échouer les e2e en
> **429 dès le `login()`** — un échec qui ressemble à une régression fonctionnelle et n'en est pas :
>
> ```
> docker compose exec -T -e NODE_ENV=test app npx jest src/sharing --runInBand
> ```

## Piège : le conteneur de dev ne recharge pas toujours le code

`docker-compose.yml` monte le source en bind (`.:/app`) et lance `npm run start:dev` (watch Nest).
Sur Docker Desktop / Windows, les événements inotify **ne traversent pas le montage** de façon
fiable : le process peut servir un build vieux de plusieurs heures alors que le source sur disque est
à jour. Symptôme typique — un client qui appelle une route qu'on vient d'ajouter reçoit un `404`
« Cannot POST /v1/… » alors que le controller existe.

Diagnostic en une commande, sans lire les logs :

```
curl -s http://localhost:3000/docs-json | jq -r '.paths | keys[]'
```

Si la route manquante n'y est pas, c'est le process, pas le code : `docker compose restart app`
(≈50 s avant que `/health` réponde). Les migrations, elles, sont indépendantes — vérifier séparément
avec `docker compose exec -T app npx prisma migrate status`.

## Déploiement

- Cible : conteneur **Podman** (unité Quadlet systemd) derrière un reverse proxy TLS ; MariaDB hébergée séparément (pas de conteneur DB en prod).
- Matériel de déploiement : `acervatim-backend/deploy/` ; doc humaine : [../../developpeur/deploiement.md](../../developpeur/deploiement.md).
- Montée de version d'un service déjà en place (deux images `--target runtime` / `--target migrate`,
  ordre des opérations, retour arrière) : [../../developpeur/mise-a-jour.md](../../developpeur/mise-a-jour.md).
  Les images se construisent hors de l'hôte (`--platform linux/arm64`) et se transfèrent en `.tar` ;
  les migrations ne sont **jamais** jouées par le conteneur d'API (CLI `prisma` = devDep).
- DB scripts : `acervatim-backend/db/` ; builds : `dist/`.

## Jobs planifiés (cron applicatifs NestJS)

| Job | Fichier | Rôle |
|---|---|---|
| Réconciliation abonnements | `src/subscriptions/subscriptions.cron.ts` | Re-vérifie les subs proches de l'expiration (index `[status, expiresAt]`) |
| Purge cache / rate-limit | via `common/cache` / `common/rate-limit` | Suppression des rows expirées (pas de TTL natif MariaDB) |

## Pièges PowerShell connus

PS 5.1 : pas de `&&`/`||` (utiliser `; if ($?) { … }`) ; UTF-8 sans BOM via .NET (pas `Out-File -Encoding utf8`) ; sources `.ps1` en ASCII pur ; toujours le wrapper plutôt que `docker compose` direct.
