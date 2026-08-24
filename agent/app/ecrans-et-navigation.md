# acervatim-app — Écrans & navigation

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`

## Navigation

- Graphe : `ui/navigation/AcervatimNavGraph.kt`
- Routes + builders : `ui/navigation/AcervatimDestinations.kt` (objet `AcervatimDestinations`, args dans `AcervatimNavArgs`).

## Table routes ↔ écrans ↔ ViewModels

| Route | Écran (composable) | ViewModel | Rôle |
|---|---|---|---|
| `collection` (start) | `ui/list/CollectionScreen.kt` | `ui/list/CollectionViewModel.kt` | Collection active, dans la densité choisie (liste / grille / galerie), à plat ou par séries |
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
| `sharing` | `ui/sharing/SharingScreen.kt` | `ui/sharing/SharingViewModel.kt` | Partages **émis** : créer un code (N collections × statuts), nommer, voir les membres, révoquer (propriétaire premium). Sans argument : un partage n'appartient plus à une collection |
| `shares/received` | `ui/sharing/received/ReceivedSharesScreen.kt` | `ui/sharing/received/ReceivedSharesViewModel.kt` | Partages **reçus** : rejoindre par code, nommer pour soi. Ouvert à **tous** les comptes, premium ou non |
| `shared/collection/{collectionId}` | `ui/sharing/received/SharedCollectionScreen.kt` | `ui/sharing/received/SharedCollectionViewModel.kt` | Consultation d'une collection partagée : **lecture seule, en ligne**, bandeau de portée permanent, pagination cursor |
| `shared/series/{collectionId}/{seriesId}` | `ui/sharing/received/SharedSeriesDetailScreen.kt` | `ui/sharing/received/SharedSeriesDetailViewModel.kt` | Détail d'une série partagée. Corps commun (`ui/series/SeriesDetailBody.kt`) ; chargement ciblé du nœud + ses seuls tomes, aucune action |
| `shared/item/{itemId}` | `ui/sharing/received/SharedItemDetailScreen.kt` | `ui/sharing/received/SharedItemDetailViewModel.kt` | Détail d'un élément partagé. Aucune action : elles ne sont pas désactivées, elles ne sont pas construites |

## Composants réutilisables & UI annexe

| Fichier | Rôle |
|---|---|
| `ui/list/CollectionDrawer.kt` | Drawer de sélection du type de collection actif + stats ; porte « Mes partages » (comptes premium seulement) et « Partagé avec moi » (**tous** les comptes — c'est le propriétaire qui paie le stockage) |
| `ui/list/FilterSheet.kt` | Bottom sheet de filtres/tri de la liste |
| `ui/sharing/CreateShareSheet.kt` | Bottom sheet de création : libellé, collections à exposer avec leurs statuts cochés, nombre de personnes, expiration. Zéro statut coché = collection non partagée |
| `ui/sharing/ShareCodeDialog.kt` | Affichage **unique** du code créé (copier / partager via…) — il n'est jamais réaffiché ; rappelle ce que le code ouvre |
| `ui/sharing/ShareLabelDialog.kt` | Renommage d'un partage, des deux côtés. Le libellé est privé de chaque côté : aucun ne franchit la frontière |
| `ui/sharing/received/JoinShareDialog.kt` | Saisie d'un code + libellé privé. Pas de capitalisation automatique : un code base64url est sensible à la casse |
| `ui/list/SeriesGrid.kt` | Grille des séries (densité grille, défaut manga) |
| `ui/list/SeriesList.kt` | Liste des séries (densité liste) — même contenu, même sélection que la grille |
| `ui/list/CollectionTileBody.kt` | Corps **en tuiles** : grille (une rubrique sans titre) et galerie (rubriques d'écoute). Porte aussi `DisplayModeAction`, la bascule liste/grille/galerie de la barre de recherche |
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
