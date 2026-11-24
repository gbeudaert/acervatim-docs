# Flux métier bout-en-bout (app ↔ backend)

> Remonter : [../INDEX.md](../INDEX.md) · Chemins app relatifs à `acervatim-app/app/src/main/java/app/acervatim/`, backend à `acervatim-backend/src/`.
> **Constellation** (`acervatim/backend-services`, cible décidée le 2026-11-16) : mêmes flux vus de l'app ;
> le `gateway` vérifie le JWT, signe un principal (`X-Acervatim-Principal`), applique premium et débits,
> puis route vers le service qui porte la route (table : `gateway/src/routing/route-table.ts`).

## 1. Connexion Google (mode connecté)

```
AccountScreen → AuthRepositoryImpl → GoogleSignInClient (idToken)
  → POST /v1/auth/google (AuthApi)
  → backend: google.provider (vérif idToken) → identity-resolver (HMAC sub → User) → JWT
  → app: TokenStore (persistance) ; AuthInterceptor ajoute le Bearer ensuite
```
Détails app : [../app/reseau-sync-compte.md](../app/reseau-sync-compte.md) · backend : [../backend/securite-et-privacy.md](../backend/securite-et-privacy.md)

## 2. Scan d'un code-barres → ajout enrichi

```
BarcodeScanScreen (CameraX) → BarcodeAnalyzer (ML Kit) → BarcodeScanViewModel
  → LookupByBarcodeUseCase → ExternalSearchRepositoryImpl
  → GET /v1/search?type=<vinyl|manga>&barcode=<EAN|ISBN>
  → backend: SearchService → SourceAdapter du type (Discogs / MalAdapter : BnF → MangaDex → MAL en repli)
  → app: ExternalMatch → ScanPrefillHandoff → route record/add?barcode= ou manga/add?barcode=
  → AddRecordUseCase (Room local) [+ sync ultérieure si connecté]
```

## 3. Import d'une série manga complète

```
Scan ISBN d'un tome → SeriesImportScreen (route series/import?barcode=)
  → GET /v1/search (identifie le tome + titleFr)
  → GET /v1/search/edition-mapping?title=<titreFr>&edition=<mention>&publisher=<éditeur>&mangaId=
  → backend: BnfService.enumerateEdition (SRU BnF + edition-records.ts) → liste des tomes de l'édition
  → app: ImportScannedSeriesUseCase → UpsertSeriesUseCase + création des tomes (owned/wishlist)
```

## 3bis. Ajout d'une série manga par titre, sans tome possédé (SD1 · AD1 · AD2)

Fusionné dans `main` le 2026-11-14 (monolithe, app, constellation) ; en production par la
constellation depuis le 2026-11-23. Cf.
[../../spec-macro-import-serie-manga.md](../../spec-macro-import-serie-manga.md).

```
Vue séries manga (connecté) → feuille d'ajout « Rechercher une série » → route series/search
  → SeriesSearchViewModel (debounce 400 ms, 3 caractères) → SearchMangaSeriesUseCase
  → GET /v1/search?type=manga&q=<texte>      (MangaDex sans jeton → SeriesCandidate)
  → choix du candidat → GET /v1/search/editions?mangaId=<uuid>&title=<saisie>  (BnF → SeriesEditionOption)
     └─ aucune édition / candidat sans mangaId → création manuelle pré-remplie (AddEditSeries)
  → choix de l'édition → WarmEditionImportUseCase : POST /v1/search/import-jobs { title, edition,
     publisher, mangaId } + polling (même écran d'avancement que le scan)
  → ImportScannedSeriesUseCase.importFromSearch → série (édition, éditeur, mangaId) + tous les tomes
     en WISHLIST, dédup par ISBN
```

## 3ter. Rattrapage des nouveaux tomes (SD3 · AD3)

```
App, ouverture d'une série (connecté) → SeriesDetailViewModel → SeriesCatchUpUseCase.check
  éligible : manga, titre + éditeur connus, parution non terminée, dernière vérification ≥ 7 j
  → GET /v1/search/edition-mapping?…&publisher=&since=<dernière vérif − 7 j>   (bib.creationdate)
  → premium : + GET /v1/series-updates (SeriesUpdatesCache 10 min/compte ; fusion par nodeId ↔ remoteId)
  → tomes absents (ISBN normalisé ET n°) → bannière « N nouveaux tomes » → addToWishlist / Plus tard
Serveur, lundi 04:00 → file edition-watch → editionMapping(since) par édition suivie
  → état ApiCache edition-watch:v1:<sha1> → servi par GET /v1/series-updates
```

## 4. Synchronisation locale ↔ cloud

```
Déclenchement manuel (écran Compte) → SyncManagerImpl (+ SyncStateStore)
  (SyncWorker.schedule existe mais n'est appelé nulle part : pas de sync périodique)
  → CRUD /v1/collections, /v1/collections/:id/nodes, /v1/collections/:id/items (AcervatimApi)
  → mapping DTO ↔ Room via RemoteDtoMapper / FacadeMapper / LocalCollectionResolver
```
Room reste la vérité locale ; `unifiedData` serveur = vérité curée synchronisée.

## 5. Achat premium (Google Play)

```
App: achat Play Billing → POST /v1/subscriptions/verify { purchaseToken }
  (côté app : pas encore de Play Billing ; le premium s'obtient par invitation, § 6)
  → backend: google-play.service (vérif auprès de l'API Play) → upsert Subscription
Asynchrone: Google → POST /webhooks/google-play/rtdn (Pub/Sub, signé)
  → pubsub-verifier → subscriptions.service (transition de statut, subscription-status.ts)
Cron: subscriptions.cron re-vérifie les subs proches d'expiration
Gate: premium.guard (402, ecriture) + collection-premium.guard (402, lecture owner-aware)
```

## 6. Invitation (beta / comp)

```
Admin: POST /v1/admin/invitations (admin-token.guard) → code en clair retourné une fois, HMAC stocké
User: AccountScreen → POST /v1/invitations/redeem { code }
  → backend: HMAC(code) → Invitation.codeHash → InvitationRedemption + PremiumGrant éventuel
```

## 7. Connexion d'un compte externe (Discogs / MAL) et clé TMDB

```
App: GET /v1/oauth/{provider}/start → URL d'autorisation (navigateur)
Provider → GET /v1/oauth/{provider}/callback
  → backend: adapter (oauth1.ts pour Discogs, pkce.ts pour MAL) → tokens chiffrés AES → oauth_credentials
App: GET /v1/oauth (liste) · DELETE /v1/oauth/{provider} (révocation)
TMDB (pas d'OAuth) : PUT / DELETE /v1/sources/tmdb/token → clé chiffrée, provider `tmdb`
```
Constellation : service `connections` (flux, stockage, résolution du jeton demandée par `search`).

## 8. Export / import backup local (sans backend)

```
SettingsScreen → ExportBackupUseCase → BackupGatewayImpl → BackupZip (JSON versionné + covers)
Import : ImportBackupUseCase → validation BackupSchema → upserts Room + BackupCoverIo
```

## 9. Suppression de compte (RGPD)

```
AccountScreen → DELETE /v1/me
  → backend: users.service → delete User → cascades Prisma (collections, items, nodes,
    oauth_credentials, subscription, grants, redemptions) + AuditLog 'user.delete'
Export préalable possible : GET /v1/me/export
```
Constellation : saga orchestrée par `identity` (chaque service purge son schéma, participants
idempotents, rejeu et alerte) — `backend-services/identity/README.md` § 2. Pas encore éprouvée en
panne réelle.

## 10. Scan d'un jeu de société (constellation seulement, pending)

```
App (à écrire, F13) → GET /v1/search/boardgame/barcode/:code[?q=&format=]  (auth user, pas premium)
  → gateway → search: BoardGameBarcodeService → normalizeGtin (400 si invalide)
  → GameUPC (cache → file gameupc) → verified | choix | unknown | unavailable (+ stale)
  → fiche BGG si BGG_API_TOKEN (vide aujourd'hui : licence pending)
Choix utilisateur → POST /v1/search/boardgame/barcode/:code/choice (30/min)
  → file gameupc-feedback (vote sous pseudonyme HMAC)
```
Détail : [../backend/gameupc-bgg.md](../backend/gameupc-bgg.md). Cible : `bgg-data` seul appelant BGG (F7, F8).
