---
title: Accueil
layout: default
nav_order: 1
---

# Acervatim — documentation

> *acervatim* (latin) — « en monceaux, par accumulation ». De *acervus*, « tas, amas, collection ».

**Acervatim** est une application Android libre (Apache 2.0) pour cataloguer vos collections. Elle
démarre avec les **disques vinyles** et les **mangas**, et son architecture est pensée pour s'étendre
à d'autres types (BD, LEGO, jeux de société, livres, films…). L'application est **100 % hors-ligne par
défaut** ; un **backend optionnel** ajoute l'enrichissement de métadonnées et la synchronisation.

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
| `mvc` | Application Android | Kotlin · Jetpack Compose · Room · Hilt · Retrofit |
| `acervus-backend` | Backend / API | Node 24 · NestJS 10 · Prisma 6 · MariaDB |
| `acervatim-docs` | **Cette documentation** | Markdown · Jekyll (just-the-docs) |
| `acervatim-infra` | Infra AWS (cible future, non activée) | OpenTofu · AWS |

> Le code fait toujours foi : cette documentation est la vue **normalisée**, les dépôts applicatifs
> restent la source de vérité pour les détails fins.
