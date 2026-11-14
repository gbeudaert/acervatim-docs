# acervatim-app — Données locales (Room, DataStore, covers, backup)

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`

## Room (`data/local/db/`)

Base : `AcervatimDatabase.kt` (**version 8**). Migrations manuelles : `Migrations.kt`. Convertisseurs de types : `Converters.kt`.

`MIGRATION_7_8` (branche `feat/manga-recherche-serie`) : `series` + `edition`, `publisherFr`, `mangaId`,
`normalizedPublisher`, `normalizedEdition` (NOT NULL DEFAULT `''`) et index de clé naturelle recréé ;
`records` + `normalized_title` (NOT NULL DEFAULT `''`), rempli dans la migration par un curseur Kotlin
(`normalizeArtistName` n'a pas d'équivalent SQL) puis à chaque écriture. Schéma exporté
`app/schemas/…/8.json` ; test `Migration7To8Test` sur une vraie base v7.

### Entités et DAOs

| Entité | DAO | Contenu |
|---|---|---|
| `RecordEntity.kt` | `RecordDao.kt` | Item de collection (album vinyle ou tome manga) : métadonnées, statut, rating, dernière écoute (`lastPlayedAt`) **et compteur d'écoutes** (`playCount`), lien série/volume |
| `ArtistEntity.kt` + `RecordArtistCrossRef.kt` | `ArtistDao.kt` | Artistes normalisés, relation N-N avec les records |
| `SeriesEntity.kt` | `SeriesDao.kt` | Séries manga (titre, wishlist, extras). Depuis la **v8** : `edition`, `publisherFr`, `mangaId`, et la clé naturelle unique `index_series_natural_key` = (`collectionType`, `normalizedTitle`, `normalizedAuthor`, **`normalizedPublisher`**, **`normalizedEdition`**) — deux éditions d'une même série coexistent ; « Pika éd. » ≡ « Pika » (mots génériques retirés, `domain/model/EditionKey.kt`). Une série d'avant la v8 (sans éditeur ni édition), de même titre et auteur, est reprise et complétée plutôt que dupliquée — **seulement si ses tomes sont compatibles** (aucun tome, ou éditeur et édition des tomes identiques à l'import ou inconnus : `ImportScannedSeriesUseCase.isCompatibleLegacy`) |
| `CollectionEntity.kt` | `CollectionDao.kt` | Collections locales (miroir des collections backend / types) |

### Projections & mappers

- `RecordWithArtistsRow.kt` — row record + artistes joints.
- `RecordQueries.kt` — construit la requête « collection filtrée **et triée** » (`SupportSQLiteQuery`, exécutée en `@RawQuery`) : l'`ORDER BY` dépend du `RecordSort`, ce qu'une `@Query` figée ne sait pas exprimer. Absents en fin de liste dans les deux sens, départage `titre` puis `id`. La recherche texte compare `records.normalized_title` (v8) et `artists.normalized_name` à la saisie normalisée (`normalizeArtistName`) : insensible à la casse **et aux accents**, comme la vue séries et le filtre en mémoire (`domain/model/RecordFilterMatching.kt`).
- `SeriesWithCountsRow.kt` — série + compteurs (tomes possédés / total) + agrégats d'écoute des tomes (`MAX(lastPlayedAt)`, `SUM(playCount)`), qui alimentent les rubriques de la galerie.
- `RecordMapper.kt`, `SeriesMapper.kt` — entity ↔ modèle domaine.
- `data/repository/LocalCollectionResolver.kt` — résolution de la collection locale cible (par type) lors des écritures.

## Préférences (`data/local/settings/`)

`DataStoreSettingsRepository.kt` — implémentation `SettingsRepository` sur Jetpack DataStore : type de collection actif, visibilité des types, mode de liste, densité d'affichage **et tri** par type (`list_mode_<TYPE>` / `display_mode_<TYPE>` / `sort_<TYPE>`, ce dernier au format `CHAMP:SENS`), réglage doublons, thème, etc.

`DataStoreSeriesCatchUpStore.kt` (AD3) — implémentation `SeriesCatchUpStore` sur un DataStore Preferences
**dédié** (`acervatim_series_catch_up`, fourni par `di/SettingsModule.kt`) : une entrée JSON par série
avec la date de dernière vérification, les tomes en attente et la date du dernier « Plus tard ». Pas
de colonne Room : ce n'est pas une donnée de collection (ni synchro, ni backup) et la perdre ne coûte
qu'une vérification de plus. Règles pures dans `domain/model/SeriesCatchUp.kt` (intervalle 7 j,
marge `since` 7 j, première vérification 7 j après la création de la série). L'entrée d'une série est
purgée à sa suppression (`DeleteSeriesUseCase`) ; tout le store est vidé à la déconnexion et à la
suppression du compte (`ClearSeriesCatchUpUseCase`, appelé par `AccountViewModel`).

## Stockage des couvertures (`data/local/storage/`)

`CoverStorageImpl.kt` — implémentation `CoverStorage` : fichiers image sur le stockage interne de l'app (pas de base64 en DB).

## Backup / restauration (`data/backup/`)

Export/import complet de la collection en **ZIP** (JSON + images), déclenché depuis Réglages.

| Fichier | Rôle |
|---|---|
| `BackupSchema.kt` | Schéma JSON versionné du backup |
| `BackupMapper.kt` | Domaine ↔ schéma backup |
| `BackupZip.kt` | Lecture/écriture de l'archive ZIP |
| `BackupCoverIo.kt` | Inclusion/extraction des fichiers de couvertures |
| `BackupGatewayImpl.kt` | Orchestration (implémente `BackupGateway`) |

Use cases associés : `domain/usecase/ExportBackupUseCase.kt`, `domain/usecase/ImportBackupUseCase.kt`.

## À savoir avant de modifier

- Tout changement de schéma Room ⇒ nouvelle migration dans `Migrations.kt` + bump de version dans `AcervatimDatabase.kt`.
- Un changement des entités impacte potentiellement : mappers Room, `BackupSchema` (compat des backups existants), et le mapping sync (`data/remote/mapper/`, voir [reseau-sync-compte.md](reseau-sync-compte.md)).
