# Réflexion — Pivot vs pipeline d'enrichissement multi-sources

> Remonter : [INDEX.md](INDEX.md) · Document de **réflexion** (pas une décision actée, pas une page du
> site public). But : évaluer si l'architecture actuelle du pivot BnF→MangaDex→MAL (résolution ad hoc,
> dupliquée à chaque point d'entrée, sans redondance de source) doit évoluer vers un **pipeline** plus
> résilient (identité résolue une fois et réutilisée, sources de repli à chaque étage). Déclencheur :
> [sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md) décrit le modèle actuel en
> détail — ce document questionne ses limites structurelles, pas son fonctionnement. Deux classes
> d'échec distinctes y sont documentées : §1-2 (rapprochement MangaDex trop strict) et §3 (absence
> totale de la BnF, sans repli).
> Dernière mise à jour : 2026-07-25.

## 1. Le déclencheur — *Kingdom* introuvable côté jaquettes

Log observé :

```
[MangaDexResolver] mangadex: no manga match title="Kingdom" mangaId=- mal=- authors=? -> repli ISBN
```

Vérification en direct sur l'API MangaDex : **l'œuvre existe**, titre exact `"Kingdom"`, `mal=16765`,
auteur `Hara Yasuhisa`, en position 8/10 des candidats sur la requête `title=Kingdom`. Ce n'est donc pas
une lacune de MangaDex — c'est notre logique de rapprochement qui a rejeté un candidat au titre
**strictement identique**.

Cause immédiate : cet appel passe par `findManga` → `chooseCoverCandidate`
(`src/common/sources/mangadex/mangadex.resolver.ts:307-340`), emprunté quand ni `mangaId` ni `malId`
ne sont connus (`mal=-` dans le log). Ce chemin **exige un match auteur obligatoire** :

```ts
if (!matchesAuthor(authors, extractAuthors(entity.relationships))) return; // ligne 317
```

Il n'a **aucun** filet de sécurité « titre seul très fort accepté sans auteur », contrairement à
`chooseIdentity` (utilisé au moment du scan, ligne 213-215) :

```ts
const accepted = b.authorMatched
  ? b.titleScore >= IDENTIFY_TITLE_FLOOR   // 0.5, auteur confirme
  : b.titleScore >= PIVOT_TITLE_STRONG;    // 0.85, titre seul accepté
```

Si les auteurs BnF transmis pour cet appel sont vides ou ne partagent aucun token avec `"Hara
Yasuhisa"`, le candidat au titre exact est rejeté **même si aucune ambiguïté ne subsiste** (un seul
manga s'appelle « Kingdom » dans les 10 candidats retournés).

## 2. Pourquoi ça peut se reproduire — deux résolutions, deux règles

L'identification MangaDex d'une série arrive par **deux points d'entrée indépendants**, avec des
scorings différents :

| Point d'entrée | Fonction | Fallback titre-seul (sans auteur) |
|---|---|---|
| Scan (`identifySeries`) | `chooseIdentity` | ✅ `titleScore ≥ 0.85` |
| Jaquettes par tome (`resolveSeriesCovers` / `cachedSeriesCovers`, via `findManga`) | `chooseCoverCandidate` | ❌ auteur obligatoire, sans exception |

`sources-et-pivot-isbn.md` documente ce deuxième comportement comme un **garde-fou volontaire** — le
cas *One Piece* : le titre seul retiendrait une **variante** (édition colorisée) dont les jaquettes ne
correspondent pas. C'est un choix défendable pour les séries à variantes multiples. Mais il pénalise
uniformément **toutes** les séries sans variante connue (dont *Kingdom*), qui n'ont aucune raison
d'exiger l'auteur.

Le vrai filet de sécurité, dans l'architecture actuelle, est censé être **en amont** : `mangaId`/`malId`
sont produits une fois par `identify` au scan, portés dans `metadata.pivot`, puis **renvoyés par l'app**
à `edition-mapping`/`import-jobs` (cf. `sources-et-pivot-isbn.md` §Énumération des tomes). Tant que ce
fil est intact, `findManga` utilise `mangaId` directement (étape 1, la plus fiable) et
`chooseCoverCandidate` n'est **jamais** sollicité. Le cas `mal=- mangaId=-` du log signifie que ce fil
**a été rompu** pour cet appel : pas d'identité de scan disponible, ou pas transmise par l'appelant.
Plusieurs causes possibles (à distinguer avant de corriger, cf. §4) :

- l'utilisateur a lancé l'import/l'énumération d'une série **sans passer par le flux de scan** (recherche
  manuelle, réimport, refresh) → aucun `mangaId`/`malId` n'a jamais existé pour cet appel ;
- l'identité a bien été produite au scan, mais son cache a expiré (`HIT_TTL_SECONDS`) et l'app n'a pas
  reconservé `metadata.pivot` côté client pour le rappel ;
- l'identité de scan a échoué (repli MAL, ou `bnf_only`), donc aucun `mangaId`/`malId` n'existait à
  transmettre — dans ce cas le rejet en `chooseCoverCandidate` est **cohérent** (on est légitimement dans
  le chemin dégradé sans identifiant fiable).

## 3. Deuxième classe d'échec — absence totale de la BnF (fraîcheur du dépôt légal)

Logs observés le même jour, sur des scans ISBN distincts (barres UI d'un même utilisateur) :

```
[MalAdapter] pivot: start isbn=9782413047070
[BnfService] bnf: parsing isbn=9782413047070 numberOfRecords=0 parsed=0
[BnfService] bnf: not found isbn=9782413047070
[MalAdapter] pivot: bnf unresolved isbn=9782413047070 reason=bnf_not_found -> no candidate
[HTTP] GET /v1/search?type=manga&barcode=9782413047070&limit=10 200 229ms …
```

Identification des ISBN en échec (recherche web, la BnF ne les connaissant pas) :

| ISBN | Titre | Éditeur | Sortie |
|---|---|---|---|
| `9782344068649` | **Sakamoto Days** Tome 22 | Glénat | 2026-07-01 |
| `9782413047070` | **Hellsing Perfect Edition** T05 | Delcourt | 2026-07-09 |
| `9782203296633` | Les Années bleues Vol. 2 (*ito*) | Casterman | ~2026-07-08 |
| `9782808703437` | Sentenced to be a Hero T1 | Vega Dupuis | 2026-05-22 |
| `9789203296373` | non identifié (préfixe EAN inhabituel `978-92-03`) | — | — |

**Sakamoto Days et Hellsing sont deux séries « de premier ordre »** (Weekly Shōnen Jump en cours de
sérialisation pour la première, classique culte pour la seconde) — l'observation de l'utilisateur est
juste : ce n'est pas un problème d'obscurité du titre. Point commun des quatre titres identifiés :
**toutes des sorties très récentes** (0 à 8 semaines avant le scan, la moitié dans le mois). Le pattern
pointe vers un **délai de fraîcheur du dépôt légal BnF**, pas vers un défaut de catalogage permanent : le
SRU renvoie `numberOfRecords=0` (recherche réellement vide côté BnF, pas un bug de parsing local, cf.
`BnfService.resolveByIsbn`, `src/common/sources/bnf/bnf.service.ts:111-157`) parce que l'ISBN n'a
**simplement pas encore été indexé** — la notice finira probablement par apparaître (le dépôt légal n'est
pas instantané), mais rien dans le backend ne re-tente le scan plus tard.

**Conséquence architecturale — pas de repli du tout.** Contrairement au cas *Kingdom* (§1-2), où
l'échec dégrade vers `bnf_only` (la notice BnF, au moins, est retournée), ici la BnF est **le tout
premier maillon de la chaîne** : sans notice BnF, il n'y a **aucun titre** à interroger côté MangaDex ou
MAL (les deux sont cherchés par titre, jamais par ISBN — cf. `sources-et-pivot-isbn.md` §Le problème).
`MalAdapter.searchByBarcode` court-circuite donc immédiatement :

```ts
// src/oauth/providers/mal.adapter.ts:279-285
const resolution = await this.bnf.resolveByIsbn(isbn);
if (!resolution.ok) {
  this.logger.warn(`pivot: bnf unresolved isbn=${isbn} reason=${resolution.reason} -> no candidate`);
  return { items: [], nextCursor: null };
}
```

Résultat pour l'utilisateur : un scan d'un manga **légitimement acheté et publié** (Sakamoto Days,
Hellsing) renvoie une liste **vide** — pas de fiche dégradée, pas de message explicite, juste « rien
trouvé ». C'est le pire des trois issues possibles (enrichi > `bnf_only` > vide), et c'est celle qui
frappe justement les sorties les plus fraîches — donc probablement les scans les plus fréquents (achat
day-one, sortie du jour).

### Ce qu'on sait (publiquement) du délai réel dépôt légal → catalogue consultable

Recherche web, aucune source ne donne de **SLA chiffré officiel** — mais plusieurs éléments recoupés
permettent une hypothèse raisonnable :

- **Le dépôt lui-même n'est pas le facteur de retard.** L'obligation légale est de déposer **au plus
  tard le jour de la mise en circulation** (dépôt direct : 48 h avant la vente ; envoi postal : 3 jours
  avant), donc l'exemplaire arrive à la BnF au moment de la sortie, pas des semaines après — cf.
  [Dépôt légal éditeur : mode d'emploi](https://www.bnf.fr/fr/centre-d-aide/depot-legal-editeur-mode-demploi),
  [SNE — Dépôt légal](https://www.sne.fr/le-b-a-ba/depot-legal/).
- **Une « pré-notice » est créée dès réception**, mais elle est générée à partir des métadonnées
  éditeur et **pas encore vérifiée par un catalogueur** — cf.
  [Dépôt légal éditeur : mode d'emploi](https://www.bnf.fr/fr/centre-d-aide/depot-legal-editeur-mode-demploi).
  Or `bib.fuzzyISBN` (l'index SRU utilisé par `BnfService`) n'a renvoyé **aucun** résultat pour ces
  ISBN, pas même une pré-notice — donc l'écart observé se situe **avant même** cette étape rapide, pas
  seulement dans l'enrichissement ultérieur.
- **Le catalogage complet ("selon les normes internationales… et la politique de catalogage de la
  BnF") n'a pas de délai publié** en jours/semaines sur les pages officielles BnF consultées.
- **Signaux indirects de délai en semaines** : le réseau universitaire Sudoc **importe/actualise les
  notices BnF chaque semaine** via ce même service SRU — cf.
  [PUNKTOKOMO — Nouvel import Sudoc](https://punktokomo.abes.fr/2021/02/01/nouvel-import-sudoc-les-notices-du-catalogue-general-de-la-bnf-par-son-service-sru/) ;
  la *Bibliographie nationale française* (le produit curé, distinct du catalogue général lui-même) est
  diffusée par **lots périodiques** (produits mensuels via l'API) — cf.
  [api.bnf.fr — Produits mensuels](https://api.bnf.fr/fr/produits_bibliographiques_mensuels) ; un retour
  d'expérience isolé (forum d'auteurs autoédités) mentionne **plus de 3 semaines** d'attente après dépôt.
  Aucune de ces sources ne documente spécifiquement les mangas/BD.
- Conclusion prudente : l'ordre de grandeur plausible est de **quelques semaines**, cohérent avec
  l'observation (0 à 8 semaines, la moitié des cas dans le mois) — mais ce n'est **pas vérifié**, juste
  une hypothèse convergente de plusieurs indices indirects.

**La mesure la plus fiable resterait interne** : ré-interroger périodiquement le SRU sur les ISBN en
échec (`bnf_not_found` loggués) jusqu'à apparition, ce qui donnerait une **distribution réelle** du délai
plutôt qu'une estimation par recoupement de sources publiques.

## 4. Le cadre de la réflexion — pivot ad hoc vs pipeline stable

Nommer le problème structurel : le système actuel est un **pivot** — chaque point d'entrée (scan,
jaquettes, énumération) **réimplémente sa propre résolution d'identité**, avec ses propres règles
d'acceptation, et ne fait confiance à un résultat antérieur que si l'appelant **repasse explicitement**
les identifiants obtenus (`mangaId`/`malId`). Rien ne garantit que ce fil survit à tous les chemins
d'appel (réimport, refresh, nouvel utilisateur qui importe la même série sans être passé par le scan…).

Une alternative serait un **pipeline** : l'identification d'une série devient une étape de première
classe, produisant un artefact stable (`SeriesIdentity`), **persisté indépendamment de l'appelant**
(clé = série normalisée ou ARK BnF, pas seulement `malId`), que toutes les étapes downstream
(jaquettes, résumé, sync MAL) **consomment sans jamais re-dériver leur propre match**.

| | Pivot (actuel) | Pipeline (alternative) |
|---|---|---|
| Où vit la règle de matching | Dupliquée par point d'entrée (`chooseIdentity`, `chooseCoverCandidate`, `choosePivotCandidate` côté MAL) | Une seule fois, dans l'étape d'identification |
| Fiabilité du 2ᵉ appel (jaquettes) | Dépend du **fil transmis par l'appelant** (`metadata.pivot`) ; si rompu → re-matching dégradé | Dépend du **cache d'identité** (clé série), indépendant de l'appelant |
| Redondance de source à l'entrée | **Aucune** — BnF est un maillon unique et bloquant (§3) ; son absence stoppe toute la chaîne | Une étape d'identification pourrait consulter une **source de repli** (Google Books par ISBN) avant d'abandonner |
| Coût | Faible — pas de nouvel artefact, résolution locale à chaque appel | Plus élevé — cache/table d'identité à maintenir, invalidation à gérer |
| Risque actuel illustré par *Kingdom* (§1-2) | Un identifiant déjà résolu peut se "perdre" en route → repli sur une règle plus stricte que nécessaire | Éliminé par construction (l'identité ne se reperd jamais une fois produite) |
| Risque actuel illustré par Sakamoto Days/Hellsing (§3) | Une source manquante = résultat vide, sans retry ni repli | Un pipeline avec sources redondantes à l'entrée absorbe l'absence temporaire d'une source |

Ce n'est pas un verdict — le pivot actuel a des raisons d'être (mêmes helpers partagés dans
`manga-matching.ts`, architecture best-effort/remplaçable documentée dans
[scan-asynchrone-bullmq.md](scan-asynchrone-bullmq.md)). La question est de savoir si le coût d'un
pipeline (persistance d'identité, sources redondantes) est justifié par la fréquence réelle de ces deux
classes d'échec, ou si des correctifs locaux suffisent.

## 5. Pistes de correction (non tranchées)

**a. Correctif minimal — aligner `chooseCoverCandidate` sur `chooseIdentity`.**
Ajouter le même fallback titre-seul (`titleScore ≥ PIVOT_TITLE_STRONG`) quand aucun auteur ne matche.
Rapide, ne touche qu'une fonction. Risque : réintroduit le faux-positif *One Piece* (variante colorisée)
que ce garde-fou visait à éviter — il faudrait alors une règle plus fine (ex. fallback titre-seul
**seulement si un unique candidat** dépasse le seuil, pas juste le meilleur score).

**b. Unifier la règle d'acceptation.**
Faire porter la décision « ce candidat EST la série » par **une seule fonction partagée**, appelée aussi
bien au scan qu'à la résolution de jaquettes, plutôt que deux implémentations parallèles
(`chooseIdentity` / `chooseCoverCandidate`) qui peuvent diverger silencieusement (comme ici). Réduit la
dette, mais demande de fusionner deux besoins légèrement différents (l'un renvoie une identité complète,
l'autre juste un `mangaId` pour joindre des couvertures).

**c. Fiabiliser le fil `mangaId`/`malId` plutôt que la règle de matching.**
Si le vrai bug est que l'identité de scan **existe** mais ne redescend pas jusqu'à l'appel jaquettes
(cache expiré, app qui ne renvoie pas `metadata.pivot`, réimport hors scan), le correctif utile n'est
pas d'assouplir `chooseCoverCandidate` mais de **retrouver l'identité déjà connue** — ex. cache
d'identité indexé par **titre de série normalisé** (pas seulement `malId`), consulté par
`findManga` avant de retomber sur la recherche+matching à l'aveugle.

**d. Aller vers un pipeline (§4).**
Persister `SeriesIdentity` (résultat de `identify`) dans une table/un cache durable, clé = série BnF
normalisée. `findManga` la consulte **avant** toute recherche MangaDex ad hoc. Plus structurant, aligne
avec la direction déjà prise par `scan-asynchrone-bullmq.md` (cache inter-user, réchauffage gratuit à
l'identify). C'est l'option qui **élimine la classe de bug** plutôt que de la contourner cas par cas.

**e. Repli ISBN→titre quand la BnF est absente (§3).**
Aujourd'hui, `resolution.reason === 'bnf_not_found'` produit un résultat vide immédiat — aucune autre
source n'est tentée. Google Books résout pourtant certains ISBN **directement** (`q=isbn:<isbn>`, déjà
utilisé comme repli jaquette, cf. `sources-et-pivot-isbn.md` §Jaquettes par tome) et pourrait fournir un
**titre minimal** (+ jaquette) le temps que la BnF rattrape son retard de dépôt légal — un item dégradé
(source `google_books`, sans enrichissement série) valant mieux qu'une liste vide pour une sortie du
jour. Alternative moins invasive : ne rien changer côté résolution, mais **exposer un message explicite**
côté app (« pas encore catalogué, réessayez dans quelques jours ») plutôt qu'un échec silencieux
indiscernable d'un vrai ISBN inconnu.

## 6. Questions ouvertes avant de trancher

- Fréquence réelle : le cas *Kingdom* est-il un incident isolé (réimport hors scan) ou un pattern
  fréquent (à instrumenter via les logs `mangadex: no manga match` en prod) ?
- Si (c)/(d) : quelle clé de cache pour l'identité de série — titre normalisé seul (collision entre
  séries homonymes de maisons d'édition différentes ?), ou titre + auteur ?
- Faut-il garder l'exigence auteur stricte **à la première identification** d'une série jamais vue (anti
  faux-positif, cf. *One Piece*) mais l'assouplir pour une **ré-résolution** d'une série déjà identifiée
  avec succès par le passé (le risque d'homonyme a déjà été tranché une fois) ?
- Où vivrait `SeriesIdentity` si persisté : `ApiCacheService` (TTL, déjà en place) suffit-il, ou faut-il
  une vraie table Prisma (identité durable, pas re-dérivable après expiration cache) ?
- **BnF absente (§3)** : faut-il traiter `bnf_not_found` comme un état **transitoire** plutôt que
  définitif ? Un cache négatif court (au lieu d'aucun cache) permettrait un **retry naturel** au prochain
  scan de l'utilisateur sans re-développement — à vérifier si un cache négatif existe déjà sur ce chemin
  (à date, `resolveByIsbn` ne semble rien cacher côté négatif, cf. `bnf.service.ts`).
- Quelle échelle de temps réelle pour le dépôt légal BnF sur les mangas (jours ? semaines ?) — aucune
  source publique BnF ne donne de SLA chiffré (cf. §3 « Ce qu'on sait ») ; l'hypothèse « quelques
  semaines » n'est **pas vérifiée**. La seule mesure fiable serait interne : ré-interroger le SRU sur les
  ISBN en échec jusqu'à apparition, pour obtenir une distribution réelle avant de calibrer un délai de
  retry.

## Fichiers clés

- `src/common/sources/mangadex/mangadex.resolver.ts` — `chooseIdentity` (ligne 174-217, scan),
  `chooseCoverCandidate` (ligne 307-340, jaquettes), `findManga` (ligne 269-298, ordre de fiabilité
  mangaId › malId › auteur).
- `src/common/sources/manga-matching.ts` — helpers partagés (`titleSimilarity`, `matchesAuthor`,
  `PIVOT_TITLE_STRONG`).
- `src/search/search.service.ts` (`buildEditionMapping`, ligne ~158) — construit le `lookup` transmis à
  `resolveSeriesCovers`/`cachedSeriesCovers` à partir de `opts.malId`/`opts.mangaId` (fournis par
  l'appelant) + `mapping.authors` (BnF, toujours dispo).
- `src/oauth/providers/mal.adapter.ts:272-285` (`searchByBarcode`) — court-circuit `{ items: [], ... }`
  immédiat si `resolution.ok === false` (§3), sans repli.
- `src/common/sources/bnf/bnf.service.ts:111-157` (`resolveByIsbn`) — `numberOfRecords=0` →
  `bnf_not_found` ; pas de cache négatif identifié sur ce chemin.
- Contexte architecture actuelle : [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md)
  (notamment §Jaquettes par tome, cas *One Piece*).
- Contexte infra : [scan-asynchrone-bullmq.md](scan-asynchrone-bullmq.md) (cache inter-user, réchauffage
  gratuit d'identité).
