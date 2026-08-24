# acervatim-backend — Contrats API V1

> Remonter : [INDEX.md](INDEX.md) · Résumé opérationnel : [../conventions.md](../conventions.md) § Contrats API V1
> **À lire avant d'écrire un controller ou un DTO.** Ces règles s'appliquent à **tout endpoint
> applicatif** ; les webhooks publics font exception explicite.

## Versioning

- Préfixe global : tous les endpoints sous `/v1/...` (posé via `app.setGlobalPrefix('v1')` dans `main.ts`).
- `/health/*` reste **hors versioning** (infra, pas API métier).
- Une bump vers `/v2/` ne se fait que pour un changement de contrat incompatible. La V1 reste maintenue en parallèle pendant la transition.

## Validation : Zod via nestjs-zod

`nestjs-zod` fournit `ZodValidationPipe` (global dans `main.ts`) et `createZodDto`. Un schéma Zod par DTO, exporté à côté de la classe :

```typescript
const CreateCollectionSchema = z.object({
  typeCode: z.string().max(32),
  name: z.string().max(255),
  description: z.string().optional(),
});
export class CreateCollectionDto extends createZodDto(CreateCollectionSchema) {}
```

`class-validator` a été entièrement retiré de `src/` — ne pas le réintroduire.

**Un `unifiedData` venu du client se valide par `safeParse` + `ZodValidationException`**, jamais par
un `.parse()` nu. Le pipe global ne couvre que la forme du corps ; la re-validation par le profil de
type (ou le `nodeSchema` d'un niveau) a lieu dans le service, où une `ZodError` non enveloppée n'est
pas reconnue par le filtre RFC 9457 et sortirait en `500` alors que la faute est au corps envoyé.
Réf. `ItemsService.validateUnified` et `NodesService.validateNodeUnified`.

**Le garde-fou de taille est partagé** : `common/validation/bounded-json.ts`
(`boundedJsonRecord`, `UNIFIED_DATA_MAX_BYTES`). Tout DTO portant un `unifiedData` le réutilise —
c'est un plafond de volume, pas une règle métier, et il ne doit pas diverger d'un DTO à l'autre.

## Erreurs : RFC 9457 (Problem Details)

Content-Type `application/problem+json`. Payload standard :

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

- `requestId` toujours présent (issu du `CorrelationIdInterceptor`).
- `type` : URI stable par classe d'erreur (`/probs/validation-error`, `/probs/not-found`, `/probs/forbidden`, …). Implémenté par `src/common/filters/problem-details.filter.ts`.
- Pas de leak d'internals (stack trace, requête SQL…) en prod.

## Pagination : cursor only

- Query : `?cursor=<id>&limit=<n>` (limit max 100, défaut 50).
- Réponse standard pour toute liste paginée :

```json
{
  "data": [ {}, {} ],
  "meta": { "pagination": { "nextCursor": "abc123", "limit": 50 } }
}
```

- `nextCursor: null` quand il n'y a plus de page suivante.
- Les ressources unitaires (`GET /v1/collections/:id`) retournent l'objet **brut**, sans wrapper `{ data }` — le wrapper ne concerne que les listes.

## Filtres : `field[in]` (OR) et `field[all]` (AND)

- `?status[in]=active,pending` → `WHERE status IN ('active', 'pending')` (OR sur un même champ).
- `?tags[all]=jazz,blues` → la ressource doit posséder **tous** les éléments listés (AND ; utile pour les relations many-to-many ou les colonnes JSON tableau).
- Plusieurs filtres se combinent en AND : `?type[in]=vinyl,manga&tags[all]=jazz,rock` = `(type IN (...)) AND (tags ALL (...))`.
- Parsing centralisé : `parseFilters(query, schema)` → objet Prisma `where` (`src/common/filters/parse-filters.ts`).
- Les champs filtrables sont **whitelistés par endpoint** (schéma Zod) : pas de filtre arbitraire client → SQL.

### Deux familles d'opérateurs — piège

| Helper | Valeurs | Sémantique de `[in]` / `[all]` |
|---|---|---|
| `filterIn` / `filterAll` | whitelistées (enum) | **égalité exacte** (`WHERE IN`) — cas par défaut |
| `searchFilter` | texte libre | **substring insensible à la casse** ; `[in]`/`[all]` combinent des _termes de recherche_ en OR/AND, **pas** une appartenance exacte |

Seul usage actuel de `searchFilter` : `GET /v1/collections?items[in]=...&items[all]=...` (recherche sur le contenu des items).

### Règle dure — recherche insensible à la casse sur JSON

Ne **jamais** utiliser le `string_contains` Prisma sur une colonne JSON pour un contains CI. Sur MariaDB, `JSON_UNQUOTE(...)` ressort en collation `utf8mb4_bin` (sensible à la casse) et MySQL n'a pas de `mode: 'insensitive'`. Passer par du SQL paramétré avec `COLLATE utf8mb4_general_ci` — référence : `CollectionsService.matchingCollectionIds` (`src/collections/collections.service.ts`).

## Documentation OpenAPI

- `openapi.json` généré automatiquement à chaque bootstrap (`main.ts`) via `@nestjs/swagger` + `patchNestJsSwagger()` (nestjs-zod). Artefact ignoré par git — ne jamais l'éditer à la main.
- UI Swagger sur `GET /docs`, exposée uniquement quand `NODE_ENV !== 'production'`.
- Tout nouveau controller doit générer un OpenAPI valide : `@ApiBearerAuth()` sur les routes JWT-protected, `@ApiTags()` par module si besoin de regrouper.
