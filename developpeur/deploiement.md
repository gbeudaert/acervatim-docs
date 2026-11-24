---
title: Déploiement du backend
layout: default
parent: Développeur
nav_order: 5
---

# Déploiement du backend — pod Podman

Depuis le 23 novembre 2026, la production tourne sur la **constellation de micro-services** (dépôt
`acervatim-services`) : onze services dans un **pod Podman rootless** (unité **Quadlet** systemd),
derrière un reverse proxy TLS qui ne vise que le gateway. La base MariaDB est hébergée
**séparément** de l'hôte applicatif, avec un schéma par service. Un conteneur **Redis** local, hors
du pod, héberge les files **BullMQ**. Cette page décrit le *comment* du déploiement, indépendamment de
l'infrastructure qui l'héberge.

> Le **monolithe** (`acervatim-backend`), qui servait la production jusque-là, est arrêté et gardé
> pour un retour : voir [Retour au monolithe](#retour-au-monolithe).

## Topologie

```
        Internet
           │  HTTPS (TLS)
           ▼
   [ Reverse proxy ]
           │  http://<hôte applicatif>:3100   (seul port publié)
           ▼
┌───────────────────────────────────────────────┐        ┌─────────────────────────┐
│  Hôte applicatif (Podman rootless)            │        │  Serveur de base de     │
│  ┌─ pod acervatim ──────────────────────────┐ │  3306  │  données                │
│  │  migrate (initContainer, avant tout)     ├─┼───────▶│  MariaDB                │
│  │  gateway :3100 ─▶ dix autres services    │ │  mysql │  onze schémas `acv_*`   │
│  │  (joints en localhost, ports non publiés)│ │        └─────────────────────────┘
│  └──────────────────┬───────────────────────┘ │
│                     │ 6379, réseau `acervatim-services`
│  conteneur acervatim-services-redis (BullMQ)  │
└───────────────────────────────────────────────┘
```

| Élément | Choix |
|---|---|
| Services | onze conteneurs dans **un seul pod** : `gateway`, `identity`, `billing`, `connections`, `sharing`, `search`, `catalog-generic`, `catalog-manga`, `catalog-vinyl`, `catalog-lego`, `rebrickable-data`. Ils partagent la pile réseau du pod et se joignent en `localhost` |
| Port publié | **3100**, le gateway seul. Les routes `/internal/*` servent entre services et à l'exploitation : le reverse proxy ne doit pas les relayer |
| Conteneurisation | **Podman rootless**, Quadlet : `acervatim.kube` joue le manifeste du pod ; Redis a son unité `.container` à part ; réseau `acervatim-services` |
| Images | construites sur un poste de build pour l'architecture de l'hôte (sous émulation), livrées dans **une archive** avec le script de déploiement. L'hôte n'a ni sources ni outils de build |
| Base de données | **MariaDB distante**, un schéma `acv_*` par service sur la même instance, un seul compte SQL |
| Migrations Prisma | **initContainer** `migrate`, joué avant les services à chaque démarrage du pod. Un échec empêche le pod de démarrer : aucun service ne tourne sur un schéma à moitié migré |
| File d'attente | **Redis** local (conteneur `acervatim-services-redis`), aucun port publié sur l'hôte |
| Secrets | dans le manifeste joué `~/acervatim-services/config/acervatim.yaml` (droits 600), recomposé par le script, jamais dans un dépôt |
| Persistance du service | `loginctl enable-linger` : le pod survit à la fermeture de session et démarre avec l'hôte |

## Installation : une archive, un script

**Sur le poste de build**, depuis un commit étiqueté :

```powershell
./deploy/build-archive.ps1 -Version <version>
```

Le script construit les douze images (onze services et l'image de migration), prend Redis et un
client MariaDB dans la même architecture, contrôle l'architecture de chaque image, puis écrit l'archive
et `deploiement.sh` à côté.

**Sur l'hôte**, avec le compte qui fait tourner les services :

```sh
sh deploiement.sh acervatim-services-<version>-arm64.tar --jusqua 7   # à l'avance, sans toucher à la production
sh deploiement.sh acervatim-services-<version>-arm64.tar               # jusqu'au bout
sh deploiement.sh statut                                                 # à tout moment
```

Le script déroule douze étapes notées. Une relance reprend à la première qui n'est pas faite.

| Étapes | Ce qui se passe |
|---|---|
| 0 à 2 | vérifications de l'hôte, extraction de l'archive, chargement des images |
| 3 à 5 | secrets, schémas sur l'instance MariaDB, unités Quadlet et manifeste joué |
| 6 et 7 | démarrage de Redis puis du pod (migrations en initContainer), tests de fumée |
| 8 et 9 | reprise des données, écritures gelées, puis relecture au nom de chaque utilisateur |
| 10 | bascule du reverse proxy, à la main : le script dit quoi changer et attend `FAIT` |
| 11 | nettoyage, et mise de côté de l'ancienne unité |

Le détail de chaque étape est dans `deploy/README.md`, section 3, du dépôt
[`acervatim-services`](https://github.com/gbeudaert/acervatim-services/blob/main/deploy/README.md).
La reprise des données est décrite dans `db/reprise/README.md` du même dépôt.

## Vérification

```sh
systemctl --user status acervatim
podman pod ps
curl -s -o ready.json -w '%{http_code}\n' localhost:3100/health/ready   # attendu : 200
jq -e '.status == "ok"' ready.json                                     # attendu : true
```

`/health/ready` du gateway interroge la sonde de disponibilité des services qu'il route : la
constellation que l'app atteint est complète quand il répond `200` et `"status": "ok"`.

## Points d'attention

Vus au déploiement de production du 23 novembre 2026 :

- **Contrôleur mémoire des cgroups.** Le manifeste fixe des limites mémoire : un noyau démarré avec
  `cgroup_disable=memory` fait échouer le démarrage du pod. Il faut `cgroup_enable=memory` sur la
  ligne de commande du noyau.
- **Reverse proxy sur une autre machine.** Il vise l'adresse de l'hôte applicatif et le port 3100,
  pas `localhost`, et le pare-feu de l'hôte doit laisser passer ce port.
- **Journaux.** Les sorties des conteneurs vont au journal systemd :
  `journalctl CONTAINER_NAME=acervatim-<service>`, avec `sudo` si le compte n'a pas accès au journal
  système.
- **Ne pas couper un premier démarrage.** Après un redémarrage de l'hôte, le pod démarre seul et joue
  les migrations. Le relancer ou l'arrêter pendant une première migration la laisse « commencée sans
  fin », et Prisma refuse ensuite toute migration de ce schéma.
- **Mise à jour.** Le passage d'une version de la constellation à la suivante n'est pas encore
  outillé : le script refuse une archive quand une version est déjà installée.

## Retour au monolithe

Tant qu'aucun utilisateur n'a écrit depuis la bascule, `sh deploiement.sh retour` relance le
monolithe sur sa base, restée intacte, et rappelle de repointer le reverse proxy sur son port. Après
des écritures, un retour les perdrait. Le déploiement du monolithe lui-même (unité Quadlet, `.env`,
Redis) est décrit dans le dossier `deploy/` du dépôt `acervatim-backend`, et sa montée de version dans
[Mise à jour du backend](mise-a-jour.md).

## Source de vérité

Dépôt `acervatim-services`, dossier `deploy/` : `README.md` (procédure), `acervatim.kube` et
`acervatim.yaml` (unité et manifeste commentés), `deploiement.sh` et `build-archive.ps1`.

## Voir aussi

- [Architecture § Déploiement](architecture.md#déploiement).
- [Mise à jour du backend](mise-a-jour.md) — montée de version du monolithe.
