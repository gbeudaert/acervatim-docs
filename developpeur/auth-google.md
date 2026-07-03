---
title: Authentification Google
layout: default
parent: Développeur
nav_order: 4
---

# Authentification Google (OAuth)

Comment Acervatim authentifie un utilisateur : l'app obtient un `id_token` Google, le backend le
**vérifie**, hache le `sub` Google et émet un **JWT interne**. Cette page couvre le setup Google Cloud
(app + backend) et le flux de vérification.

> **Source de vérité** : le code des dépôts `acervatim-backend` (vérification du token, config
> OAuth serveur) et `acervatim-app` (intégration Credential Manager).

## Modèle — deux clients OAuth, un écran de consentement

Dans **un seul projet Google Cloud**, il faut **2 clients OAuth** + 1 écran de consentement :

| Client OAuth | Rôle | Où va la valeur |
|---|---|---|
| **Web** | Identité que le **backend** présente ; c'est l'`audience` du token. | `google_server_client_id` (app) **et** `GOOGLE_CLIENT_ID` (backend) |
| **Android** | Autorise l'APK (via empreinte SHA-1) à demander un token. | Rien dans le code — déclaré dans la console via la SHA-1 |

L'app référence le **client Web** (pas l'Android) : c'est le `serverClientId` passé au Credential
Manager. Le client Android est seulement « connu » de Google via sa SHA-1.

> `google_server_client_id` est un **identifiant public** (transite en clair, extractible de l'APK) —
> le commiter est l'usage standard. Le **client secret** Web reste exclusivement côté backend, et
> n'est même pas requis pour la simple vérification d'`id_token`.

## Setup Google Cloud

1. **Projet** : `console.cloud.google.com` → nouveau projet `Acervatim`.
2. **Écran de consentement OAuth** : type *Externe* ; scopes de base `openid`, `email`, `profile`
   (le backend ne lit que le `sub`). En mode *Testing*, ajouter les comptes Gmail testeurs (sinon
   connexion refusée).
3. **Client Web** (*Application Web*) → fournit le `Client ID` (`…apps.googleusercontent.com`) à
   placer dans `strings.xml` (`google_server_client_id`) **et** dans le backend (`GOOGLE_CLIENT_ID`).
   Pas d'URI de redirection nécessaire pour la vérification d'`id_token` mobile.
4. **Client Android** (*Android*) : package `app.acervatim` + **empreinte SHA-1**. Un client par
   empreinte.

### Les trois SHA-1 à déclarer

- **Debug** : clé auto-générée par Android Studio (builds locaux).
- **Upload** : la clé qui signe l'AAB envoyé à Google.
- **Play App Signing** : la clé qui signe l'app **livrée aux utilisateurs** — la plus importante ;
  son absence cause un `DEVELOPER_ERROR` (`10`) au Sign-In en production.

```powershell
# Debug
keytool -list -v -alias androiddebugkey -keystore $env:USERPROFILE\.android\debug.keystore -storepass android -keypass android
# Release / upload
keytool -list -v -alias <ton_alias> -keystore <chemin\ton.keystore>
```

## Configuration backend

Dans `.env` (jamais commité) :

```dotenv
# audience de l'id_token = WEB client ID
GOOGLE_CLIENT_ID="xxxxxxxxx.apps.googleusercontent.com"
```

Lu au boot par `GoogleIdentityProvider.onModuleInit()` ; absente, l'app refuse de démarrer
(`GOOGLE_CLIENT_ID must be set`).

## Flux de vérification

1. App : Google Sign-In (Credential Manager) → `id_token`.
2. App → `POST /v1/auth/google` `{ idToken }`.
3. Backend : vérifie la signature et l'`audience` (= `GOOGLE_CLIENT_ID`), extrait le `sub`, calcule
   `HMAC-SHA-256(sub, SUB_HASH_PEPPER)` → `users.googleSubHash`. **L'e-mail est ignoré, jamais
   stocké.**
4. Backend : émet un **JWT interne RS256** (UUID utilisateur opaque en sujet).
5. App : stocke le JWT chiffré, l'injecte en `Authorization: Bearer`.

Vérification rapide : `POST http://localhost:3000/v1/auth/google` avec un `id_token` réel → `200` + JWT.

## Pièges fréquents

- **`audience` mismatch** (`401 Invalid Google id_token`) : app et backend doivent utiliser
  *exactement* le même client **Web** (et non l'Android).
- **SHA-1 manquante / mauvaise** : `GetCredentialException` / `DEVELOPER_ERROR` au Sign-In — penser à
  la SHA-1 **release ET** Play App Signing une fois publié.
- **Compte non testeur** en mode *Testing* : connexion refusée.
- Le **client secret** Web ne doit **jamais** être commité ni embarqué dans l'app.

## Récapitulatif des valeurs

| Valeur | Source | Destination |
|---|---|---|
| Client ID **Web** | Client Web | `strings.xml` (`google_server_client_id`) + backend (`GOOGLE_CLIENT_ID`) |
| Client **secret** Web | Client Web | Backend uniquement (souvent inutile pour la simple vérif) |
| Client ID **Android** | Client Android | Aucune (lié à la SHA-1, connu de Google) |
| SHA-1 debug / upload / Play | `keytool` / Play Console | Console Google (clients Android) |
| `GOOGLE_CLIENT_ID` | Client Web | `.env` backend |

## Voir aussi

- [Le serveur Acervatim](backend.md) · [Architecture § Authentification](architecture.md#1-authentification).
- Publication : [Play Store](publication-play-store.md).
