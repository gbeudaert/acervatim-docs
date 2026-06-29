# Plan de consolidation de la documentation Acervatim

> Objectif : une **doc centrale unique et normalisée** (`acervatim-docs`, dépôt public) organisée par
> **public cible** — **Utilisateur** et **Développeur** — incluant les éléments obligatoires (légal,
> confidentialité). Les artefacts **de travail** (contexte IA, revues de code, brouillons) **n'y
> figurent pas** : ils restent dans les projets. Les `docs/interne/` des projets ne servent qu'aux
> **sprints**.

Statut : **plan** (non exécuté). Révisé le 2026-06-28.

## 1. Principe directeur

La doc centrale est un **produit documentaire**, pas un dépotoir d'internes. Deux espaces, un socle légal :

- **Espace Utilisateur** — à quoi sert Acervatim, comment l'utiliser, ce qui est privé.
- **Espace Développeur** — architecture, API, setup, déploiement, conventions.
- **Socle légal / obligatoire** — politique de confidentialité, CGU, mentions légales, Data Safety.

**Ne vont PAS dans la doc centrale** (restent matière de travail, dans les projets, gitignored) :
`context-ia.md`, les `reviews/`, `memo-a-analyser.md` et tout brouillon/notes d'agent. Ce sont des
supports de production, pas de la documentation.

## 2. Décisions actées

| Décision | Choix |
|---|---|
| Hôte | **`acervatim-docs`** = doc centrale unique (site public GitHub Pages inclus) |
| Visibilité | Dépôt **public** |
| Organisation | **Par public cible** : Utilisateur / Développeur (+ socle légal) |
| Portée | **Migration curée et normalisée** — pas un déversement de l'interne |
| Exclus de la centrale | `context-ia`, `reviews`, brouillons → restent dans les projets |
| `docs/interne/` des projets | **Sprints uniquement** |

> Cette révision **remplace** la précédente version « tout migrer / dump intégral de l'interne » :
> on ne publie que ce qui a une valeur documentaire normalisée + le légal.

`acervatim-docs` n'est pas encore un dépôt git → `git init` + remote public en phase 1.

## 3. Cible — arborescence de `acervatim-docs`

```
acervatim-docs/                      # dépôt public, GitHub Pages (just-the-docs)
  _config.yml                        # thème + nav (déplacé depuis mvc/docs)
  index.md                           # accueil : pitch + 2 portes (Utilisateur / Développeur)

  utilisateur/
    presentation.md                  # ex-mvc/docs/application.md (normalisé)
    fonctionnalites.md               # ex-mvc/docs/roadmap.md (dispo / à venir)
    mode-connecte.md                 # le serveur vu côté utilisateur (ex-backend.md, volet user)

  developpeur/
    architecture.md                  # vue système macro (ex-acervatim-docs/architecture.md)
    backend.md                       # stack réelle, rôles, privacy by design (volet technique)
    api-reference.md                 # contrat /v1 normalisé ; source de vérité = openapi.json
    auth-google.md                   # setup OAuth Google (app + backend)
    deploiement-raspberry-pi.md      # Pi 4 / Podman / Quadlet, MariaDB sur NAS
    scan-enrichissement.md           # flux scan → /v1/search
    publication-play-store.md        # release app

  legal/
    confidentialite.md               # version canonique unique
    cgu.md                           # à rédiger si absent
    mentions-legales.md              # à rédiger si absent
    data-safety.md                   # formulaire Google Play (lié au sprint 08 backend)

# NON migré — reste dans les projets :
acervus-backend/docs/interne/   context-ia.md, api-mobile-spec.md (source brute),
                                reviews/, etudes/, archives AWS, sprints/
mvc/docs/interne/               sprints app, sprint-realises, reviews, mémos, assets de travail
```

> Nommage indicatif (`utilisateur/`, `developpeur/`, `legal/`) — à figer en phase 1.

## 4. Mapping de migration

### Vers la doc centrale (curé + normalisé)

| Source actuelle | Destination | Action |
|---|---|---|
| `mvc/docs/index.md` | `index.md` | Refondre en page d'accueil à 2 portes |
| `mvc/docs/application.md` | `utilisateur/presentation.md` | Reprise |
| `mvc/docs/roadmap.md` | `utilisateur/fonctionnalites.md` | Reprise |
| `mvc/docs/backend.md` | `utilisateur/mode-connecte.md` + `developpeur/backend.md` | **Scinder** volet user / volet technique |
| `mvc/docs/confidentialite.md` **ou** `acervus-backend/docs/politique-confidentialite.md` | `legal/confidentialite.md` | **Dédoublonner** → une seule version canonique |
| `acervatim-docs/architecture.md` | `developpeur/architecture.md` | Reprise (déjà à jour Pi/Podman) |
| `acervus-backend/docs/interne/api-mobile-spec.md` | `developpeur/api-reference.md` | **Normaliser** (la source brute reste côté backend) |
| `acervus-backend/docs/interne/setup-google-cloud-oauth.md` + `mvc/docs/interne/google-oauth-setup.md` | `developpeur/auth-google.md` | **Fusionner** app+backend |
| `acervus-backend/docs/interne/guide-implementation-nas-synology.md` | `developpeur/deploiement-raspberry-pi.md` | **Réécrire** Pi/Podman |
| `mvc/docs/interne/guide-scan-enrichissement-app.md` | `developpeur/scan-enrichissement.md` | Reprise |
| `mvc/docs/interne/playstore-deployment.md` | `developpeur/publication-play-store.md` | Reprise (relire secrets) |

### NON migré — reste dans les projets (matière de travail / sprints)

| Doc | Reste où | Pourquoi |
|---|---|---|
| `context-ia.md` | `acervus-backend/docs/interne/` | Contexte agent IA, pas de la doc produit |
| `reviews/`, `mvc/docs/review/` | projets | Revues internes, non publiables |
| `sprints/…`, `sprint-09…16`, `sprint-realises.md`, `memo-a-analyser.md` | `docs/interne/` des projets | Couplés au code ; flux sprint par session |
| `etudes/authz-casl-vs-opa.md` | `acervus-backend/docs/interne/` | Étude exploratoire interne |
| `guide-implementation-backend-aws-archive.md` | `acervus-backend/docs/interne/archives/` | Archive d'une stack non retenue |
| `api-mobile-spec.md` (source brute) | `acervus-backend/docs/interne/` | Source de vérité couplée au code ; la centrale en publie une version normalisée |

## 5. Garde-fous (dépôt public)

- **Privacy by design** : aucun secret en clair (constat actuel : il n'y en a pas dans les docs ;
  les guides OAuth rappellent que les secrets vivent en `.env`). Ajouter un **grep CI** sur le dépôt
  doc : `GOCSPX`, `client_secret`, `-----BEGIN`, `*_PEPPER=`, `ENCRYPTION_KEY=`, clés JWT.
- **Historique propre** : commit initial à partir des fichiers seuls — ne pas importer un historique
  git pouvant contenir d'anciens secrets.
- **Pas de réintroduction AWS** : vérifier qu'aucun doc migré ne présente la stack AWS/DynamoDB comme
  « actuelle » (corrigé récemment dans `contexte-global.md` et `architecture.md`).

## 6. Doublons à résoudre

- **Politique de confidentialité** : `mvc/docs/confidentialite.md` **et**
  `acervus-backend/docs/politique-confidentialite.md` → une seule `legal/confidentialite.md`, les
  autres pointent dessus (ou sont supprimées).
- **Description backend** : volet **user** (`mode-connecte.md`) vs volet **dev** (`backend.md`) —
  scinder proprement, pas de copie intégrale dupliquée.

## 7. GitHub Pages / Jekyll

- Déplacer `_config.yml` + thème `just-the-docs` dans `acervatim-docs`.
- Navigation `just-the-docs` à 2 sections parentes (`Utilisateur`, `Développeur`) + `Légal`
  (`nav_order`, `parent`, `has_children`).
- Activer Pages sur le nouveau dépôt ; **domaine** à décider (URL dépôt vs domaine custom `CNAME`).
- **Redirection** depuis l'ancienne URL Pages du dépôt `mvc` pour ne pas casser les liens existants.
- Ajuster `baseurl`.

## 8. Réécriture des liens

Les liens relatifs inter-dépôts (`../acervus-backend/…`, `../mvc/docs/…`) deviennent **internes** à
`acervatim-docs` → réécrire. Les liens vers le **code** (pas la doc) pointent vers le dépôt GitHub
(URL absolue). Passage `grep '](../'` après migration pour traquer les liens morts.

## 9. Plan d'exécution par phases

1. **Init** : `git init` `acervatim-docs`, remote public, `.gitignore` (`_site/`, `.jekyll-cache/`).
2. **Figer** l'arborescence (§3) et le nommage.
3. **Migrer Utilisateur** : présentation, fonctionnalités, volet user du serveur.
4. **Migrer Développeur** : architecture, backend, API normalisée, auth, déploiement Pi, scan, Play Store.
5. **Socle légal** : confidentialité canonique (dédoublonnée) ; rédiger CGU / mentions légales / Data Safety si absents.
6. **Réécrire** : guide NAS → Pi/Podman ; scinder backend user/dev ; liens (§8).
7. **Nettoyer les projets** : `docs/interne/` = sprints uniquement ; déplacer context-ia/reviews/études/archives en clair comme matière de travail (rester gitignored) ; mettre à jour les `CLAUDE.md`.
8. **Pages** (§7) + redirection.
9. **Vérifier** : build Pages OK, zéro lien mort, zéro secret (grep CI), nav cohérente.

## 10. Risques & points d'attention

- Fuite via l'historique git (→ commit initial propre).
- URL Pages cassée si abandon du dépôt `mvc` (→ redirection).
- Liens relatifs morts post-migration.
- Dérive « la centrale redevient un dump » : tenir la règle — **seulement** Utilisateur / Développeur / Légal, normalisé.

## 11. Décisions restantes

- Nommage définitif des sections (`utilisateur` / `developpeur` / `legal`).
- Domaine GitHub Pages (URL dépôt vs domaine custom).
- `api-reference` : page normalisée rédigée à la main **ou** rendu direct d'`openapi.json` (ex. Redoc/Swagger) ?
- CGU / mentions légales : à rédiger (n'existent pas encore) — bloquant pour publication Play Store ?
