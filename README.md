# acervatim-docs

**Documentation centrale d'Acervatim** — site unique (GitHub Pages, thème *just-the-docs*) regroupant
toute la documentation **normalisée** des dépôts Acervatim, organisée par public cible.

> Acervatim — application Android de gestion de collections (vinyles, mangas…), *offline-first*, avec
> un backend optionnel d'enrichissement et de synchronisation.

## Structure

| Espace | Dossier | Contenu |
|---|---|---|
| 👤 Utilisateur | [`utilisateur/`](utilisateur/) | Présentation, fonctionnalités, sources de données, mode connecté. |
| 🛠️ Développeur | [`developpeur/`](developpeur/) | Architecture, backend, référence API, auth Google, déploiement Pi, pivot ISBN, Play Store. |
| ⚖️ Légal | [`legal/`](legal/) | Confidentialité (canonique), CGU, mentions légales, Data Safety. |

La page d'accueil du site est [`index.md`](index.md).

## Périmètre

Cette doc est **normalisée et curée** : elle ne contient pas les artefacts de travail (contexte IA,
revues de code, brouillons, sprints). Ceux-ci restent dans les dépôts applicatifs :

- `acervus-backend/docs/interne/` — sprints backend + matière de travail.
- `mvc/docs/interne/` — sprints app + matière de travail.

Le **code reste la source de vérité** ; les pages renvoient aux dépôts `mvc` et `acervus-backend`
pour le détail fin. Voir le [plan de consolidation](plan-consolidation-documentation.md) pour le
contexte de la migration.

## Développement local

Site Jekyll + just-the-docs (chargé via `jekyll-remote-theme`) :

```sh
bundle exec jekyll serve   # rendu local sur http://localhost:4000
```

Sur GitHub Pages : Settings → Pages → branche `main`, racine du dépôt.

## Statut

Dépôt **en cours de constitution** (consolidation de la doc auparavant éclatée entre `mvc/docs` et
`acervus-backend/docs`). Voir [`plan-consolidation-documentation.md`](plan-consolidation-documentation.md).
