# acervatim-app — Catalogue des fonctionnalités → fichiers

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`
> Pour chaque feature : UI → logique → données. Commencer par le ViewModel listé.

## Collection & consultation

| Fonctionnalité | UI | Logique / données |
|---|---|---|
| Liste de la collection (albums ou séries selon type actif) | `ui/list/CollectionScreen.kt`, `CollectionViewModel`, `SeriesGrid`, `RecordRow` | `ObserveFilteredRecordsUseCase`, `ObserveSeriesWithCountsUseCase`, `ObserveActiveCollectionTypeUseCase` |
| Filtres (genre, statut, recherche, mode liste) | `ui/list/FilterSheet.kt` | `domain/model/RecordFilter.kt`, `ListMode`, `ObserveDistinctGenresUseCase`, `ListModeForCollectionTypeUseCases`. Recherche texte insensible à la casse **et aux accents** dans les deux vues (liste plate : `records.normalized_title`, Room v8) |
| Tri de la collection (artiste, titre, année, dernière écoute, ajout, note) | `ui/list/SortAction.kt` (barre de recherche, **distinct** du FilterSheet) | `domain/model/RecordSort.kt`, `data/local/db/RecordQueries.kt` (ORDER BY), `SortForCollectionTypeUseCases` (persisté par type) |
| Densité d'affichage (liste / grille / galerie) | `ui/list/CollectionTileBody.kt` (`DisplayModeAction`), `SeriesList`, `SeriesGrid` | `domain/model/DisplayMode.kt`, `DisplayModeForCollectionTypeUseCases` (persisté par type) |
| Galerie : derniers écoutés, plus écoutés, oubliés | `ui/list/CollectionTileBody.kt` | `CollectionUiState.from` (rubriques bornées, vides masquées) ; `Record.playCount` / agrégats `SeriesWithCounts` |
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
| Préremplissage du formulaire depuis un scan ou une recherche | `ui/scan/ScanPrefillHandoff.kt` | routes `record/add?barcode=` / `?prefillKey=`, `manga/add?barcode=` |
| Import d'une série manga complète (« toute la série d'un coup ») | `ui/series/SeriesImportScreen.kt`, `SeriesImportViewModel` | `ImportScannedSeriesUseCase`, `WarmEditionImportUseCase` (job serveur + polling), `/v1/search/edition-mapping` (BnF, avec l'éditeur du tome scanné), `domain/model/EditionMapping.kt` |
| **Ajout d'une série manga par titre**, sans tome possédé (AD1, branche `feat/manga-recherche-serie`) | `ui/series/SeriesSearchScreen.kt`, `SeriesSearchViewModel` (route `series/search`, entrée dans la feuille d'ajout de la vue séries, connecté) | `SearchMangaSeriesUseCase` → `/v1/search?type=manga&q=` (MangaDex, sans jeton) ; `ExternalSearchRepository.listEditions` → `/v1/search/editions` ; `WarmEditionImportUseCase` puis `ImportScannedSeriesUseCase.importFromSearch` (tous les tomes en wishlist) ; repli création manuelle pré-remplie |
| **Nouveaux tomes d'une série en cours** (AD3) | `ui/series/NewVolumesBanner.kt` dans le détail série | `SeriesCatchUpUseCase` (`edition-mapping?since=` + `series-updates` en premium), `ImportScannedSeriesUseCase.importNewTomes`, état dans `DataStoreSeriesCatchUpStore` |
| Recherche externe **par texte** (disque sans code-barres) | `ui/search/ExternalSearchScreen.kt`, `ExternalSearchViewModel` (saisie debouncée 400 ms) | `SearchExternalByTextUseCase` → `ExternalSearchRepository.searchByText` → `/v1/search?q=` ; résultats regroupés par `domain/model/PressingGroup.kt` |
| Choisir le bon pressage | groupes dépliables, ligne discriminante année · label · pays · format · n° de catalogue | `ExternalMatch.masterId` regroupe, `catalogNumber`/`label`/`country` départagent — cf. [../backend/recherche-textuelle-discogs.md](../backend/recherche-textuelle-discogs.md) |
| Refus de jeton de source (`403 source-token-required`) | message actionnable + « Connecter une source » (→ Réglages → Gérer les collections), **jamais** « aucun résultat » | traduit dès `data/remote/search/` en `domain/model/ExternalSearchFailure.kt` |
| Code-barres d'un résultat de recherche | `AddEditRecordViewModel.adoptMatchBarcode` | **Piège** : `records.barcode` est un index **unique**, et Discogs annonce souvent le même EAN sur plusieurs pressages d'un album. Le champ n'est repris que si le code est **libre**, sinon le second pressage devient impossible à ajouter. Un doublon réel sort en `AddRecordEvent.DuplicateBarcode` (snackbar), jamais en exception |

## État utilisateur sur les items

| Fonctionnalité | Logique |
|---|---|
| Note (rating) | `UpdateRatingUseCase` |
| « Écouté / lu » + dernière écoute + **compteur d'écoutes** | `MarkAsPlayedUseCase` (incrémente `Record.playCount` avec l'horodatage), `ResetLastPlayedUseCase`, `ui/util/RelativeTime.kt` |
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
| Suppression de compte | idem | `DELETE /v1/me` — `GET /v1/me/quota` a été **supprimé** (S2/A2 : plus de quota d'offre) |
| Partage de collections (propriétaire) | `ui/sharing/SharingScreen.kt`, `SharingViewModel`, `CreateShareSheet`, `ShareCodeDialog`, `ShareLabelDialog` ; entrée « Mes partages » dans `ui/list/CollectionDrawer.kt` | `SharingRepository` → `data/remote/sharing/SharingRepositoryImpl.kt`, `domain/model/sharing/SharingModels.kt` — `POST/GET /v1/shares`, `PATCH/DELETE /v1/shares/:id` |
| Consultation d'un partage reçu (membre) | `ui/sharing/received/` : `ReceivedSharesScreen`, `JoinShareDialog`, `SharedCollectionScreen`, `SharedItemDetailScreen` ; entrée « Partagé avec moi » dans `ui/list/CollectionDrawer.kt` | `SharingRepository` (`redeem`, `receivedShares`) + `SharedCollectionRepository` → `data/remote/sharing/SharedCollectionRepositoryImpl.kt`, `domain/model/sharing/SharedBrowsing.kt` — `POST /v1/shares/redeem`, `GET /v1/shares/received`, puis les routes de lecture normales |
