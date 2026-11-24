---
title: Référence API
layout: default
parent: Développeur
nav_order: 3
---

# Référence API `/v1`

Vue normalisée du contrat HTTP, identique pour la constellation (`acervatim-services`, en production
depuis le 23 novembre 2026) et pour le monolithe (`acervatim-backend`, gardé pour un retour). **Source de vérité** : le code et l'`openapi.json`
généré au démarrage ; dans la constellation, le gateway agrège celui de chaque service.
En dev (`NODE_ENV !== 'production'`), un Swagger UI est exposé sur `GET /docs`.

## Conventions transverses

- **Versioning** : tous les endpoints applicatifs sous `/v1/...`. `/health/*` reste hors versioning
  (infra). Une bump `/v2` n'arrive que pour un changement de contrat incompatible.
- **Auth** : `Authorization: Bearer <JWT>` (RS256, émis par `POST /v1/auth/google`). Voir
  [Authentification Google](auth-google.md).
- **Validation** : schémas **Zod** (`nestjs-zod`), whitelist stricte — tout champ inattendu est rejeté.
- **Erreurs** : **RFC 9457 / Problem Details**, `Content-Type: application/problem+json`, avec
  `requestId` (corrélation) toujours présent.
- **Corrélation** : chaque réponse porte un `requestId` (en-tête / payload d'erreur).

### Forme d'une erreur

```json
{
  "type": "https://api.acervatim/probs/validation-error",
  "title": "Invalid request payload",
  "status": 400,
  "detail": "name: String must contain at most 255 characters",
  "instance": "/v1/collections",
  "requestId": "f1c0…",
  "errors": [{ "field": "name", "code": "too_long", "message": "…" }]
}
```

`type` est une URI stable par classe d'erreur (`/probs/validation-error`, `/probs/not-found`,
`/probs/forbidden`, `/probs/too-many-requests`, `/probs/source-token-required`, …). Aucun internal
(stack, SQL) en production.

Certaines erreurs portent des **membres d'extension** (RFC 9457). En particulier
`/probs/source-token-required` (**403**) ajoute `provider` — la source à connecter — pour que l'app
propose le bon parcours : **connecter la source** (`/v1/oauth/:provider/start`) **ou passer premium**.

## Pagination — cursor only

- Query : `?cursor=<id>&limit=<n>` (`limit` max 100, défaut 50).
- Réponse de liste :

```json
{
  "data": [ { }, { } ],
  "meta": { "pagination": { "nextCursor": "abc123", "limit": 50 } }
}
```

- `nextCursor: null` = plus de page. Les **ressources unitaires** (`GET …/:id`) renvoient l'objet
  **brut** (pas de wrapper `{ data }`).

## Filtres — `field[in]` (OR) et `field[all]` (AND)

- `?status[in]=active,pending` → `status IN (...)` (OR sur un même champ).
- `?tags[all]=jazz,blues` → la ressource possède **tous** les éléments (AND).
- Plusieurs filtres se combinent en AND. Champs filtrables **whitelistés par endpoint** (Zod).
- Deux familles : égalité exacte (enum whitelisté) vs recherche texte libre (substring insensible à la
  casse, ex. `GET /v1/collections?items[in]=…`).

## Catalogue des endpoints

> Récapitulatif ; les payloads exacts sont dans `openapi.json`.

### Authentification & compte
| Méthode | Route | Rôle |
|---|---|---|
| POST | `/v1/auth/google` | Échange un `id_token` Google contre un JWT interne (crée le compte au 1er login). |
| GET | `/v1/me` | Profil minimal + statut premium. |
| GET | `/v1/me/export` | Export complet des données (portabilité RGPD), partages émis et rejoints compris. |
| DELETE | `/v1/me` | Suppression du compte + données (cascade). |

> **Synchronisation : premium uniquement.** Le stockage serveur (collections / items / nœuds) a un
> coût réel, il est donc réservé au palier payant : toutes ces routes répondent `402`
> (`/probs/payment-required`) à un compte non premium, en écriture **comme en lecture**. Il n'y a
> plus de palier gratuit borné par des quotas — la notion de quota d'offre a disparu.
>
> En lecture, le statut vérifié est celui du **propriétaire** de la collection, pas du demandeur :
> c'est ce qui permet à un membre non premium de consulter une collection qu'on lui partage.
>
> Il subsiste des plafonds purement **techniques** (anti-emballement), identiques pour tous, très
> au-dessus de tout usage réel, que rien n'expose et qu'aucun abonnement ne lève ; les atteindre
> renvoie `409` (`/probs/technical-limit`).

### Collections
| Méthode | Route | Rôle |
|---|---|---|
| POST | `/v1/collections` | Créer une collection (typée). |
| GET | `/v1/collections` | Lister (pagination cursor, filtres). |
| GET | `/v1/collections/:id` | Détail. |
| PATCH | `/v1/collections/:id` | Mise à jour. |
| DELETE | `/v1/collections/:id` | Suppression. |

### Items
| Méthode | Route | Rôle |
|---|---|---|
| POST | `/v1/collections/:collectionId/items` | Créer un item. |
| GET | `/v1/collections/:collectionId/items` | Lister les items d'une collection. |
| GET | `/v1/items/:id` | Détail item. |
| PATCH | `/v1/items/:id` | Mise à jour. |
| DELETE | `/v1/items/:id` | Suppression. |
| GET | `/v1/items/:id/sources` | Snapshots de sources externes liés. |
| POST | `/v1/items/:id/sources` | Attacher un snapshot de source. |

### Nœuds (séries / sets)
| Méthode | Route | Rôle |
|---|---|---|
| POST | `/v1/collections/:collectionId/nodes` | Créer un nœud (série, coffret). |
| GET | `/v1/collections/:collectionId/nodes` | Lister les nœuds. |
| GET | `/v1/nodes/:id` | Détail + drill-down. |
| PATCH | `/v1/nodes/:id` | Mise à jour. |
| GET | `/v1/nodes/:id/sources` | Snapshots de sources. |
| POST | `/v1/nodes/:id/sources` | Attacher un snapshot. |

### Recherche externe unifiée
| Méthode | Route | Rôle |
|---|---|---|
| GET | `/v1/search?type=…&q=…` **ou** `?type=…&barcode=…` | Proxy unifié (`q`/`barcode` mutuellement exclusifs) → `UnifiedItem`. Le manga par texte est servi par MangaDex, sans jeton. Voir [Sources & pivot ISBN](sources-et-pivot-isbn.md). |
| GET | `/v1/search/editions?mangaId=…[&title=…]` | Éditions françaises (BnF) d'une série MangaDex : une ligne par (éditeur, mention d'édition). Voir [Recherche d'une série par titre](sources-et-pivot-isbn.md#recherche-serie-titre). |
| GET | `/v1/search/edition-mapping?title=…[&edition=…][&publisher=…][&since=AAAAMMJJ]` | Tomes d'une édition (BnF) ; `since` = notices créées depuis une date. Voir [Énumération des tomes](sources-et-pivot-isbn.md#enumeration-des-tomes). |
| POST | `/v1/search/import-jobs` | Import asynchrone d'une édition complète (`title`, `edition`, `publisher`, `mangaId`) ; `GET /v1/search/import-jobs/:id` pour l'avancement. |
| GET | `/v1/series-updates` | **Premium.** Nouveaux tomes des séries synchronisées détectés par la vérification hebdomadaire. Voir [Nouveaux tomes](sources-et-pivot-isbn.md#nouveaux-tomes). |
| GET | `/v1/search/cover?isbn=…[&title=…&volume=…&edition=…]` | Jaquette d'un tome (MangaDex, puis Google Books). |
| GET | `/v1/search?type=lego&q=…` **ou** `?type=lego&barcode=…` | **Constellation seulement.** Sets LEGO, lus dans `rebrickable-data` (aucun appel externe pour le texte). Au scan, `meta.barcodeLookup` dit d'où vient la réponse (`cache`, `brickset`, `miss`, `pending`, `disabled`, `exhausted`, `unavailable`). Voir [Sources & pivot ISBN](sources-et-pivot-isbn.md). |
| GET | `/v1/search/boardgame/barcode/:code[?q=…][&format=…]` | **Constellation seulement.** Code-barres d'un jeu de société → candidats (GameUPC) et fiche BoardGameGeek. Voir [Jeux de société](sources-et-pivot-isbn.md#jeux-de-societe). |
| POST | `/v1/search/boardgame/barcode/:code/choice` | **Constellation seulement.** Jeu choisi par l'utilisateur, renvoyé à GameUPC sous pseudonyme ; 30 par minute au plus. |

> **En production depuis la bascule du 23 novembre 2026** : `editions`, le paramètre `publisher` et
> `since` d'`edition-mapping`, et `series-updates`, fusionnés dans `main` le 14 novembre 2026. Le
> monolithe déployé, construit fin août 2026, ne les a pas.

> **Résolution des jetons** : chaque appel sortant utilise le jeton de l'utilisateur (**BYOT**) s'il a
> connecté la source ; sinon, pour un **premium**, un repli sur les jetons Acervatim ; sinon un mode
> **dégradé** (cache partagé + sources sans clé, ex. BnF) puis l'erreur actionnable
> `source-token-required` (403). Détail : [Sources & pivot ISBN](sources-et-pivot-isbn.md#résolution-des-jetons-byot--repli-premium--mode-dégradé).

### Comptes externes (OAuth)
| Méthode | Route | Rôle |
|---|---|---|
| GET | `/v1/oauth/:provider/start` | Démarre le flux de connexion (Discogs, MAL). |
| GET | `/v1/oauth/:provider/callback` | Callback OAuth. |
| GET | `/v1/oauth` | Comptes connectés. |
| DELETE | `/v1/oauth/:provider` | Révocation. |

### Sources externes — clé personnelle (BYOT hors OAuth)
| Méthode | Route | Rôle |
|---|---|---|
| PUT | `/v1/sources/tmdb/token` | Enregistre/remplace la **clé API TMDB personnelle** (TMDB n'a pas d'OAuth). Stockée chiffrée (`oauth_credentials`, provider `tmdb`). |
| DELETE | `/v1/sources/tmdb/token` | Révoque la clé TMDB personnelle. |

### Partage de collection
Toutes les routes sont ancrées sur `/v1/shares` : un partage porte **plusieurs collections**, chacune
restreinte aux statuts choisis (`OWNED`, `WISHLIST`, `IGNORED`), en lecture seule pour les membres.

| Méthode | Route | Rôle |
|---|---|---|
| POST | `/v1/shares` | Émettre un code de partage (**premium**) : `{ label?, collections: [{ collectionId, statuses[] }], maxUses?, expiresAt? }`. Le code en clair n'est renvoyé qu'ici : seule son empreinte est stockée. |
| GET | `/v1/shares[?collectionId=…]` | Partages émis, avec leurs collections, statuts et membres. |
| PATCH | `/v1/shares/:id` | Changer le libellé du partage, et rien d'autre. |
| DELETE | `/v1/shares/:id` | Révoquer un partage : le code cesse de fonctionner et tous ses membres perdent l'accès. |
| DELETE | `/v1/shares/:id/members/:memberUserId` | Retirer un seul membre ; le code reste valide pour les autres. |
| POST | `/v1/shares/redeem` | Rejoindre un partage avec un code (`{ code, label? }`). **Ne demande pas le premium** : c'est le propriétaire qui paie le stockage. |
| GET | `/v1/shares/received` | Partages rejoints. |
| PATCH | `/v1/shares/received/:shareId` | Changer son propre libellé du partage reçu. |

> **Lecture** : un membre lit les collections partagées par les routes habituelles
> (`/v1/collections/:id/items`…), filtrées côté serveur sur les statuts partagés ; les données
> personnelles du propriétaire (`userData`) sont masquées hors liste blanche. Le premium compte
> côté **propriétaire**.
>
> **Pourquoi un code et pas une invitation par email** : les comptes sont anonymes, il n'y a ni
> email ni nom en base. Le code voyage donc hors-bande (message, papier, à voix haute). Chaque
> libellé reste privé de son côté.
>
> Un code inconnu, révoqué, expiré ou épuisé renvoie la **même** erreur `404`
> (`/probs/share-code-invalid`) : la réponse ne dit jamais si un code existe.

### Premium & invitations
| Méthode | Route | Rôle |
|---|---|---|
| POST | `/v1/subscriptions/verify` | Vérifie un achat Google Play Billing. |
| POST | `/v1/webhooks/google-play/rtdn` | Webhook RTDN (public, `@SkipThrottle`). |
| POST | `/v1/invitations/redeem` | Échange un code d'invitation premium. |

### Infra
| Méthode | Route | Rôle |
|---|---|---|
| GET | `/health` | Vivacité (hors `/v1`). |
| GET | `/health/ready` | Disponibilité, base comprise : **constellation**, en cours de correction ; absent du monolithe. |

## Voir aussi

- [Le serveur Acervatim](backend.md) · [Architecture](architecture.md).
- Contrat brut : `openapi.json` (dépôts `acervatim-backend` et `acervatim-services`).
