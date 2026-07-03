# acervatim-app — Carte du dépôt Android

> Remonter : [../INDEX.md](../INDEX.md) · Règles de code : `acervatim-app/CLAUDE.md`

## Identité

- **Package** : `app.acervatim` · **Nom** : Acervatim
- **Stack** : Kotlin (JVM 17), Jetpack Compose + Material 3, Hilt, Room, Retrofit + kotlinx-serialization, DataStore, CameraX + ML Kit Barcode (modèle embarqué offline), WorkManager (sync).
- **SDK** : minSdk 24, compile/target 34. Build : Gradle Kotlin DSL + version catalog `gradle/libs.versions.toml`.
- **Modèle** : offline-first. Room est la vérité locale ; le backend est optionnel (mode connecté = sync + recherche externe + premium).

## Racine des sources

Tout le code applicatif vit sous `acervatim-app/app/src/main/java/app/acervatim/`.
Dans les fiches app, les chemins courts (ex. `ui/scan/BarcodeScanScreen.kt`) sont relatifs à cette racine.

```
app/acervatim/
├── AcervatimApplication.kt     — @HiltAndroidApp
├── MainActivity.kt             — unique Activity, host Compose + NavGraph
├── ui/          — écrans Compose, ViewModels, navigation, thème        → ecrans-et-navigation.md
├── domain/      — modèles métier, interfaces repository, use cases    → architecture.md
├── data/        — Room, DataStore, Retrofit, sync, backup, covers     → donnees-locales.md · reseau-sync-compte.md
└── di/          — modules Hilt (Database, Network, Repository, Settings, Storage)
```

## Fiches détaillées

| Fiche | Contenu |
|---|---|
| [architecture.md](architecture.md) | Couches, DI, catalogue des use cases et repositories |
| [ecrans-et-navigation.md](ecrans-et-navigation.md) | Routes de navigation ↔ écrans ↔ ViewModels |
| [donnees-locales.md](donnees-locales.md) | Room (entités/DAO/migrations), DataStore, stockage covers, backup ZIP |
| [reseau-sync-compte.md](reseau-sync-compte.md) | Client API, auth Google, tokens, sync, compte |
| [catalogue-fonctionnalites.md](catalogue-fonctionnalites.md) | Feature → fichiers impliqués (point d'entrée recommandé) |

## Tests

- Unit tests JVM : `acervatim-app/app/src/test/` (JUnit + Turbine + coroutines-test). Les ViewModels y sont testés (ex. `BarcodeScanViewModelTest`).
- Instrumentation : `acervatim-app/app/src/androidTest/` (Compose UI testing).

## Vocabulaire du code (pièges de nommage)

- **`Record`** = un item de collection (album vinyle **ou** tome manga), pas seulement un disque. Cf. `domain/model/Record.kt`.
- **`Series`** = série manga (nœud parent des tomes). Cf. `domain/model/Series.kt`.
- **`CollectionType`** = type de collection (vinyl, manga, …), avec visibilité gérable par l'utilisateur.
- Le terme « manga » dans `ui/manga/` désigne les écrans de **tome** ; les écrans de **série** sont dans `ui/series/`.
