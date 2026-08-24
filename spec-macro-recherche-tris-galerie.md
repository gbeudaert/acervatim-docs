# Spec macro — Recherche, tris et vue galerie

> **Document interne** (exclu du site publié, comme `sprints-conformite-offre.md`).
> Créé le 2026-08-20. Vue d'ensemble transverse de trois évolutions indépendantes.
> Sprints d'exécution : `acervatim-backend/docs/interne/sprints/decouverte/` et
> `acervatim-app/docs/interne/sprints/decouverte/` (répertoires **gitignorés**).

Trois évolutions, une même finalité : **retrouver ce qu'on cherche** — dans les bases externes
quand on ajoute, dans sa propre collection quand on la parcourt.

| # | Évolution | Où est le travail |
|---|---|---|
| 1 | [Recherche textuelle vers les bases externes](#1--recherche-textuelle-vers-les-bases-externes) | ~90 % app (le backend sait déjà le faire) |
| 2 | [Tris et filtres dans l'app](#2--tris-et-filtres-dans-lapp) | App (backend seulement pour le web) |
| 3 | [Vue galerie](#3--vue-galerie) | App — la donnée qui manquait côté backend est livrée (SD3) |

Elles sont **indépendantes** : ni ordre imposé, ni dépendance croisée. Deux liens utiles seulement :
l'évolution 3 profite des tris de l'évolution 2, et les deux profitent de la synchronisation
complète (`A1bis`/`S1bis` du chantier partage) pour rester cohérentes entre appareils.

---

## 1 — Recherche textuelle vers les bases externes

### Le besoin

Certains disques n'ont **pas de code-barres** (pressages indépendants, autoproductions —
*Degiheugi* en est un cas typique) alors qu'ils sont parfaitement référencés sur Discogs.
Aujourd'hui, ils sont introuvables : le seul chemin d'ajout enrichi passe par le scan.

### L'état réel du code (audit du 2026-08-20)

**Le backend sait déjà faire.** Ce n'est pas une fonctionnalité à construire, c'est une
fonctionnalité à **brancher** :

- `GET /v1/search?type=…&q=…` existe (`q` XOR `barcode`), documenté et routé vers l'adapter du type.
- `DiscogsAdapter.search(query)` est implémenté (`runSearch({ q: query })`).
- `AcervatimApi.search()` côté app expose **déjà** le paramètre `q`.

**Ce qui manque est côté app :**

- `ExternalSearchRepository` n'expose que `searchByBarcode()` — aucune méthode de recherche texte.
- Le seul point d'entrée utilisateur est `BarcodeScanScreen` (caméra). Il n'existe aucun écran pour
  taper un titre ou un artiste.

### Le vrai problème de conception : choisir le bon pressage

Un code-barres identifie **une** édition. Un texte en renvoie des dizaines : Discogs référence
souvent 30 pressages du même album (pays, année, label, format). Une liste de résultats qui
n'affiche que « titre — artiste » est inutilisable : l'utilisateur ne peut pas choisir.

C'est là que se situe le travail réel, et il est partagé :
- **Backend** : garantir que la réponse porte de quoi discriminer (année, label, pays, format,
  jaquette) et gérer la pagination des résultats.
- **App** : une UI de sélection qui rend ces différences lisibles.

### Point d'attention — jetons

`/v1/search` passe par la résolution de jetons (BYOT → repli premium → mode dégradé → 403
`source-token-required`). Une recherche texte Discogs **exige un jeton** : un utilisateur connecté
gratuit sans compte Discogs relié recevra un 403. L'UI doit le dire clairement et proposer l'action
(connecter Discogs, ou passer premium), pas afficher « aucun résultat ».

---

## 2 — Tris et filtres dans l'app

### Le besoin

Trier par date de dernière écoute, par genre, etc. Aujourd'hui l'ordre est **figé**.

### L'état réel du code

- `RecordFilter` porte des filtres (`query`, `status`, `genre`, `format`,
  `notPlayedSinceSixMonths`) mais **aucun champ de tri**.
- La requête Room ordonne par `r.id, ra.position` (ordre technique, nécessaire au regroupement des
  artistes), puis `RecordRepositoryImpl` retrie **en mémoire** avec un comparateur figé
  (`RECORD_BY_PRIMARY_ARTIST`, ou `RECORD_BY_VOLUME` pour les tomes).
- Il y a donc déjà un tri, mais il n'est **pas choisissable**.

### Les deux points durs

**Le tri en mémoire.** Trier après regroupement suppose de charger toute la collection. C'est
indolore sur 200 disques, moins sur 5 000 — et l'évolution 3 (galerie) augmente la pression. Le tri
doit descendre dans SQL. Attention : le regroupement disque × artistes impose de trier sur une
requête de disques, pas sur les lignes jointes.

**Le genre.** L'app est **mono-genre** (colonne `genre` unique, filtre en égalité stricte), le
serveur est **pluriel** (`genre: string[]`). Trier ou filtrer « par genre » sur un modèle mono-genre
donnera un résultat pauvre dès qu'un disque relève de deux genres. Passer l'app au pluriel est un
changement large (modèle, formulaire, UI, mapper) — c'est une dette connue, à arbitrer
explicitement : la traiter dans ce chantier, ou assumer le mono-genre et le documenter.

### Backend : pas nécessaire maintenant

La liste de l'app lit Room. Le backend n'intervient pas. **Sauf pour le web** : `GET
/v1/collections/:id/items` a une pagination **cursor** avec un ordre fixe — un tri arbitraire
oblige à encoder la clé de tri dans le curseur (curseur composite). C'est un vrai travail, à faire
au moment du web, pas avant. Le sprint backend correspondant est écrit mais explicitement différé.

---

## 3 — Vue galerie

### Le besoin

Une vue inspirée de *Photos* : grandes images, mise en avant des **derniers écoutés** et des **plus
écoutés**. La vue séries manga doit suivre la même logique.

### Le blocage : « les plus écoutés » n'avait aucune donnée

À l'audit du 2026-08-20, il n'existait **aucun compteur d'écoutes** dans le projet, ni côté app ni
côté serveur. `MarkAsPlayedUseCase` appelle `updateLastPlayed(id, now)` : un unique horodatage,
**écrasé** à chaque écoute. On sait *quand* un disque a été écouté pour la dernière fois, jamais
*combien de fois*.

Le serveur sait désormais le stocker (`userData.playCount`, sprint SD3, 2026-08-24) ; **l'app ne
compte toujours rien** — c'est AD3 qui doit incrémenter et pousser.

État des rubriques :

| Rubrique voulue | Faisable aujourd'hui ? |
|---|---|
| Les X derniers écoutés | ✅ Oui — tri sur `lastPlayedAt` |
| Les plus écoutés | 🔸 Côté serveur oui (`userData.playCount`, SD3) — **l'app ne compte toujours pas** |
| Jamais écoutés / pas depuis 6 mois | ✅ Oui (filtre déjà présent) |

Deux options étaient ouvertes :

- **Compteur simple** (`playCount`) — peu coûteux, répond à « les plus écoutés », mais ne saura
  jamais dire « les plus écoutés **ce mois-ci** ».
- **Historique d'écoutes** (une ligne par écoute) — répond à tout, y compris des rubriques
  temporelles à la *Photos*, mais c'est une nouvelle table, une croissance de données à borner, et
  un sujet de vie privée (un historique d'écoute est une donnée de comportement, plus sensible
  qu'un compteur).

**Tranché le 2026-08-24 : compteur simple**, l'historique reste ouvert si des rubriques temporelles
deviennent nécessaires — ce serait alors une décision produit **et** privacy, pas un simple ajout
technique. Le volet serveur est livré (sprint SD3) : `userData.playCount`, entier ≥ 0, absent ⇒ 0,
projeté en liste. Reste le volet app (AD3) : le compteur doit exister des deux côtés (local +
`userData.playCount`) pour survivre à un changement d'appareil — sinon il repart de zéro à chaque
réinstallation, ce qui vide la rubrique de son sens.

Limite assumée du modèle : le PATCH `userData` est un merge de valeurs, pas un incrément. L'app
envoie une valeur absolue, donc deux appareils qui écoutent hors ligne puis synchronisent
s'écrasent mutuellement (dernier écrivain gagne) au lieu de s'additionner.

### L'axe d'affichage n'existe pas encore

`ListMode` (`FLAT` / `SERIAL`) décrit la **hiérarchie** (liste plate vs liste de séries), pas la
densité visuelle. La galerie est donc un **axe orthogonal** à introduire (mode d'affichage :
liste / grille / galerie), pas une troisième valeur de `ListMode`. Confondre les deux enfermerait
la galerie dans un seul type de collection.

Une brique existe déjà : `SeriesGrid.kt` (grille de tomes de l'écran détail série) — à regarder
avant d'en écrire une autre.

### Images

Les jaquettes viennent de deux sources : `coverPath` (fichier local) et `coverRemoteUrl`
(provider). Une galerie affiche de **grandes** images, en nombre : le dimensionnement, le cache et
le préchargement deviennent des sujets de performance, alors qu'ils sont indolores sur une liste à
vignettes.

---

## Points ouverts (transverses)

- **Genre mono vs pluriel** (évolution 2) : traiter la dette ou l'assumer ?
- ~~**Compteur vs historique d'écoutes** (évolution 3)~~ : tranché le 2026-08-24 — **compteur**
  (`userData.playCount`), livré côté serveur par SD3. L'historique reste écarté, pas oublié.
- **Tri serveur** : différé au chantier web, mais le curseur composite est à concevoir à ce
  moment-là, pas à bricoler.

## Documents liés

| Document | Contenu |
|---|---|
| `acervatim-backend/docs/interne/sprints/decouverte/` | Sprints backend SD1→SD3 (gitignoré) |
| `acervatim-app/docs/interne/sprints/decouverte/` | Sprints app AD1→AD3 (gitignoré) |
| [spec-macro-partage-collection.md](spec-macro-partage-collection.md) | Chantier partage — `A1bis`/`S1bis` synchronisent `lastPlayedAt`, utile ici |
| [agent/transverse/sources-externes.md](agent/transverse/sources-externes.md) | Résolution des jetons (BYOT / repli premium / dégradé) |
