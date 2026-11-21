# Étude : une archive et un script pour déployer la constellation

> Document de travail, exclu du site. Rédigé le 2026-11-21, à la demande de Guillaume : « j'importe
> le tout, je lance le script avec le chemin de l'archive et le script déroule par étape ».
> Les six décisions ont été prises le 2026-11-21 ; l'archive, le script et la répétition sous WSL
> ont été faits le même jour, et l'étiquette de la première version déplacée sur eux (sections 8
> et 9).

## Suivi

| Décision | Retenu | État |
|---|---|---|
| D1. Où vit la base de la constellation | Sur l'instance MariaDB du monolithe : onze schémas `acv_*` à côté de `acervatim`, aucun conteneur MariaDB | tranché le 2026-11-21 |
| D2. D'où viennent les secrets | Tous ceux du monolithe, compte SQL compris ; seuls les secrets internes sont générés | tranché le 2026-11-21 |
| D3. Bascule du reverse proxy | Étape manuelle : le script s'arrête, affiche quoi changer, attend la confirmation | tranché le 2026-11-21 |
| D4. Répétition avant l'hôte | Sous Podman dans WSL, avec une archive amd64, dans une distribution dédiée (section 8) | tranché le 2026-11-21, répétition faite |
| D5. La reprise des données dans le script | Oui, en mode bascule seulement : écritures gelées, API du monolithe arrêtée | tranché le 2026-11-21 |
| D6. Dossier de déploiement sur l'hôte | `~/acervatim-services/`, pas `~/acervatim/` (celui du monolithe) | tranché le 2026-11-21 |

---

## 1. Ce qui existe déjà

- **Images** : `deploy/build-images.ps1` construit les douze images (onze services et l'image de
  migration). Il accepte `-Platform linux/arm64`. Le manifeste `deploy/acervatim.yaml` épingle leur
  version.
- **Construction arm64 sur ce poste** : Docker Desktop exécute des conteneurs arm64 par émulation
  (vérifié le 2026-11-21 : `uname -m` rend `aarch64`). Une image construite avec
  `--platform linux/arm64` embarque le moteur Prisma arm64. Le piège documenté est l'inverse :
  construire en amd64 puis copier sur l'hôte.
- **Le monolithe fait déjà ainsi** : construction arm64 ici, `docker save`, `scp`, `podman load`
  sur l'hôte, qui n'a ni sources ni outils de build (`developpeur/mise-a-jour.md`).
- **Schémas** : `db/init/01-schemas.sql` crée les schémas ; les migrations des onze schémas tournent
  en initContainer du pod (`deploy/migrate.sh`). Un échec empêche le pod de démarrer.
- **Reprise** : `db/reprise/reprise.sh` (dump, load, transform, verify, registry, clean,
  reset-targets) et `db/reprise/test/parcours.js` (relecture par le gateway, avec des jetons signés
  pour les utilisateurs repris). La reprise refuse un monolithe dont les migrations ne sont pas
  exactement les 8 connues ; ce sont celles de la production (vérifié le 2026-11-21).
- **Jamais exécuté** : aucun `podman kube play`, aucune image construite sous Podman ou en arm64,
  aucune migration contre la base de l'hôte (`deploy/README.md`, section 7). La première exécution
  du script sera donc aussi le premier essai de tout cela.

## 2. Contenu de l'archive

Un dossier à copier sur l'hôte, `acervatim-services-<version>-arm64/`, qui contient l'archive
`acervatim-services-<version>-arm64.tar` (non recompressée : les images, qui en font presque tout
le poids, le sont déjà) et, à côté, `deploiement.sh`, le même que dans l'archive. Sur l'hôte :
`sh deploiement.sh acervatim-services-<version>-arm64.tar`.

```
acervatim-services-<version>/
├── MANIFEST              version, commit, brouillon ou non, architecture, images, place requise
├── SHA256SUMS            empreinte de chaque fichier, au format de sha256sum -c
├── CHANGELOG.md
├── deploiement.sh        le script (section 4)
├── images.tar.gz         un seul `docker save` : les douze images, Redis et un client MariaDB
├── deploy/               unités Quadlet, manifeste, gabarit de secrets
│   ├── outils/           secrets.js (composition des secrets), controle-image.js
│   └── tests/            fumee.sh et relecture.js (section 5)
└── db/                   init/ (liste des schémas), reprise/ (reprise.sh, sql/)
```

Redis et le client MariaDB viennent de Docker Hub, pris par l'empreinte de leur variante arm64 et
renommés `localhost/acervatim-tiers/…` : l'hôte n'a rien à télécharger. Le client sert aux étapes
4, 8 et 9, l'hôte n'en ayant pas (la base du monolithe est distante).

Aucun secret dans l'archive : le gabarit seulement. Les fichiers viennent de git, pas de l'arbre
de travail : fins de ligne LF, et aucun fichier ignoré n'y entre. L'archive peut donc transiter
et être gardée comme les archives du monolithe (artefact local, jamais versionné).

## 3. Construction de l'archive, sur ce poste

`deploy/build-archive.ps1 -Version <version> [-Architecture arm64|amd64] [-Brouillon]
[-SansConstruction]` :

1. **Refuse** un arbre de travail modifié, ou un commit sans l'étiquette `v<version>` : une archive
   correspond toujours à une version étiquetée et à rien d'autre. Les douze `package.json` et les
   images épinglées par le manifeste doivent porter la même version. `-Brouillon` lève les deux
   premières conditions pour une répétition : l'archive est marquée, et `deploiement.sh` le dit.
2. Construit les douze images (`build-images.ps1`), en `linux/arm64` par défaut : sous émulation,
   donc plusieurs fois plus lent qu'en natif ; durée à mesurer au premier essai. Redis et le client
   MariaDB sont tirés par l'empreinte de leur variante, pour ne pas remplacer les images amd64
   locales du même nom (stockage d'images classique de Docker Desktop).
3. **Contrôle chaque image** : son `node` tourne dans l'architecture cible, et chaque moteur Prisma
   qu'elle porte est un binaire de cette architecture. Le contrôle lit l'en-tête ELF au lieu
   d'exécuter le moteur : sous émulation, un binaire amd64 s'exécuterait quand même. C'est le
   contrôle qui manquait au piège documenté.
4. Un seul `docker save` des quatorze images, compressé (`images.tar.gz`), les fichiers tirés de
   git, le `MANIFEST`, les empreintes (`SHA256SUMS`), puis l'archive.

**Taille**, relevée sur la construction amd64 du 2026-11-21 (l'arm64 devrait être du même ordre) :
un `docker save` des douze images fait **4,3 Gio brut, 1,3 Gio compressé** (gzip). Les couches
communes ne sont stockées qu'une fois, mais chaque service a ses propres dépendances Node (environ
500 Mo par image, 1,4 Go pour l'image de migration). En gardant `images.tar` compressé dans
l'archive, `podman load` le lit tel quel : l'hôte a besoin d'environ 1,3 Gio pour l'archive et
4,3 Gio pour les images chargées, soit **6 Gio libres** avec une marge, sans compter les images de
la version précédente gardées pour le retour arrière. Alléger les images est possible mais hors du
périmètre de cette étude.

## 4. Le script `deploiement.sh <archive>`

### Principes

- **POSIX `sh`**, lancé par l'utilisateur rootless qui fait tourner les unités, jamais en root.
- **Une étape à la fois**, chacune avec un contrôle de sortie. Une étape réussie est notée dans
  `~/acervatim-services/etat-<version>` ; relancer le script reprend à la première étape non faite.
  `--jusqua <n>` s'arrête après l'étape n, `--depuis <n>` rejoue à partir de l'étape n,
  `--simulation` ne joue que les vérifications.
- **Les étapes 0 à 7 ne touchent pas au monolithe** : elles peuvent être jouées des jours avant
  (`--jusqua 7`). Elles éprouvent sur l'hôte l'arm64, Podman, Quadlet et l'instance de la base,
  constellation démarrée sur des schémas vides et proxy inchangé. Le jour de la bascule, le script
  reprend à l'étape 8.
- **Confirmation tapée** avant chaque geste irréversible : `GELER` avant d'arrêter le monolithe,
  `FAIT` après la bascule du proxy, `SUPPRIMER` avant une désinstallation.
- **Journal** horodaté de chaque exécution dans `~/acervatim-services/journal/`. Aucun secret
  affiché ni journalisé, aucun mot de passe sur une ligne de commande : fichiers en `600`, tubes,
  et `-e MYSQL_PWD` pour les clients SQL (même règle que `reprise.sh`).
- **Refus d'emblée** (étape 0) si l'architecture n'est pas celle de l'archive, si le script tourne
  en root, si Podman (4.4 au moins, pour Quadlet), `systemd --user` ou le linger manquent, si le
  port 3100 est pris ou si la place ou la mémoire ne suffisent pas.

### Les étapes

| N° | Étape | Contrôle de sortie | Réversible |
|---|---|---|---|
| 0 | **Vérifications** : architecture, utilisateur, Podman et Quadlet, `systemd --user`, linger, place disque (sous `$HOME` et sous le stockage de Podman), mémoire disponible, port 3100, monolithe sur 3000 | tout vert, sinon arrêt avec la cause | rien n'est fait |
| 1 | **Extraction** dans `~/acervatim-services/<version>/`, sauf les images, qui restent dans l'archive ; empreintes de `SHA256SUMS`, images comprises | empreintes identiques | oui |
| 2 | **Chargement des images** : `podman load` lit les images dans l'archive ; Redis garde l'image du monolithe si elle est là | les quatorze références présentes, dans l'architecture du `MANIFEST` | oui |
| 3 | **Secrets** (D2) : lus dans l'environnement du conteneur `acervatim-api` en service (ce qu'il voit réellement), repli sur `~/acervatim/.env` ; compte SQL repris, onze `DATABASE_URL_*` dérivées de sa `DATABASE_URL` ; `INTERNAL_*` et le pepper GameUPC générés ; clé GameUPC de production demandée ; `CORS_ORIGINS` et `HTTP_USER_AGENT` du monolithe reportés dans le ConfigMap | chaque `secretKeyRef` a sa clé ; fichiers en `600` dans `config/` | oui |
| 4 | **Schémas** (D1) sur l'instance du monolithe, avec son compte. S'il n'en a pas le droit, le script affiche le SQL (onze `CREATE DATABASE`, un `GRANT` sur `` `acv\_%` ``, jamais `ON *.*`) et le joue avec un compte d'administration saisi, ou attend qu'on le joue | onze schémas créés, aucun occupé ; le compte crée et supprime `acv_legacy` | oui (schémas vides) |
| 5 | **Unités** Quadlet (`acervatim.kube`, Redis, réseau ; jamais celles du monolithe), manifeste joué composé avec l'adresse de la base, `daemon-reload` | `systemctl --user cat acervatim` répond | oui |
| 6 | **Démarrage** de Redis puis du pod : les migrations tournent en initContainer | `/health/ready` à 200 et `ok` sous 4 minutes | oui (arrêt du pod) |
| 7 | **Tests de fumée** sans données (section 5) | tous verts | rien n'est fait |
| 8 | **Reprise** (D5) : empreintes des secrets critiques comparées entre monolithe et constellation, files du monolithe vides, `GELER`, arrêt de `acervatim-api`, vidage, `reprise.sh all` puis `registry` ; chaque sous-étape notée | `reprise conforme`, registre sans écart | oui jusqu'à l'étape 10 (`retour`) |
| 9 | **Relecture** : `relecture.js`, au nom de chaque utilisateur repris, à travers le gateway | chacun retrouve ses collections et leurs items | rien n'est fait |
| 10 | **Bascule du reverse proxy** (D3) : le script affiche quoi changer, attend `FAIT`, puis vérifie que l'URL publique répond par la constellation | `/health/ready` public servi par le gateway | oui tant qu'aucun utilisateur n'a écrit |
| 11 | **Après** : `acv_legacy` supprimé ; unité du monolithe mise de côté (sur `OUI`), son image, son `.env` et sa base restent ; vidage rappelé, à détruire à la fin de la fenêtre de retour | — | — |

Sous-commandes à côté du déroulé : `statut` (étapes faites, état des unités), `retour` (retour au
monolithe, section 6) et `desinstaller` (avant la bascule ou après un retour : pod, unités, Redis
et son volume, images, schémas `acv_*` ; jamais le monolithe, son dossier ni sa base).

**Versions suivantes.** Pas encore pris en charge : le script refuse une archive d'une autre
version que celle installée. La mise à jour reprendrait les étapes 0, 1, 2, puis le manifeste joué
recomposé, le redémarrage du pod (les migrations nouvelles tournent en initContainer) et les tests
de fumée, la version précédente restant chargée pour le retour arrière.

## 5. Les tests

- **Sans données** (étape 7, `deploy/tests/fumee.sh`, `curl` et Podman seulement) : `/health` et
  `/health/ready` du gateway, refus `401` sans jeton, avec un jeton invalide et sur une route
  admin, `GET /v1/admin/queues` avec `ADMIN_API_TOKEN`, spécification OpenAPI agrégée, sonde de
  disponibilité de chacun des dix services derrière le gateway (de l'intérieur du pod), Redis.
- **Sur données** (étape 9, `deploy/tests/relecture.js`) : pour chaque utilisateur du vidage,
  `/v1/me`, toutes ses collections par `/v1/collections` (pagination comprise), et le nombre
  d'items de chacune, comparés à `acv_legacy`, donc au monolithe. Il tourne dans le conteneur
  `identity`, qui a la clé de signature. `parcours.js` reste le test du jeu de répétition : il
  vise des identifiants fixes et ne sert pas sur des données réelles.
- **Ce qui n'est pas testé** : le comportement de l'app elle-même. Un scan et une lecture de
  collection depuis l'app, sur un compte connu, restent à faire à la main entre l'étape 9 et la
  décision de l'étape 10.

## 6. Bascule et retour

- **Pas de mode répétition dans le script** (D5) : la répétition se fait sous WSL (section 8), et
  les étapes 0 à 7 se jouent à l'avance sur l'hôte. Le jour J, le script vérifie que les secrets
  critiques sont identiques et que les files du monolithe sont vides, demande `GELER`, puis arrête
  l'API du monolithe (pas sa base). La procédure est celle de `db/reprise/README.md` § 3,
  automatisée.
- **Retour** (`sh deploiement.sh retour`) : avant l'étape 10, la constellation s'arrête et le
  monolithe redémarre sur sa base intacte. Après, le script demande d'abord de repointer le proxy
  sur 3000. Tant qu'aucun utilisateur n'a écrit sur la constellation, rien n'est perdu ; au-delà,
  un retour perd les écritures faites depuis la bascule : il n'existe pas de reprise en sens
  inverse.

## 7. Risques et points à vérifier

- **Première reprise sur des données de production le jour J** (D5) : aucune répétition sur un
  vidage de production. Les contrôles de la reprise arrêtent le script avant la bascule du
  proxy, et le retour reste simple jusque-là ; mais un cas de données que les deux jeux de
  répétition ignorent ne se verra que ce jour-là, fenêtre de gel ouverte.
- **Compte SQL du monolithe** (D1, D2) : il doit recevoir les droits sur `` `acv\_%` `` (étape 4).
  Le script affiche le `GRANT` exact et le joue avec un compte d'administration saisi, ou attend
  qu'on le joue ; jamais `ON *.*`.
- **Connexions de l'instance** : elle est partagée avec le monolithe. Au repos, la constellation
  en tient 26 ; chaque pool Prisma peut monter à 2 × cœurs + 1. L'étape 4 relève la marge. En
  répétition, quinze redémarrages du pod enchaînés en pleine initialisation ont laissé 150
  connexions orphelines (refus `1040`) ; un redémarrage normal n'en laisse aucune.
- **Podman de l'hôte** : Quadlet exige un Podman assez récent, que la Debian de ce poste n'a pas.
  L'hôte fait déjà tourner le monolithe sous Quadlet ; l'étape 0 vérifie la version.
- **Reverse proxy** : sa configuration n'est décrite dans aucun dépôt (D3). Le script ne la touche
  pas ; il vérifie seulement, après `FAIT`, que l'URL publique répond par la constellation. Il
  rappelle de ne pas relayer `/internal/`.
- **Durée de construction** en arm64 sous émulation : à mesurer à la première archive arm64.

## 8. Répétition sous WSL (D4), le 2026-11-21

**Environnement.** Le Podman de la Debian de ce poste étant trop ancien pour Quadlet, la répétition
tourne dans une distribution WSL dédiée, `acervatim-repetition` (Debian à jour, Podman rootless,
systemd, linger), qui se supprime d'une commande (`wsl --unregister acervatim-repetition`). Elle
y figure l'hôte :

- une instance MariaDB, en root, à une adresse factice (192.0.2.10:3306) : vue du pod, elle
  est distante, comme celle du monolithe ;
- le monolithe de la production, reconstruit en amd64 depuis son étiquette, avec ses unités Quadlet, son
  `.env` dans `~/acervatim/` et un compte SQL limité à `acervatim` ; ses données : le vidage de
  développement (34 utilisateurs) et le jeu complémentaire de la reprise ;
- un nginx sur 8080 en guise de reverse proxy.

Sous WSL2, les distributions partagent leur pile réseau : le gateway de développement de Docker
Desktop tenait le port 3100. L'étape 0 l'a refusé ; il a été arrêté pendant la répétition, puis
redémarré.

**Résultat.** Archive amd64 de 1 400 Mio (4,7 Gio d'images brutes, Redis et client MariaDB
compris), construite en 5 minutes à partir des images existantes. Déroulé final d'un bout à
l'autre avec le script de l'archive : étapes 0 à 7 à l'avance, puis 8 à 11. Secrets repris
(empreintes identiques), compte d'administration à l'étape 4, premier `kube play` du manifeste,
onze schémas migrés sur l'instance distante, 18 contrôles de fumée verts, reprise conforme
(22 tables, 5 invariants, registre de 18 entrées), `relecture.js` et `parcours.js` conformes,
bascule du proxy détectée, gel de 24 secondes sur ce jeu minuscule. Puis `retour` et
`desinstaller` : la base du monolithe sort identique, à la somme de contrôle près. Aucun secret
dans les journaux. Mémoire des onze services au repos : environ 575 Mo.

**Défauts trouvés et corrigés en chemin** : `migrate.sh` en CRLF dans l'image de migration
(l'initContainer ne démarrait pas) ; des clés vides
refusées par `billing` (`GOOGLE_PLAY_*`, désormais laissées hors du Secret quand le monolithe ne
les a pas) ; un `podman run -i` qui mangeait les réponses tapées ; un compteur qui écrasait la
boucle des étapes ; une entrée fermée qui faisait boucler une pause manuelle.

**Reste à éprouver sur l'hôte** : l'arm64, l'instance et le reverse proxy réels. Les étapes 0 à
7, jouées à l'avance, le font sans toucher au monolithe.

## 9. Avancement

1. `deploy/build-archive.ps1`, `MANIFEST` et `SHA256SUMS` : **fait**.
2. `deploiement.sh`, étapes 0 à 11, `statut`, `retour`, `desinstaller` : **fait**.
3. `deploy/tests/fumee.sh` et `relecture.js` : **fait**.
4. Répétition sous WSL (D4) : **faite** le 2026-11-21 (section 8).
5. Version : **tranché le 2026-11-21**. L'étiquette locale de la première version, jamais poussée,
   est déplacée sur l'outillage et la correction de l'image de migration : la première version
   est celle qui se déploie.
6. Archive arm64 : **en cours** le 2026-11-21. Puis étapes 0 à 7 sur l'hôte, à l'avance.
7. Bascule : étapes 8 à 11, le jour choisi.
