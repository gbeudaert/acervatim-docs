# acervatim-app — Réseau, auth, sync, compte

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`
> Contrat serveur : [../backend/api-endpoints.md](../backend/api-endpoints.md)

## Clients HTTP (Retrofit, fournis par `di/NetworkModule.kt`)

| Interface | Rôle |
|---|---|
| `data/remote/api/AcervatimApi.kt` | **Toute l'API métier** `/v1/*` (JWT requis) : me/quota, collections, items, nodes, search, edition-mapping, oauth, subscriptions, invitations. Chaque méthode est annotée avec le § de la spec mobile. |
| `data/remote/api/AuthApi.kt` | `/v1/auth/google` (échange idToken Google → JWT backend), hors intercepteur auth |

## Intercepteurs & erreurs

| Fichier | Rôle |
|---|---|
| `data/remote/interceptor/AuthInterceptor.kt` | Ajoute `Authorization: Bearer <JWT>` ; gère le refresh de session |
| `data/remote/interceptor/ErrorInterceptor.kt` | Convertit les non-2xx (Problem Details RFC 9457) en `ApiException` |
| `data/remote/error/ApiException.kt` | Exception typée côté app (status, type, detail, requestId) |
| `data/remote/dto/ProblemDetailsDto.kt` | DTO du payload d'erreur |

## Auth Google (`data/remote/auth/`)

| Fichier | Rôle |
|---|---|
| `GoogleSignInClient.kt` | Lance le flow Google Sign-In (Credential Manager) → idToken |
| `AuthRepositoryImpl.kt` | Implémente `domain/repository/AuthRepository` : login/logout, état de session (`domain/model/auth/AuthState.kt`) |
| `TokenStore.kt` | Persistance sécurisée du JWT backend |
| `SessionContracts.kt` | Contrats internes de session |

## Compte (`data/remote/account/`)

`AccountRepositoryImpl.kt` — implémente `AccountRepository` : `/v1/me`, quota, suppression de compte, statut premium, vérification d'achat Play, rédemption d'invitation. Mapping DTO : `data/remote/mapper/AccountMapper.kt`. UI : `ui/account/AccountScreen.kt`.

## Recherche externe (`data/remote/search/`)

`ExternalSearchRepositoryImpl.kt` — implémente `ExternalSearchRepository` : appelle `GET /v1/search` (texte ou code-barres, XOR) et `GET /v1/search/edition-mapping` (énumération d'une édition manga via BnF). Résultats mappés vers `domain/model/ExternalMatch.kt` / `EditionMapping.kt`.

## Sync (`data/remote/sync/`)

| Fichier | Rôle |
|---|---|
| `SyncManagerImpl.kt` | Implémente `SyncRepository` : orchestration de la synchronisation Room ↔ backend (collections/nodes/items) |
| `SyncWorker.kt` | Worker WorkManager (sync en arrière-plan) |
| `SyncStateStore.kt` | État/curseurs de sync persistés |
| `domain/model/sync/` | `SyncState`, `SyncStatus`, `RemoteModels` |

## DTOs & mappers (`data/remote/dto/`, `data/remote/mapper/`)

- DTOs par domaine : `AuthDto`, `AccountDto`, `CollectionDto`, `ItemDto`, `NodeDto`, `SearchDto`, `SourceDto`, `UnifiedDataDto`, `PageDto` (pagination cursor), `ProblemDetailsDto`.
- `RemoteDtoMapper.kt` — DTO ↔ domaine (items/nodes/collections).
- `FacadeMapper.kt` — façade de mapping entre `unifiedData` serveur et modèles locaux.

## Points d'attention

- Toute évolution du contrat backend ⇒ modifier `AcervatimApi.kt` + DTOs + mappers, et vérifier `SyncManagerImpl`.
- Le backend expose `unifiedData` en JSON typé par profil (vinyl/manga) : le mapping app est centralisé dans `UnifiedDataDto.kt` + `FacadeMapper.kt`.
