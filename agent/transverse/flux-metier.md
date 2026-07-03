# Flux métier bout-en-bout (app ↔ backend)

> Remonter : [../INDEX.md](../INDEX.md) · Chemins app relatifs à `acervatim-app/app/src/main/java/app/acervatim/`, backend à `acervatim-backend/src/`.

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
  → backend: SearchService → SourceAdapter du type (Discogs / MAL+BnF)
  → app: ExternalMatch → ScanPrefillHandoff → route record/add?barcode= ou manga/add?barcode=
  → AddRecordUseCase (Room local) [+ sync ultérieure si connecté]
```

## 3. Import d'une série manga complète

```
Scan ISBN d'un tome → SeriesImportScreen (route series/import?barcode=)
  → GET /v1/search (identifie le tome + titleFr)
  → GET /v1/search/edition-mapping?title=<titreFr>&edition=<mention>
  → backend: BnfService.enumerateEdition (SRU BnF + unimarc.parser) → liste des tomes de l'édition
  → app: ImportScannedSeriesUseCase → UpsertSeriesUseCase + création des tomes (owned/wishlist)
```

## 4. Synchronisation locale ↔ cloud

```
SyncWorker (WorkManager) / déclenchement manuel → SyncManagerImpl (+ SyncStateStore)
  → CRUD /v1/collections, /v1/collections/:id/nodes, /v1/collections/:id/items (AcervatimApi)
  → mapping DTO ↔ Room via RemoteDtoMapper / FacadeMapper / LocalCollectionResolver
```
Room reste la vérité locale ; `unifiedData` serveur = vérité curée synchronisée.

## 5. Achat premium (Google Play)

```
App: achat Play Billing → POST /v1/subscriptions/verify { purchaseToken }
  → backend: google-play.service (vérif auprès de l'API Play) → upsert Subscription
Asynchrone: Google → POST /webhooks/google-play/rtdn (Pub/Sub, signé)
  → pubsub-verifier → subscriptions.service (transition de statut, subscription-status.ts)
Cron: subscriptions.cron re-vérifie les subs proches d'expiration
Gate: premium.guard (402) + quota.service (limites free)
```

## 6. Invitation (beta / comp)

```
Admin: POST /v1/admin/invitations (admin-token.guard) → code en clair retourné une fois, HMAC stocké
User: AccountScreen → POST /v1/invitations/redeem { code }
  → backend: HMAC(code) → Invitation.codeHash → InvitationRedemption + PremiumGrant éventuel
```

## 7. Connexion d'un compte externe (Discogs / MAL / TMDB)

```
App: GET /v1/oauth/{provider}/start → URL d'autorisation (navigateur)
Provider → GET /v1/oauth/{provider}/callback
  → backend: adapter (oauth1.ts pour Discogs, pkce.ts pour MAL) → tokens chiffrés AES → oauth_credentials
App: GET /v1/oauth (liste) · DELETE /v1/oauth/{provider} (révocation)
```

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
