# Jeux de société : code-barres → BoardGameGeek (GameUPC + BGG)

> Remonter : [INDEX.md](INDEX.md) · Sources : [../transverse/sources-externes.md](../transverse/sources-externes.md)
> Code : `acervatim/backend-services/search/src/{common/barcode,sources/gameupc,sources/bgg,search/boardgame}/`
> Mesures faites le **2026-11-04** sur `api.gameupc.com/test` ; fixtures : `search/test/fixtures/{gameupc,bgg}/`.

## TL;DR

- **Aucune source francophone ne remplace GameUPC.** Les bases FR qui connaissent les EAN (MyLudo,
  les e-commerçants) n'ont pas d'API ouverte ; les bases ouvertes (Open Products Facts, GS1) ne
  donnent pas d'identifiant BGG. GameUPC propose un candidat pour les 8 EAN d'éditeurs français
  testés, mais n'en vérifie qu'un : **c'est la boucle de vote des utilisateurs qui fera la
  couverture française**.
- **Le Swagger GameUPC est faux sur des points qui coûtent** : nom du statut, validation des codes
  (inexistante), indexation par chaîne (`019962194719` ≠ `0019962194719`), 403 non documenté.
- Implémenté dans **`search`** (pas de nouveau service) : normalisation GTIN, client GameUPC derrière
  une interface, deux files (consultation, feedback), client BGG `thing` **désactivé tant que la
  licence BGG n'est pas accordée**, deux routes `/v1`. 338 tests `search` et 31 gateway verts.

## 1. Où ça s'insère dans `backend-services`

Le chantier en cours de la constellation est le découpage (README de `backend-services`, § 6) et la
recherche manga (`feat/manga-recherche-serie`). Les jeux de société n'y figuraient pas. La spec en
brouillon du 2026-11-03,
[spec-catalog-boardgame-bgg-data.md](../../_bmad-output/implementation-artifacts/spec-catalog-boardgame-bgg-data.md),
prévoit `catalog-boardgame` et un service de référence local `bgg-data` à la manière de
`rebrickable-data`, conditionné à la licence BGG (copie locale non couverte explicitement). La
résolution d'un code-barres, elle, est de la **recherche externe** : elle va dans `search`, qui
porte déjà les clés serveur, les files BullMQ et `api_cache`, et en respecte les règles (aucun
appel sortant hors worker, aucun secret dans Redis, aucune clé de cache par utilisateur).

**Conflit avec la spec, à trancher** : elle réserve les appels BGG au moissonnage de `bgg-data`
(jamais sur le chemin d'une requête, 1 req / 5 s), alors que la file `bgg` d'ici appelle `thing` à
la demande (1 req / 2 s). Deux appelants sur un même jeton dépasseraient les deux plafonds.
Recommandation : `bgg-data` seul appelant, `search` lit ses fiches et lui demande un id en
priorité ; `BggClient` est une interface, son implémentation se remplace. Vue d'ensemble et autres
décisions : [../../dossier-jeux-de-societe-bgg-gameupc.md](../../dossier-jeux-de-societe-bgg-gameupc.md).

Le flux « ci-dessus » du brief ne m'a pas été transmis : celui du § 4 est **reconstitué** à partir
des contraintes et des livrables du brief et de la consigne d'intégration GameUPC (choix parmi les
suggestions **avec** un champ de recherche). À confronter à l'original.

## 2. Sources francophones : y a-t-il mieux que GameUPC ?

| Source | Nature | Accès programmatique | EAN → jeu | Verdict |
|---|---|---|---|---|
| **MyLudo** | App FR de ludothèque, scan d'EAN | Aucune API publique trouvée | Oui, dans l'app | Concurrent direct ; partenariat seulement |
| **Tric Trac** | Base FR de fiches jeux | Aucune API (scrapers tiers seulement) | EAN non confirmés | Non |
| **Philibert, Play-in** | E-commerçants FR (Philibert : groupe Asmodee depuis 2025) | Aucune API ; EAN visibles sur les fiches | Indirect | Scraping exclu (CGU, droit *sui generis* des bases) ; flux affilié à demander |
| **Open Products Facts** | Base ouverte d'origine française | API sans clé ; ODbL (commercial permis, attribution, partage à l'identique) | 1 EAN sur 5 testés | Trop maigre ; au mieux un indice de nom |
| **Verified by GS1 / GS1 France** | Registre officiel des GTIN | 30 requêtes/jour gratuites, API pour les membres | Donne l'entreprise et un libellé, pas l'id BGG | Indice de nom au mieux |
| **GameUPC** | Base communautaire UPC/EAN → BGG (15 000+ codes validés) | API à clé, serveur de test public | Voir ci-dessous | **Retenu** |

EAN d'éditeurs français interrogés sur le serveur de test :

| EAN | Produit | Statut GameUPC | Premier candidat |
|---|---|---|---|
| 3421272101337 | 6 qui prend ! (Gigamic) | `verified` | 6 nimmt! — juste |
| 3664824001215 | Kingdomino (Blue Orange) | `choose_from…` | Kingdomino (63) — juste |
| 3664824000591 | Kingdomino Duel | `choose_from…` | Kingdomino Duel (82) — juste |
| 9782370990822 | Dixit 2022 FR (EAN au format ISBN) | `choose_from…` | Dixit (93) — juste |
| 3558380073574 | Dixit multilingue | `choose_from…` | Dixit (82) — juste |
| 3421272888115 | 6 qui prend ! 30 ans | `choose_from…` | 6 nimmt! / 6 nimmt! 25 Jahre, ex æquo (29) |
| 3558380107071 | Dixit Disney | `choose_from…` | Dixit (63) — **faux**, le bon est 3ᵉ (11) |
| 3421272418824 | 6 qui surprend | `choose_from…` | 6 nimmt! 25 Jahre (63) — **faux** |

Les versions BGG portent la langue : Kingdomino a plus de 60 versions dont 3 françaises, toutes à
confiance 0. Pour un utilisateur français, l'édition est donc presque toujours à choisir.

## 3. Synthèse du Swagger GameUPC

Spécification : `https://gameupc.com/gameupc-oas.yaml` (OAS 3.0, « GameUPC REST API v1.0 »),
rendue par `https://gameupc.com/api.html`. Démos : `demo.html`, `demo.py`.

**Serveurs**

| URL | Rôle |
|---|---|
| `https://api.gameupc.com/test` | Développement ; clé publique `test_test_test_test_test` ; **vidé périodiquement** |
| `https://api.gameupc.com/dev` | Branche de développement, instable |
| `https://api.gameupc.com/v1` | Production ; clé **sur demande par courriel** (adresse donnée sur gameupc.com) ; la clé de test y est refusée (403) |

**Authentification** : en-tête `x-api-key` (schéma `ApiKeyAuth`), sur toutes les routes.

**Routes**

| Méthode et chemin | Paramètres | Réponses | Usage ici |
|---|---|---|---|
| `GET /upc/{upc}` | `search` (termes utilisateur), `search_mode` = `speed` (défaut) \| `quality` (~500 ms de plus, meilleures suggestions) | 200 `UpcInfo` · 400 · 429 | Consultation (`quality`) |
| `POST /upc/{upc}/bgg_id/{bgg_id}` | corps `{ "user_id": "…" }`, ≥ 8 caractères, stable par utilisateur | 200 `UpcInfo` mis à jour · 400 · 429 | Vote |
| `POST /upc/{upc}/bgg_id/{bgg_id}/version/{bgg_version}` | idem | idem | Vote avec édition |
| `DELETE` sur les deux chemins précédents | même `user_id` | 200 `UpcInfo` | Non utilisé (annulation) |
| `GET /upc/{upc}/user_id/{user_id}` | — | 200 `VoteResult` · 404 | Arbitrage d'un vote retenté |
| `GET /qr/game/{bgg_id}`, `GET /qr/user/{user_name}` | `include_avatar` | PNG | Non utilisé |
| `GET /warmup` | — | 200 | Non utilisé (réduit la latence du premier appel, ~1 s) |

Codes de test : `111111111117` (vérifié), `222222222224` (à choisir), `333333333331` (sans données).

**`UpcInfo`** : `upc`, `name`, `searched_for` (à pré-remplir dans le champ de recherche),
`bgg_info_status`, `bgg_info[]` (`id`, `name`, `published`, `thumbnail_url`, `image_url`,
`page_url`, `data_url`, `update_url`, `confidence`, `version_status`, `versions[]`), `status`.
Une version : `version_id`, `name`, `published`, `language`, `language_id`, images, `update_url`,
`confidence`. `version_status` : `verified` (`versions[0]` est l'édition du code), `none`,
`choose_from_versions`.

### Écarts mesurés (Swagger → réalité)

1. Le statut s'appelle **`bgg_info_status`**, pas `bgg_status`. Le client lit les deux.
2. `thumbail_url` (coquille) est en réalité `thumbnail_url` ; `published` est une **chaîne**
   (`"2016"`) ; `language_id`, `stage` et `new` existent sans être documentés ; `bgg_alternatives`
   et `alt_versions`, cités dans des descriptions, n'existent pas.
3. **Aucune validation du code**, malgré le 400 documenté : `abc` devient `000000000000` et rend un
   jeu au hasard, `123` devient `000000000123` (« Blob »), une clé de contrôle fausse ou un GTIN-14
   **créent un enregistrement** (`new: true`). Seuls les paramètres du vote sont contrôlés
   (`bgg_id` numérique, corps présent, `user_id` ≥ 8 caractères).
4. **Indexation par chaîne** : `019962194719` (UPC-A) est vérifié, `0019962194719` (même produit en
   EAN-13) est un autre enregistrement, non vérifié. Un UPC-E (`01234565`) est complété par des zéros
   (`000001234565`) au lieu d'être développé (`012345000065`).
5. **Une consultation a un effet de bord** : un code inconnu est créé chez GameUPC.
6. Clé absente ou fausse : **403 `{"message":"Forbidden"}`** de la passerelle, non documenté ; la
   description du Swagger affirme même que les clés ne sont pas exigées.
7. **Un vote par (code, `user_id`)** : revoter remplace, renvoyer le même vote rend la même réponse,
   un `DELETE` répété rend 200. Sur `/test`, **un seul vote** suffit à passer en `verified` ; la démo
   parle de « deux votes non contestés » en général.
8. La réponse d'un vote nomme les versions autrement que la consultation
   (`Kingdomino (K-Domino\'Z)` contre `Kingdomino (French trial edition)`), apostrophes échappées
   comprises.
9. `data_url` vise `api.geekdo.com/xmlapi2` : un jeton BGG y serait ignoré. `update_url` porte
   l'environnement de la consultation (`/test/`). Les deux sont retirés du modèle et les chemins
   reconstruits.
10. Réponses lourdes : la liste de versions n'est pas bornée (Kingdomino : ~160 Ko).
11. **Aucune limite de débit ni condition d'utilisation publiée** ; seul un `429 Overloaded` est
    documenté.

### Écarts au brief

- Le brief compte quatre issues (verified, choose_from…, inconnu, erreur). GameUPC n'a **que deux
  statuts** : « inconnu » est `choose_from_bgg_info_or_search` avec un `bgg_info` vide (et
  `new: true`), « erreur » un statut HTTP.
- « Aligné sur ses conditions une fois vérifiées » : il n'y en a pas de publiées. Les débits retenus
  sont provisoires (§ 5) et figurent dans les questions ouvertes.
- Le vote existe en deux formes (jeu seul, jeu et édition) et se lit (`GET …/user_id/…`) : cette
  lecture sert à ne pas écraser un choix plus récent par un vote retenté.

## 4. Le flux implémenté

```
App (scan ML Kit : valeur + format)
  GET /v1/search/boardgame/barcode/:code[?q=&format=]          gateway → search, auth user
    normalizeGtin  ── invalide ──> 400 (rien ne part chez GameUPC)
    GameUPC (cache → file `gameupc`)
      verified + bgg_info[0]      → status verified, 1 candidat, details = fiche BGG (si licence)
      choose_from… + candidats    → status choose, candidats + searchedFor
      choose_from… + bgg_info []  → status unknown
      statut non documenté        → choose s'il y a des candidats, sinon unknown
      injoignable / sans clé      → dernière réponse connue (stale: true) sinon unavailable
  POST /v1/search/boardgame/barcode/:code/choice { bggId, versionId? }   202
    jeu (ou version) jamais proposé pour ce code → 409
    déjà vérifié à l'identique   → feedback skipped (already-verified)
    sinon                        → vote enfilé (`gameupc-feedback`), pseudonyme HMAC
    réponse                      → { feedback, reason, details }
```

**Normalisation** (`common/barcode/gtin.ts`) : EAN-13 (préfixes ISBN compris), UPC-A à 12 ou
11 chiffres, EAN-8, UPC-E **développé** si l'app annonce `format=upc_e`, GTIN-14 d'indicateur 0 ;
clé de contrôle vérifiée. Forme canonique GTIN-13 pour l'app, **forme UPC-A à 12 chiffres** pour
GameUPC dès que le GTIN-13 commence par 0.

**Pseudonyme du votant** : `acv-` + 22 caractères de HMAC-SHA-256(`GAMEUPC_VOTER_PEPPER`, userId).
Stable (GameUPC s'en sert pour retirer en bloc les votes d'un fauteur de troubles), non réversible
sans le pepper ; l'identifiant Acervatim ne sort jamais.

**Garde-fou anti-abus** : un vote ne part que pour un jeu (et une version) effectivement proposé pour
ce code, toutes recherches confondues sur 24 h (`gameupc:offered:<upc>`). Sinon, notre API
permettrait d'associer n'importe quel id à n'importe quel code sous nos pseudonymes — et c'est la
clé Acervatim que GameUPC révoquerait. La route de choix a son propre plafond au gateway (30/min).

## 5. Cache, files et débit

| Clé `api_cache` | Contenu | TTL |
|---|---|---|
| `gameupc:upc:<upc>` | Consultation sans recherche | 7 j si vérifié, 24 h sinon |
| `gameupc:upc:<upc>:q:<sha1>` | Consultation avec termes (minuscules, espaces compactés) | idem |
| `gameupc:last:<upc>` | Dernière réponse connue, servie `stale` si GameUPC tombe | 90 j |
| `gameupc:offered:<upc>` | Jeux et versions proposés (garde-fou du vote) | 24 h |
| `bgg:thing:<id>` | Fiche BGG (`thing: null` = id inconnu) | 30 j (7 j si absent) |

Aucune clé ne porte d'identifiant utilisateur. Un vote réussi remplace `gameupc:upc:<upc>` par
l'état renvoyé.

| File | Débit | Tentatives | Sur 429 |
|---|---|---|---|
| `gameupc` (consultation) | 2 req/s, concurrence 1, single-flight par code | 1 (+1 HTTP courte) ; attente 8 s | file en pause (`Retry-After`, sinon 30 s) |
| `gameupc-feedback` (votes) | 1 req/s | 6, exponentiel dès 30 s (~15 min) ; refus = `UnrecoverableError` | idem |
| `bgg` (`thing`) | **1 req / 2 s**, aucune retentative HTTP | 4, exponentiel dès 10 s ; attente 6 s | file en pause sur **429 et 503** (`Retry-After`, sinon 60 s) |

**Idempotence du vote** : `jobId` = sha1(code, jeu, version, votant) déduplique les envois en cours
(double tap, rejeu réseau) ; `removeOnComplete: true` pour qu'un retour au premier choix (A, B, A)
ne soit pas avalé. Rejouer un job est sans effet chez GameUPC (un vote par votant). Sur une
nouvelle tentative, le worker relit le vote enregistré : déjà identique → rien à faire ; plus
récent que le choix → `superseded`.

Les trois files apparaissent dans `GET /v1/admin/queues`.

## 6. Contraintes BGG et GameUPC du brief

| Contrainte | Où elle tient |
|---|---|
| Appels GameUPC et BGG côté serveur, jeton BGG jamais chez le client | Clés lues dans les clients HTTP de `search` seulement ; ni dans Redis, ni dans le cache, ni dans les réponses ; `data_url`/`update_url` retirés |
| BGG ≤ 1 req / 2 s, backoff 429/503 | Limiter de la file `bgg`, une requête par job, pause de file sur 429/503 |
| Cache long sur `thing` | 30 jours |
| `boardgamegeek.com` sans `www` | URL construite par le client ; `BGG_BASE_URL` en `www.` **refusée au démarrage** |
| Débit raisonnable vers GameUPC | 2 + 1 req/s, provisoire |
| Logo « Powered by BGG » lié à boardgamegeek.com | **App** : sur l'écran de résultat (`pageUrl` fourni par candidat) |
| Pas de redistribution par une API tierce publique | Routes sous `auth: 'user'`, servies à l'app seule |
| Licence BGG non accordée | `BGG_API_TOKEN` vide : aucun appel, `details: null`, la résolution vit des données GameUPC |

Configuration (`.env.example`, `deploy/`) : `GAMEUPC_BASE_URL`, `GAMEUPC_API_KEY`,
`GAMEUPC_VOTER_PEPPER`, `BGG_API_TOKEN`, `BGG_BASE_URL`.

## 7. Tests

`../scripts/dev.ps1 test search` (tous verts au 2026-11-04 : 26 suites, 338 tests ; gateway 31).

| Fichier | Couvre |
|---|---|
| `common/barcode/gtin.spec.ts` | EAN-13 et ISBN, UPC-A 12/11 chiffres, EAN-13 et GTIN-14 à zéro de tête, UPC-E (4 règles de développement), EAN-8, clé fausse, `abc`, longueurs, GTIN-14 carton |
| `sources/gameupc/gameupc.client.spec.ts` | Fixtures `verified`, `choose-from` (Kingdomino FR réel), `unknown`, vote, vote lu, 404 ; erreurs 400, 403, 429 avec `Retry-After`, 503, réseau, corps illisible ; nom de statut du Swagger ; clé absente |
| `sources/gameupc/gameupc.processor.spec.ts` | TTL par statut, dernière réponse connue, pause sur 429 ; vote envoyé, **rejoué deux fois**, retry sur panne et 429, refus non retenté, arbitrage `superseded` |
| `sources/gameupc/gameupc.service.spec.ts` | Single-flight, `jobId` déterministe du vote, options de retry, files injoignables |
| `search/boardgame/boardgame-barcode.service.spec.ts` | Chaque branche de statut, statut non documenté, normalisation avant l'appel, 400 ; repli `stale`, `unavailable`, recherche sans `stale`, sans licence BGG ; choix : pseudonyme, 409, déjà vérifié, version, pepper absent, file injoignable |
| `sources/bgg/bgg.spec.ts` | Parseur `thing` (entités double-échappées, extension, `Not Ranked`), URL sans `www`, Bearer, 429/503, 401, 202, pause de file |

Fixtures GameUPC : réponses réelles du serveur de test, réduites. Fixture BGG : **construite**
d'après le format documenté (aucun appel BGG sans licence), à remplacer par une capture.

## 8. Questions ouvertes

1. **Clé de production GameUPC.** À demander par courriel. À faire préciser dans la même demande :
   usage par une app gratuite (palier premium payant prévu plus tard), attribution attendue, une
   clé par environnement ?
2. **Quotas GameUPC.** Aucune limite publiée ; 2 + 1 req/s retenus par prudence. Demander la limite
   réelle et la conduite attendue sur 429.
3. **Vote de version.** Faut-il faire choisir l'édition ? Pour un utilisateur français, c'est elle
   qui distingue l'EAN FR (Kingdomino : 3 versions françaises sur 60+, toutes à confiance 0).
   Recommandation : oui, en option, versions de la langue de l'utilisateur et multilingues en tête.
   À confirmer avec GameUPC : un vote de version compte-t-il aussi pour le jeu ?
4. **Seuil de vérification en production** : un vote sur `/test`, « deux votes non contestés » selon
   la démo. Conditionne le délai avant qu'un EAN FR devienne `verified` pour tous.
5. **Représentativité du serveur de test.** Les mesures du § 2 viennent de `/test`, vidé
   périodiquement ; à refaire sur `/v1` avec la clé de production.
6. **RGPD.** Les choix partent chez GameUPC sous pseudonyme. Faut-il les retirer à la suppression du
   compte ? Il faudrait alors garder la liste des votes par utilisateur, ce que `search` s'interdit ;
   sans elle, le lien se rompt de fait avec l'identifiant. À trancher, et à mentionner dans
   `legal/confidentialite.md`.
7. **Licence BGG.** Page officielle « Using the XML API » (version du 2025-07-02) relue le
   2026-11-13, détail au § 3.1 du dossier : licence **non commerciale** demandée pour l'instant
   (app gratuite, personne physique), licence commerciale à obtenir avant le premium (en général
   gratuite jusqu'à 100 utilisateurs payants) ; appels serveur et cache recommandés ; un jeton par application,
   jamais demandé à l'utilisateur final (**pas de BYOT BGG**) ; dump téléchargeable avec le jeton ;
   logo officiel lisible, lié à BGG. Les « XML API terms of use » restent à relire. À préciser dans
   la demande : comptage des utilisateurs payants, cache de 30 jours, hotlink des images
   `cf.geekdo-images.com` (GameUPC les renvoie déjà), emplacement du logo.
8. **Type de collection.** `game` (servi par `catalog-generic`) ou un futur `boardgame` ? Le code
   `game` est ambigu avec le jeu vidéo.
9. **Premium.** Les deux routes sont ouvertes à tous (`premium: 'none'`), comme le reste de la
   recherche. À confirmer.
10. **Repli sans GameUPC.** Ni `stale` ni réponse : l'app propose la saisie manuelle. Une recherche
    texte BGG (`xmlapi2/search`) est possible une fois la licence obtenue ; non implémentée.
11. **Compléments FR.** Partenariat MyLudo, flux affilié Philibert, GS1 ou Open Products Facts comme
    indice de nom pour les codes inconnus : décisions produit, aucune n'est nécessaire au lancement.
12. **App.** Envoyer `format` (indispensable pour un UPC-E), afficher le logo BGG, proposer la
    recherche pré-remplie par `searchedFor` à côté des candidats.
