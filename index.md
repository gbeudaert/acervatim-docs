---
title: Accueil
layout: default
nav_order: 1
---

# Acervatim — documentation

> *acervatim* (latin) — « en monceaux, par accumulation ». De *acervus*, « tas, amas, collection ».

**Acervatim** est une application Android libre (Apache 2.0) pour cataloguer vos collections. Elle
gère aujourd'hui les **disques vinyles** et les **mangas** ; les **films** et le **LEGO** sont les
prochains types prévus, les **jeux de société** suivront (voir
[D'où viennent les informations](utilisateur/sources-de-donnees.md)). L'application est **100 %
hors-ligne par défaut** ; le **mode connecté** optionnel ajoute l'enrichissement des fiches depuis
des bases spécialisées, avec vos propres jetons d'accès ; le palier **premium** permet d'utiliser les
jetons Acervatim, la synchronisation cloud et le partage de collections.

![Statut](https://img.shields.io/badge/statut-pre--alpha-orange)
![Plateforme](https://img.shields.io/badge/platform-Android%207%2B-blue)
![Licence](https://img.shields.io/badge/license-Apache%202.0-green)

---

## Par où commencer

Cette documentation est organisée par **public** :

| Espace | Pour qui | Contenu |
|---|---|---|
| 👤 **[Utilisateur](utilisateur/)** | Toute personne qui utilise l'application | Présentation, guide d'utilisation, fonctionnalités à venir, sources de données. |
| 🛠️ **[Développeur](developpeur/)** | Contributeurs et intégrateurs | Architecture, backend, référence API, authentification, déploiement, enrichissement. |
| ⚖️ **[Légal](legal/)** | Tous | Politique de confidentialité, CGU, mentions légales, Data Safety. |

## En deux mots

- **100 % hors-ligne par défaut.** Vos données restent sur votre appareil : pas de compte
  obligatoire, pas de télémétrie, pas de connexion requise.
- **Multi-collections.** Une seule application pour tous vos objets de collection, avec une expérience
  adaptée à chaque type.
- **Vos données vous appartiennent.** Import, export et sauvegarde complète (ZIP) à tout moment, dans
  des formats ouverts (JSON, CSV).
- **Privacy by design.** Aucune donnée personnelle directe stockée côté serveur (cf.
  [politique de confidentialité](legal/confidentialite.md)).

## Les dépôts

| Dépôt | Rôle | Stack |
|-------|------|-------|
| [`acervatim-app`](https://github.com/gbeudaert/acervatim-app) | Application Android | Kotlin · Jetpack Compose · Room · Hilt · Retrofit |
| [`acervatim-backend`](https://github.com/gbeudaert/acervatim-backend) | Serveur actuel, en production : un seul programme (le « monolithe ») | Node.js · NestJS · Prisma · MariaDB · Redis |
| [`acervatim-services`](https://github.com/gbeudaert/acervatim-services) | Serveur cible, découpé en micro-services ; la production y basculera, la date n'est pas fixée | Node.js · NestJS · Prisma · MariaDB · Redis |
| [`acervatim-docs`](https://github.com/gbeudaert/acervatim-docs) | **Cette documentation** | Markdown · Jekyll (just-the-docs) |

Les versions des outils et des dépendances ne sont pas reprises ici : les fichiers de build de chaque
dépôt font foi.

> Le code fait toujours foi : cette documentation est la vue **normalisée**, les dépôts applicatifs
> restent la source de vérité pour les détails fins.
