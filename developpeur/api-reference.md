---
title: Référence API
layout: default
parent: Développeur
nav_order: 3
---

# Référence API `/v1`

Vue normalisée du contrat HTTP. **Source de vérité** : le code du dépôt `acervatim-backend` et son
`openapi.json` généré au bootstrap.
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
`/probs/forbidden`, `/probs/too-many-requests`, …). Aucun internal (stack, SQL) en production.

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
| GET | `/v1/me/quota` | Consommation des quotas du palier gratuit. |
| GET | `/v1/me/export` | Export complet des données (portabilité RGPD). |
| DELETE | `/v1/me` | Suppression du compte + données (cascade). |

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
| GET | `/v1/search?type=…&q=…` **ou** `?type=…&barcode=…` | Proxy unifié (`q`/`barcode` mutuellement exclusifs) → `UnifiedItem`. Voir [Sources & pivot ISBN](sources-et-pivot-isbn.md). |

### Comptes externes (OAuth)
| Méthode | Route | Rôle |
|---|---|---|
| GET | `/v1/oauth/:provider/start` | Démarre le flux de connexion (Discogs, MAL…). |
| GET | `/v1/oauth/:provider/callback` | Callback OAuth. |
| GET | `/v1/oauth` | Comptes connectés. |
| DELETE | `/v1/oauth/:provider` | Révocation. |

### Premium & invitations
| Méthode | Route | Rôle |
|---|---|---|
| POST | `/v1/subscriptions/verify` | Vérifie un achat Google Play Billing. |
| POST | `/v1/webhooks/google-play/rtdn` | Webhook RTDN (public, `@SkipThrottle`). |
| POST | `/v1/invitations/redeem` | Échange un code d'invitation premium. |

### Infra
| Méthode | Route | Rôle |
|---|---|---|
| GET | `/health` | Liveness (hors `/v1`). Un `/health/ready` avec ping DB est prévu. |

## Voir aussi

- [Le serveur Acervatim](backend.md) · [Architecture](architecture.md).
- Contrat brut : `openapi.json` (dépôt `acervatim-backend`).
