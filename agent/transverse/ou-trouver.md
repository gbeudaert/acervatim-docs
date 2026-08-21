# « Où trouver X ? » — table de lookup rapide

> Remonter : [../INDEX.md](../INDEX.md) · Point d'entrée le plus rapide pour un agent : chercher son intention ici, ouvrir directement le fichier cible.
> Préfixes : `app:` = `acervatim-app/app/src/main/java/app/acervatim/` · `be:` = `acervatim-backend/src/`

## Par intention

| Je veux… | Fichier(s) cible(s) | Fiche détaillée |
|---|---|---|
| Ajouter/modifier un endpoint HTTP | `be:<feature>/<feature>.controller.ts` + `.service.ts` + `dto/` | [backend/api-endpoints.md](../backend/api-endpoints.md) |
| Modifier le schéma DB serveur | `acervatim-backend/prisma/schema.prisma` (+ migration via dev.ps1) | [backend/modele-de-donnees.md](../backend/modele-de-donnees.md) |
| Modifier le schéma DB app | `app:data/local/db/*Entity.kt` + `Migrations.kt` + `AcervatimDatabase.kt` | [app/donnees-locales.md](../app/donnees-locales.md) |
| Ajouter un écran Android | composable dans `app:ui/<domaine>/` + route dans `app:ui/navigation/AcervatimDestinations.kt` + `AcervatimNavGraph.kt` | [app/ecrans-et-navigation.md](../app/ecrans-et-navigation.md) |
| Changer le contrat app ↔ backend | `app:data/remote/api/AcervatimApi.kt` + DTOs/mappers ; côté serveur controller+DTO Zod | [app/reseau-sync-compte.md](../app/reseau-sync-compte.md) |
| Toucher à la logique métier app | `app:domain/usecase/*UseCase.kt` | [app/architecture.md](../app/architecture.md) |
| Ajouter un type de collection | recette complète | [backend/modules.md](../backend/modules.md) § recette |
| Ajouter une source externe / un adapter | `be:oauth/providers/*.adapter.ts` + `be:common/sources/` | [transverse/sources-externes.md](sources-externes.md) |
| Comprendre un flux complet (scan, sync, premium…) | — | [transverse/flux-metier.md](flux-metier.md) |
| Toucher auth / JWT / identité | `be:auth/`, `be:common/guards/jwt-auth.guard.ts` ; `app:data/remote/auth/` | [backend/securite-et-privacy.md](../backend/securite-et-privacy.md) |
| Lancer le backend / une migration / les tests | `acervatim-backend/scripts/dev.ps1` | [backend/operations-dev.md](../backend/operations-dev.md) |
| Builder/tester l'app | `.\gradlew.bat assembleDebug / test / lint` | [conventions.md](../conventions.md) |
| Mettre à jour la doc publique (humains) | dépôt `acervatim-docs/` (page concernée) | [conventions.md](../conventions.md) § où documenter |

## Par mot-clé

| Mot-clé | Emplacement |
|---|---|
| JWT, Bearer, guard | `be:common/guards/jwt-auth.guard.ts` · `app:data/remote/interceptor/AuthInterceptor.kt` |
| Google Sign-In | `app:data/remote/auth/GoogleSignInClient.kt` · `be:auth/providers/google.provider.ts` |
| subHash, pepper, HMAC | `be:common/crypto/hash.service.ts` · `be:auth/identity-resolver.service.ts` |
| AES, chiffrement tokens | `be:common/crypto/aes.service.ts` |
| Problem Details, erreurs API | `be:common/filters/problem-details.filter.ts` · `app:data/remote/interceptor/ErrorInterceptor.kt` |
| Pagination, cursor | `be:common/pagination/paginate.ts` · `app:data/remote/dto/PageDto.kt` |
| Filtres `[in]` / `[all]` | `be:common/filters/parse-filters.ts` |
| Plafond technique d'items | `be:common/limits/limits.service.ts` (anti-emballement, pas une limite d'offre) |
| Gate premium sur la sync | `be:premium/premium.guard.ts` (écriture) · `be:premium/collection-premium.guard.ts` (lecture, statut du **propriétaire**) |
| Throttle appels sortants (limiters BullMQ) | `be:src/oauth/providers/*.processor.ts` · `be:src/common/sources/googlebooks/googlebooks.processor.ts` (cf. [scan-asynchrone-bullmq.md](../backend/scan-asynchrone-bullmq.md)) |
| Cache API externe | `be:common/cache/api-cache.service.ts` |
| Audit log | `be:common/audit/audit-log.service.ts` |
| unifiedData, sources, userData | `be:collections/types/` (schémas) · `be:items/dto/item-user-data.schema.ts` · `be:nodes/dto/node-user-data.schema.ts` · [backend/modele-de-donnees.md](../backend/modele-de-donnees.md) |
| « le serveur connaît-il ce champ ? » | [backend/contrat-app-serveur.md](../backend/contrat-app-serveur.md) (tableau app ↔ serveur, item et nœud) |
| Profil vinyl / manga | `be:collections/types/vinyl.ts` / `manga.ts` / `registry.ts` |
| Série, node, hiérarchie | `be:nodes/` · `app:ui/series/` · `app:domain/model/Series.kt` |
| Tome, volume | `be:items/` (unique userId+nodeId+volume) · `app:ui/manga/` |
| Scan, barcode, ML Kit | `app:ui/scan/` · `app:domain/usecase/LookupByBarcodeUseCase.kt` |
| ISBN, BnF, UNIMARC, édition | `be:common/sources/bnf/` · `be:search/` |
| MangaDex, identité manga, jaquettes/tome | `be:common/sources/mangadex/` (identify + covers) · `be:common/sources/manga-matching.ts` (rapprochement partagé + `isSpecialArtEdition`) |
| Discogs | `be:oauth/providers/discogs.adapter.ts` (+ `oauth1.ts`) |
| MAL, MyAnimeList (identité repli + pivot) | `be:oauth/providers/mal.adapter.ts` (+ `pkce.ts`) — `searchByBarcode` orchestre BnF → MangaDex → MAL |
| TMDB | `be:oauth/providers/tmdb.adapter.ts` |
| Recherche externe | `be:search/search.service.ts` · `app:data/remote/search/` |
| Sync, WorkManager | `app:data/remote/sync/` (SyncManagerImpl, SyncWorker) |
| Backup, export/import ZIP | `app:data/backup/` |
| Couvertures, images | `app:data/local/storage/CoverStorageImpl.kt` · `app:ui/components/CoverPicker.kt` |
| Premium, abonnement, Play Billing | `be:premium/` · `be:subscriptions/` · `app:ui/account/` |
| RTDN, webhook Google Play | `be:subscriptions/pubsub-verifier.service.ts` + controller |
| Invitations, codes | `be:invitations/` |
| Wishlist | `app:domain/usecase/ToggleWishlistUseCase.kt` · `CollectionNode.isWishlist` |
| Rating, note, dernière écoute | `app:domain/usecase/UpdateRatingUseCase.kt`, `MarkAsPlayedUseCase.kt` |
| Thème, couleurs | `app:ui/theme/` |
| Settings, préférences, DataStore | `app:data/local/settings/DataStoreSettingsRepository.kt` · `app:ui/settings/` |
| Variables d'environnement | `be:config/env.validation.ts` |
| Seed, types de collection en base | `acervatim-backend/prisma/seed.ts` |
| Déploiement backend | `acervatim-backend/deploy/` · `acervatim-docs/developpeur/deploiement.md` |
| Cron, jobs planifiés | `be:subscriptions/subscriptions.cron.ts` |
| requestId, corrélation | `be:common/interceptors/correlation-id.interceptor.ts` |
