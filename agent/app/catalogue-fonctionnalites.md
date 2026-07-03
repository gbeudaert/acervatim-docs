# acervatim-app — Catalogue des fonctionnalités → fichiers

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`
> Pour chaque feature : UI → logique → données. Commencer par le ViewModel listé.

## Collection & consultation

| Fonctionnalité | UI | Logique / données |
|---|---|---|
| Liste de la collection (albums ou séries selon type actif) | `ui/list/CollectionScreen.kt`, `CollectionViewModel`, `SeriesGrid`, `RecordRow` | `ObserveFilteredRecordsUseCase`, `ObserveSeriesWithCountsUseCase`, `ObserveActiveCollectionTypeUseCase` |
| Filtres & tri (genre, statut, recherche, mode liste) | `ui/list/FilterSheet.kt` | `domain/model/RecordFilter.kt`, `ListMode`, `ObserveDistinctGenresUseCase`, `ListModeForCollectionTypeUseCases` |
| Drawer types de collection + statistiques | `ui/list/CollectionDrawer.kt` | `ObserveCollectionTypeStatsUseCase`, `ObserveCollectionStatsUseCase`, `SetActiveCollectionTypeUseCase` |
| Détail album vinyle | `ui/detail/RecordDetailScreen.kt` | `ObserveRecordByIdUseCase` |
| Détail tome manga | `ui/manga/MangaDetailScreen.kt` | idem + `MangaExtras` |
| Détail série (tomes possédés / manquants / wishlist) | `ui/series/SeriesDetailScreen.kt`, `VolumeTile` | `ObserveSeriesByIdUseCase`, `ObserveSeriesItemsUseCase` |

## Ajout & édition

| Fonctionnalité | UI | Logique / données |
|---|---|---|
| Ajout/édition album | `ui/edit/AddEditRecordScreen.kt` | `AddRecordUseCase`, `UpdateRecordUseCase`, `RecordInput`, `ObserveArtistSuggestionsUseCase` |
| Ajout/édition tome manga | `ui/manga/AddEditMangaScreen.kt` | idem + `UpsertSeriesUseCase` |
| Création/édition série | `ui/series/AddEditSeriesScreen.kt` | `UpsertSeriesUseCase` |
| Suppression item / série | écrans détail | `DeleteRecordUseCase`, `DeleteSeriesUseCase` |
| Choix / zoom de couverture | `ui/components/CoverPicker.kt`, `ZoomableCover.kt` | `domain/repository/CoverStorage` → `data/local/storage/CoverStorageImpl.kt` |

## Scan & enrichissement externe

| Fonctionnalité | UI | Logique / données |
|---|---|---|
| Scan code-barres (EAN/ISBN) | `ui/scan/BarcodeScanScreen.kt`, `BarcodeAnalyzer` (ML Kit), `BarcodeScanViewModel` | `LookupByBarcodeUseCase` → `ExternalSearchRepository` → backend `/v1/search?barcode=` |
| Préremplissage du formulaire depuis un scan | `ui/scan/ScanPrefillHandoff.kt` | routes `record/add?barcode=`, `manga/add?barcode=` |
| Import d'une série manga complète (« toute la série d'un coup ») | `ui/series/SeriesImportScreen.kt`, `SeriesImportViewModel` | `ImportScannedSeriesUseCase`, `/v1/search/edition-mapping` (BnF), `domain/model/EditionMapping.kt` |
| Recherche externe par texte | écrans d'ajout | `ExternalSearchRepositoryImpl`, `domain/model/ExternalMatch.kt` |

## État utilisateur sur les items

| Fonctionnalité | Logique |
|---|---|
| Note (rating) | `UpdateRatingUseCase` |
| « Écouté / lu » + dernière écoute | `MarkAsPlayedUseCase`, `ResetLastPlayedUseCase`, `ui/util/RelativeTime.kt` |
| Wishlist | `ToggleWishlistUseCase`, `UpdateItemStatusUseCase`, `domain/model/ItemStatus.kt` |

## Réglages & données

| Fonctionnalité | UI | Logique / données |
|---|---|---|
| Réglages (thème, doublons…) | `ui/settings/SettingsScreen.kt` | `SettingsRepository` (DataStore), `ObserveDuplicateAlbumsSettingUseCase` |
| Gestion des types de collection visibles | `ui/settings/ManageCollectionTypesScreen.kt` | `CollectionTypeVisibilityUseCases` |
| Export / import backup ZIP | `ui/settings/SettingsScreen.kt` | `ExportBackupUseCase`, `ImportBackupUseCase`, `data/backup/*` — voir [donnees-locales.md](donnees-locales.md) |

## Mode connecté (compte, sync, premium)

| Fonctionnalité | UI | Logique / données |
|---|---|---|
| Connexion Google | `ui/account/AccountScreen.kt`, `AccountViewModel` | `AuthRepository` → `GoogleSignInClient`, `TokenStore`, `POST /v1/auth/google` |
| Synchronisation cloud | idem | `SyncRepository` → `SyncManagerImpl`, `SyncWorker` — voir [reseau-sync-compte.md](reseau-sync-compte.md) |
| Premium (vérification achat Play) | idem | `AccountRepository` → `POST /v1/subscriptions/verify` |
| Code d'invitation | idem | `POST /v1/invitations/redeem` |
| Suppression de compte / quota | idem | `DELETE /v1/me`, `GET /v1/me/quota` |
