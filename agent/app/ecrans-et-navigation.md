# acervatim-app — Écrans & navigation

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`

## Navigation

- Graphe : `ui/navigation/AcervatimNavGraph.kt`
- Routes + builders : `ui/navigation/AcervatimDestinations.kt` (objet `AcervatimDestinations`, args dans `AcervatimNavArgs`).

## Table routes ↔ écrans ↔ ViewModels

| Route | Écran (composable) | ViewModel | Rôle |
|---|---|---|---|
| `collection` (start) | `ui/list/CollectionScreen.kt` | `ui/list/CollectionViewModel.kt` | Liste principale de la collection active (albums ou grille de séries) |
| `record/add?barcode=` | `ui/edit/AddEditRecordScreen.kt` | `ui/edit/AddEditRecordViewModel.kt` | Ajout d'un album (préremplissable par scan) |
| `record/{recordId}` | `ui/detail/RecordDetailScreen.kt` | `ui/detail/RecordDetailViewModel.kt` | Détail album |
| `record/{recordId}/edit` | `ui/edit/AddEditRecordScreen.kt` | idem | Édition album |
| `scan?collectionType=&seriesImport=&seriesId=` | `ui/scan/BarcodeScanScreen.kt` | `ui/scan/BarcodeScanViewModel.kt` | Scan code-barres (CameraX + ML Kit) |
| `series/import?barcode=` | `ui/series/SeriesImportScreen.kt` | `ui/series/SeriesImportViewModel.kt` | Import d'une série manga complète depuis un ISBN scanné |
| `series/add?collectionType=` | `ui/series/AddEditSeriesScreen.kt` | `ui/series/AddEditSeriesViewModel.kt` | Création série |
| `series/{seriesId}` | `ui/series/SeriesDetailScreen.kt` | `ui/series/SeriesDetailViewModel.kt` | Détail série (tomes possédés/wishlist) |
| `series/{seriesId}/edit` | `ui/series/AddEditSeriesScreen.kt` | idem | Édition série |
| `manga/add?seriesId=&barcode=&volumeNumber=` | `ui/manga/AddEditMangaScreen.kt` | `ui/manga/AddEditMangaViewModel.kt` | Ajout d'un tome |
| `manga/{recordId}` | `ui/manga/MangaDetailScreen.kt` | `ui/manga/MangaDetailViewModel.kt` | Détail tome |
| `manga/{recordId}/edit` | `ui/manga/AddEditMangaScreen.kt` | idem | Édition tome |
| `settings` | `ui/settings/SettingsScreen.kt` | `ui/settings/SettingsViewModel.kt` | Réglages (thème, doublons, backup export/import…) |
| `settings/collection-types` | `ui/settings/ManageCollectionTypesScreen.kt` | `ui/settings/ManageCollectionTypesViewModel.kt` | Visibilité/gestion des types de collection |
| `account` | `ui/account/AccountScreen.kt` | `ui/account/AccountViewModel.kt` | Compte Google, sync, premium, invitations, suppression |

## Composants réutilisables & UI annexe

| Fichier | Rôle |
|---|---|
| `ui/list/CollectionDrawer.kt` | Drawer de sélection du type de collection actif + stats |
| `ui/list/FilterSheet.kt` | Bottom sheet de filtres/tri de la liste |
| `ui/list/SeriesGrid.kt` | Grille des séries (mode manga) |
| `ui/series/VolumeTile.kt` | Tuile d'un tome dans le détail série |
| `ui/components/RecordRow.kt` | Ligne d'item dans la liste |
| `ui/components/RecordCover.kt` | Affichage pochette/couverture |
| `ui/components/CoverPicker.kt` | Sélection/remplacement d'image de couverture |
| `ui/components/ZoomableCover.kt` | Zoom plein écran de la couverture |
| `ui/scan/BarcodeAnalyzer.kt` | Analyse ML Kit des frames caméra |
| `ui/scan/ScanPrefillHandoff.kt` | Transfert du résultat de scan vers l'écran d'ajout (préremplissage) |
| `ui/util/RelativeTime.kt` · `ui/util/SectionLetter.kt` | Helpers d'affichage (temps relatif, index alphabétique) |

## Thème

`ui/theme/` : `Theme.kt` (point d'entrée), `Color.kt`, `ColorSchemeMapper.kt` (mapping thème dynamique/choisi), `Type.kt`, `Shape.kt`. Toujours passer par `MaterialTheme.colorScheme/typography/shapes`.
