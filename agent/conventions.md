# Conventions transverses — règles non négociables

> Résumé opérationnel des `CLAUDE.md` des deux dépôts. En cas de doute, lire l'original :
> `acervatim-app/CLAUDE.md` · `acervatim-backend/CLAUDE.md`.

## Privacy by design (backend — règle absolue)

- **Aucune donnée perso directe en base** : pas d'email, nom, photo, ni `sub` Google en clair.
- Identité = `users.googleSubHash` (HMAC-SHA-256 du `sub` avec `SUB_HASH_PEPPER`). `users.id` = UUID v4 opaque.
- Jamais de `sub`, email ou token OAuth en clair dans les logs (préfixe de hash pour debug : `subHash.slice(0, 8) + '…'`).
- Tokens OAuth externes stockés chiffrés AES-256-GCM (`AesService`).
- Secrets non rotables sans plan de migration : `SUB_HASH_PEPPER`, `ENCRYPTION_KEY`, `JWT_PRIVATE_KEY`, `INVITE_CODE_PEPPER`.

## Commandes backend — toujours via le wrapper

`npm` / `npx` / `node` / `prisma` / `nest` ne se lancent **jamais** directement sous Windows : toujours `acervatim-backend/scripts/dev.ps1` (exécution dans le conteneur `app`). Voir [backend/operations-dev.md](backend/operations-dev.md).

## Commandes app — toujours le wrapper Gradle

`.\gradlew.bat assembleDebug | installDebug | test | connectedAndroidTest | lint` — jamais un `gradle` système.

## Contrats API V1 (backend)

- Préfixe global `/v1/` (sauf `/health` et webhooks publics).
- Validation **Zod via nestjs-zod** (pas de class-validator dans le nouveau code).
- Erreurs RFC 9457 Problem Details (`application/problem+json`, `requestId` toujours présent).
- Pagination **cursor only** : `?cursor=&limit=` (max 100, défaut 50) ; listes wrappées `{ data, meta.pagination }`, ressources unitaires brutes.
- Filtres `field[in]` (OR) / `field[all]` (AND), champs whitelistés par endpoint via Zod (`src/common/filters/parse-filters.ts`).
- Jamais de `string_contains` Prisma sur colonne JSON pour un contains insensible à la casse (MariaDB → collation bin) ; SQL paramétré avec `COLLATE utf8mb4_general_ci` (réf. `CollectionsService.matchingCollectionIds`).

## Architecture app (Android)

- Single-activity (`MainActivity`) + Compose ; nouveaux écrans = composables, jamais d'Activity/Fragment.
- Couches strictes `ui/` ← `domain/` ← `data/` ; Compose ne touche jamais `data/` directement.
- `ViewModel` + `StateFlow<UiState>` collecté avec `collectAsStateWithLifecycle()`.
- DI = Hilt. Coroutines/Flow, dispatchers injectés.
- Dépendances : uniquement via `gradle/libs.versions.toml`.
- Texte UI : `stringResource(...)` obligatoire, pas de littéraux.

## Où mettre la documentation (règle commune aux deux dépôts)

| Type de doc | Emplacement |
|---|---|
| Lisible par humains (utilisateur/développeur/légal) | ce dépôt `acervatim-docs/` (site Jekyll public) : [`../utilisateur/`](../utilisateur/) · [`../developpeur/`](../developpeur/) · [`../legal/`](../legal/) |
| Index pour agents IA (cette doc) | `acervatim-docs/agent/` (exclu du site publié) |

Aucune autre documentation : ne rien référencer ni créer ailleurs (en particulier pas de doc gitignorée dans les dépôts de code). La politique de confidentialité **canonique** vit dans [`../legal/confidentialite.md`](../legal/confidentialite.md).

## Hors-sujet — ne pas faire

- Pas de refactor d'opportunité hors périmètre du sprint en cours.
- Pas de dossier `lib/`/`utils/` global côté backend : tout dans `src/common/<sous-dossier explicite>/`.
- Pas d'ajout de dépendance sans discussion préalable (les deux dépôts).
- Pas d'emojis dans le code ou les commits.
