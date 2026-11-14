# Jeux de société : le dossier BoardGameGeek et GameUPC

> **Document interne** (exclu du site publié). Il rassemble tout ce qui concerne les jeux de société
> au **2026-11-04** : les études, ce qui est construit, ce qui est prévu, ce qui reste à faire et ce
> qui reste à trancher. Il consolide quatre sources, qui restent les références de détail :
>
> - [agent/backend/gameupc-bgg.md](agent/backend/gameupc-bgg.md) : fiche technique de ce qui est
>   construit (index pour les agents) ;
> - [spec-catalog-boardgame-bgg-data.md](_bmad-output/implementation-artifacts/spec-catalog-boardgame-bgg-data.md) :
>   spec en brouillon du catalogue et de la base de référence BGG ;
> - l'étude des conditions BGG du 2026-11-03, complétée le 2026-11-13 par la page officielle
>   « Using the XML API » ;
> - les échanges de conception du 2026-11-04 (place dans l'architecture, service de feedback).

## En bref

- Scanner une boîte de jeu pose deux questions : **quel est ce jeu**, puis **que sait-on de lui**.
  BoardGameGeek (BGG) décrit tous les jeux mais ne sait pas lire un code-barres. GameUPC fait le pont
  entre le code-barres et l'identifiant BGG.
- **Aucune source francophone ne remplace GameUPC.** Il reconnaît les boîtes françaises, mais ne les
  a presque jamais vérifiées : ce sont les choix de nos utilisateurs, renvoyés sous pseudonyme, qui
  feront la couverture française.
- **BGG : licence non commerciale pour l'instant.** Acervatim est gratuit, édité par une personne
  physique, et ne vend rien. Le palier premium, prévu plus tard, exigera une licence commerciale
  obtenue avant son lancement (en général gratuite jusqu'à 100 utilisateurs payants). Tant
  qu'aucune licence n'est accordée, aucun appel BGG ne part : le code est prêt, mais désactivé.
- **La résolution du code-barres est construite** dans le service `search` et testée, mais **pas
  encore commitée**.
- **Le catalogue des jeux (`catalog-boardgame`) et la base de référence BGG (`bgg-data`) sont
  spécifiés en brouillon**, pas construits.
- Quatre décisions conditionnent la suite : qui appelle BGG et à quel débit, faut-il effacer les
  votes à la suppression d'un compte, faut-il faire choisir l'édition, et que faire si BGG refuse.

---

## 1. Le problème, expliqué simplement

Un collectionneur pose sa boîte de *Kingdomino* devant le téléphone, et l'app lit le code-barres
`3664824001215`. Pour remplir la fiche, il faut répondre à deux questions :

1. **De quel jeu s'agit-il ?** Un code-barres n'est qu'un numéro de produit attribué par l'éditeur.
   Il ne contient ni le titre, ni l'auteur, ni l'édition.
2. **Que sait-on de ce jeu ?** Nombre de joueurs, durée, auteurs, image, extensions.

Acervatim résout déjà ce problème pour les LEGO, avec deux sources : Brickset donne le code-barres
d'un set, Rebrickable sa fiche. Les jeux de société suivent exactement le même schéma :

| Question | Source jeux de société | Équivalent LEGO |
|---|---|---|
| Quel jeu derrière ce code-barres ? | **GameUPC** | Brickset |
| Que sait-on de ce jeu ? | **BoardGameGeek** | Rebrickable |

### Petit lexique

| Terme | Ce qu'il veut dire ici |
|---|---|
| **EAN-13** | Le code-barres européen à 13 chiffres, celui de presque toutes les boîtes vendues en France. Le dernier chiffre est une **clé de contrôle**, calculée à partir des douze autres : elle permet de détecter une lecture fausse. |
| **UPC-A** | Le code nord-américain à 12 chiffres. C'est un EAN-13 qui commence par 0, sans ce 0. |
| **UPC-E, EAN-8** | Des formes courtes à 8 chiffres. UPC-E est un UPC-A compressé (il faut le « développer » pour retrouver les 12 chiffres) ; EAN-8 est le format court européen. À 8 chiffres, on ne peut pas savoir lequel des deux on lit sans que le lecteur le dise. |
| **GTIN** | Le nom de toute cette famille. Le serveur ramène chaque code à une forme canonique à 13 chiffres. |
| **Id BGG** | L'identifiant d'un jeu chez BoardGameGeek (*Kingdomino* : `204583`). C'est la clé qui relie tout. |
| **Version (édition)** | Chez BGG, un jeu a des versions : langue, éditeur, année. *Kingdomino* en a plus de 60, dont 3 françaises. Une boîte correspond à une version. |
| **`thing`** | La route de l'API BGG qui rend la fiche d'un jeu. |
| **Vote (feedback)** | Le message renvoyé à GameUPC pour lui dire « ce code-barres, c'est ce jeu ». |
| **File BullMQ** | Une file de tâches dans Redis. Elle borne le débit vers une source, retente plus tard en cas d'échec et évite d'appeler une source pendant une requête de l'app. |
| **Cache d'API** | La table `api_cache` de `search` : une réponse obtenue pour un utilisateur sert à tous les suivants. |

---

## 2. Les deux sources

### 2.1 BoardGameGeek, la référence

BGG est la base communautaire de référence du jeu de société : fiches, versions, extensions,
classements, notes. On y accède de deux façons :

- **L'API XML2**, qui répond en XML (pas en JSON). Elle exige une application enregistrée et un
  jeton envoyé en en-tête `Authorization: Bearer`. Piège : les requêtes doivent viser
  `boardgamegeek.com` **sans `www`**, sinon le jeton est ignoré sans erreur. BGG peut aussi
  répondre `202` (« réponse en préparation, revenez plus tard »).
- **Le dump CSV des classements** (id, nom, année, rangs, notes). Il exige une application
  approuvée, et se télécharge en étant connecté au site ou avec le jeton d'application.

BGG n'a **pas d'API d'écriture** : on ne peut rien lui renvoyer.

### 2.2 GameUPC, le pont

GameUPC est une base communautaire qui associe des codes UPC et EAN à des identifiants BGG. Elle
annonce plus de 15 000 codes validés. Son fonctionnement est simple à comprendre :

1. On lui envoie un code-barres.
2. Elle répond soit **`verified`** (des humains ont confirmé le jeu), soit **`choose_from_…`** (des
   suggestions classées par confiance, à faire choisir par l'utilisateur).
3. Quand l'utilisateur choisit, on **vote** pour ce choix.
4. Quand les votes concordent, le code devient `verified` pour tout le monde.

C'est un cercle vertueux : chaque utilisateur qui choisit fait gagner du temps aux suivants.

GameUPC expose trois serveurs : `/test` (clé publique `test_test_test_test_test`, vidé
périodiquement), `/dev` (instable) et `/v1` (production, clé **à demander par courriel**). La clé
s'envoie dans l'en-tête `x-api-key`.

---

## 3. Les études menées

### 3.1 Les conditions de BGG (2026-11-03, complétées le 2026-11-13)

L'étude du 2026-11-03 n'avait pas pu lire les pages de BGG (accès refusé à l'outil). Le
2026-11-13, la page officielle **« Using the XML API »** (version du 2025-07-02) a été relue à
partir du texte transmis par l'utilisateur. Ce qui suit en tient compte.

**Quelle licence, et quand.** BGG classe comme commerciale toute application d'une organisation à
but lucratif, ou qui collecte de l'argent de quelque façon, ou qui affichera de la publicité, ou qui
offrira un avantage contre paiement. Aujourd'hui, Acervatim n'entre dans aucun de ces cas : il est
édité par une personne physique, il est gratuit, sans publicité, et rien n'y est vendu. **On
demande donc une licence non commerciale** (décision du 2026-11-13). BGG l'accorde en général sans
frais, mais ses limites d'usage peuvent différer de celles d'une licence commerciale.

Ça changera avec le **palier premium**, prévu plus tard : il fera payer un enrichissement par
jetons serveur, donc un avantage contre paiement. La licence commerciale devra être obtenue
**avant** le lancement du premium. Créer une société pour éditer l'app, ou afficher de la
publicité, aurait le même effet.

Deux précautions pour la demande non commerciale :

- **Annoncer le premium prévu.** Les CGU l'annoncent déjà, et la définition de BGG parle d'une app
  qui « affichera » de la publicité ou « offrira » un avantage contre paiement. Le taire pourrait
  passer pour une demande trompeuse, et coûter la licence au moment de passer en commercial.
- **Ne pas tenir ce passage pour acquis.** La licence commerciale peut être refusée (voir le risque
  principal ci-dessous). Les jeux de société devraient alors continuer sans BGG.

**Ce que coûtera la licence commerciale.** Le tarif dépend de la façon dont l'application gagne de
l'argent. Pour une application payée par ses utilisateurs, comme le sera Acervatim, BGG accorde
**en général, pour l'instant, une licence gratuite jusqu'à 100 utilisateurs payants**. Au-delà, le
prix est fixé au cas par cas, et BGG prévient qu'il peut changer. Pour comparaison : le seuil est
de 1 000 utilisateurs pour une application financée par la publicité, et une boutique en ligne
paie dès le départ.

Ce seuil est bas : le centième abonné premium peut suffire à déclencher une redevance. Il restera à
savoir si BGG compte tous les abonnés d'Acervatim ou seulement ceux qui ont des jeux de société
(question au § 9).

**Le risque principal**, quelle que soit la licence : BGG peut refuser une app qui concurrence « une partie quelconque » de son
activité ou qui lui nuit « de quelque façon ». Il peut aussi retirer la licence d'une app déjà
approuvée pour les mêmes raisons. BGG fait lui-même du suivi de collection : le refus est une issue
réaliste, d'où la question 7.6. La page prévient aussi qu'une application d'entreprise qui dépend
de ces données le fait « à ses risques », puisque l'API et ses règles peuvent changer à tout moment.

| Obligation BGG | Ce que ça implique pour nous |
|---|---|
| Enregistrer l'application (`boardgamegeek.com/applications`) ; réponse en une semaine ou plus | En « Non commercial » maintenant, en « Commercial » avant le premium ; lancer la démarche tôt |
| Jeton `Authorization: Bearer <jeton>` (une espace, pas de deux-points), vers `boardgamegeek.com` sans `www` ; pas de renouvellement pour l'instant | Déjà fait : une URL en `www` est refusée au démarrage |
| Un jeton par application ; ne jamais demander à un utilisateur final d'obtenir le sien | **Pas de BYOT pour BGG**, contrairement à TMDB ou MAL : le jeton serveur est la seule voie, y compris pour les comptes gratuits |
| Appels depuis nos serveurs, résultats en cache, le moins de requêtes possible ; un jeton exposé côté client peut être révoqué | Déjà fait : appels dans les workers de `search`, cache de 30 jours |
| Débit non chiffré, qui peut dépendre de la licence ; la consommation se lit sur `boardgamegeek.com/applications` (« Usage ») | Débit prudent, un seul appelant (question 7.1) ; surveiller « Usage » |
| Aucun service tiers qui ouvre les données à d'autres applications | Déjà fait : routes réservées aux utilisateurs connectés de l'app |
| Seuls l'API XML, les collections et le dump sont sous licence ; les autres API privées du site ne le sont pas | N'appeler que `xmlapi2` ; le `data_url` de GameUPC (`api.geekdo.com`) est déjà écarté |
| Logo « Powered by BGG », lié à BGG, dans toute app publique : n'importe quel fichier officiel, à une taille où le texte reste lisible | Côté app, sur l'écran de résultat |
| Le dump CSV de tous les jeux exige une application approuvée ; il se télécharge connecté au site **ou avec le jeton** | Une fois la licence obtenue, le téléchargement automatique devient possible (§ 7.7) |
| Le téléchargement des collections d'utilisateurs (XML ou CSV) relève du même accord | Un import de collection BGG, non prévu, serait couvert par la même licence |
| Ne pas modifier les données ; pas d'entraînement d'IA | La page relue n'en parle pas : à vérifier dans les conditions d'utilisation |
| Licence révocable à tout moment | Tout ce qui vient de BGG doit pouvoir être purgé d'un coup |
| La copie locale complète n'est pas couverte explicitement | À demander explicitement dans la demande de licence |
| Aucun support technique ; les changements sont annoncés sur le forum « Geek Tools News » | Suivre ce forum |

La page se contredit sur le dump : elle dit d'abord qu'on peut le télécharger sans enregistrement
en étant connecté, puis qu'il exige une application approuvée. Pour Acervatim, la question ne se
pose pas : l'application sera de toute façon enregistrée, pour l'API.

**Ce qui reste à relire.** Les **conditions d'utilisation de l'API** (« XML API terms of use »)
sont une page distincte, pas encore relue. C'est là que devraient figurer l'interdiction de
modifier les données et celle d'entraîner une IA, et peut-être la réponse sur la copie locale et
les durées de cache. **Elles sont à relire** avant d'envoyer la demande.

### 3.2 Les sources francophones (2026-11-04)

La question posée : existe-t-il une source francophone plus pertinente que GameUPC pour passer d'un
EAN français à un jeu ?

| Source | Ce que c'est | Accès par programme | EAN → jeu | Verdict |
|---|---|---|---|---|
| **MyLudo** | App française de ludothèque, avec scan | Aucune API publique | Oui, dans l'app | Concurrent direct ; un partenariat au mieux |
| **Tric Trac** | Base française de fiches de jeux | Aucune API | EAN non confirmés | Non |
| **Philibert, Play-in** | E-commerçants français | Aucune API ; EAN visibles sur les fiches | Indirect | Aspirer leurs fiches est exclu (CGU, droit des bases de données) ; un flux affilié est à demander |
| **Open Products Facts** | Base ouverte d'origine française | API sans clé, licence ODbL | 1 EAN sur 5 testés | Trop maigre ; au mieux un indice de nom |
| **GS1** (Verified by GS1) | Registre officiel des codes-barres | 30 requêtes par jour gratuites | Donne l'entreprise et un libellé, pas l'id BGG | Au mieux un indice de nom |
| **GameUPC** | Base communautaire EAN/UPC → BGG | API à clé | Voir ci-dessous | **Retenu** |

Le banc d'essai : huit EAN d'éditeurs français, interrogés sur le serveur de test.

| EAN | Produit | Statut GameUPC | Premier candidat |
|---|---|---|---|
| 3421272101337 | 6 qui prend ! (Gigamic) | `verified` | 6 nimmt! — juste |
| 3664824001215 | Kingdomino (Blue Orange) | à choisir | Kingdomino — juste |
| 3664824000591 | Kingdomino Duel | à choisir | Kingdomino Duel — juste |
| 9782370990822 | Dixit 2022 FR | à choisir | Dixit — juste |
| 3558380073574 | Dixit multilingue | à choisir | Dixit — juste |
| 3421272888115 | 6 qui prend ! 30 ans | à choisir | deux candidats ex æquo |
| 3558380107071 | Dixit Disney | à choisir | Dixit — **faux**, le bon est 3ᵉ |
| 3421272418824 | 6 qui surprend | à choisir | 6 nimmt! 25 Jahre — **faux** |

**Ce qu'il faut en retenir :**

- GameUPC **reconnaît** les boîtes françaises (8 sur 8 ont des candidats) mais ne les a **presque
  jamais vérifiées** (1 sur 8).
- Son premier candidat est faux une fois sur quatre : l'app ne doit jamais le prendre pour acquis.
  Elle présente les candidats **avec** un champ de recherche, comme le recommande GameUPC lui-même.
- L'**édition** est presque toujours à choisir : les trois versions françaises de *Kingdomino* ont
  toutes une confiance nulle.
- Ces mesures viennent du serveur de test, vidé périodiquement. **Elles sont à refaire en
  production** une fois la clé obtenue.

### 3.3 Le Swagger GameUPC face à la réalité (2026-11-04)

Le Swagger est la description officielle de l'API (routes, paramètres, réponses). On l'a confronté
à des appels réels. Les écarts ne sont pas des détails : plusieurs auraient produit des erreurs
silencieuses.

| Ce que dit le Swagger | Ce qu'on a mesuré | Pourquoi ça compte |
|---|---|---|
| Un code invalide rend `400` | **Aucune validation** : `abc` rend un jeu au hasard, une clé de contrôle fausse crée un enregistrement | On doit valider nous-mêmes **avant** d'appeler, sinon on pollue GameUPC et on affiche n'importe quoi |
| (rien) | **Une consultation crée l'enregistrement** d'un code inconnu | Même conséquence : ne jamais envoyer un code douteux |
| (rien) | **Le code est indexé comme une chaîne** : `019962194719` est vérifié, `0019962194719` (le même produit) ne l'est pas | On doit envoyer la bonne forme (12 chiffres pour un UPC-A), sinon on rate un code vérifié |
| (rien) | Un UPC-E est complété par des zéros au lieu d'être développé | Le serveur développe lui-même l'UPC-E, à condition que l'app annonce le format |
| Le statut s'appelle `bgg_status` | Il s'appelle `bgg_info_status` | Le client lit les deux noms |
| Les clés ne seraient pas exigées | Clé absente ou fausse : `403` non documenté ; la clé de test est refusée en production | Le `403` est traité comme une erreur de configuration |
| Quatre issues dans le brief (vérifié, à choisir, inconnu, erreur) | GameUPC n'a **que deux statuts** ; « inconnu » est un « à choisir » sans candidat | Le serveur reconstitue les quatre issues pour l'app |
| (rien) | **Un vote par couple (code, votant)** : revoter remplace, renvoyer le même vote ne change rien | Les votes peuvent être retentés sans risque |
| Limite de débit | **Aucune limite ni condition d'utilisation publiée** | Débits provisoires, à faire confirmer |

Écarts mineurs : coquilles et types (`published` est une chaîne), champs non documentés, réponses
lourdes (environ 160 Ko pour *Kingdomino*, à cause des versions).

---

## 4. Ce qui est construit (backend, 2026-11-04)

### 4.1 Où ça vit, et pourquoi pas dans un service à part

Le backend d'Acervatim est découpé en microservices. L'un d'eux, `search`, porte **toutes les
recherches auprès de sources externes** : les clés serveur, les files BullMQ et le cache d'API. Il
obéit à trois règles :

- aucun appel réseau depuis un contrôleur, tout passe par une file ;
- aucun secret dans Redis ;
- aucun identifiant d'utilisateur dans les clés de cache, puisque les données de catalogue sont
  publiques et partagées.

Reconnaître le jeu d'un code-barres est une recherche externe, comme Discogs pour un vinyle : elle
va dans `search`.

**Pourquoi pas un microservice GameUPC ?** Il ne posséderait aucune donnée propre, et coûterait
toute l'infrastructure d'un service (Dockerfile, déploiement, copies de `contracts.ts`, gateway,
health check) pour un client, deux files et un cache.

**Pourquoi pas un service dédié aux votes ?** Parce que les votes sont **liés à la consultation** :

- le garde-fou anti-abus et le test « déjà vérifié » lisent le cache de consultation ;
- un vote accepté met ce cache à jour, pour que l'utilisateur suivant voie le code vérifié ;
- votes et consultations partagent la même clé et le même quota GameUPC ;
- le volume est minuscule (au plus un vote par scan d'un code non vérifié) ;
- les pertes sont déjà couvertes : Redis écrit sur disque en production, un vote est retenté six
  fois, et le rejouer ne change rien.

Un service séparé devrait rappeler `search` ou travailler à l'aveugle. Deux évolutions changeraient
la réponse : garder la liste des votes de chaque utilisateur (question 7.3), ou contribuer à
plusieurs sources avec de la modération.

### 4.2 Le parcours d'un scan

Suivons la boîte de *Kingdomino* :

1. **L'app interroge le serveur** :
   `GET /v1/search/boardgame/barcode/3664824001215?format=ean_13`. Le format est celui que donne le
   lecteur de code-barres ; il est indispensable pour un UPC-E.
2. **Le serveur normalise le code.** Il vérifie la clé de contrôle et calcule la forme à envoyer à
   GameUPC. Un code invalide est refusé tout de suite (`400`), sans rien envoyer à GameUPC.
3. **Il consulte le cache**, sinon met une consultation dans la file `gameupc` et attend au plus
   8 secondes.
4. **Il traduit la réponse en l'un des quatre statuts que l'app comprend** :

   | Statut | Situation | Ce que fait l'app |
   |---|---|---|
   | `verified` | Code confirmé : un seul candidat | Affiche le jeu, et sa fiche BGG si la licence est là |
   | `choose` | Des suggestions | Fait choisir, avec un champ de recherche pré-rempli (`searchedFor`) |
   | `unknown` | Aucune suggestion | Propose la recherche, puis la saisie manuelle |
   | `unavailable` | GameUPC injoignable et rien en mémoire | Propose de réessayer ou de saisir à la main |

   Si GameUPC ne répond pas mais qu'on connaît une réponse ancienne, le serveur la sert avec
   `stale: true`, plutôt que rien.
5. **L'utilisateur choisit** son jeu, et éventuellement son édition :
   `POST /v1/search/boardgame/barcode/3664824001215/choice` avec `{ bggId: 204583, versionId? }`.
   Le serveur répond `202` (« accepté, traité en arrière-plan ») avec la fiche BGG si disponible.
6. **Le vote part en arrière-plan** par la file `gameupc-feedback`, sous pseudonyme. Une fois
   accepté par GameUPC, le cache est mis à jour : le prochain utilisateur qui scanne ce code voit
   directement le jeu vérifié.

```
App (scan : valeur + format)
  GET  /v1/search/boardgame/barcode/:code[?q=&format=]
         normalisation ── invalide ──> 400, rien ne part
         cache, sinon file `gameupc`
           verified        → 1 candidat (+ fiche BGG si licence)
           à choisir       → candidats + searchedFor
           sans candidat   → unknown
           injoignable     → dernière réponse connue (stale) sinon unavailable
  POST /v1/search/boardgame/barcode/:code/choice { bggId, versionId? }  → 202
         jeu jamais proposé pour ce code → 409
         déjà vérifié à l'identique      → pas de vote
         sinon                           → vote dans la file `gameupc-feedback`
```

### 4.3 Les garde-fous, et ce qu'ils évitent

| Garde-fou | Ce qu'il évite |
|---|---|
| Normalisation stricte avant tout appel | Créer des fiches fantômes chez GameUPC ; rater un code vérifié parce qu'on l'a envoyé sous une autre forme |
| UPC-E développé seulement si l'app annonce ce format | Confondre un UPC-E et un EAN-8, qui ont tous deux 8 chiffres |
| Un vote n'est accepté que pour un jeu proposé pour ce code dans les dernières 24 h (sinon `409`) | Que notre API serve à associer n'importe quel jeu à n'importe quel code sous nos pseudonymes, ce qui ferait révoquer **notre** clé |
| Au plus 30 choix par minute et par utilisateur (gateway) | Le vote en rafale |
| Pseudonyme `acv-…` calculé par HMAC à partir de l'identifiant | GameUPC ne voit jamais l'identifiant Acervatim ; le pseudonyme est stable, pour qu'il puisse écarter un tricheur |
| Jeton BGG et clé GameUPC lus uniquement dans les workers | Qu'un secret passe par Redis, le cache, une réponse, ou atteigne l'app |
| `www.boardgamegeek.com` refusé au démarrage | Un jeton BGG ignoré sans erreur |
| Routes réservées aux utilisateurs connectés de l'app | Redistribuer les données BGG par une API publique, ce que BGG interdit |
| BGG désactivé tant que `BGG_API_TOKEN` est vide | Un appel BGG sans licence |

### 4.4 Cache et files

Le cache évite de redemander ce qu'on sait déjà. Ses durées suivent la fiabilité de la réponse :

| Ce qui est gardé | Durée |
|---|---|
| Consultation d'un code vérifié | 7 jours |
| Consultation d'un code non vérifié (il peut changer avec les votes) | 24 heures |
| Dernière réponse connue (servie `stale` en cas de panne) | 90 jours |
| Jeux proposés pour un code (garde-fou du vote) | 24 heures |
| Fiche BGG | 30 jours (7 jours pour un id inconnu) |

Les files bornent le débit vers chaque source :

| File | Débit | En cas d'échec | Sur un refus pour excès (`429`) |
|---|---|---|---|
| `gameupc` (consultation) | 2 requêtes/s, une à la fois | Une tentative ; l'app reçoit la dernière réponse connue | File en pause (`Retry-After`, sinon 30 s) |
| `gameupc-feedback` (votes) | 1 requête/s | Six tentatives sur environ 15 minutes ; un refus définitif n'est pas retenté | Idem |
| `bgg` (fiche) | 1 requête toutes les 2 s | Quatre tentatives, espacées de plus en plus | Pause sur `429` et `503` (sinon 60 s) |

Les débits GameUPC sont **provisoires**, faute de limite publiée.

**Pourquoi un vote ne peut pas partir deux fois par erreur.** Trois situations piégeuses sont
couvertes :

- **Le double appui** : deux envois identiques portent le même identifiant de tâche, et la file
  n'en garde qu'un.
- **La nouvelle tentative après une coupure** : avant de renvoyer, le worker relit le vote
  enregistré chez GameUPC. S'il est déjà là, il n'envoie rien.
- **Le changement d'avis** (A, puis B) : si le vote enregistré est plus récent que le choix que la
  tâche porte, celle-ci s'arrête (`superseded`) au lieu d'écraser le choix le plus récent.

### 4.5 Ce qui a été vérifié

- 338 tests du service `search` et 31 du gateway passent ; typage et lint propres.
- Les tests couvrent la normalisation, chaque statut, les replis, l'idempotence et les nouvelles
  tentatives du vote.
- Un essai du vrai client contre le serveur de test GameUPC donne les résultats attendus (code
  vérifié, mauvaise clé, pseudonyme trop court).
- Les fixtures GameUPC sont des réponses réelles, réduites. Pour en capturer une, un vote a été
  envoyé sur le serveur de test (*Kingdomino*, votant `acervatim-probe-0001`), puis annulé.
- La fixture BGG est **construite** d'après le format documenté, puisqu'aucun appel BGG n'a été fait
  sans licence. Elle sera à remplacer par une vraie réponse.

### 4.6 L'état du code

- **Rien n'est commité.** Le travail se trouve dans l'arbre de travail de `backend-services`, sur la
  branche `feat/manga-recherche-serie`, qui est celle de la recherche manga. Il faudra le déplacer
  sur une branche dédiée avant de commiter.
- Emplacements : `search/src/common/barcode/` (normalisation), `search/src/sources/gameupc/`,
  `search/src/sources/bgg/`, `search/src/search/boardgame/` (service et routes), fixtures dans
  `search/test/fixtures/{gameupc,bgg}/`. Les deux routes sont déclarées au gateway.
- Configuration : `GAMEUPC_BASE_URL`, `GAMEUPC_API_KEY`, `GAMEUPC_VOTER_PEPPER` (sans lui, aucun
  vote ne part), `BGG_API_TOKEN` (vide), `BGG_BASE_URL`. Elles sont présentes dans
  `.env.example` et `deploy/`.
- **Une règle est cassée.** `rebrickable-data` copie `src/common/**` de `search` à l'identique. Deux
  modifications de `search` n'y sont pas reportées : le client HTTP qui conserve désormais le
  statut d'origine d'un `502`, et le nouveau dossier `barcode/`. À reporter avant le commit.

---

## 5. Ce qui est prévu : `catalog-boardgame` et `bgg-data`

La spec du 2026-11-03 est **en brouillon**, ni validée ni implémentée. Elle prévoit deux nouveaux
services.

**`catalog-boardgame`** gère les collections de jeux de l'utilisateur :

- un nouveau type de collection, `boardgame`, distinct de `game` qui désigne déjà le jeu vidéo ;
- un **nœud « jeu »** dont les items sont des **boîtes** : la base et ses extensions ;
- l'id BGG est rangé dans les sources de l'item, pas dans ses données, comme pour le vinyle ;
- c'est un catalog comme les autres : clone de `catalog-manga`, aucun code métier hors de son profil.

**`bgg-data`** est une base de référence BGG locale, sur le modèle de `rebrickable-data` :

- elle importe le dump CSV de BGG, **poussé par un opérateur** (jamais téléchargé automatiquement) ;
- elle complète les fiches par l'API, **en tâche de fond seulement** : 500 appels par jour au plus,
  20 jeux par appel, une requête toutes les 5 s, chaque nuit à 03:30 UTC ;
- elle ne répond qu'aux autres services (`/internal/*`) et aux opérateurs (`/admin/*`) ;
- elle se purge en une commande, pour le cas où BGG révoquerait la licence ;
- sans jeton BGG, elle ne moissonne rien.

Brancher `search` sur `bgg-data` est explicitement **le lot suivant** de cette spec.

Deux liens avec ce qui est construit :

- La spec signale un piège : le client HTTP commun aplatit les `429` en `502`, si bien que le worker
  ne peut pas mettre sa file en pause. La modification faite dans `search` corrige précisément ce
  piège, et `bgg-data` en héritera en copiant `src/common`.
- La spec et le brief de la résolution **se contredisent sur BGG** : voir la question 7.1.

Pour l'historique : le plan initial de l'app (`acervatim-app/docs/contexte-global.md`) prévoyait une
recherche texte BGG pour les jeux de société. Le passage par le code-barres et GameUPC est venu
ensuite. La recherche texte BGG n'est pas construite.

---

## 6. Les décisions déjà prises

| Décision | Pourquoi | Ce qui la remettrait en cause |
|---|---|---|
| GameUPC pour passer du code-barres au jeu | Aucune source francophone n'a d'API qui donne l'id BGG (§ 3.2) | Un partenariat avec MyLudo ou un e-commerçant |
| La résolution vit dans `search` | C'est une recherche externe ; `search` a déjà clés, files et cache (§ 4.1) | — |
| Pas de service dédié aux votes | Votes et consultation partagent cache, clé et quota (§ 4.1) | Garder la liste des votes par utilisateur ; contribuer à plusieurs sources |
| Licence BGG non commerciale d'abord, commerciale avant le premium | Acervatim est gratuit, édité par une personne physique, sans publicité (§ 3.1, décision du 2026-11-13) | Le lancement du premium ou de la publicité, la création d'une société |
| BGG désactivé tant qu'aucune licence n'est accordée | Aucun appel sans application approuvée (§ 3.1) | La licence |
| Normaliser le code avant tout appel | GameUPC ne valide rien et indexe des chaînes (§ 3.3) | — |
| Type `boardgame` distinct de `game` | `game` désigne déjà le jeu vidéo | À confirmer en validant la spec (7.7) |

---

## 7. Ce qui reste à trancher

### 7.1 Qui appelle BGG, quand, et à quel débit ?

**Pourquoi la question se pose.** Le brief de la résolution demande d'appeler BGG pour la fiche d'un
jeu, avec un cache long et une requête toutes les 2 s au plus. La spec `bgg-data`, elle, interdit
d'appeler BGG pendant une requête et fixe une requête toutes les 5 s. Si les deux coexistent, deux
services appellent BGG avec le **même jeton** et deux limiteurs indépendants : le débit total
dépasse alors les deux plafonds, et c'est notre licence qui est en jeu.

**Options :**

- (a) Garder les deux appelants. Simple, mais le débit total n'est plus maîtrisé.
- (b) `search` seul appelle BGG, à la demande. On perd la base locale et sa purge centralisée.
- (c) **`bgg-data` seul appelle BGG.** `search` lit la fiche chez `bgg-data`. Pour un jeu que
  `bgg-data` n'a pas encore, `search` lui demande de le récupérer en priorité, dans sa propre file.

**Recommandation : (c), à 5 s.** Un seul jeton, un seul limiteur, une seule purge. L'app affiche
tout de suite ce que donne GameUPC (nom, année, vignette) et reçoit la fiche complète un peu plus
tard. Le client BGG de `search` est déjà derrière une interface : il suffira d'en changer
l'implémentation. Il faut alors ajouter à la spec une route de récupération prioritaire.

### 7.2 Faut-il faire choisir l'édition ?

**Pourquoi.** C'est l'édition qui distingue un EAN français : *Kingdomino* a plus de 60 versions,
dont 3 françaises, toutes à confiance nulle. Sans vote d'édition, GameUPC ne saura jamais quelle
version correspond à la boîte française.

**Recommandation : oui, en option**, avec les versions dans la langue de l'utilisateur et les
versions multilingues en tête. À demander à GameUPC : un vote d'édition compte-t-il aussi pour le
jeu ?

### 7.3 Faut-il retirer les votes d'un utilisateur quand il supprime son compte ?

**Pourquoi.** Les votes partent chez GameUPC sous pseudonyme. Pour les retirer, il faudrait garder
la liste des votes de chaque utilisateur, c'est-à-dire une donnée d'utilisateur, que `search`
s'interdit de stocker. Et `catalog-boardgame` ne peut pas l'héberger : sa spec lui interdit tout
code métier hors de son profil.

**Recommandation : ne pas garder cette liste.** Le pseudonyme est calculé à partir de l'identifiant
et d'un secret, et GameUPC ne voit jamais cet identifiant. Une fois le compte supprimé, plus
personne ne peut relier un vote à la personne, et un vote dit seulement « tel code est tel jeu ».
En complément, à la suppression d'un compte, retirer les votes **encore en file** pour ce compte
(un vote en échec garde le pseudonyme jusqu'à sept jours dans Redis). Ce point est à valider côté
juridique et à écrire dans la politique de confidentialité.

### 7.4 Les routes sont-elles réservées au premium ?

Aujourd'hui, elles sont ouvertes à tous les utilisateurs connectés, comme le reste de la recherche.
À confirmer.

### 7.5 Que faire quand GameUPC ne sait pas ?

Pour un code inconnu ou une panne, l'app propose aujourd'hui la saisie manuelle. D'autres replis
sont possibles : une recherche texte BGG une fois la licence obtenue, ou GS1 et Open Products Facts
comme simples indices de nom. Aucun n'est nécessaire au lancement.

### 7.6 Et si BGG refuse la licence ?

**Pourquoi.** C'est une issue réaliste (§ 3.1), à l'enregistrement comme au passage en
commercial. Ce qui resterait :

- GameUPC, qui donne déjà nom, année et vignettes. Ces données viennent de BGG : il faut savoir si
  les afficher relève de la licence BGG (question à poser, § 9) ;
- la saisie manuelle ;
- des pistes **non étudiées** : Wikidata, qui référence l'identifiant BGG de nombreux jeux, ou un
  partenariat francophone.

**Recommandation** : ne pas construire de plan B avant la réponse de BGG, mais ne rien promettre
publiquement qui dépende de BGG d'ici là (§ 8, documentation).

### 7.7 Valider la spec `catalog-boardgame` / `bgg-data`

Elle est en brouillon. Sa validation fixe le type `boardgame`, et c'est le bon moment pour y
intégrer la réponse à 7.1 (route de récupération prioritaire, débit unique).

La spec interdit aussi de télécharger le dump automatiquement : un opérateur le pousse. Or la page
officielle relue le 2026-11-13 prévoit le téléchargement avec le jeton d'application. Une fois la
licence obtenue, un téléchargement planifié devient envisageable, si BGG en accepte la fréquence
(question au § 9).

### 7.8 Compléments francophones

Partenariat MyLudo, flux affilié Philibert, indices GS1 ou Open Products Facts : ce sont des
décisions produit, sans urgence.

---

## 8. Ce qui reste à faire

### Démarches externes

- [ ] **GameUPC** : demander la clé de production par courriel (questions au § 9).
- [x] **BGG** : relire la page « Using the XML API » (fait le 2026-11-13, § 3.1).
- [ ] **BGG** : relire les conditions d'utilisation de l'API (« XML API terms of use »).
- [ ] **BGG** : enregistrer l'application en « Non commercial », en annonçant le palier premium
  prévu (questions au § 9), puis suivre le forum « Geek Tools News », où BGG annonce ses
  changements.
- [ ] **BGG**, avant le lancement du premium : obtenir la licence commerciale.

### Backend

- [ ] Reporter dans `rebrickable-data/src/common` les deux modifications de `search/src/common`.
- [ ] Déplacer le travail sur une branche dédiée, puis commiter.
- [ ] À la suppression d'un compte, retirer les votes en file (si 7.3 est suivie).
- [ ] Avec la clé de production : refaire les mesures du § 3.2 et recaler les débits GameUPC.
- [ ] Avec la licence BGG : remplacer la fixture BGG construite par une vraie réponse.
- [ ] Après validation de la spec : construire `catalog-boardgame` et `bgg-data`.
- [ ] Puis brancher `search` sur `bgg-data` et retirer son appel direct à BGG (si 7.1 (c) est
  retenue).

### App

- [ ] Envoyer le format du code scanné (indispensable pour un UPC-E).
- [ ] Afficher le logo « Powered by BGG », lié à la fiche BGG du jeu (`pageUrl`).
- [ ] Écran de choix : les candidats **et** un champ de recherche pré-rempli par `searchedFor` ;
  les éditions de la langue de l'utilisateur en tête.
- [ ] Gérer `unavailable` (réessayer, saisir à la main) et signaler une réponse `stale`.

### Documentation et juridique

- [ ] Politique de confidentialité (`legal/confidentialite.md`) : l'envoi des choix à GameUPC sous
  pseudonyme, et l'usage de BGG.
- [ ] Vérifier si cet envoi à GameUPC se déclare dans `legal/data-safety.md`.
- [ ] La page publique `utilisateur/fonctionnalites.md` annonce déjà BoardGameGeek comme source :
  la formulation doit rester au conditionnel tant que la licence n'est pas obtenue.

---

## 9. Ce qu'il faut demander

### À GameUPC

1. Une clé de production pour une app Android gratuite, avec un palier premium payant prévu plus
   tard : conditions aujourd'hui et au lancement du premium, attribution attendue, une clé par
   environnement ?
2. La limite de débit réelle, et la conduite attendue sur un `429`.
3. Le seuil de vérification en production : la démo parle de « deux votes non contestés », un seul
   suffit sur le serveur de test.
4. Un vote d'édition compte-t-il aussi pour le jeu ?
5. Peut-on retirer en bloc les votes d'un `user_id`, au cas où la question 7.3 serait tranchée
   autrement ?

### À BGG

1. Quelles limites d'usage pour une licence non commerciale ? Le passage en commercial, avant le
   premium, se fait-il sur la même application ou par une nouvelle demande ?
2. La copie locale complète (dump et fiches moissonnées) est-elle autorisée, et pour quelle durée ?
   Peut-on télécharger le dump avec le jeton, et à quelle fréquence ?
3. Les durées de cache prévues (30 jours dans `search`, 90 jours dans `bgg-data`) sont-elles
   acceptables ?
4. Peut-on afficher les images en lien direct depuis `cf.geekdo-images.com` ?
5. Le logo « Powered by BGG » sur l'écran de résultat, lié à la fiche BGG du jeu, suffit-il ? (Sa
   forme est réglée : un fichier officiel, à une taille où le texte reste lisible.)
6. Quel débit est acceptable pour notre licence, et les lots de 20 jeux par requête ?
7. Couper une description trop longue, ou laisser l'utilisateur corriger sa propre fiche, est-ce une
   « modification des données » ?
8. Les données BGG reçues par l'intermédiaire de GameUPC (noms, vignettes) sont-elles couvertes par
   notre licence ?

Plus tard, au passage en commercial : le seuil de la licence gratuite (100 utilisateurs payants)
compte-t-il tous les abonnés premium d'Acervatim, ou seulement ceux qui utilisent les jeux de
société ? Quel tarif au-delà, et à quel rythme ?

---

## 10. Où trouver quoi

| Sujet | Où |
|---|---|
| Détail technique de ce qui est construit | [agent/backend/gameupc-bgg.md](agent/backend/gameupc-bgg.md) |
| Spec du catalogue et de la base BGG | [spec-catalog-boardgame-bgg-data.md](_bmad-output/implementation-artifacts/spec-catalog-boardgame-bgg-data.md) |
| Fiche des sources externes | [agent/transverse/sources-externes.md](agent/transverse/sources-externes.md) |
| Modèle suivi pour LEGO | [spec-rebrickable-data-service.md](_bmad-output/implementation-artifacts/spec-rebrickable-data-service.md) |
| Code | `acervatim/backend-services/search/src/{common/barcode,sources/gameupc,sources/bgg,search/boardgame}/` |
| Fixtures | `acervatim/backend-services/search/test/fixtures/{gameupc,bgg}/` |
| Résumé côté service | `acervatim/backend-services/search/README.md`, § 7 |

Sources consultées :

- Page BGG « Using the XML API », version du 2025-07-02, relue le 2026-11-13 à partir du texte
  transmis par l'utilisateur ([boardgamegeek.com/using_the_xml_api](https://boardgamegeek.com/using_the_xml_api))
- [Swagger GameUPC](https://gameupc.com/gameupc-oas.yaml) et [gameupc.com](https://gameupc.com/)
- [Annonce de GameUPC sur BGG](https://boardgamegeek.com/thread/2579359/new-rest-service-for-upc-bgg-lookups)
- [Notes d'intégration BGG d'un autre projet (shelf #167)](https://github.com/dgahagan/shelf/issues/167)
- Fiches e-commerçants : [Philibert, Kingdomino](https://www.philibertnet.com/en/blue-orange/44925-kingdomino-3664824001215.html),
  [Philibert, 6 qui prend](https://www.philibertnet.com/en/gigamic/6064-6-qui-prend-boite-metal--3421272101337.html),
  [Play-in, Dixit 2022](https://www.play-in.com/fr/produit/394813/dixit-edition-2022)
- [Verified by GS1](https://support.gs1.org/support/solutions/articles/43000734077-what-is-verified-by-gs1-)
- [Open Products Facts](https://www.data.gouv.fr/datasets/open-products-facts/community-resources)
