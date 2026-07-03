# acervatim-docs

**Documentation centrale d'Acervatim** — site unique (GitHub Pages, thème *just-the-docs*) regroupant
toute la documentation **normalisée** des dépôts Acervatim, organisée par public cible.

> Acervatim — application Android de gestion de collections (vinyles, mangas…), *offline-first*, avec
> un mode connecté optionnel d'enrichissement de métadonnées.

## Structure

| Espace | Dossier | Contenu |
|---|---|---|
| 👤 Utilisateur | [`utilisateur/`](utilisateur/) | Présentation, fonctionnalités, sources de données, mode connecté. |
| 🛠️ Développeur | [`developpeur/`](developpeur/) | Architecture, backend, référence API, auth Google, déploiement, pivot ISBN, Play Store. |
| ⚖️ Légal | [`legal/`](legal/) | Confidentialité (canonique), CGU, mentions légales, Data Safety. |

La page d'accueil du site est [`index.md`](index.md).

## Périmètre

Cette doc est **normalisée, curée et auto-portante** : elle ne contient pas d'artefacts de travail
(contexte IA, revues de code, brouillons, sprints) et ne renvoie vers aucune documentation interne
des dépôts applicatifs.

Le **code reste la source de vérité** : en cas de doute, les dépôts `acervatim-app` (application)
et `acervatim-backend` (API) font foi.

## Développement local

Site Jekyll + just-the-docs (chargé via `jekyll-remote-theme`) :

```sh
bundle exec jekyll serve   # rendu local sur http://localhost:4000
```

Sur GitHub Pages : Settings → Pages → branche `main`, racine du dépôt.

## Statut

Dépôt **en cours de constitution** (consolidation de la doc auparavant éclatée entre les dépôts
applicatifs). Voir [`plan-consolidation-documentation.md`](plan-consolidation-documentation.md).
