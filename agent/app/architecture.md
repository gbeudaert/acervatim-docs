# acervatim-app — Architecture (couches, DI, use cases)

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`

## Couches

```
ui/  (Compose + ViewModel)  →  domain/ (modèles, interfaces, use cases)  ←  data/ (implémentations)
```

- `ui/` ne référence que `domain/` (jamais `data/`).
- `domain/repository/*` définit les interfaces ; `data/**` les implémente ; le binding est fait dans `di/RepositoryModule.kt`.
- État d'écran : `ViewModel` + `StateFlow<UiState>`, collecte via `collectAsStateWithLifecycle()`.

## Modules Hilt (`di/`)

| Fichier | Fournit |
|---|---|
| `di/DatabaseModule.kt` | Room `AcervatimDatabase` + DAOs |
| `di/NetworkModule.kt` | Retrofit, OkHttp (+ `AuthInterceptor`, `ErrorInterceptor`), `AcervatimApi`, `AuthApi` |
| `di/RepositoryModule.kt` | Bindings interfaces `domain/repository` → impls `data/` |
| `di/SettingsModule.kt` | DataStore / `SettingsRepository` |
| `di/StorageModule.kt` | `CoverStorage` |

## Interfaces repository (`domain/repository/`)

| Interface | Rôle | Implémentation |
|---|---|---|
| `RecordRepository` | CRUD + flux des items (albums/tomes) | `data/repository/RecordRepositoryImpl.kt` |
| `SeriesRepository` | CRUD + flux des séries manga | `data/repository/SeriesRepositoryImpl.kt` |
| `SettingsRepository` | Préférences (type actif, mode liste, doublons, thème…) | `data/local/settings/DataStoreSettingsRepository.kt` |
| `CoverStorage` | Fichiers image des pochettes/couvertures | `data/local/storage/CoverStorageImpl.kt` |
| `BackupGateway` | Export/import ZIP | `data/backup/BackupGatewayImpl.kt` |
| `AuthRepository` | Sign-in Google, session, tokens | `data/remote/auth/AuthRepositoryImpl.kt` |
| `AccountRepository` | Profil `/me`, suppression compte, premium, invitations | `data/remote/account/AccountRepositoryImpl.kt` |
| `ExternalSearchRepository` | Recherche externe via backend (`/v1/search`) | `data/remote/search/ExternalSearchRepositoryImpl.kt` |
| `SyncRepository` | Synchronisation locale ↔ backend | `data/remote/sync/SyncManagerImpl.kt` |
| `SharingRepository` | Partages, **les deux sens** : émis (créer / lister / renommer / révoquer) et reçus (rejoindre / lister / renommer) | `data/remote/sharing/SharingRepositoryImpl.kt` |
| `SharedCollectionRepository` | Lecture **en ligne** d'une collection reçue en partage. Aucune persistance : ni Room, ni export, ni ZIP | `data/remote/sharing/SharedCollectionRepositoryImpl.kt` |

## Modèles domaine (`domain/model/`)

Principaux : `Record` (item générique), `Series` / `SeriesWithCounts`, `Artist` / `RecordArtistEntry`, `CollectionType`, `RecordFilter` (filtres liste) et `RecordSort` (tri : champ + sens, **à côté** du filtre), `ItemStatus` (owned/wishlist…), `ListMode` (hiérarchie) et `DisplayMode` (densité : liste/grille/galerie, axe orthogonal), `AcervatimFormat`, `RecordingSpeed` (vinyle), `MangaExtras` / `MangaDemographic` / `EditionMapping` / `Isbn` (manga), `ExternalMatch` (résultat recherche externe).
Sous-dossiers : `model/auth/` (`AuthState`, `SignInCancelledException`), `model/account/` (`AccountModels`), `model/sync/` (`RemoteModels`, `SyncState`, `SyncStatus`).

## Use cases (`domain/usecase/`) — catalogue par thème

### Items (albums / tomes)
`AddRecordUseCase` · `UpdateRecordUseCase` · `DeleteRecordUseCase` · `RecordInput` (modèle d'entrée) · `ObserveRecordByIdUseCase` · `ObserveRecordsUseCase` · `ObserveFilteredRecordsUseCase` (applique `RecordFilter` + `RecordSort`) · `ObserveCollectionEntriesUseCase`

### État utilisateur sur un item
`UpdateRatingUseCase` · `MarkAsPlayedUseCase` (horodate **et** incrémente `playCount`) · `ResetLastPlayedUseCase` (efface les deux) · `ToggleWishlistUseCase` · `UpdateItemStatusUseCase`

### Séries manga
`UpsertSeriesUseCase` · `DeleteSeriesUseCase` · `ObserveSeriesByIdUseCase` · `ObserveSeriesItemsUseCase` · `ObserveSeriesWithCountsUseCase` · `ObserveSeriesByCollectionTypeUseCase` · `ImportScannedSeriesUseCase` (import d'une série complète depuis un scan)

### Types de collection & réglages
`ObserveActiveCollectionTypeUseCase` · `SetActiveCollectionTypeUseCase` · `CollectionTypeVisibilityUseCases` · `ListModeForCollectionTypeUseCases` · `DisplayModeForCollectionTypeUseCases` · `SortForCollectionTypeUseCases` · `ObserveDuplicateAlbumsSettingUseCase`

### Statistiques & suggestions
`ObserveCollectionStatsUseCase` · `ObserveCollectionTypeStatsUseCase` · `ObserveDistinctGenresUseCase` · `ObserveArtistSuggestionsUseCase`

### Recherche externe & backup
`LookupByBarcodeUseCase` (code-barres → recherche externe) · `SearchExternalByTextUseCase` (titre/artiste → pressages regroupés par album ; court-circuite le réseau si la requête fait moins de 3 caractères ou si aucune session n'est ouverte) · `ExportBackupUseCase` · `ImportBackupUseCase`

## Points d'entrée

- `AcervatimApplication.kt` — `@HiltAndroidApp`.
- `MainActivity.kt` — unique Activity ; pose le thème et `AcervatimNavGraph`.
