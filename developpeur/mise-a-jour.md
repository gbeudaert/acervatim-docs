---
title: Mise à jour du backend
layout: default
parent: Développeur
nav_order: 6
---

# Mise à jour du backend — montée de version

Cette page décrit la **procédure de montée de version** du monolithe : ce qu'on construit, dans
quel ordre on l'applique, et comment on revient en arrière.

> Depuis le 23 novembre 2026, la production tourne sur la constellation de micro-services, décrite
> dans [Déploiement du backend](deploiement.md). Le monolithe est arrêté et gardé pour un retour :
> cette procédure ne vaut plus que pour lui. Pour la constellation, le passage d'une version à la
> suivante n'est pas encore outillé ; ses migrations tournent dans un initContainer à chaque
> démarrage du pod.

Le principe tient en trois points :

1. Les images sont construites **ailleurs que sur l'hôte de production**, puis transférées sous forme
   d'archive OCI (`docker save` / `podman load`). L'hôte applicatif n'a ni toolchain ni sources.
2. Les **migrations Prisma ne sont pas jouées par le conteneur d'API** : le CLI `prisma` est une
   devDependency, absente de l'image runtime. Elles sont jouées **une fois**, par une image dédiée,
   **API arrêtée**.
3. Le schéma se migre **en avant uniquement**. Prisma ne génère pas de migration descendante : le
   retour arrière d'un déploiement qui a touché au schéma passe par une **restauration de dump**.

## Les deux images

| Image | Cible du `Dockerfile` | Rôle | Ordre de grandeur |
|---|---|---|---|
| `acervatim-api` | `--target runtime` | Service au long cours : `dist/` + `node_modules` de production, non-root, healthcheck `/health`. C'est elle que référence l'unité Quadlet | ~500 Mo |
| `acervatim-migrate` | `--target migrate` | **One-shot**, jamais un service : contient le CLI `prisma`, le schéma et le dossier `migrations/`. Sa commande est `npx prisma migrate deploy` | ~1,3 Go |

Les deux sortent du **même `Dockerfile`** et de la même étape de build : le client Prisma généré dans
l'image de migration est exactement celui qu'embarque le runtime. Construire les deux au même moment,
depuis le même arbre de sources, est ce qui garantit que le schéma appliqué et le code qui l'attaque
sont d'accord.

> **Architecture.** L'hôte applicatif est en `linux/arm64` ; le poste de build ne l'est pas
> forcément. Le client Prisma embarque un moteur natif : on construit **pour l'architecture de
> déploiement** (`--platform linux/arm64`, émulation QEMU/binfmt côté build), jamais en copiant une
> image d'une autre architecture.

## Procédure

### 0. Pré-vol

- Lire les notes de la version visée dans le `CHANGELOG.md` du dépôt `acervatim-backend` :
  **nouvelles variables d'environnement**, ruptures d'API, migrations destructrices, compatibilité
  avec les versions de l'application. Les notes propres à une version vivent là, pas sur cette page.
- La liste faisant foi des variables exigées au démarrage est le schéma de validation d'env du
  backend (`src/config/env.validation.ts`) — le service **refuse de démarrer** s'il en manque une.
  Compléter le `.env` **avant** de basculer.
- Vérifier la place disque sur l'hôte : les archives sont volumineuses et `podman load` recopie.

### 1. Construire (poste de build)

```sh
docker build --platform linux/arm64 --target runtime -t acervatim-api:<version> .
docker build --platform linux/arm64 --target migrate -t acervatim-migrate:<version> .
```

Le second réutilise le cache du premier : seule la première construction paie l'émulation.

### 2. Exporter et transférer

```sh
docker save -o acervatim-api-<version>-arm64.tar     acervatim-api:<version>
docker save -o acervatim-migrate-<version>-arm64.tar acervatim-migrate:<version>
scp acervatim-*-<version>-arm64.tar <hôte>:~/
```

Ces archives sont des **artefacts locaux, jamais versionnés** (`*.tar` est ignoré par git et exclu du
contexte de build).

### 3. Charger sur l'hôte

```sh
podman load -i acervatim-api-<version>-arm64.tar
podman load -i acervatim-migrate-<version>-arm64.tar
podman images | grep acervatim
```

### 4. Arrêter l'API

```sh
systemctl --user stop acervatim-api
```

Non négociable dès qu'une migration touche le schéma : l'ancien code écrirait contre une base en
cours de transformation, et une migration destructrice supprimerait des lignes qu'il vient d'écrire.
Redis peut rester debout — les jobs sont idempotents et rejouables.

### 5. Sauvegarder la base

```sh
mariadb-dump --single-transaction --routines --databases acervatim > acervatim-pre-<version>.sql
```

C'est le **seul** chemin de retour arrière sur le schéma. À faire même quand la migration paraît
anodine, et **obligatoirement** quand les notes de version signalent une suppression de données.

### 6. Jouer les migrations

```sh
podman run --rm --env-file ~/acervatim/.env acervatim-migrate:<version>
```

`prisma migrate deploy` est **non interactif et idempotent** : il applique, dans l'ordre, les
migrations absentes de la table `_prisma_migrations`, et ne fait rien si la base est déjà à jour. Il
ne réclame **pas** de shadow database (contrairement à `migrate dev` en développement) : le compte de
la base de production n'a pas besoin du droit de créer des bases.

Sortie attendue : la liste des migrations appliquées, puis `All migrations have been successfully
applied`. **Ne pas enchaîner sur l'étape suivante si elle échoue** — voir *En cas d'échec* plus bas.

### 7. Basculer le tag et redémarrer

L'unité Quadlet référence `localhost/acervatim-api:latest` : la bascule est un re-tag.

```sh
podman tag localhost/acervatim-api:<version> localhost/acervatim-api:latest
systemctl --user restart acervatim-api
```

### 8. Vérifier

```sh
systemctl --user status acervatim-api
podman logs --tail 20 acervatim-api      # doit afficher : acervus-backend v<version> démarré
curl -s localhost:3000/health            # { "status": "ok", "redis": "up", ... }
podman ps                                # colonne STATUS : (healthy)
```

Le log de version au démarrage est la vérification qui compte : il prouve que le conteneur qui tourne
est bien l'image qu'on vient de charger, et non l'ancienne restée sous `:latest`.

Puis un aller-retour applicatif réel depuis l'app (connexion, lecture d'une collection, un push) avec
un compte de test — le healthcheck ne dit rien de la base.

### 9. Nettoyer (plus tard)

Garder l'image précédente **au moins jusqu'à validation** : c'est la moitié « code » du rollback.
Ensuite seulement, `podman image prune` et suppression des archives transférées.

## Retour arrière

| Situation | Manœuvre |
|---|---|
| Migrations **pas encore jouées** | Re-taguer `:latest` sur l'image précédente, redémarrer. Rien d'autre à faire |
| Migrations jouées, **schéma additif** (aucune colonne ni table supprimée) | Idem : l'ancien code ignore ce qu'il ne connaît pas |
| Migrations jouées, **schéma destructif ou données supprimées** | Restaurer le dump de l'étape 5 (`mariadb acervatim < acervatim-pre-<version>.sql`), **puis** re-taguer et redémarrer. Toute écriture postérieure à la migration est perdue |

### En cas d'échec d'une migration

Prisma marque la migration en échec dans `_prisma_migrations` et **refuse toute exécution ultérieure**
tant que l'état n'est pas tranché. Deux issues, à choisir en connaissance de cause :

- corriger la cause et rejouer `migrate deploy` après avoir marqué la migration comme annulée
  (`npx prisma migrate resolve --rolled-back <nom>`) — la base doit vraiment avoir été remise dans
  l'état d'avant, ce que la restauration du dump garantit ;
- déclarer la migration appliquée (`--applied <nom>`) **seulement** après avoir vérifié en base que
  son effet est bel et bien présent.

## Voir aussi

- [Déploiement du backend](deploiement.md) — topologie, unités Quadlet, installation initiale.
- [Publication Play Store](publication-play-store.md) — le versant app d'une livraison.
