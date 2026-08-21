# Chantier — Scan asynchrone & passerelle sources (BullMQ + Redis)

> Remonter : [INDEX.md](INDEX.md) · Document de **suivi de chantier** (pas une page du site public).
> But : charger ce fichier dans une session Claude pour reprendre le travail avec tout le contexte.
> Dernière mise à jour : 2026-07-20. Branche : `feat/mangadex-pivot-principal` (poussée ; la branche `feat/p0-bullmq-gbooks` a été **repliée dedans puis supprimée** — tout P0→P2 + P3 partiel y est). P0/P1/BnF/TMDB/MAL/**Discogs** + **MangaDex** commités et poussés. **P2 terminé**, token bucket SQL retiré. Ajout du pivot **MangaDex source principale** (`c91f4cb`) — cf. [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md).

## TL;DR

On transforme les appels aux sources externes (Google Books, BnF, MAL, Discogs, TMDB) en une
**passerelle d'appels sortants mutualisée** adossée à **BullMQ + Redis** : throttle **global** (tous
users confondus), **single-flight** (dédup des demandes concurrentes), **cache inter-user**. Objectif :
supprimer les rafales de `429/503` et le timeout client qui faisaient échouer les scans (incident
`0.5.3`). **P0 (Google Books), P1 (import série async) et P2 (BnF + TMDB + MAL + Discogs) sont
implémentés, validés et poussés** (Discogs = `0508a9d`). Le **token bucket SQL** (`TokenBucketService`
+ modèle `RateLimitBucket`) a été **entièrement retiré** — toutes les sources sont derrière un limiter
BullMQ. **P3 démarré** : le **circuit-breaker Redis** est en place (`a712cf7`) — Redis down → `503`
immédiat sur le chemin interactif au lieu d'un hang de ~15 s ; et les **métriques de files** (`70eacaf`).
**Reste P3** : Bull Board, calibrage effectif des limiters (à partir de ces métriques), WorkManager
app ; et la validation app bout-en-bout.

## 1. Pourquoi — l'incident 0.5.3

Analyse du log `acervatim-backend/logs/prod_0.5.3.log` :

- `GET /v1/search/edition-mapping` résout **une jaquette Google Books par tome** (jusqu'à ~30) au
  premier scan « série complète » d'une édition à froid.
- Google Books renvoyait des **`503` en rafale** ; `HttpClientService` retente 3× avec backoff
  (`[250, 750, 2250]` ms) → `edition-mapping` dépassait le **read-timeout de 10 s** d'OkHttp côté app
  (aucun override dans `NetworkModule.kt`) → l'app coupait → **scan en erreur**.
- Le backend ne plantait pas (Google Books est best-effort `null`) : c'était un **timeout client**.
  Au rescan, cache chaud → succès. Les échecs JJK/SNK du log sont exactement ce schéma.
- Cause aggravante interne : le bucket `gbooks:global` (`TokenBucketService`, une ligne SQL à CAS
  optimiste) **livelockait** sous rafale (cf. commentaire dans `src/search/search.service.ts`).

## 2. Décisions actées

| Sujet | Décision |
|---|---|
| Moteur | **BullMQ + Redis** (le Pi n'est pas saturé ; Redis assumé comme nouveau composant) |
| Périmètre | **Toutes les sources** passent par la file, **phase 1 (scan) incluse** |
| Patrons | **enqueue-and-await** (phase 1 → reste synchrone pour l'app) ; **fire-and-poll** (import série → `202` + UUID, l'app **re-tape `edition-mapping`** une fois `done`) |
| Éligibilité | Tout le monde connecté ; on garde l'**identité du demandeur** pour résoudre **son** token (BYOT) ; sans token → règles actuelles (repli premium → dégradé → `403`). Token résolu **dans le worker**, **jamais stocké dans Redis** |
| Dédup inter-user + cache | **Cache indépendant des clés** : si l'info est en cache pour un user, on la sert à un autre **sans rappel**. On ne fait **jamais** d'appel avec le token d'autrui — on mutualise seulement les **réponses** (endpoints publics). Seuls les demandeurs **token-capables** enfilent un job ; token-less = cache-only sinon `403` |
| Cache | **Seules les réponses 2xx** sont mises en cache (un `200`-vide = absence légitime, cacheable ; `4xx/429/5xx` jamais → re-tentable) |
| Files | **Une file par source** (`gbooks`, `bnf`, `mal`, `discogs`, `tmdb`, `mangadex`), **sans split byot/shared** pour l'instant (le limiter protège le credential partagé ; les BYOT héritent de la même borne — rouvrable). La file `mangadex` (`concurrency:1`, `limiter 4/s`) porte **deux jobs** discriminés par `job.name` : `series-covers` (jaquettes) et `identify` (pivot d'identité) |
| Throttle DB | `TokenBucketService` + modèle `RateLimitBucket` **supprimés** (remplacés par les limiters BullMQ) — migration de drop `20260708120000_drop_rate_limit_buckets` |
| Interactif vs série | **Séparés** par **priorité BullMQ** (interactif `priority:1`, série `priority:10`) + `queue:bnf` en `concurrency:2` (seul point de contention : lookup ISBN vs énumération ; la priorité ne préempte pas un job en cours) |
| Redis down | **`503` propre immédiat** sur le chemin interactif (circuit-breaker `RedisHealthService`, `a712cf7` — pas d'attente des ~15 s de `waitUntilFinished`, pas de repli appel-direct). Sur le chemin best-effort (jaquettes), on dégrade en `null` tout de suite |

## 3. Architecture cible

La file est un **proxy sortant** : chaque handler consulte d'abord le cache (`ApiCache`), sinon
**enfile** un job (dédup par `jobId` = clé publique normalisée `<source>:<requête>`) et attend/poll le
résultat ; le **worker** résout le token du demandeur, appelle la source, met en cache (2xx), renvoie.

- **Single-flight** : `jobId` public → deux demandes identiques concurrentes = **un** appel sortant.
- **Throttle global** : concurrence worker basse + `limiter` BullMQ par file → débit sortant plafonné
  tous users confondus (remplace le token bucket).
- **Phase 1** reste synchrone pour l'app (enqueue-and-await) ; garde-fous : **priorité** aux jobs
  interactifs + **max-wait** avant bascule `202`+poll (sinon on recrée le timeout 0.5.3).

Phasage : **P0** file pilote `gbooks` → **P1** import série async (job + endpoints statut + progression)
→ **P2** généraliser MAL/BnF/Discogs/TMDB + supprimer le token bucket → **P3** durcissement (priorités,
circuit-breaker Redis, Bull Board, calibrage limiters).

## 4. État d'avancement — P0 IMPLÉMENTÉ (Google Books)

Branche `feat/p0-bullmq-gbooks` (backend). Ce qui a été fait :

**Infra**
- `package.json` : `bullmq` + `@nestjs/bullmq`.
- `docker-compose.yml` : service `redis:7-alpine` (`--maxmemory-policy noeviction`, healthcheck) +
  `REDIS_HOST=redis`/`REDIS_PORT` injectés à l'app + `depends_on: redis (healthy)`.
- `deploy/acervatim-redis.container` : Quadlet Redis prod (+ note dans `deploy/acervatim-api.container`).
- `src/config/env.validation.ts` + `.env.example` : `REDIS_HOST` (défaut `localhost`) / `REDIS_PORT` (6379).
- `src/app.module.ts` : `BullModule.forRootAsync` (connexion Redis depuis la config).
- `src/main.ts` : `app.enableShutdownHooks()` (drain des workers BullMQ au SIGTERM).

**Google Books éclaté en 3 responsabilités** (`src/common/sources/googlebooks/`)
- `googlebooks.resolver.ts` — `GoogleBooksResolver.fetchCover(isbn, hint)` : résolution **réseau pure**
  (isbn: puis repli `intitle:`), **jette** sur échec (permet le cache-2xx-only). Toute la logique de
  matching tome/édition/`bookDisplayNumber` vit ici.
- `googlebooks.processor.ts` — `GoogleBooksProcessor` (`@Processor('gbooks', { concurrency: 1,
  limiter: { max: 4, duration: 1000 } })`) : **seul** à appeler Google Books. Met en cache
  **uniquement sur 2xx** (une absence d'image = 200 → cache négatif TTL court ; un throw → rien).
  **Cadence baissée** (2→1 concurrence, 10→4 req/s) le 2026-07-08 après récidive des 429/503 :
  l'énumération d'une édition (~31 tomes) enfile autant de jobs, le limiter ne borne que leur
  démarrage, et chaque job fait jusqu'à 2 appels × 3 retries HTTP → engorgement. `concurrency: 1`
  sérialise les appels entre tomes.
- `googlebooks.service.ts` — `GoogleBooksCoverService` (producteur, **API publique inchangée**) :
  `cache → enqueue single-flight (jobId = clé de cache) → waitUntilFinished`, **best-effort** (Redis
  down / échec / timeout 15 s → `null`, jamais d'exception). `removeOnComplete: {age:60}` (fenêtre pour
  `waitUntilFinished`). **Retry job-level** (`attempts: 3`, `backoff exponential delay: 15_000` →
  ≈15 s puis ≈30 s) ajouté le 2026-07-08 : les 503 Google arrivent en **vagues de 20-30 s** que le
  retry HTTP (3 tentatives en ~2 s) ne traverse pas ; BullMQ retente donc le job **après** la vague,
  en arrière-plan, et met en cache dès qu'une tentative réussit. `waitUntilFinished` rend `null` sur
  la 1ʳᵉ vague, mais le poll suivant d'`edition-mapping` sert le cache réchauffé (cache warming).
  `removeOnFail: true` ne s'applique **qu'après les 3 tentatives épuisées** → l'échec définitif reste
  re-tentable au prochain scan (jobId libéré) ; pendant les retries (`delayed`) le single-flight tient.
- `googlebooks.types.ts` — types + constantes partagés (`GBOOKS_QUEUE`, TTL, `CoverHint`,
  `CoverResult`, `CachedCover`, `CoverJobData`, `normalizeIsbn`, `coverCacheKey`).
- `googlebooks.module.ts` — `BullModule.registerQueue({ name: 'gbooks' })` + 3 providers.

**Token bucket** : **entièrement retiré** (fin de P2). `TokenBucketService`, sa spec, `RateLimitModule`
et le modèle Prisma `RateLimitBucket` sont supprimés ; migration de drop
`20260708120000_drop_rate_limit_buckets`. Plus aucun consommateur — le throttle sortant passe
exclusivement par les limiters BullMQ (un par file source). Aucun cron ne le référençait.

**Validation** (conteneur `node:24` jetable — pas de node sur l'hôte Windows, cf. `scripts/dev.ps1`) :
- `npm run build` (typecheck complet) : **OK**.
- Specs Google Books unitaires : **27/27** (`resolver.spec` 16, `service.spec` 8, `processor.spec` 3).
- Suite unitaire complète : **361 pass** ; seuls échecs = `*.e2e.spec.ts` (exigent MariaDB/Redis).
- **E2E BullMQ contre un vrai Redis** : `googlebooks.e2e.spec.ts` (résolution bout-en-bout,
  single-flight, best-effort re-tentable, hit de cache) — voir §6 pour la commande.

## 4bis. État d'avancement — P1 IMPLÉMENTÉ (import série async)

Fire-and-poll de l'import « série complète », dans `src/search/import/` :
- `POST /v1/search/import-jobs` (`{ title, edition? }`) → `202 { jobId, state }`. `jobId`
  **déterministe** (`sha1(titre+édition normalisés)`, cf. `editionImportJobId`) → **dédup inter-user**.
  Idempotent : job en file/en cours/terminé renvoyé tel quel ; un échec est purgé puis relancé.
- `GET /v1/search/import-jobs/:id` → `{ jobId, state: queued|running|done|failed, progress: { phase,
  done, total } | null }`. `404` si inconnu.
- File BullMQ `edition-import` (`concurrency: 1`, tâche de fond → n'affame pas les scans interactifs).
  `EditionImportProcessor` appelle `SearchService.editionMapping(title, edition, onProgress)` : énumère
  la BnF puis résout les jaquettes **via la file `gbooks`** (throttlée) — effet de bord : **réchauffe
  le cache**. Quand `done`, l'app **re-tape `edition-mapping`** (servi du cache, < 1 s).
- `SearchService.editionMapping` a reçu un **callback de progression** optionnel (incrément par tome
  résolu). Rétro-compatible (les appels existants passent 2 args).
- Fichiers : `search/import/edition-import.{types,service,processor,controller}.ts`,
  `search/dto/create-import-job.dto.ts`, `search.module.ts` (registerQueue + providers + controller).
- **Validation** : build OK ; **21 specs** `src/search` (service, processor, search.service) ; **E2E
  contre un vrai Redis** `edition-import.e2e.spec.ts` **2/2** (cycle enqueue→worker→progression→done ;
  dédup deux demandes identiques → un seul job).

## 5. Reste à faire

- **App (P1 côté client) — IMPLÉMENTÉ + validé JVM** (repo `acervatim-app`, non commité).
  `SeriesImportViewModel` pilote la machine à états `OPTIONS → LAUNCHING → POLLING → IMPORTING → DONE` :
  `POST import-jobs` → **polling adaptatif** (`import-jobs/:id`, backoff 500 ms→2 s, garde-fou 120 s)
  → au `done`, l'import local (`ImportScannedSeriesUseCase.importFromList`, **inchangé**) re-tape
  `edition-mapping` (cache chaud). Le job async **réchauffe seulement le cache** ; toute erreur (POST
  KO, Redis `503`, job `failed`, `404`, timeout) tombe sur l'import local qui dégrade en « tome scanné
  seul ». Écran d'avancement (`ImportProgressContent` : indéterminé en énumération, `LinearProgressIndicator`
  done/total en phase jaquettes). Résilience choisie : **polling ViewModel** (job durable+idempotent) ;
  **WorkManager reporté en P3**. Fichiers : `data/remote/dto/SearchDto.kt` (DTO job),
  `data/remote/api/AcervatimApi.kt`, `domain/model/ImportJob.kt`,
  `domain/repository/ExternalSearchRepository.kt` (+Impl : `startImportJob` / `importJobStatus`,
  `404`→`success(null)`), `ui/series/SeriesImportViewModel.kt` + `SeriesImportScreen.kt`, strings
  `scan_series_import_{launching,phase_enumerating,phase_covers,phase_covers_count,finalizing}`.
  **Validation** : `:app:testDebugUnitTest` (VM + use case) vert ; **reste** : vérif device/emulateur
  bout-en-bout contre un backend avec Redis (cf. §6).
- **P2 — généralisation** ✅ **TERMINÉ** : **BnF (`7fbc75a`), TMDB (`5f57ff6`), MAL (`e160cca`) et
  Discogs (`0508a9d`) migrés et poussés**. `TokenBucketService` + spec + `RateLimitModule` + modèle
  `RateLimitBucket` **supprimés** (migration de drop `20260708120000_drop_rate_limit_buckets` ; aucun
  cron ne le référençait).
  - **Patron token-source établi (TMDB)** répliqué : le producteur (adapter) gate le mode dégradé
    (`none` → cache-only / `403`) + fast-fail typé (repli premium sans clé serveur → `503`), puis
    enfile `{ userId, url/params SANS secret }` ; le **worker résout le token/clé** (via
    `TokenResolverService`) et l'injecte à l'appel → **jamais de secret dans Redis**. Échec worker/Redis
    → `502`. `jobId = sha1(cléCache)`.
  - **Piège BullMQ** : un `jobId` **ne peut contenir ni `:` ni espace** (message trompeur « Custom Id
    cannot contain : »). gbooks/bnf OK (jobIds sans espace : ISBN/hash) ; TMDB/MAL (texte de requête) →
    jobId hashé (`sha1(cléCache)`).
  - **MAL — FAIT** (`e160cca`, décision actée) : le bucket **par-user** `mal:<userId>` (60/min) est
    **supprimé, pas remplacé** — l'anti-abus repose sur le `@nestjs/throttler` **entrant** (100/min/IP)
    et, pour les appels Bearer, sur le quota que MAL applique déjà au jeton de l'utilisateur. Le bucket
    **partagé** `acervatim:mal` est remplacé par le limiter de file. **Spécificité MAL vs TMDB** : le
    worker choisit le **header** selon le jeton résolu — `Authorization: Bearer` (user) ou
    `X-MAL-CLIENT-ID` serveur (repli premium) — là où TMDB n'ajoute qu'un `api_key` d'URL ;
    `MalProcessor` lit donc `MAL_CLIENT_ID` en config. Le pivot `searchByBarcode` conserve sa logique
    dégradé **inline** (none → cache-only → `bnf_only`) et enfile via le même `enqueueFetch` que
    `malResolvedGet`. Fichiers : `src/oauth/providers/mal.{types,processor,adapter}.ts` (+ specs +
    `mal.e2e.spec.ts` 3/3 Redis réel), `oauth.module.ts` (registerQueue + MalProcessor).
  - **Discogs — FAIT** (`0508a9d`) : file `discogs` (limiter ~1/s ≈ 60/min authentifié). Spécificité
    vs TMDB/MAL : **OAuth 1.0a** → la **signature** (HMAC-SHA1 sur méthode+URL+params) est calculée
    **dans le worker** (`DiscogsProcessor`) à partir du `userId` — ni consumer secret, ni token user,
    ni personal token Acervatim ne transitent par Redis (seuls `userId` + URL publique). Worker :
    `user` → signature token user ; `fallback` → `Discogs token=` personal si configuré (images) sinon
    signature consumer-only ; `none` → 403. Bucket **par-user** `discogs:<userId>` supprimé, pas
    remplacé (comme MAL) ; `acervatim:discogs` → limiter de file. Le **flux OAuth** (start/callback)
    reste un appel direct hors file. Fichiers : `src/oauth/providers/discogs.{types,processor,adapter}.ts`
    (+ specs + `discogs.e2e.spec.ts` 3/3 Redis réel), `oauth.module.ts` (registerQueue + DiscogsProcessor).
- **P3 — durcissement** (EN COURS) :
  - **Circuit-breaker Redis — FAIT** (`a712cf7`). `RedisHealthService` (`src/common/redis/`, `@Global`) :
    connexion ioredis dédiée dont l'état (`ready`/`close`) pilote `isAvailable()`, mis à jour
    ~immédiatement sur coupure (n'émet aucune commande ; `enableOfflineQueue: false`). Les producteurs
    consultent `isAvailable()` **avant** d'enfiler → Discogs/MAL/TMDB : `503` immédiat ; Google Books :
    `null` tout de suite (évite ~15 s × 30 tomes) ; import série (POST + status) : `503`. `/health`
    expose `redis: up|down` (mais `status` reste `ok` : Redis non vital au boot). ioredis ajouté en
    dépendance directe (déjà transitif via bullmq).
  - **Métriques de files — FAIT** (`70eacaf`). `QueueMetricsService` (`src/metrics/`) : `snapshot()`
    agrège `getJobCounts()` des files (waiting/active/delayed = backlog, completed = proxy débit
    ~60 s vu `removeOnComplete age:60`, failed) sans connexion Redis supplémentaire. Log périodique
    conditionnel (`@Interval` 60 s, uniquement si backlog/activité, court-circuité si Redis down) +
    endpoint `GET /admin/queues` (AdminTokenGuard). `failed` ~0 par conception (`removeOnFail:true` →
    échecs tracés par les workers) ; compteur d'échecs dédié = raffinement ultérieur.
  - **Reste P3** : Bull Board, **calibrage effectif des limiters** (à partir de ces métriques, sous
    charge réelle). App : WorkManager (reporté de P1).
- **État branche** : `feat/mangadex-pivot-principal`, **poussée sur `origin`** (l'ancienne
  `feat/p0-bullmq-gbooks` y a été repliée puis supprimée, local + distant). Commits : P0 `a4019a2`,
  P1 `a048d69`, P2 BnF `7fbc75a`, P2 TMDB `5f57ff6`, P2 MAL `e160cca`, P2 Discogs + drop token bucket
  `0508a9d`, P3 circuit-breaker Redis `a712cf7`, **P3 métriques files `70eacaf`**, **pivot MangaDex
  source principale `c91f4cb`**. App-side P1 (repo `acervatim-app`) : non commité. Calibrage des
  limiters en conditions réelles : à faire (P3, désormais outillé par les métriques).
- **Edge connu** (pré-existant, légèrement aggravé par le single-flight) : le cache jaquette est indexé
  **par ISBN seul** (ignore le `hint`). Un appel *sans hint* (ex. `mal.adapter.resolveCover(isbn)` sur
  le repli `bnf_only`) peut cacher un **négatif 7 j** qui prive ensuite le chemin *hint-armé* d'
  `edition-mapping` (cas Ki-oon). À traiter si gênant (clé de cache incluant la présence de hint, ou
  ne pas cacher négatif sans hint).

## 6. Développer & valider

Workflow normal (Windows) : **toujours** via `acervatim-backend/scripts/dev.ps1` (jamais npm/prisma
direct — binaires natifs incompatibles conteneur). `./scripts/dev.ps1 up` démarre désormais aussi
Redis (service `redis` du compose).

Validation hors dev.ps1 utilisée pour ce chantier (conteneur jetable, `node_modules` en volume nommé) :

```sh
# Build + specs gbooks (pas de DB/Redis nécessaires : Prisma/Queue mockés)
docker run --rm -v "C:/Developpement/acervatim-backend":/app -v acv_nodemodules:/app/node_modules \
  -w /app node:24-slim bash -lc "npm install && npx prisma generate && npm run build && \
  NODE_ENV=test npx jest src/common/sources/googlebooks"

# E2E contre un vrai Redis (réseau Docker dédié)
docker network create acv_e2e_net; docker run -d --name acv_redis_e2e --network acv_e2e_net redis:7-alpine
docker run --rm --network acv_e2e_net -e REDIS_HOST=acv_redis_e2e -e NODE_ENV=test \
  -v "C:/Developpement/acervatim-backend":/app -v acv_nodemodules:/app/node_modules -w /app \
  node:24-slim bash -lc "npx jest googlebooks.e2e.spec.ts --runInBand --forceExit"
docker rm -f acv_redis_e2e; docker network rm acv_e2e_net
```

(Sous Git Bash sur Windows, préfixer les `docker run` par `MSYS_NO_PATHCONV=1` pour éviter la
conversion des chemins `/app`.)

## 7. Fichiers clés

- Backend code : `src/common/sources/googlebooks/*` (resolver / processor / service / types / module),
  `src/app.module.ts`, `src/main.ts`, `src/config/env.validation.ts`.
- Infra : `docker-compose.yml`, `deploy/acervatim-redis.container`, `deploy/acervatim-api.container`.
- Déploiement Redis : [../../developpeur/deploiement.md](../../developpeur/deploiement.md) (§ Redis).
- Contexte sources/pivot : [sources-externes.md](../transverse/sources-externes.md),
  [../../developpeur/sources-et-pivot-isbn.md](../../developpeur/sources-et-pivot-isbn.md).
- MAL (P2, `e160cca`) : `src/oauth/providers/mal.{types,processor,adapter}.ts` + specs + `mal.e2e.spec.ts`.
- Discogs (P2, `0508a9d`) : `src/oauth/providers/discogs.{types,processor,adapter}.ts` + specs +
  `discogs.e2e.spec.ts`. Signature OAuth 1.0a réutilisée depuis `src/oauth/providers/oauth1.ts`.
- **Supprimé** (P2, `0508a9d`) : `src/common/rate-limit/` (tout le dossier), modèle Prisma
  `RateLimitBucket`. Migration de drop : `prisma/migrations/20260708120000_drop_rate_limit_buckets`.
- Circuit-breaker (P3, `a712cf7`) : `src/common/redis/redis-health.{service,module}.ts` (+ spec) ;
  garde `isAvailable()` dans les `enqueueFetch`/producteurs de chaque source + `src/app.service.ts`
  (`/health`).
- Métriques (P3, `70eacaf`) : `src/metrics/queue-metrics.{service,controller,module}.ts` (+ spec +
  e2e). Endpoint `GET /admin/queues` (AdminTokenGuard).
