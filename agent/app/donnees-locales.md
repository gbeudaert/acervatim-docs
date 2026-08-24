# acervatim-app — Données locales (Room, DataStore, covers, backup)

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`

## Room (`data/local/db/`)

Base : `AcervatimDatabase.kt`. Migrations manuelles : `Migrations.kt`. Convertisseurs de types : `Converters.kt`.

### Entités et DAOs

| Entité | DAO | Contenu |
|---|---|---|
| `RecordEntity.kt` | `RecordDao.kt` | Item de collection (album vinyle ou tome manga) : métadonnées, statut, rating, dernière écoute (`lastPlayedAt`) **et compteur d'écoutes** (`playCount`), lien série/volume |
| `ArtistEntity.kt` + `RecordArtistCrossRef.kt` | `ArtistDao.kt` | Artistes normalisés, relation N-N avec les records |
| `SeriesEntity.kt` | `SeriesDao.kt` | Séries manga (titre, wishlist, extras) |
| `CollectionEntity.kt` | `CollectionDao.kt` | Collections locales (miroir des collections backend / types) |

### Projections & mappers

- `RecordWithArtistsRow.kt` — row record + artistes joints.
- `RecordQueries.kt` — construit la requête « collection filtrée **et triée** » (`SupportSQLiteQuery`, exécutée en `@RawQuery`) : l'`ORDER BY` dépend du `RecordSort`, ce qu'une `@Query` figée ne sait pas exprimer. Absents en fin de liste dans les deux sens, départage `titre` puis `id`.
- `SeriesWithCountsRow.kt` — série + compteurs (tomes possédés / total) + agrégats d'écoute des tomes (`MAX(lastPlayedAt)`, `SUM(playCount)`), qui alimentent les rubriques de la galerie.
- `RecordMapper.kt`, `SeriesMapper.kt` — entity ↔ modèle domaine.
- `data/repository/LocalCollectionResolver.kt` — résolution de la collection locale cible (par type) lors des écritures.

## Préférences (`data/local/settings/`)

`DataStoreSettingsRepository.kt` — implémentation `SettingsRepository` sur Jetpack DataStore : type de collection actif, visibilité des types, mode de liste, densité d'affichage **et tri** par type (`list_mode_<TYPE>` / `display_mode_<TYPE>` / `sort_<TYPE>`, ce dernier au format `CHAMP:SENS`), réglage doublons, thème, etc.

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
