---
title: Développeur
layout: default
nav_order: 3
has_children: true
---

# Espace Développeur

Documentation technique pour contribuer à Acervatim ou intégrer son API.

- [Architecture](architecture.md) — vue système, flux clés, modèle de données façade.
- [Le serveur Acervatim](backend.md) — rôles, stack réelle, privacy by design, API.
- [Référence API](api-reference.md) — contrat `/v1`, conventions, erreurs, pagination.
- [Authentification Google](auth-google.md) — setup OAuth côté app et côté backend.
- [Déploiement du backend](deploiement.md) — conteneur Podman, reverse proxy, migrations.
- [Sources de données & pivot ISBN](sources-et-pivot-isbn.md) — provenance des champs, flux scan, pivot BnF → MyAnimeList.
- [Publication Play Store](publication-play-store.md) — release de l'application Android.

> Le code reste la source de vérité : les dépôts `acervatim-app` (application) et
> `acervatim-backend` (API) font foi pour le détail. Ces pages en donnent la vue normalisée.
