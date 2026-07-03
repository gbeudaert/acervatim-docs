# acervatim-backend — Opérations & développement

> Remonter : [INDEX.md](INDEX.md) · Référence complète : `acervatim-backend/CLAUDE.md` (§ Commandes, § Workflow Prisma)

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
- E2E : `*.e2e.spec.ts` (auth, invitations, items, users).

## Déploiement

- Cible : conteneur **Podman** (unité Quadlet systemd) derrière un reverse proxy TLS ; MariaDB hébergée séparément (pas de conteneur DB en prod).
- Matériel de déploiement : `acervatim-backend/deploy/` ; doc humaine : [../../developpeur/deploiement.md](../../developpeur/deploiement.md).
- DB scripts : `acervatim-backend/db/` ; builds : `dist/`.

## Jobs planifiés (cron applicatifs NestJS)

| Job | Fichier | Rôle |
|---|---|---|
| Réconciliation abonnements | `src/subscriptions/subscriptions.cron.ts` | Re-vérifie les subs proches de l'expiration (index `[status, expiresAt]`) |
| Purge cache / rate-limit | via `common/cache` / `common/rate-limit` | Suppression des rows expirées (pas de TTL natif MariaDB) |

## Pièges PowerShell connus

PS 5.1 : pas de `&&`/`||` (utiliser `; if ($?) { … }`) ; UTF-8 sans BOM via .NET (pas `Out-File -Encoding utf8`) ; sources `.ps1` en ASCII pur ; toujours le wrapper plutôt que `docker compose` direct.
