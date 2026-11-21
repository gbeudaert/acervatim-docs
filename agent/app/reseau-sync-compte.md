# acervatim-app — Réseau, auth, sync, compte

> Remonter : [INDEX.md](INDEX.md) · Chemins relatifs à `acervatim-app/app/src/main/java/app/acervatim/`
> Contrat serveur : [../backend/api-endpoints.md](../backend/api-endpoints.md)

## Clients HTTP (Retrofit, fournis par `di/NetworkModule.kt`)

| Interface | Rôle |
|---|---|
| `data/remote/api/AcervatimApi.kt` | **Toute l'API métier** `/v1/*` (JWT requis) : me, collections, items, nodes, search, edition-mapping, search/editions, import-jobs, series-updates, oauth, subscriptions, invitations. Chaque méthode est annotée avec le § de la spec mobile. |
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

Deux repositories, deux responsabilités qu'il ne faut pas confondre :

| Repository | Rôle | Routes |
|---|---|---|
| `SharingRepositoryImpl` | **Gérer** les partages, dans les deux sens | `POST/GET /v1/shares`, `PATCH/DELETE /v1/shares/:id`, `DELETE /v1/shares/:id/members/:memberUserId`, `POST /v1/shares/redeem`, `GET /v1/shares/received`, `PATCH /v1/shares/received/:shareId` |
| `SharedCollectionRepositoryImpl` | **Lire** une collection reçue | Aucune route dédiée : les routes de lecture normales (`GET /v1/collections/:id/items`, `GET /v1/items/:id`), dont l'autorisation est étendue serveur-side |

Un partage porte **N collections**, chacune avec son jeu de statuts (`OWNED`/`WISHLIST`/`IGNORED`) :
il n'appartient plus à une collection, d'où l'ancrage sur `/v1/shares` et la disparition du champ
`scope`. Les anciennes portées ne survivent que comme raccourcis d'affichage (`ShareScopePreset`).

- **Sans cache, des deux côtés.** Un partage n'a pas de vie locale : ni Room, ni DataStore. Seule
  exception au « tout distant » : `sharableCollections()` lit les `remoteId` des collections locales
  (`CollectionDao`) — le partage porte sur la collection **serveur**, donc une collection jamais
  synchronisée n'est pas partageable (elle n'apparaît pas dans le sélecteur).
- **Les collections reçues ne sont jamais persistées.** Elles ne sont pas les données de
  l'utilisateur : les mettre en Room les ferait entrer dans son export et sa sauvegarde ZIP, et
  laisserait une copie fantôme après révocation. Un test verrouille l'absence de dépendance à un
  `Dao`/`Database`/`DataStore` dans `SharedCollectionRepositoryImpl`.
- **Le code n'existe qu'une fois.** `POST /v1/shares` le renvoie en clair, le serveur n'en garde que
  le HMAC. Il ne transite ni par un log, ni par `SavedStateHandle`, ni par aucun stockage —
  `SharingViewModel` le tient dans un `MutableStateFlow` vidé à la fermeture du dialogue (test dédié).
- **Le libellé est privé de chaque côté.** Le propriétaire et le membre nomment le partage chacun
  pour soi ; aucun des deux ne franchit la frontière. Pour l'**effacer**, on envoie la chaîne vide et
  non `null` : le `Json` de l'app est en `explicitNulls = false`, un `null` ne serait pas sérialisé
  et le schéma strict du serveur (`label` requis) refuserait le corps vide.
- **Erreurs traduites dans la couche data** : `ApiException`/`IOException` →
  `domain/model/sharing/SharingException`, pour que l'UI n'ait pas à connaître les statuts HTTP.
  Le `404` n'a pas le même sens des deux côtés — `GONE` sur une action de propriétaire, mais
  `CODE_INVALID` sur un `redeem`, où le serveur confond volontairement cinq refus pour ne pas offrir
  d'oracle. Le `402` est un **état**, pas une panne : côté propriétaire le premium a expiré (la
  révocation reste ouverte, seule la création se ferme) ; côté membre le **propriétaire** a perdu son
  premium, ce qui suspend l'accès sans le supprimer.

Mapping DTO : `data/remote/mapper/SharingMapper.kt` · DTOs : `data/remote/dto/ShareDto.kt` ·
Modèles : `domain/model/sharing/SharingModels.kt` (gestion) et `SharedBrowsing.kt` (lecture membre).

## Recherche externe (`data/remote/search/`)

`ExternalSearchRepositoryImpl.kt` — implémente `ExternalSearchRepository` : appelle `GET /v1/search` (texte ou code-barres, XOR) et `GET /v1/search/edition-mapping` (énumération d'une édition manga via BnF). Résultats mappés vers `domain/model/ExternalMatch.kt` / `EditionMapping.kt`.

Ajouts AD2 (branche `feat/manga-recherche-serie`), tous mappés vers des modèles domaine neutres :

| Méthode | Endpoint | Modèle |
|---|---|---|
| `searchMangaSeries(query)` | `GET /v1/search?type=manga&q=` | `SeriesCandidate` (`domain/model/SeriesSearch.kt`) ; `mangaId` gardé seulement s'il est un UUID — sinon `null`, ce qui évite le `400` de `search/editions` sur un candidat venu du repli MAL |
| `listEditions(mangaId, title?)` | `GET /v1/search/editions` | `SeriesEditionOption` ; `publisher` peut être `null` (affiché « Éditeur inconnu ») |
| `editionMapping(titleFr, edition, publisher?, mangaId?, since?)` | `GET /v1/search/edition-mapping` | `EditionMapping` (+ `publisher` retenu, `complete` — vrai si absent —, `EditionTome.noticeCreatedAt`) ; `since` envoyé en `AAAAMMJJ` |
| `startImportJob(titleFr, edition, publisher?, mangaId?)` | `POST /v1/search/import-jobs` | `ImportJob` |
| `seriesUpdates()` | `GET /v1/series-updates` | `SeriesUpdate` ; hors premium (`402`), échec `ExternalSearchFailure.UNKNOWN`, ignoré par le rattrapage. Lu via `domain/usecase/SeriesUpdatesCache.kt` : un téléchargement au plus toutes les **10 min par compte** (le job serveur est hebdomadaire) |

Le scan transmet désormais `ExternalMatch.publisherFr` et `ExternalMatch.mangaId` (lu dans
`metadata.pivot`) à `edition-mapping` et `import-jobs` : sans éditeur, le serveur retient l'éditeur
dominant.

## Sync (`data/remote/sync/`)

| Fichier | Rôle |
|---|---|
| `SyncManagerImpl.kt` | Implémente `SyncRepository` : orchestration de la synchronisation Room ↔ backend (collections/nodes/items) |
| `SyncWorker.kt` | Worker WorkManager (sync en arrière-plan) |
| `SyncStateStore.kt` | État/curseurs de sync persistés |
| `domain/model/sync/` | `SyncState`, `SyncStatus`, `RemoteModels` |

### Ce qui est réellement synchronisé

**Vinyle et manga**, depuis le serveur déployé le 2026-08-24 et l'app du même jour. Le filtre `VINYL` du push et du pull est
levé, et les **nœuds (séries) transitent** : `FacadeMapper.seriesToRemote()` / `remoteToSeries()`,
qui existaient sans appelant, sont désormais câblés.

**L'ordre est structurant, au push comme au pull** : collections → **séries** → items. Un tome se
rattache à sa série par l'`id serveur` du nœud, qui n'existe qu'une fois la série poussée ; et un
tome tiré avant sa série n'aurait aucune série locale où se rattacher. Un tome dont la série n'est
pas encore synchronisée est **reporté** (compté en `skipped`) plutôt que poussé détaché — le lien
serait sinon perdu sans que rien ne le signale.

Trois pièges du contrat, tous verrouillés par `SyncRequestContractTest` :

- **`node` et `nodeId` sont exclusifs.** `nodeId` (id serveur) est le chemin de la synchronisation ;
  `node` (référence provider, upsert côté serveur) celui des imports enrichis MAL. Les envoyer
  ensemble est un `400`.
- **`explicitNulls = false` rend « effacer » et « ne pas toucher » indistinguables.** Sur le `PATCH`
  d'un tome, `nodeId` et `volume` sont donc typés `JsonElement` et passent par `nodeRef()` /
  `volumeRef()` : `JsonNull` s'écrit `null`, un `null` Kotlin serait omis et le serveur garderait
  l'ancienne valeur. Sans ça, détacher un tome de sa série resterait local.
- **`isWishlist` n'est jamais envoyé.** `PATCH /nodes/:id` **supprime** le nœud quand il reçoit
  `isWishlist: false` sur un nœud sans tome. Or `RemoteNode.isWishlist` vaut `false` par défaut et
  l'app ne modélise pas ce drapeau : le transmettre aurait purgé toute série dont les tomes ne sont
  pas encore poussés, pour la recréer à la sync suivante — en boucle.

Deux points d'attention côté pull :

- **Un tome n'a pas de `creators`.** `MangaItemSchema` est `.strict()` et n'en porte pas : l'auteur
  d'un tome est celui de sa **série**, exactement comme le fait la saisie locale
  (`AddEditMangaViewModel`), avec le même repli `"Inconnu"`. Sans ce repli, tout tome tiré du serveur
  tomberait sur l'invariant « au moins un artiste ».
- **Dédup des séries par clé naturelle** (type + titre + auteur normalisés) en plus du `remoteId`,
  comme les items le font par code-barres/ISBN : une série créée hors-ligne existe déjà localement
  sous un autre id, et l'upsert buterait sinon sur `index_series_natural_key`.

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
- Depuis **AD2** (branche `feat/manga-recherche-serie`) : `edition` sur l'item
  (`unifiedData.edition`, `MangaItemSchema`) et `edition` / `publisherFr` / `mangaId` sur le nœud
  série, aller et retour (`FacadeMapper.seriesToRemote()` / `remoteToSeries()`, `NodeDto`,
  `UnifiedDataDto`). **Ordre de déploiement** : `MangaItemSchema` est `.strict()` — un serveur sans
  SD2 rejette en `400` tout tome manga portant `edition` : le backend part **avant** l'app.
- **Volontairement local** : `coverPath` (photo de l'utilisateur — le serveur ne stocke aucune
  image) et `extras.tomeSynopsis` (pas de home serveur ; `tomeSynopsis` n'est **pas**
  `userData.note`, qui est une note privée saisie, pas un synopsis provider).
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
