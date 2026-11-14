# Étude — leviers pour lever les limitations de la recherche manga

> **Document interne** (exclu du site publié, comme `audit-limitations-recherche-manga.md`).
> Étude du 2026-10-27 : sources officielles et tests en direct des API, à la révision `0.9.0`
> backend (`0732683`) / `fdd0c80` app. Aucune ligne de code n'a été modifiée.
> Répond à [audit-limitations-recherche-manga.md](audit-limitations-recherche-manga.md), qui
> constate ; les décisions vivent dans [spec-macro-import-serie-manga.md](spec-macro-import-serie-manga.md),
> amendée le même jour.

## En une phrase

MangaDex suffit à ouvrir la recherche de série par titre **sans jeton** (11 titres français sur 12
retrouvés), et la BnF sait déjà **lister les éditions** et **rattraper les nouveaux tomes** — avec
**plusieurs semaines** de retard sur la sortie ; avant d'y brancher un écran, l'énumération d'édition
doit corriger deux défauts mesurés : la **collision d'éditeur** et l'**exclusion des pré-notices**.

## Arbitrages rendus le 2026-10-27

| Question | Décision |
|---|---|
| Source de la recherche texte manga | **MangaDex seul**, sans nouveau provider ; MAL ne sert plus qu'en repli |
| Rattrapage des tomes parus après l'import | **Deux déclencheurs** : l'app à l'ouverture d'une série en cours, et un job serveur périodique |
| Portée de ce chantier | **L'étude seule** : aucun code ; la spec macro porte le découpage |

## 1. Les sources testées

Banc du 2026-10-27 : **12 titres français différents du titre original** — *L'attaque des titans*,
*Le Pacte des Yokai*, *Les Carnets de l'apothicaire*, *Moi, quand je me réincarne en Slime*, *Les
Brigades immunitaires*, *Les Enfants de la baleine*, *La Rose de Versailles*, *L'Habitant de
l'infini*, *Le Requiem du roi des roses*, *Les Gouttes de Dieu*, *Le Sommet des dieux*, *Les
Vacances de Jésus et Bouddha* — plus le piège « Frieren » de
[spec-repli-google-books-pivot.md](spec-repli-google-books-pivot.md) § 4.

| Source | Résultat mesuré | Verdict |
|---|---|---|
| **MangaDex** `GET /manga?title=` + `order[followedCount]=desc` + exclusion des tags Doujinshi et Anthology | **11/12** retrouvés, dont **7 au rang 1** (rangs 2 à 4 pour les autres ; raté : *L'Habitant de l'infini*). « Frieren » → *Sousou no Frieren* **au rang 1**, 3 résultats au lieu de 11 | **Retenu** |
| AniList GraphQL, `format_in: [MANGA, ONE_SHOT]`, tri `SEARCH_MATCH, POPULARITY_DESC` | **10/12 au rang 1** (ratés : *Slime*, *Jésus et Bouddha*) ; synonymes FR, `idMal`, traducteur français listé dans le staff | Repli possible plus tard |
| Wikidata `wbsearchentities` (`language=fr`) | rang 1 sur **9/9** requêtes abouties, avec les identifiants MAL (P4087), AniList (P8731), MangaUpdates (P11149), BnF (P268, rare) — puis **HTTP 429 après ~25 requêtes** anonymes | Écarté en appel direct ; exploitable hors ligne (CC0) |
| Jikan v4 (façade non officielle de MAL) | **injoignable** : timeout à 21 s sur 3 essais | Écarté |
| MangaUpdates `POST /v1/series/search` | titre FR retrouvé via les noms associés, mais deux doujinshi devant *Sousou no Frieren* | Écarté |
| Open Library Covers | **0/5** jaquettes (Black Torch T1-5, Ki-oon) | Écarté |
| Google Books, sans clé | **HTTP 429** « Queries per day » : le quota anonyme est partagé, et il était épuisé | `GOOGLE_BOOKS_API_KEY` à configurer en production (repli du scan) |
| BnF SRU, texte imprimé | des notices d'édition française pour **12/12** | Source d'édition confirmée |

Deux réglages MangaDex font tout le travail sur « Frieren » : le tri **par nombre de suivis** — en
tri `relevance` sans exclusion, la vraie série n'est même pas dans les cinq premiers — et
l'**exclusion des tags de format** (identifiants lus dans `GET /manga/tag`) :

| Tag | Identifiant | Pourquoi l'exclure |
|---|---|---|
| Doujinshi | `b13b2a48-c720-44a9-9c77-39c9979373fb` | œuvres de fans, l'essentiel du bruit |
| Anthology | `51d83883-4103-437c-b4b1-731cb73d786c` | anthologies dérivées |
| Official Colored | `320831a8-4026-470b-94f6-8353740e6f04` | variante colorisée (le piège *One Piece*) |
| Fan Colored | `7b2ce280-79ef-4c09-9b58-12b7c23a9b78` | idem, non officielle |

MangaDex ne référence que de la bande dessinée : le filtre de type que l'audit trouve absent de
`search()` (`MANGA_MEDIA_TYPES`, `mal.adapter.ts:60`) n'a pas d'équivalent à écrire pour les light
novels.

## 2. Le délai de publication et de catalogage BnF

### Ce qui est mesuré

Les quatre ISBN en échec dans les logs de juillet (identifiés dans
[agent/backend/pivot-vs-pipeline-enrichissement.md](agent/backend/pivot-vs-pipeline-enrichissement.md) § 3),
relus le 2026-10-27 :

| Tome | Éditeur | Sortie | Notice créée (`100$a`) | Délai |
|---|---|---|---|---|
| Sakamoto Days T22 | Glénat | 2026-07-01 | 2026-07-28 | **+27 j** |
| Les Années bleues vol. 2 | Casterman | 2026-07-08 | 2026-07-30 | **+22 j** |
| Sentenced to be a Hero T1 | Vega-Dupuis (Belgique) | 2026-05-22 | 2026-08-17 | **+87 j** |
| Hellsing Perfect Edition T05 | Delcourt | 2026-07-09 | — | **toujours absent à J+110** |

C'est un ordre de grandeur, pas une distribution : **3 à 4 semaines** chez les grands éditeurs
français, **environ 3 mois** pour un éditeur hors de France, et des cas plus longs encore.
L'hypothèse « quelques semaines » de pivot-vs-pipeline § 3 tient sur ces cas, sans être
généralisée. Deux tentatives d'élargir l'échantillon ont échoué :

- Google Books, qui aurait daté les nouveautés par éditeur : quota anonyme épuisé (429) ;
- la BnF elle-même : **`bib.LegalDepositDate` vaut la date de création de la notice**, pas la
  sortie — vérifié par dichotomie sur Sakamoto Days (28/07) et Sentenced (17/08) ; *Les Années
  bleues* ne répond pas du tout à ce critère. La BnF ne date donc pas une parution.

### À quoi ressemble une notice récente : la pré-notice

Sakamoto Days T22, relue en entier (UNIMARC, extrait) :

```
100 $a 20260728d2026 …                                    ← création de la notice
200 $a Sakamoto days $h vol. 22
214 $a Grenoble $c Glénat $d 2026
225 $a Shonen manga                                       ← collection de l'éditeur, pas la série
500 $3 87362361 $a Sakamoto days $h vol. 22 $m français   ← titre uniforme + lien d'œuvre
702 $3 18035194 $a Suzuki $b Yūto                         ← l'auteur, en 702 et non en 700
856 $u 1274342 $b Première de couverture                  ← jaquette (cf. § 4)
```

Ni `461` (ensemble), ni `205` (mention d'édition), ni `700/701`. Les index SRU la trouvent pourtant
tous : `bib.title`, `bib.doctype any "a"`, `bib.publisher any "glenat"`, `bib.author all "suzuki"`
(l'index couvre `702`) et `bib.creationdate`.

### Ce que ça change

1. **Le rattrapage verra un tome des semaines, voire des mois, après sa sortie.** Le job serveur peut
   donc être **hebdomadaire** : un passage quotidien n'apporte rien à cette échelle. Requête
   `bib.creationdate >= "<dernière vérification − 7 j>"` — marge contre le délai d'indexation,
   doublons écartés par ISBN — au format strict `AAAAMMJJ` : `"2025-01-01"` renvoie 0 notice **sans
   erreur**. Côté app, la formulation ne promet pas l'immédiat : les tomes apparaissent quelques
   semaines après leur sortie.
2. **La vraie distribution se mesure gratuitement.** Quand le job découvre un tome, journaliser sa
   date `100$a` face à sa première observation côté Acervatim (scan en `bnf_not_found`, repli
   Google Books). C'est la mesure interne que pivot-vs-pipeline § 6 réclamait.
3. **Le scan d'une nouveauté** reste l'affaire du repli Google Books pendant ces semaines — d'où la
   clé API (§ 1).
4. **Les pré-notices doivent être lisibles par l'énumération**, ce qu'elles ne sont pas aujourd'hui
   (§ 3).

## 3. Deux défauts mesurés dans `enumerateEdition`

Mesurés en rejouant, hors du dépôt, la logique de `bnf.service.ts:214-261` sur les vraies réponses
SRU : clé de série `461$t → 225$a → 200$a`, édition `205$a` normalisée, n° de tome
`461$v → 225$v → 200$h`, premier arrivé gardé. À reproduire en test sur fixture au début de SD1.

### Collision d'éditeur

*L'attaque des titans*, édition standard (146 notices) : la logique retient 34 tomes, mais **les
tomes 11 et 17 viennent des notices 2-en-1 France Loisirs** (« 11-12 », « 17-18 ») et non de Pika
— ISBN, titre et jaquette faux. Le groupe « standard » réunit en fait une dizaine d'éditeurs :
Pika, France Loisirs, Pika roman (des romans), @anime (des DVD), Larousse, 404 éditions, les
Éditions de l'Opportun… parce qu'aucun n'a de mention `205`.

⇒ **Une édition s'identifie par le couple (éditeur normalisé, `205$a`)**, et la requête filtre le
texte imprimé (`bib.doctype any "a"`). La normalisation retire les mots génériques (« éditions »,
« éd. ») : « Pika éd. » et « Pika édition » se rejoignent, « Pika roman » reste une autre marque. Ce
couple doit aussi entrer dans le contrat (SD2) et dans la clé naturelle de la série côté app
(AD2), sans quoi la standard Pika et la 2-en-1 France Loisirs se confondent.

### Pré-notices exclues

Sur une pré-notice, faute de `461`, la clé de série retombe sur `225$a` — « Shonen manga », la
collection de Glénat — qui ne vaut jamais « sakamoto days » : **le tome le plus récent disparaît de
l'énumération**, précisément celui que le rattrapage cherche.

⇒ L'appartenance devient : **l'un des** champs `500$a`, `461$t` ou `200$a` égal à la clé de série,
au lieu du premier champ non vide de la chaîne ; `225$a` ne sert plus qu'en dernier recours.

### Deux corollaires

- **Les auteurs.** `extractAuthors` (`bnf.service.ts:418`) ne lit que `700/701`, puis `200$f`. Une
  série dont on ne connaît que des pré-notices n'a donc aucun auteur pour valider MangaDex : lire
  aussi `702`.
- **L'édition d'une pré-notice est inconnue.** Sans `205`, un tome Perfect ou Colossale récent serait
  classé standard et y écraserait le tome de même numéro : appliquer aussi `isSpecialArtEdition`
  (`manga-matching.ts:135`) au titre (`200$a`, `200$e`). Invérifiable aujourd'hui : le seul cas
  connu, Hellsing Perfect T05, n'est toujours pas catalogué.

## 4. Ce que le SRU BnF sait faire et que le code n'utilise pas

Relevé de `operation=explain` et tests du 2026-10-27.

| Capacité | Test | Usage proposé |
|---|---|---|
| `bib.title startswith` | « attaque des tit » → 145 notices (146 en `all` sur le titre entier) | saisie partielle ou approximative |
| Titre original indexé quand la notice le porte | « shingeki no kyojin » → 112 ; « sousou no frieren » → 0 (les notices Ki-oon n'en ont pas) | pont romaji → titre FR, partiel |
| `bib.doctype any "a"` | Frieren 24 → 19 ; *L'attaque des titans* 146 → 140 | écarter DVD, CD et Blu-ray, systématiquement |
| `bib.publisher`, `bib.author` | *L'attaque des titans* + `bib.publisher any "pika"` → 129 | resserrer le lot une fois l'édition choisie |
| `bib.creationdate`, `bib.lastmodificationdate` (`>=`, `<`) | Frieren depuis `"20250101"` → 8 ; avec `"2025-01-01"` → 0, sans erreur | rattrapage (§ 2) |
| `bib.work2bib any "<n° d'œuvre>"`, n° lu en `500$3` (présent dès la pré-notice) | *L'attaque des titans* (`17716689`) → 88 notices : standard 63, colossale 11, limitée 9, couleur 2… ; tomes 31-34 non rattachés | signal d'appartenance et liste des éditions d'une œuvre — en complément, pas comme énumération |
| `maximumRecords` jusqu'à 1000 | le code pagine par 100 (`bnf.service.ts:45`) | moins d'allers-retours sur les grosses séries |
| Pas de tri (`sortKeys` absent) | — | le classement reste côté Acervatim |
| Jaquette en `856 $b Première de couverture`, servie par `https://catalogue.bnf.fr/couverture?appName=NE&idImage=<856$u>&couverture=1` | JPEG de 551 Ko pour Sakamoto Days T22, dès la pré-notice ; *L'attaque des titans* colossale **16/21** notices, dont les T3/T4 absents de Google Books ; standard 66/88 ; Black Torch 0/10 | **à instruire** (§ 6) : jaquette par ISBN, donc juste aussi pour les éditions d'art |

## 5. Le levier par limitation de l'audit

| § audit | Limitation | Levier | Sprint |
|---|---|---|---|
| 1 | Pas de recherche de série par titre dans l'app | Entrée « Rechercher une série » dans la feuille d'ajout manga, à côté du scan (`CollectionScreen.kt:623-641`, aujourd'hui vinyle seulement) ; candidats, puis éditions | AD1 |
| 2 | MAL seul, sous jeton | `GET /v1/search?type=manga&q=` servi par **MangaDex**, sans jeton : un job `search` de plus sur la file `mangadex` existante (`mangadex.processor.ts:32`, 4 req/s) ; MAL seulement si MangaDex ne renvoie rien **et** qu'un jeton existe | SD1 |
| 2 | Pas de titre français | MangaDex cherche dans ses `altTitles` (11/12) ; chaque candidat expose son `titleFr`, extrait comme dans `identify` (`mangadex.resolver.ts:135`). Le titre FR **exact** d'une édition se lit dans les notices BnF, jamais dans la saisie | SD1 |
| 2 | Pas de filtre de type | rien à écrire pour les light novels ; exclusion des quatre tags du § 1 | SD1 |
| 2 | Le titre seul ne départage pas | tri par suivis + exclusions, puis **choix de l'utilisateur** ; le `mangaId` du candidat part vers `edition-mapping` et `import-jobs` (paramètre déjà prévu, `edition-mapping.query.ts:19`) → jointure directe, première étape de `findManga` (`mangadex.resolver.ts:297`), sans aucun rapprochement | SD1 · AD1 |
| 2 | Seuils d'acceptation | inchangés pour le scan ; sans objet pour un candidat choisi | — |
| 2 | Cache texte peu efficace | clé normalisée (minuscules, sans accents ni ponctuation) dans une famille `mangadex:search` mesurée comme `discogs:search:q`, TTL selon l'idiome HIT/MISS/FAIL (`mangadex.types.ts:29-32`). L'enjeu baisse : 4 req/s partagées, contre ~60/min chez Discogs | SD1 |
| 2 · 4 | Aucune liste d'éditions | `listEditions` = projection sur `fetchEditionRecords`, groupée par **(éditeur normalisé, `205$a`)**, notices filtrées texte imprimé ; par ligne : nombre de tomes, dates, `sourceVolumeRange` | SD1 |
| 4 | Il faut le titre FR exact | titre lu dans les notices ; `startswith` pour la saisie ; filtres auteur et éditeur une fois le candidat choisi | SD1 |
| 4 | Troncature à 1000 notices | les filtres réduisent le lot ; pages jusqu'à 1000 notices | SD1 |
| 4 | Page SRU vide cachée 7 jours | TTL d'absence de 24 h, comme `NOT_FOUND_CACHE_TTL_SECONDS` (`bnf.service.ts:41`) | SD1 |
| 4 | Dépendance au catalogage | appartenance « l'un des champs » (§ 3), lecture de `702`, `work2bib` en complément | SD1 |
| 4 | Séries en cours sans rattrapage | `edition-mapping?since=AAAAMMJJ`. **App** : à l'ouverture d'une série en cours dont la dernière vérification a plus de 7 jours. **Serveur** : job hebdomadaire sur les séries synchronisées, une requête SRU **par édition** et non par utilisateur, nouveaux tomes signalés à l'app (« N nouveaux tomes »), import confirmé par l'utilisateur. Délai assumé (§ 2) | SD3 · AD3 |
| 3 | Trous de jaquettes | MangaDex a les jaquettes `ja` des Black Torch T1-5 : le trou relevé chez Google Books vient d'un MangaDex non joint, pas d'une absence de données (à confirmer dans les logs `mangadex: no manga match`) — le `mangaId` transmis ferme ce cas. Jaquettes BnF à instruire (§ 4, § 6) ; sinon trous assumés sur les éditions d'art | SD1 · à instruire |
| 3 | BnF muette sur les nouveautés | délai mesuré (§ 2) ; repli Google Books inchangé, avec clé API | — |
| 3 | `bookDisplayNumber` vide | irréductible : la donnée n'existe pas chez Google | — |
| 5 | L'édition absente du contrat | `edition` et `publisherFr` sur `MangaSerieSchema`, `edition` sur `MangaItemSchema` (`collections/types/manga.ts:17`, `:34`) ; clé naturelle Room v8 = (type, titre, auteur, **éditeur, édition**) | SD2 · AD2 |
| 6 | Recherche locale ASCII seulement | colonne `records.normalized_title` remplie par `normalizeArtistName`, comparée à la saisie normalisée ; `artists.normalized_name` existe déjà (`ArtistEntity.kt:15`). Dans **la même** migration v8 qu'AD2, pour ne migrer qu'une fois | AD2 |
| 6 | `/v1/search` mono-type, sans items utilisateur | hors périmètre | — |

## 6. Licences et conditions d'usage

| Source | Règle | Conséquence |
|---|---|---|
| **MangaDex** | Crédit obligatoire ; « *You CANNOT run ads or paid services on your website and/or apps* », pas de profit tiré de l'API, exceptions possibles pour un service légitime ; `User-Agent` réel, pas d'en-tête `Via` ; ~5 req/s par IP ; une image chargée en hotlink peut être servie fausse | **Bloquant avant le lancement du palier payant** : demander une exception écrite. Le risque existe déjà sur le scan ; « MangaDex seul » le concentre. Le backend envoie déjà un `User-Agent` obligatoire (`HTTP_USER_AGENT`, `http-client.service.ts:64`) ; l'app, elle, charge les jaquettes depuis `uploads.mangadex.org` : ça fonctionne, la conformité reste à confirmer |
| MangaDex — crédit | — | **Écart** : aucun crédit dans l'app. `strings.xml:257-259` déclare `source_discogs`, `source_myanimelist` et `source_bnf`, pas de `source_mangadex` ; seul le site de documentation crédite MangaDex |
| BnF — jaquettes | Rien de spécifique au service `couverture` ; les règles générales soumettent l'usage commercial d'une reproduction à autorisation et redevance, et une œuvre protégée à l'accord de l'ayant droit | Ne rien exploiter avant une réponse écrite de la BnF (`reproduction@bnf.fr`) ; image de 551 Ko, à réduire avant tout affichage en liste |
| AniList (si repli un jour) | Gratuit en usage commercial sous 150 $ de revenus mensuels, licence au-delà ; ni accumulation ni miroir | Compatible avec un palier unique à prix bas, au départ |
| Wikimedia | `User-Agent` descriptif avec contact ; limites resserrées en 2026 | 429 en anonyme : jamais un appel par saisie |

## 7. Ce qui reste irréductible

- **Le délai BnF lui-même.** Parmi les sources ouvertes testées, seule Google Books connaît un tome
  français avant la BnF, avec un titre non normalisé (spec-repli § 4). Les bases commerciales
  (Electre, Dilicom, ISBNdb) n'ont pas été testées.
- **Le n° de tome des titres Google** : `seriesInfo.bookDisplayNumber` reste vide.
- **Les jaquettes des éditions d'art** absentes de Google Books, tant que la piste BnF n'est pas
  instruite.
- **Le pont romaji → titre FR à la BnF** pour les éditeurs dont les notices n'ont pas de titre
  original (Ki-oon) : il passe par MangaDex.

## Annexe — rejouer les mesures

Rejouer à la main, sans boucle : Wikidata et Google Books en anonyme ont répondu 429 pendant
l'étude.

```
# MangaDex — « Frieren », tri par suivis, sans doujinshi ni anthologies
#   → 3 résultats, Sousou no Frieren (mal 126287) en tête
https://api.mangadex.org/manga?title=Frieren&limit=10&order[followedCount]=desc&excludedTags[]=b13b2a48-c720-44a9-9c77-39c9979373fb&excludedTags[]=51d83883-4103-437c-b4b1-731cb73d786c

# BnF — base des requêtes (la requête CQL va, encodée, dans `query`)
https://catalogue.bnf.fr/api/SRU?version=1.2&operation=searchRetrieve&recordSchema=unimarcxchange&maximumRecords=100&query=…

bib.title all "frieren" and bib.creationdate >= "20250101"      → 8
bib.title all "frieren" and bib.creationdate >= "2025-01-01"    → 0 (format refusé en silence)
bib.fuzzyISBN all "9782344068649"                               → la pré-notice Sakamoto Days T22
bib.fuzzyISBN all "9782344068649" and bib.author all "suzuki"   → 1 (l'index couvre 702)
bib.work2bib any "17716689"                                     → 88 (œuvre L'attaque des titans)
bib.title all "attaque des titans"                              → 146 (collision France Loisirs, § 3)

# BnF — jaquette d'une notice (856$u)
https://catalogue.bnf.fr/couverture?appName=NE&idImage=1274342&couverture=1
```

## Sources

- MangaDex — [politique d'usage](https://api.mangadex.org/docs/),
  [limites et exigences](https://api.mangadex.org/docs/2-limitations),
  [liste des tags](https://api.mangadex.org/manga/tag)
- AniList — [conditions d'utilisation](https://docs.anilist.co/guide/terms-of-use),
  [limites de débit](https://docs.anilist.co/guide/rate-limiting)
- BnF — [service SRU](https://www.bnf.fr/fr/service-sru-catalogue-general-de-la-bnf),
  [explain](https://catalogue.bnf.fr/api/SRU?version=1.2&operation=explain),
  [utilisation commerciale d'une reproduction](https://www.bnf.fr/fr/faire-une-utilisation-commerciale-dune-reproduction),
  [dépôt légal éditeur](https://www.bnf.fr/fr/centre-d-aide/depot-legal-editeur-mode-demploi)
- Wikimedia — [politique d'accès aux API](https://www.mediawiki.org/wiki/Wikimedia_APIs/Access_policy)
- Jikan — [documentation v4](https://docs.api.jikan.moe/)

## Documents liés

| Document | Contenu |
|---|---|
| [audit-limitations-recherche-manga.md](audit-limitations-recherche-manga.md) | Les limitations auxquelles cette étude répond |
| [spec-macro-import-serie-manga.md](spec-macro-import-serie-manga.md) | Les décisions et le découpage, amendés le 2026-10-27 |
| [spec-repli-google-books-pivot.md](spec-repli-google-books-pivot.md) | Le repli Google Books du scan, ses mesures, le piège « Frieren » |
| [agent/backend/pivot-vs-pipeline-enrichissement.md](agent/backend/pivot-vs-pipeline-enrichissement.md) | Les ISBN de juillet et l'hypothèse de délai que le § 2 vérifie |
| [agent/transverse/sources-externes.md](agent/transverse/sources-externes.md) | Les providers, la résolution des jetons, le mode dégradé |
