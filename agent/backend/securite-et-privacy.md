# acervatim-backend — Sécurité & privacy

> Remonter : [INDEX.md](INDEX.md) · Règles complètes : `acervatim-backend/CLAUDE.md` (§ Privacy by design, § Secrets)

## Chaîne d'authentification

1. App : Google Sign-In → `idToken` → `POST /v1/auth/google`.
2. `auth/providers/google.provider.ts` vérifie l'idToken (audience, signature).
3. `auth/identity-resolver.service.ts` : `sub` → HMAC-SHA-256 avec `SUB_HASH_PEPPER` (`common/crypto/hash.service.ts`) → lookup/création `users.googleSubHash`.
4. Émission d'un JWT interne (clé `JWT_PRIVATE_KEY`) porté ensuite en `Authorization: Bearer`.
5. `common/guards/jwt-auth.guard.ts` protège toutes les routes `/v1` applicatives ; `@CurrentUserId()` extrait le userId.

Exceptions au JWT : `POST /v1/auth/google` (public), `GET /health`, `POST /webhooks/google-play/rtdn` (vérifié par `subscriptions/pubsub-verifier.service.ts`), `POST /v1/admin/invitations` (guard `invitations/admin-token.guard.ts` — token admin statique).

## Invariants privacy (règle absolue)

- Aucune donnée perso directe en base : pas d'email, nom, photo, `sub` en clair.
- Aucun log avec `sub`, email ou token OAuth en clair ; debug = `subHash.slice(0, 8) + '…'`.
- Tout nouveau champ `users` ⇒ check de réidentification avant merge.
- Codes d'invitation : seul le HMAC est stocké (`INVITE_CODE_PEPPER`), le code en clair n'est retourné qu'à la création.

## Chiffrement

| Quoi | Comment | Où |
|---|---|---|
| Tokens OAuth externes (Discogs/MAL/TMDB) | AES-256-GCM, clé `ENCRYPTION_KEY`, versionnée (`encryptedWithKeyVersion`) | `common/crypto/aes.service.ts`, table `oauth_credentials` |
| Identité Google | HMAC-SHA-256 + `SUB_HASH_PEPPER` | `common/crypto/hash.service.ts` |
| Codes d'invitation | HMAC-SHA-256 + `INVITE_CODE_PEPPER` | idem |

## Secrets critiques (non rotables sans plan de migration)

`SUB_HASH_PEPPER` (perdre = perdre tous les comptes) · `ENCRYPTION_KEY` · `JWT_PRIVATE_KEY` · `INVITE_CODE_PEPPER`. Validés au boot par `config/env.validation.ts`. Jamais committés (`.env` gitignored).

## Garde-fous applicatifs

| Mécanisme | Fichier |
|---|---|
| Guard premium (402 Payment Required) | `premium/premium.guard.ts` + `payment-required.exception.ts` |
| Quota items free/premium | `common/quota/quota.service.ts` |
| Rate limit providers externes | `common/rate-limit/token-bucket.service.ts` |
| Erreurs sans leak d'internals (RFC 9457) | `common/filters/problem-details.filter.ts` |
| Corrélation requêtes (`requestId`) | `common/interceptors/correlation-id.interceptor.ts` |
| Audit append-only (login, delete, export, grants) | `common/audit/audit-log.service.ts` |
| Validation env stricte | `config/env.validation.ts` |
| Whitelist des filtres query (anti injection) | `common/filters/parse-filters.ts` |
