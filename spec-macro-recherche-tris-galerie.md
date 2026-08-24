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
| 3 | [Vue galerie](#3--vue-galerie) | ✅ Livré — serveur (SD3) puis app (AD3), 2026-08-24 |

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

Le serveur sait le stocker depuis le sprint SD3 (`userData.playCount`) et **l'app compte depuis
AD3** (2026-08-24) : `Record.playCount` en base (migration Room v6 → v7, l'existant repart de 0),
incrémenté par `MarkAsPlayedUseCase` dans la même écriture que `lastPlayedAt`, poussé en valeur
absolue dans `userData.playCount`. Le blocage est levé.

État des rubriques :

| Rubrique voulue | État |
|---|---|
| Les X derniers écoutés | ✅ Livré — tri sur `lastPlayedAt`, borné à 12 |
| Les plus écoutés | ✅ Livré — `playCount` des deux côtés (local + `userData`) |
| Jamais écoutés / pas depuis 6 mois | ✅ Livré — une seule rubrique « oubliés » |

Deux options étaient ouvertes :

- **Compteur simple** (`playCount`) — peu coûteux, répond à « les plus écoutés », mais ne saura
  jamais dire « les plus écoutés **ce mois-ci** ».
- **Historique d'écoutes** (une ligne par écoute) — répond à tout, y compris des rubriques
  temporelles à la *Photos*, mais c'est une nouvelle table, une croissance de données à borner, et
  un sujet de vie privée (un historique d'écoute est une donnée de comportement, plus sensible
  qu'un compteur).

**Tranché le 2026-08-24 : compteur simple**, l'historique reste ouvert si des rubriques temporelles
deviennent nécessaires — ce serait alors une décision produit **et** privacy, pas un simple ajout
technique. Les deux volets sont livrés : serveur (SD3) `userData.playCount`, entier ≥ 0, absent ⇒ 0,
projeté en liste ; app (AD3) compteur local **et** push, pour que le compteur survive à un
changement d'appareil — sinon il repartirait de zéro à chaque réinstallation, ce qui viderait la
rubrique de son sens.

Limite assumée du modèle : le PATCH `userData` est un merge de valeurs, pas un incrément. L'app
envoie une valeur absolue, donc deux appareils qui écoutent hors ligne puis synchronisent
s'écrasent mutuellement (dernier écrivain gagne) au lieu de s'additionner.

### L'axe d'affichage : `DisplayMode`

`ListMode` (`FLAT` / `SERIAL`) décrit la **hiérarchie** (liste plate vs liste de séries), pas la
densité visuelle. La galerie est donc un **axe orthogonal**, introduit par AD3 :
`DisplayMode` (`LIST` / `GRID` / `GALLERY`), persisté **par type de collection** comme `ListMode`,
et non une troisième valeur de `ListMode` — qui aurait enfermé la galerie dans un seul type de
collection. Défaut par type : grille pour une collection de séries (ce que la vue manga a toujours
fait), liste pour une collection plate.

La brique existante `SeriesGrid.kt` reste la vue grille des séries ; les tuiles disques et séries
partagent `ui/list/CollectionTileBody.kt`, donc un seul rendu pour la grille et la galerie.

### Images

Les jaquettes viennent de deux sources : `coverPath` (fichier local) et `coverRemoteUrl`
(provider). `RecordCover` gère les deux avec repli et pose un placeholder, et Coil charge à la
taille du composable — une tuile sans jaquette ne troue pas la grille, et rien n'est décodé en
pleine résolution. Le préchargement n'est pas implémenté : le défilement sur une **grosse**
collection reste à mesurer sur appareil, il n'est pas prouvé ici.

---

## Points ouverts (transverses)

- **Genre mono vs pluriel** (évolution 2) : traiter la dette ou l'assumer ?
- ~~**Compteur vs historique d'écoutes** (évolution 3)~~ : tranché le 2026-08-24 — **compteur**
  (`userData.playCount`), livré serveur (SD3) et app (AD3). L'historique reste écarté, pas oublié.
- **Tri serveur** : différé au chantier web, mais le curseur composite est à concevoir à ce
  moment-là, pas à bricoler.

## Documents liés

| Document | Contenu |
|---|---|
| `acervatim-backend/docs/interne/sprints/decouverte/` | Sprints backend SD1→SD3 (gitignoré) |
| `acervatim-app/docs/interne/sprints/decouverte/` | Sprints app AD1→AD3 (gitignoré) |
| [spec-macro-partage-collection.md](spec-macro-partage-collection.md) | Chantier partage — `A1bis`/`S1bis` synchronisent `lastPlayedAt`, utile ici |
| [agent/transverse/sources-externes.md](agent/transverse/sources-externes.md) | Résolution des jetons (BYOT / repli premium / dégradé) |
