# acervatim-app — Réseau, auth, sync, compte

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`
> Contrat serveur : [../backend/api-endpoints.md](../backend/api-endpoints.md)

## Clients HTTP (Retrofit, fournis par `di/NetworkModule.kt`)

| Interface | Rôle |
|---|---|
| `data/remote/api/AcervatimApi.kt` | **Toute l'API métier** `/v1/*` (JWT requis) : me, collections, items, nodes, search, edition-mapping, oauth, subscriptions, invitations. Chaque méthode est annotée avec le § de la spec mobile. |
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

`AccountRepositoryImpl.kt` — implémente `AccountRepository` : `/v1/me`, suppression de compte, statut premium, vérification d'achat Play, rédemption d'invitation. Mapping DTO : `data/remote/mapper/AccountMapper.kt`. UI : `ui/account/AccountScreen.kt`.

## Partages de collection (`data/remote/sharing/`)

`SharingRepositoryImpl.kt` — implémente `domain/repository/SharingRepository` : `POST/GET
/v1/collections/:id/shares`, `DELETE /v1/shares/:id`, `DELETE /v1/shares/:id/members/:memberUserId`.
Côté app c'est **le seul** point d'entrée du partage ; le volet membre (`redeem`, `shares/received`)
n'est pas encore branché (sprint A4).

- **Sans cache.** Un partage n'a pas de vie locale : ni Room, ni DataStore. Seule exception au
  « tout distant » : `remoteCollectionId(type)` lit le `remoteId` de la collection locale
  (`CollectionDao`) — le partage porte sur la collection **serveur**, donc une collection jamais
  synchronisée n'est pas partageable (l'écran le dit au lieu d'échouer).
- **Le code n'existe qu'une fois.** `POST` le renvoie en clair, le serveur n'en garde que le HMAC.
  Il ne transite ni par un log, ni par `SavedStateHandle`, ni par aucun stockage — `SharingViewModel`
  le tient dans un `MutableStateFlow` vidé à la fermeture du dialogue (test dédié).
- **Erreurs traduites dans la couche data** : `ApiException`/`IOException` →
  `domain/model/sharing/SharingException` (`PREMIUM_REQUIRED` / `NETWORK` / `UNKNOWN`), pour que
  l'UI n'ait pas à connaître les statuts HTTP. Le 402 est traité comme un état (premium expiré entre
  l'affichage et l'action), pas comme une panne : la révocation reste ouverte, seule la création
  se ferme.

Mapping DTO : `data/remote/mapper/SharingMapper.kt` · DTOs : `data/remote/dto/ShareDto.kt`.

## Recherche externe (`data/remote/search/`)

`ExternalSearchRepositoryImpl.kt` — implémente `ExternalSearchRepository` : appelle `GET /v1/search` (texte ou code-barres, XOR) et `GET /v1/search/edition-mapping` (énumération d'une édition manga via BnF). Résultats mappés vers `domain/model/ExternalMatch.kt` / `EditionMapping.kt`.

## Sync (`data/remote/sync/`)

| Fichier | Rôle |
|---|---|
| `SyncManagerImpl.kt` | Implémente `SyncRepository` : orchestration de la synchronisation Room ↔ backend (collections/nodes/items) |
| `SyncWorker.kt` | Worker WorkManager (sync en arrière-plan) |
| `SyncStateStore.kt` | État/curseurs de sync persistés |
| `domain/model/sync/` | `SyncState`, `SyncStatus`, `RemoteModels` |

### Ce qui est réellement synchronisé

**Seul le type `vinyl` transite aujourd'hui.** `SyncManagerImpl.push()` saute tout item dont le
`collectionType` n'est pas `VINYL`, et `pull()` ne parcourt que les collections vinyle. Les
**nœuds (séries manga) ne sont ni poussés ni tirés** : `FacadeMapper.seriesToRemote()` /
`remoteToSeries()` existent et sont testés, mais **aucun appelant** dans le code de production. Le
mapping manga est donc prêt, pas actif.

### Refus du serveur : deux codes, deux comportements

| Réponse serveur | `ApiException` | `SyncStatus` | Comportement |
|---|---|---|---|
| `409 technical-limit` | `TechnicalLimit` | `LimitReached` | Sync **partielle** : la création fautive est sautée, l'entité reste `DIRTY` et repartira plus tard. Le pull a quand même lieu. Message **neutre** — ce plafond est un garde-fou anti-emballement, identique pour tous, que payer ne lève pas : surtout pas d'appel à souscrire. |
| `402 payment-required` | `PaymentRequired` | `PremiumRequired` | État **terminal** : la sync fait partie du palier premium (S2). Ni retry, ni `Failure` — et **aucune donnée locale touchée**, c'est l'invariant offline-first. |

Piège historique : jusqu'au sprint A2, l'app ne rattrapait que `402 quota-exceeded`. Le backend a
retiré les quotas (S2) et envoie désormais `409 technical-limit`, qui tombait dans `Conflict` et
n'était **pas** rattrapé — le plafond faisait donc échouer la sync entière. Toute évolution des
codes d'erreur côté serveur doit être répercutée dans `ApiException.from()` **et** dans les `catch`
de `SyncManagerImpl` : un code non rattrapé ne dégrade pas, il casse.

Il n'y a plus de canal quota : `GET /v1/me/quota` a été supprimé côté serveur, et avec lui
`QuotaResponse`/`QuotaInfo`/`AccountInfo.Known.quota` côté app.

### Résolution de conflits — règle effective

La granularité est **l'item entier**, jamais le champ. Il n'y a pas de merge à trois voies.

| Situation au pull | Issue |
|---|---|
| Aucune version locale | Le serveur est écrit |
| Local `SYNCED` (rien en attente) | Le serveur écrase le local |
| Local modifié **et** `updatedAt` local strictement postérieur au serveur | Le local gagne, l'écriture est sautée (il repartira au push suivant) |
| Local modifié mais **pas** plus récent | Le serveur écrase le local — les modifications locales en attente sont **perdues** |

Autrement dit : **last-write-wins sur `updatedAt`, à l'item entier**. Deux clients qui éditent deux
champs *différents* du même item entre deux syncs ne fusionnent pas — le perdant perd tout son
item, pas seulement son champ. C'est invisible tant que l'app Android est le seul client écrivain ;
ça deviendra visible dès qu'un second client (web) écrira. Ne pas confondre avec le **merge partiel
du `userData`**, qui est une règle *serveur* (cf. ci-dessous) et opère à la clé.

Atténuation en place : au pull, `preservedExtras` réinjecte le cliché local avant de reconstruire le
`Record`, ce qui protège les champs sans équivalent serveur (`coverPath`, extras manga sans home).

### `unifiedData` vs `userData` : deux sémantiques d'écriture opposées

| Canal | Sémantique serveur | Conséquence côté app |
|---|---|---|
| `unifiedData` | **Remplacé en bloc**, re-validé par le profil du type | Omettre une clé = la mettre à `null`. `FacadeMapper` reconstruit donc l'objet complet à chaque push. |
| `userData` | **Mergé** clé à clé sur l'existant | Omettre une clé = **conserver** la valeur distante. L'app envoie donc explicitement `null` pour les champs qu'elle possède (sinon un effacement ne se propagerait jamais), et n'écrit **jamais** `note`, qu'elle ne modélise pas. |

Ce `null` explicite est porté dans un `JsonObject` : `explicitNulls = false` (`NetworkModule`) ne
s'applique qu'aux propriétés de classes `@Serializable`, pas au contenu d'un `JsonObject`. Un test
verrouille ce comportement (`FacadeMapperTest`) — il est load-bearing.

### Correspondance champ app ↔ champ serveur (item)

Le tableau champ par champ, item **et** nœud, avec l'état de chacun (✅/⚠️/❌/🔸), est le livrable du
Sprint S1bis et vit dans **[../backend/contrat-app-serveur.md](../backend/contrat-app-serveur.md)**.
Une seule table, tenue à jour dans le sprint qui bouge un champ — ne pas la dupliquer ici.

Ce qu'il faut en retenir côté app :

- Poussé depuis **A1** : `status` (`userData`). Depuis **A1bis** : `rating`, `purchasePrice`,
  `lastPlayedAt` (`userData`) ; `recordingSpeed`, `country` et les extras manga `publisherFr`,
  `pageCount`, `sourceVolumeRange` (`unifiedData`).
- **Volontairement local** : `coverPath` (photo de l'utilisateur — le serveur ne stocke aucune
  image), `extras.edition` et `extras.tomeSynopsis` (pas de home serveur ; `tomeSynopsis` n'est
  **pas** `userData.note`, qui est une note privée saisie, pas un synopsis provider).
- **Modélisé serveur, pas consommé par l'app** (🔸) : `userData.note` d'item, et tout le niveau
  série (jaquette distante, `rating` MAL, `demographic`, note/commentaire, `isWishlist`).

Tout champ resté local est invisible pour un client sans base Room (le web) : c'est la raison d'être
d'A1bis, pas un détail d'implémentation.

### Deux pièges tranchés côté serveur — ne pas les « corriger » côté app

- **`year` → `releaseDate` n'est plus une perte de données.** L'app n'a qu'une année et pousse
  `"%04d-01-01"`. Depuis S1bis le profil `vinyl` implémente `reconcileOnUpdate` et **ne remplace
  jamais une date précise par le 1ᵉʳ janvier de la même année** ; un changement d'année passe
  normalement. Le garde-fou est côté serveur pour valoir pour tout client — inutile de complexifier
  le modèle app.
- **`barcode` ne vit que dans `sources[]`.** Il a été retiré de `VinylItemSchema` en S1bis, et une
  clé `barcode` envoyée dans `unifiedData` est **filtrée silencieusement** (pas de 400). L'app
  faisait déjà la bonne chose : ne rien changer, et surtout ne pas l'ajouter à `buildVinylUnified()`.

## DTOs & mappers (`data/remote/dto/`, `data/remote/mapper/`)

- DTOs par domaine : `AuthDto`, `AccountDto`, `CollectionDto`, `ItemDto`, `NodeDto`, `SearchDto`, `SourceDto`, `UnifiedDataDto`, `PageDto` (pagination cursor), `ProblemDetailsDto`.
- `RemoteDtoMapper.kt` — DTO ↔ domaine (items/nodes/collections).
- `FacadeMapper.kt` — façade de mapping entre `unifiedData`/`userData` serveur et modèles locaux, plus le **cliché local** (`localExtras`).

## Points d'attention

- Toute évolution du contrat backend ⇒ modifier `AcervatimApi.kt` + DTOs + mappers, et vérifier `SyncManagerImpl`.
- Le backend expose `unifiedData` en JSON typé par profil (vinyl/manga) : le mapping app est centralisé dans `UnifiedDataDto.kt` + `FacadeMapper.kt`.
- **`localExtras` n'est pas un stockage** : c'est un `JsonObject` reconstruit à la volée depuis le `Record` en base à chaque pull (`preservedExtras`). Il sert de **repli de lecture** — `FacadeMapper` lit `serveur ?: cliché local`. En retirer un champ fait silencieusement retomber sur le défaut tous les items synchronisés avant que ce champ ne parte au serveur. Y ajouter un champ ne « stocke » rien de plus.
