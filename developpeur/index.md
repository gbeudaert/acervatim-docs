---
title: Développeur
layout: default
nav_order: 3
has_children: true
---

# Espace Développeur

Documentation technique pour contribuer à Acervatim ou intégrer son API.

- [Architecture](architecture.md) — vue système, micro-services en production et ancien monolithe, flux clés, modèle de données façade.
- [Le serveur Acervatim](backend.md) — rôles, stack, privacy by design, fonctionnalités de l'API.
- [Référence API](api-reference.md) — contrat `/v1`, conventions, erreurs, pagination.
- [Authentification Google](auth-google.md) — setup OAuth côté app et côté backend.
- [Déploiement du backend](deploiement.md) — conteneur Podman, reverse proxy, migrations (monolithe).
- [Mise à jour du backend](mise-a-jour.md) — montée de version, sauvegarde, retour arrière (monolithe).
- [Sources de données](sources-et-pivot-isbn.md) — toutes les sources par type de collection (rôle, accès, débits, conditions d'usage, statut), puis le pivot ISBN manga BnF → MangaDex (MAL en repli).
- [Publication Play Store](publication-play-store.md) — release de l'application Android.
- [Build locale de l'app](build-app-locale.md) — variante debug installable à côté de celle du Play Store.

> Le code reste la source de vérité : les dépôts `acervatim-app` (application), `acervatim-services`
> (serveur en production, en micro-services) et `acervatim-backend` (ancien serveur, gardé pour un
> retour) font foi pour le détail. Ces pages en donnent la vue normalisée. Les versions des outils et des dépendances ne sont
> pas reprises ici : les fichiers de build font foi.
