# Contexte d'implémentation — repli Google Books quand la BnF est muette

> **Document interne** (exclu du site publié, comme `spec-macro-partage-collection.md`).
> Créé le 2026-08-24. Destiné à un agent qui implémente dans `acervatim-backend`.
> Page publique à mettre à jour en fin de chantier :
> [developpeur/sources-et-pivot-isbn.md](developpeur/sources-et-pivot-isbn.md).

## 1. Mission

Le scan d'un manga dont la **BnF ne connaît pas l'ISBN** renvoie aujourd'hui une liste vide : tout le
pivot est conditionné à la notice BnF. Il faut ouvrir un **chemin de repli**
`Google Books (titre par ISBN) → échelle de troncature → MangaDex`, et, à défaut d'identification,
renvoyer un **item minimal** plutôt que rien.

Ce document contient toutes les mesures nécessaires : **ne re-mesure rien**, implémente.

## 2. L'incident de départ

```
LOG  [MalAdapter] pivot: start isbn=9782808703437
LOG  [BnfService] bnf: parsing isbn=9782808703437 numberOfRecords=0 parsed=0
WARN [BnfService] bnf: not found isbn=9782808703437
WARN [MalAdapter] pivot: bnf unresolved isbn=9782808703437 reason=bnf_not_found -> no candidate
LOG  [HTTP] GET /v1/search?type=manga&barcode=9782808703437 200 20ms
```

*Sentenced to be a Hero* T1, Vega-Dupuis, paru le 22/05/2026. La BnF n'a réellement aucune notice
(vérifié en direct sur le SRU : `bib.fuzzyISBN` **et** `bib.isbn` → `numberOfRecords 0`). Deux causes
qui se cumulent et qui définissent la population visée : **parution récente** (retard de catalogage du
dépôt légal) et **éditeur non français** (préfixe 978-2-8087, belge).

La série existe pourtant sur MangaDex : `5e8b3e5a-9e19-47e7-8921-0bfe5fdc7f46`, `links.mal = 151361`.
Seul le maillon d'entrée manquait.

## 3. Déjà fait — ne pas refaire

Le **TTL négatif** est corrigé et testé (21/21 sur `src/common/sources/bnf`) :

- `NOT_FOUND_CACHE_TTL_SECONDS = 24 * 3600` : une absence n'est plus cachée 30 jours comme une notice,
  sinon une nouveauté reste morte un mois après son catalogage réel.
- `resolveByIsbn` déroule `cache.get` → `fetchSru` → `parseUnimarc` → `cache.set`, parce que le TTL
  dépend de la réponse. Une entrée servie du cache n'est **pas** réécrite (sinon le TTL se prolonge
  indéfiniment).
- `parseUnimarc` expose `isSruResponse` : un corps sans `<numberOfRecords>` (page d'erreur, diagnostic)
  redonne `bnf_unparsable` et **n'est pas caché du tout**. Avant, il partait en `numberOfRecords=0`,
  donc en fausse absence.

Fichiers : `src/common/sources/bnf/bnf.service.ts`, `unimarc.parser.ts`, `bnf.service.spec.ts`.

Reste ouvert, **hors périmètre** : `enumerateEdition` cache aussi une page SRU à 0 notice pendant
7 jours.

## 4. Mesures acquises

Échantillon : **137 tomes numérotés** (ISBN-13 + n° de tome), éditeur lu dans la notice BnF
(`210$c`/`214$c`), passés dans le chemin réel BnF → ISBN → Google Books → MangaDex.

| éditeur | tomes | résolus | titre de tome seul | coffret/goodies | absent de Google Books |
|---|---|---|---|---|---|
| Glénat | 27 | 11 | 7 | 4 | 5 |
| Pika | 20 | 17 | 1 | 2 | 0 |
| Ki-oon | 20 | 13 | 7 | 0 | 0 |
| Kurokawa | 19 | 16 | 0 | 3 | 0 |
| Kana | 13 | 11 | 0 | 2 | 0 |
| Crunchyroll | 13 | 11 | 1 | 1 | 0 |
| Ototo | 6 | 6 | 0 | 0 | 0 |
| Panini | 5 | 3 | 0 | 2 | 0 |
| Meian | 5 | 4 | 0 | 0 | 1 |
| Kazé | 3 | 3 | 0 | 0 | 0 |
| **TOTAL** | **137** | **98** | **19** | **14** | **6** |

Faits qui contraignent la conception :

1. **`seriesInfo.bookDisplayNumber` est vide sur 137/137.** Le champ structuré vanté dans
   `googlebooks.types.ts` n'existe pas sur le corpus manga FR : le n° de tome ne peut venir que du
   titre.
2. **Le titre Google n'est pas normalisé** — quatre formes au moins : `One Piece Tome 112`, `Frieren`,
   le **titre du tome seul** (`Ryomen Sakuna`, `Merci`), et du déchet
   (`Boite avec 1 tote bag, 1 serre-livre metal…`). Aucun pattern fiable à extraire.
3. **MangaDex ne tolère aucun bruit** : `title=One Piece Tome 112` → `total=0`. On ne peut donc pas
   envoyer le titre brut et laisser le scoring trancher — il n'y a aucun candidat à scorer. Corollaire
   heureux : une requête fausse ne coûte qu'un aller-retour, jamais une mauvaise série.
4. **`MANGA_SEARCH_LIMIT = 10` est trop court** : sur `title=Sentenced to be a Hero`, MangaDex renvoie
   30 résultats et la bonne série est **rang 12**.
5. **Google Books ne renvoie `authors` que dans 42 % des cas** (35/83 notices lues ; `publisher` :
   14 %). Le validateur habituel du pivot (`matchesAuthor`) est donc absent une fois sur deux.
6. Le **n° de tome tiré du reste tronqué est exact 57/57** (le seul écart, `9` vs `09`, est du
   zéro-padding BnF : comparer en numérique).
7. Le **scorer de prod** (`titleSimilarity` + bonus `exact` + départage par rang), rejoué sans auteur
   sur les pièges, se comporte ainsi :

```
Naruto                 -> Naruto                        conf=0.598   (et non "Uzumaki : Illegitimate")
My hero academia       -> Boku no Hero Academia         conf=0.597   (et non "Vigilante … Illegals")
Tokyo Revengers        -> Toukyou Revengers             conf=0.599   (et non "… Baji Keisuke")
Bleach                 -> Bleach                        conf=0.599   (et non "Bleach (Official Colored)")
One Piece / Blue Lock / Chainsaw Man                    conf≈0.600
Sentenced to be a Hero -> Yuusha-kei ni Shosu…          conf=0.488
Frieren                -> Frieren Cinnamoroll Kamigata  conf=0.500   ← FAUX POSITIF
```

C'est le bonus `exact` (+0.10) qui sauve les sept premiers. Il ne sauve pas **Frieren** : la série
réelle s'appelle `Sousou no Frieren`, aucun candidat n'a d'égalité exacte avec « Frieren », et un
crossover parasite passe devant à `titleSimilarity` 1.00. **Classe de panne à traiter : titre FR =
troncature du titre original, requête d'un seul token, aucun auteur pour arbitrer.**

## 5. Conception à implémenter

### A. Point d'entrée — `MalAdapter.searchByBarcode`

Aujourd'hui (`src/oauth/providers/mal.adapter.ts:279-285`) tout échec BnF renvoie `items: []`.
Ne dérouter que **`bnf_not_found`** :

- `bnf_not_found` → repli Google Books (ci-dessous) ;
- `bnf_unavailable` / `bnf_unparsable` → comportement inchangé (liste vide). Ce sont des pannes
  transitoires : y greffer un repli dégraderait la qualité pendant un incident BnF au lieu de laisser
  l'app retenter.

### B. Titre par ISBN chez Google Books

`GoogleBooksResolver` a déjà un `fetchVolumeByIsbn` privé, utilisé par `fetchCover`. Ajouter une
méthode publique côté `GoogleBooksCoverService` (ou un service frère) qui renvoie
`{ title, authors, publishedDate } | null`, **en passant par la file `gbooks` existante** (throttle
sortant, single-flight, cache) — jamais d'appel HTTP direct depuis l'adapter.

TTL : suivre l'idiome maison `HIT_TTL` / `MISS_TTL` / `FAIL_TTL` de `googlebooks.types.ts`. Un titre
est stable → TTL long ; une absence peut être comblée → TTL court ; un échec dur (503, dont Google est
coutumier sur `isbn:`) → `FAIL_TTL`.

### C. Échelle de troncature — la pièce centrale

**Ne code aucune liste de mots** (`Tome`, `T.`, `vol.`…) : les titres ne sont pas normalisés, et une
liste serait fausse pour la moitié du corpus. Un seul garde-fou, non linguistique :

> On ne tronque **que si la queue retirée contient un chiffre** — on n'enlève des tokens que quand on
> retire plausiblement un numéro de tome.

Sans ce garde-fou, `Instinct grégaire` (tome Ki-oon de Jujutsu Kaisen) se tronque en `Instinct` et
matche une série réellement nommée *Instinct*. Avec, il ne se tronque jamais.

```
ladder(title):
  tokens = title.split(/\s+/)
  for cut in 0 .. min(6, tokens.length - 1):
      prefix = tokens.slice(0, tokens.length - cut).join(' ').trim(' .,-:')
      rest   = tokens.slice(tokens.length - cut).join(' ')
      if cut > 0 and (rest ne contient aucun chiffre  ou  prefix.length < 4): continue
      yield { prefix, tome: dernier entier de rest }
```

Ordre : du titre complet au plus court. On s'arrête au **premier** préfixe accepté par MangaDex. En
pratique le succès tombe à la 3ᵉ requête au pire ; garder `cut ≤ 6` (le cas
`Chainsaw Man Tome 18 . Edition collector` en demande 5).

Le n° de tome retenu est celui de la queue retirée du préfixe **gagnant** — il n'est donc cru que
parce que le préfixe, lui, a matché une vraie série.

### D. Identification MangaDex

- Passer `MANGA_SEARCH_LIMIT` de 10 à **30** (`src/common/sources/mangadex/mangadex.resolver.ts:23`).
  Mesure 4 : sinon la bonne série n'est jamais dans le lot scoré. Cela profite aussi au chemin
  nominal ; le coût est la taille de la réponse, rien d'autre.
- Appeler `mangaDex.identifySeries(prefix, authors)` avec les auteurs Google **quand il y en a**
  (42 % des cas) : `matchesAuthor` retrouve alors son rôle d'arbitre.

### E. Acceptation renforcée en l'absence d'auteur

Le seuil actuel sans auteur (`titleScore >= PIVOT_TITLE_STRONG`) laisse passer le cas Frieren.
**Ne baisse ni ne monte `PIVOT_TITLE_STRONG`** — ajoute un critère d'**ambiguïté** :

> Calculer, dans `chooseIdentity`, si un **autre** candidat égale le meilleur `titleScore`. Exposer ce
> booléen dans `MangaDexIdentity` (`ambiguous`). Le chemin nominal l'ignore (l'auteur tranche) ; le
> repli Google Books **rejette** quand `ambiguous && !authorMatched`.

Calculer le drapeau **toujours** et le stocker dans l'identité cachée, plutôt que de passer un mode
d'appel : `identityCacheKey(query)` et `identityJobId(query)` sont indexés sur la seule requête, un
paramètre de mode empoisonnerait le cache entre les deux chemins.

Vérifie ce critère contre les cas de test §7 avant d'aller plus loin : il doit rejeter *Frieren
Cinnamoroll Kamigata* **sans** rejeter *Sentenced to be a Hero* (candidat unique, `total=1`).

### F. Résultat

- **Identifié** → item `source='mangadex'` comme le chemin nominal, avec
  `metadata.pivot.resolutionPath = 'gbooks+mangadex'`. Attention : `buildMangaDexItem(identity,
  notice, isbn, useOriginal)` et `buildScannedTomeMeta(notice, isbn)` exigent aujourd'hui une
  `BnfNotice`. Les rendre tolérants à son absence (paramètre optionnel) plutôt que fabriquer une
  fausse notice : `titleFr` = titre Google, `seriesTitleFr` = préfixe gagnant, `volume` = n° extrait,
  `edition` / `publisherFr` / `sourceVolumeRange` = `null`.
- **Non identifié** → item minimal `source='gbooks'`, `resolutionPath = 'gbooks_only'`, titre Google +
  tome extrait, sur le modèle de `bnfOnlyResult` (`mal.adapter.ts:599`). Objectif : que l'app propose
  la saisie manuelle au lieu d'un « aucun résultat » muet.
- **Rien chez Google non plus** → `items: []`, avec un log explicite.

Logguer chaque étape dans le style existant (`pivot: …`), en indiquant le préfixe essayé, le nombre de
requêtes MangaDex et la raison du rejet — c'est ce qui rendra le taux de succès mesurable en prod.

## 6. Ce qu'il ne faut pas faire

- Pas de liste de mots-clés de tome (« Tome », « T. », « vol. », « Edition collector »…).
- Pas de nouvelle dépendance npm (règle du dépôt : discussion préalable obligatoire).
- Pas d'appel MAL sur ce chemin : MAL n'a pas de lookup ISBN et exige un jeton, donc rien en mode
  dégradé. Le repli Google Books doit fonctionner **sans jeton**, comme le chemin MangaDex.
- Ne pas toucher au chemin nominal BnF → MangaDex, hormis `MANGA_SEARCH_LIMIT` et le drapeau
  `ambiguous`.
- Ne pas assouplir `PIVOT_TITLE_STRONG`.

## 7. Cas de test

Fixtures : **aucun appel réseau dans les tests**. Le titre Google et les candidats MangaDex se stubent ;
ces valeurs sont des relevés réels du 2026-08-23.

| classe | ISBN | titre Google | attendu |
|---|---|---|---|
| `<Série> Tome n` | 9791042021030 | `Spy X Family Tome 16` | SPY×FAMILY, tome 16 |
| » | 9791043306471 | `Blue Lock Tome 33` | Blue Lock, tome 33 |
| » | 9782820354464 | `Chainsaw Man Tome 22` | Chainsaw Man, tome 22 |
| titre nu | 9782723495943 | `Berserk` | Berserk, sans tome |
| » | 9791032711897 | `Frieren` | Sousou no Frieren, sans tome |
| bruit après le n° | 9782820352545 | `Chainsaw Man Tome 18 . Edition collector` | Chainsaw Man, tome 18 |
| zéro-padding | 9791032707821 | `Jujutsu Kaisen Tome 9` | tome `9` ≡ BnF `09` |
| unicode | 9782820353443 | `Kaiju n°8 Tome 16` | Kaiju No. 8, tome 16 |
| » | 9782377176618 | `Kaijû girl carameliser` | Otome Kaijuu Caraméliser |
| **piège spin-off** | 9782505116882 | `Naruto Edition Hokage Tome 22` | **Naruto**, jamais `Uzumaki : Illegitimate` |
| » | 9791032706084 | `My Hero Academia Tome 23` | **Boku no Hero Academia**, jamais `Vigilante … Illegals` |
| » | 9782344041611 | `Tokyo Revengers Tome 13` | **Toukyou Revengers**, jamais `… Baji Keisuke` |
| **piège édition** | 9782344012895 | `Bleach Tome 67` | **Bleach**, jamais `Bleach (Official Colored)` |
| **piège 1 token** | — | `Frieren` | Sousou no Frieren, jamais `Frieren Cinnamoroll Kamigata` |
| titre de tome seul | 9791032705544 | `Ryomen Sakuna` | pas de série → `gbooks_only` |
| » | 9791032707517 | `Instinct grégaire` | pas de série, **surtout pas** `Instinct` |
| » | 9782723478991 | `Merci` | pas de série |
| coffret/goodies | 9782380714340 | `Boite avec 1 tote bag, 1 serre-livre…` | pas de série |
| absent de Google | 9782344073674 | — | `items: []` |
| cas fondateur | 9782808703437 | `Sentenced to be a Hero Tome 1` | Yuusha-kei ni Shosu…, tome 1, `mal_id 151361` |

Tests attendus :

- **unitaire, `ladder()`** : table-driven sur les titres ci-dessus (variantes produites, n° extrait,
  garde-chiffre) — c'est une fonction pure, elle doit être testée seule.
- **unitaire, adapter** : `bnf_not_found` → repli appelé ; `bnf_unavailable` → pas de repli ;
  identification → `resolutionPath 'gbooks+mangadex'` ; échec → `gbooks_only`.
- **unitaire, `chooseIdentity`** : le drapeau `ambiguous` sur les pièges spin-off / Frieren.
- **e2e** avec `HttpClientService` stubé, sur le modèle de `bnf.e2e.spec.ts`.

## 8. Pièges d'environnement (lire avant de lancer les tests)

- **Node, npm et jest n'existent que dans le conteneur `app`** :
  `docker compose exec -T app npx jest <chemin>`.
- **Les suites e2e à Redis réel échouent tant que le serveur de dev tourne** : son worker consomme la
  même file et vole les jobs (symptôme : le stub HTTP n'est jamais appelé, et le XML reçu vient de la
  vraie BnF). Lancer contre un Redis isolé :

```
docker run -d --rm --name redis-e2e --network acervatim-backend_default redis:7-alpine
docker compose exec -T -e REDIS_HOST=redis-e2e app npx jest src/common/sources
docker rm -f redis-e2e
```

- `googlebooks.e2e.spec.ts` a 1–2 tests instables **préexistants** (résultat variable d'un run à
  l'autre, hors de ce chantier) : ne pas les imputer à tes changements, ne pas les « corriger » au
  passage.
- L'arbre de travail de `acervatim-backend` porte beaucoup de modifications non commitées :
  **ne jamais `git stash`** pour isoler un test.
- Commentaires en français, expliquant le *pourquoi* (le dépôt en est dense et cohérent) ; suivre
  l'idiome `HIT/MISS/FAIL_TTL` pour toute nouvelle constante de cache.

## 9. Définition de fini

1. Un scan de `9782808703437` renvoie la série MangaDex avec son `mal_id`, `resolutionPath
   'gbooks+mangadex'`, tome 1.
2. Les quatre pièges du §7 renvoient la série principale, jamais le spin-off ni l'édition colorisée.
3. `Instinct grégaire` et `Frieren Cinnamoroll Kamigata` sont refusés.
4. `docker compose exec -T app npx jest src/common/sources src/oauth/providers` au vert (Redis isolé),
   hors instabilité gbooks préexistante.
5. `developpeur/sources-et-pivot-isbn.md` mis à jour : le pipeline y est décrit comme BnF-gated, ce qui
   devient faux — ajouter la branche de repli, son taux de succès mesuré et ses limites (titre de tome
   seul, ISBN absent de Google). Répercuter dans `agent/backend/`.

## 10. Limite connue, à assumer

Le repli ne peut rien pour les ISBN dont Google renvoie le **titre du tome** sans nom de série
(Ki-oon, vieux One Piece Glénat) ni pour les 6/137 absents de Google Books : l'information n'existe
pas, aucun parsing ne la fabriquera. Ordre de grandeur attendu : **~72 % des tomes de l'échantillon,
~84 % en écartant coffrets/goodies et ISBN absents**. Réserve de méthode : l'échantillon est constitué
d'ISBN que la BnF **possède**, alors que le repli ne se déclenche que quand elle est muette — sur les
nouveautés et les éditeurs non français, la distribution des formes de titre est probablement moins
favorable.
