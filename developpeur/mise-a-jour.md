---
title: Mise à jour du backend
layout: default
parent: Développeur
nav_order: 6
---

# Mise à jour du backend — montée de version

Cette page décrit la **procédure de montée de version** d'un backend déjà en service : ce qu'on
construit, dans quel ordre on l'applique, et comment on revient en arrière. L'installation initiale
et la topologie sont décrites dans [Déploiement du backend](deploiement.md) ; on ne les répète pas.

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

- Lire les notes de la version visée (annexes en bas de page : [0.8.1](#annexe--081), [0.8.0](#annexe--080)) :
  **nouvelles variables d'environnement**, ruptures d'API, migrations destructrices.
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

## Annexe — 0.8.1

Contenu : **déblocage de la synchronisation manga côté serveur**. Une série peut désormais naître
d'un `unifiedData` client seul, et un tome se rattache à sa série par `nodeId`. Correctif au passage :
un `unifiedData` invalide sur `PATCH /v1/nodes/:id` renvoyait **500** au lieu de 400.

Artefact : `acervatim-api-0.8.1-arm64.tar` (image 497 Mo), `linux/arm64`, Node 24.
**Pas d'image de migration** — voir ci-dessous.

### Ce qui change pour l'exploitation

| Point | Détail |
|---|---|
| **Nouvelle variable d'env** | Aucune. Le `.env` de 0.8.0 convient tel quel |
| **Migrations** | **Aucune.** `prisma/schema.prisma` est inchangé : `items.node_id` et `items.volume` étaient déjà nullables, et l'unique `[user_id, node_id, volume]` gérait déjà le cas `NULL` |
| **Donnée supprimée** | Non |
| **Infrastructure** | Inchangée |

Conséquence pratique : la montée de version se réduit aux étapes **1-4** et **7-9** de la procédure.
Les étapes 5 et 6 (dump, migrations) n'ont rien à faire ici, et le retour arrière est un simple
re-tag de `:latest` sur `acervatim-api:0.8.0`.

> **Une seule réserve** : ceci ne vaut que si **0.8.0 est déjà en service**. Si l'on saute
> directement de 0.7.x à 0.8.1, les trois migrations de 0.8.0 restent dues — il faut alors construire
> `acervatim-migrate:0.8.1`, dumper la base et suivre la procédure complète, en lisant l'annexe 0.8.0
> (dont la **purge des partages** et la variable `SHARE_CODE_PEPPER`).

### Ruptures d'API

Aucune. Les trois changements de contrat sont **additifs** : un client qui ne les connaît pas
continue d'envoyer ce qu'il envoyait.

| Endpoint | Extension |
|---|---|
| `POST /v1/collections/:id/nodes` | `source` devient optionnel, `unifiedData` apparaît ; au moins l'un des deux. Un `unifiedData` seul crée une série sans référence provider — le chemin de la saisie manuelle |
| `POST /v1/collections/:id/items` | `nodeId` (série déjà créée) en alternative exclusive à `node` (référence provider). Sur un type hiérarchique, série et numéro de tome ne sont plus exigés : un tome orphelin ou hors numérotation est accepté |
| `PATCH /v1/items/:id` | Accepte `nodeId` et `volume` (nullables) — l'identité structurelle d'un tome devient modifiable, sans quoi un changement de série resterait local |

Un plafond technique s'ajoute côté nœuds (20 000 par compte, `409` `/probs/technical-limit`), du même
ordre que ceux qui existent sur les collections et les items : la création d'un nœud n'est plus
adossée à un appel provider, donc plus freinée par rien. Un usage humain ne le voit jamais.

### Compatibilité avec l'app

Le serveur ne bloque plus, mais **l'app ne pousse toujours pas les mangas** : `SyncManagerImpl` filtre
encore sur `VINYL` (`skip push item …: type MANGA non supporté en V1`). Déployer 0.8.1 ne change donc
rien de visible tant que le versant app n'est pas livré — c'est un déploiement de **préparation**, à
faire avant la build app qui l'exploitera, pas après.

La réserve de 0.8.0 sur l'UI de partage du sprint A3 (incompatible, à ne pas publier) reste entière.

## Annexe — 0.8.0

Contenu : synchronisation **premium-only**, statut d'item synchronisé, et **partage de collection**
(émission de codes, lecture partagée filtrée par statuts, partage multi-collections).

Artefacts : `acervatim-api-0.8.0-arm64.tar` (~512 Mo, image 497 Mo) et
`acervatim-migrate-0.8.0-arm64.tar` (~1,42 Go, image 1,38 Go), tous deux `linux/arm64`, Node 24.

### Ce qui change pour l'exploitation

| Point | Détail |
|---|---|
| **Nouvelle variable d'env obligatoire** | `SHARE_CODE_PEPPER` (≥ 32 caractères, aléatoire, distinct d'`INVITE_CODE_PEPPER`). **Sans elle, l'API refuse de démarrer.** Comme les autres peppers, elle n'est **pas rotable** : la changer invaliderait tous les codes de partage en circulation |
| **Migrations** | Trois, jouées d'affilée par l'image de migration |
| **Donnée supprimée** | Oui — voir ci-dessous. Le dump de l'étape 5 est obligatoire |
| **Infrastructure** | Inchangée : mêmes unités Quadlet, même Redis, mêmes ports |

### Les trois migrations

| Migration | Effet |
|---|---|
| `20260821182330_add_collection_shares` | Crée `collection_shares` et `collection_share_members`. Purement additif |
| `20260822114550_shares_multi_collections_statuses` | Fait passer le modèle de « un partage = une collection + une portée » à « un partage = N collections, chacune avec ses statuts » : crée `collection_share_entries`, **supprime** les colonnes `collection_id` et `scope` de `collection_shares`, ajoute les colonnes `label` |
| `20260822114900_purge_pre_s5_shares` | **`DELETE FROM collection_shares`** — rupture nette assumée : la migration précédente a privé les partages antérieurs de toute collection, leur code n'accorderait plus qu'une adhésion vide. Membres et entrées cascadent |

La purge est sans conséquence si l'on part d'une base où la fonctionnalité n'a jamais tourné (aucune
de ces tables n'existait avant 0.8.0) : les trois migrations s'appliquent alors d'une traite sur une
base vierge de partages.

### Ruptures d'API

| Rupture | Conséquence côté client |
|---|---|
| `GET /v1/me/quota` **retiré** | `404`. Le palier gratuit plafonné n'existe plus ; ce qui subsiste est un plafond technique |
| **Sync réservée au premium** | `collections` / `nodes` / `items` répondent `402` `/probs/payment-required` à un compte non premium. Un compte gratuit ne synchronise plus — l'app reste pleinement utilisable hors-ligne |
| `POST` / `GET /v1/collections/:id/shares` **n'existent plus** | Remplacés par `/v1/shares` ; `scope` a disparu du contrat au profit d'une liste de statuts par collection |

Le catalogue complet des routes de partage est dans la [référence API](api-reference.md).

### Compatibilité avec l'app

Le backend 0.8.0 se déploie **indépendamment** de l'app : rien n'exige de publier une nouvelle version
en même temps. Deux réserves :

- une build antérieure aux sprints A1/A2 continue de fonctionner pour un compte premium, mais un
  compte gratuit voit sa synchronisation s'arrêter (`402`) et son relevé de quota disparaître (`404`) ;
- l'UI **propriétaire** de partage livrée au sprint A3 parle aux routes `/v1/collections/:id/shares`
  supprimées : elle est inutilisable contre 0.8.0 et **ne doit pas être publiée** telle quelle. Sa
  reprise sur le nouveau contrat (A3bis) est en cours.

## Voir aussi

- [Déploiement du backend](deploiement.md) — topologie, unités Quadlet, installation initiale.
- [Publication Play Store](publication-play-store.md) — le versant app d'une livraison.
