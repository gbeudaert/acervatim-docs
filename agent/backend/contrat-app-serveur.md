# Contrat app ↔ serveur — tableau de correspondance des champs

> Remonter : [INDEX.md](INDEX.md) · Établi au **Sprint S1bis** (2026-08-20)
> Sources : `acervatim-app` (`domain/model/Record.kt`, `Series.kt`, `data/remote/mapper/FacadeMapper.kt`)
> et `acervatim-backend` (`collections/types/*.ts`, `items/dto/item-user-data.schema.ts`,
> `nodes/dto/node-user-data.schema.ts`).

## À quoi sert ce document

Le serveur doit être le **miroir intégral** d'une collection : un client sans stockage local (une
future version web) ne peut afficher que ce qu'il détient. Ce tableau est la réponse, champ par
champ, à « est-ce que le serveur le sait ? ». Tout champ **non** synchronisé y porte une raison
explicite — il n'y a pas de zone grise, et c'est ce qui évite de refaire l'audit à chaque sprint.

Légende : ✅ synchronisé · ⚠️ synchronisé avec une réserve · ❌ volontairement local ·
🔸 modélisé côté serveur mais **pas encore consommé** par l'app.

## Item (`Record` app ↔ `Item` serveur)

| Champ app | Home serveur | État |
|---|---|---|
| `title` | `unifiedData.title` | ✅ |
| `artists: List<Artist>` | `unifiedData.creators: string[]` | ⚠️ seuls les **noms** transitent ; `Artist.id` est un identifiant local régénéré au pull |
| `genre: String?` | `unifiedData.genre: string[]` | ⚠️ mono côté app, **pluriel côté serveur** (voir décisions) |
| `year: Int?` | `unifiedData.releaseDate` (date complète) | ⚠️ l'app n'a que l'année ; le serveur refuse la dégradation (voir décisions) |
| `label` | `unifiedData.label` | ✅ |
| `format: AcervatimFormat?` | `unifiedData.format` (`LP`/`Single`/`Shellac`/`Other`) | ✅ mappé par code |
| `recordingSpeed` | `unifiedData.recordingSpeed` (enum `RPM_*`) | ✅ |
| `country` | `unifiedData.country` | ✅ |
| `coverRemoteUrl` | `unifiedData.coverUrl` | ✅ |
| `volumeNumber` | `Item.volume` (colonne) | ✅ |
| `seriesId` | `Item.nodeId` (FK) | ✅ |
| `rating: Int?` | `userData.rating` (1–5) | ✅ l'app reborne à 1..5 avant push (le schéma rejette 0) |
| `purchasePrice` | `userData.purchasePrice` | ✅ |
| `lastPlayedAt` | `userData.lastPlayedAt` (ISO-8601) | ✅ |
| `status: ItemStatus` | `userData.status` | ✅ depuis S1 ; absent ⇒ `OWNED` |
| `barcode` | `sources[]` `provider: 'barcode'` | ✅ **seule** source de vérité depuis S1bis |
| `isbn` | `sources[]` `provider: 'isbn'` | ✅ |
| `discogsId` | `sources[]` `provider: 'discogs'` | ✅ |
| `extras.publisherFr` | `unifiedData.publisherFr` (manga) | ✅ |
| `extras.pageCount` | `unifiedData.pageCount` (manga) | ✅ |
| `extras.sourceVolumeRange` | `unifiedData.sourceVolumeRange` (manga) | ✅ |
| `extras.demographic` | `MangaSerieSchema.demographic` — niveau **série** | ⚠️ l'app le range par tome, le serveur par série : pas de home au niveau item (redondant par tome) |
| `extras.originalAuthor` | `MangaSerieSchema.author` — niveau **série** | ⚠️ idem |
| `extras.edition` | aucun | ❌ notion d'édition FR propre à l'app (regroupement d'affichage), sans équivalent dans le modèle serveur |
| `extras.tomeSynopsis` | aucun (≠ `userData.note`, voir ci-dessous) | ❌ `userData.note` est une note **privée saisie par l'utilisateur**, pas un synopsis ; les confondre mélangerait curé et subjectif |
| `coverPath` | aucun | ❌ **photo prise par l'utilisateur**. Le serveur ne stocke aucune image (invariant privacy, cf. [securite-et-privacy.md](securite-et-privacy.md)) — voir « Blocage web » |
| — | `userData.note` (note libre par item) | 🔸 existe côté serveur, **l'app ne l'expose pas**. À reprendre côté app, pas à retirer côté serveur |

## Nœud série (`Series` app ↔ `CollectionNode` serveur)

| Champ app | Home serveur | État |
|---|---|---|
| `title` | `unifiedData.title` | ✅ |
| `originalAuthor` | `unifiedData.author` | ✅ |
| `publicationStatus` | `unifiedData.status` (`ongoing`/`finished`/`hiatus`/`unknown`) | ✅ mappé depuis le libellé FR |
| `totalVolumes` | `unifiedData.totalCount` | ✅ |
| `synopsis` | `unifiedData.synopsis` | ✅ |
| `coverPath` | aucun | ❌ photo locale, même raison que pour l'item |
| — | `unifiedData.coverUrl` (jaquette série du provider) | 🔸 `Series` n'a pas de champ équivalent : la jaquette série distante n'est pas restituée au pull |
| — | `unifiedData.rating` (moyenne MAL) | 🔸 lu par `NodeDto`, absent de `Series` |
| — | `unifiedData.demographic` | 🔸 l'app le stocke dans les extras du **tome** (voir tableau item) |
| — | `userData.note` / `userData.comment` | 🔸 `NodeUserDataDto` existe côté app, mais `Series` ne porte ni note ni commentaire : le canal est ouvert et inutilisé |
| — | `isWishlist` | 🔸 `RemoteNode.isWishlist` existe ; `seriesToRemote()` ne le renseigne pas (toujours `false` au push) |

## Décisions prises en S1bis

**`barcode` — une seule source de vérité : `sources[]`.** Le code-barres vivait à deux endroits
(`VinylItemSchema.barcode` **et** `sources[]` avec `provider: 'barcode'`). C'est un identifiant
externe : sa place est `sources[]`, où l'app l'écrivait déjà et d'où elle le relisait. `barcode` est
donc **retiré de `unifiedData`**, mais **silencieusement** (clé filtrée avant validation) plutôt que
rejeté : `unifiedData` étant remplacé en bloc au PATCH, un client qui relit un item puis le repousse
tel quel renverrait la clé stockée et se prendrait un 400 sur une donnée qu'il n'a pas produite.
Aucune perte : rien côté serveur n'alimentait ce champ.

**`genre` — le serveur reste pluriel.** `genre: string[]` est déjà correct ; l'app n'en gère qu'un
et pousse un tableau à un élément (`genres.firstOrNull()` à la lecture). Aucune perte côté serveur,
et un futur client multi-genres n'aura rien à migrer. Passer l'app au pluriel est un chantier large
(modèle, formulaire, UI, mapper) sans bénéfice immédiat.

**`year` / `releaseDate` — garde-fou côté serveur.** L'app ne stocke qu'une année et repousse
`"%04d-01-01"` ; une date complète venant d'un provider (`released` chez Discogs, ex. `1959-08-17`)
était donc écrasée en `1959-01-01` au push suivant. Le profil `vinyl` implémente désormais
`reconcileOnUpdate` : **on ne remplace jamais une date précise par le 1ᵉʳ janvier de la même
année**. Un changement d'année reste un vrai changement et passe. Le correctif est côté serveur
pour valoir **pour tout client**, y compris un futur web, sans attendre une refonte du modèle app.

> `reconcileOnUpdate` est un point d'extension du profil de type
> (`collections/types/common.ts`) : il donne le dernier mot sur le `unifiedData` d'un PATCH au vu de
> l'existant. C'est la seule exception à la sémantique « `unifiedData` remplacé en bloc » — toute
> autre exception doit passer par ce même point, pas par un cas particulier dans le service.

## Blocage à trancher avant le chantier web

`coverPath` est une photo prise par l'utilisateur, stockée sur l'appareil. Le backend ne stocke
**aucune image** — invariant privacy assumé et écrit dans la doc publique
([../../developpeur/backend.md](../../developpeur/backend.md)). Conséquence : un client web ne
pourra **jamais** afficher ces jaquettes-là. Celles issues des providers (`coverUrl`) passent, ce
sont des URLs.

Ce n'est pas un oubli à corriger : c'est un arbitrage produit entre un invariant de confidentialité
et la faisabilité du web. Trois issues possibles — stockage d'images côté serveur (avec le coût
RGPD et d'hébergement que ça implique), web dégradé sans jaquettes locales, ou jaquettes servies par
l'app. **À trancher avant de lancer le chantier web**, pas pendant.

## Comment maintenir ce tableau

Il se périme dès qu'un champ bouge d'un côté ou de l'autre. Le réflexe : tout ajout de champ à
`Record`, `Series`, `ItemUserDataSchema`, `NodeUserDataSchema` ou à un schéma `unifiedData` de
`collections/types/` se répercute ici **dans le même sprint**, avec son état (✅/⚠️/❌/🔸). Un champ
sans ligne dans ce tableau est un bug de documentation.
