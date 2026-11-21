---
title: Build locale de l'app
layout: default
parent: Développeur
nav_order: 9
---

# Build locale de l'app — installable à côté de celle du Play Store

La variante **debug** de l'application porte un `applicationId` distinct de la variante **release**.
Android traite deux `applicationId` différents comme **deux applications sans rapport** : la build
locale s'installe donc sur le même téléphone que celle du Play Store, sans la remplacer ni écraser
ses données.

| | Play Store (`release`) | Locale (`debug`) |
|---|---|---|
| `applicationId` | `app.acervatim` | `app.acervatim.dev` |
| Nom affiché | Acervatim | **Acervatim Dev** |
| Icône | fond vert | fond **orange** (API 26+) |
| `versionName` | celui de `app/build.gradle.kts` | le même, suffixé `-dev` |
| Signature | clé release / Play App Signing | clé debug (`~/.android/debug.keystore`) |
| Backend visé | `https://api.acervatim.app/` | idem par défaut, surchargeable (voir plus bas) |

Chaque application a sa propre base Room, son propre DataStore et son propre JWT stocké : **aucune
donnée n'est partagée entre les deux**. Désinstaller l'une ne touche pas l'autre.

## Où c'est défini

Tout tient dans le type de build `debug` de `acervatim-app/app/build.gradle.kts` et dans le
*source set* `app/src/debug/` :

- `applicationIdSuffix = ".dev"` et `versionNameSuffix = "-dev"` ;
- `app/src/debug/res/values/strings.xml` → `app_name` = « Acervatim Dev » ;
- `app/src/debug/res/values/colors.xml` → `ic_launcher_background` orange ;
- `app/src/debug/AndroidManifest.xml` → `usesCleartextTraffic="true"`, pour pouvoir taper un
  backend local en HTTP simple. Cet attribut ne doit **jamais** remonter dans le manifest principal.

Les autorités des `provider` (FileProvider, androidx-startup) utilisent déjà `${applicationId}` :
rien à changer de ce côté, les deux installations ne se marchent pas dessus.

## Construire et installer

```powershell
cd ..\acervatim-app
.\gradlew :app:assembleDebug
adb install -r app\build\outputs\apk\debug\app-debug.apk
```

Depuis Android Studio, le bouton *Run* en variante `debug` fait la même chose. `adb devices` doit
lister le téléphone (débogage USB activé, ou `adb pair` en Wi-Fi).

Vérifier ce qui a été produit :

```powershell
aapt dump badging app\build\outputs\apk\debug\app-debug.apk | Select-Object -First 1
# package: name='app.acervatim.dev' versionCode='<n>' versionName='<version>-dev'
```

## Viser un backend local

Par défaut la build debug tape l'API publique — elle est utilisable telle quelle. Pour la faire
pointer vers un backend lancé sur le poste de dev, poser dans `acervatim-app/local.properties`
(non versionné) :

```properties
# Téléphone physique : IP LAN du poste, joignable depuis le Wi-Fi du téléphone
devApiBaseUrl=http://192.168.1.20:3000/
# Émulateur : la machine hôte est vue en 10.0.2.2
# devApiBaseUrl=http://10.0.2.2:3000/
```

La valeur est injectée dans `BuildConfig.API_BASE_URL` à la configuration Gradle. Le backend doit
écouter sur toutes les interfaces (le `docker compose` du dépôt publie déjà `3000:3000`), et le
pare-feu Windows doit laisser passer le port 3000 en réseau privé.

> `local.properties` est lu à la **configuration** Gradle. Après l'avoir modifié, relancer le build ;
> si la valeur ne bouge pas, invalider le cache de configuration
> (`.\gradlew :app:assembleDebug --no-configuration-cache`).

## Google Sign-In : un client OAuth par package

Le client OAuth **Android** est déclaré par couple *(package, empreinte SHA-1)*. Changer le package
en `app.acervatim.dev` invalide le client existant : sans nouveau client, le Sign-In échoue avec
`DEVELOPER_ERROR` (`10`).

À ajouter dans le projet Google Cloud, en plus des clients décrits dans
[Authentification Google](auth-google.md) :

| Package | SHA-1 |
|---|---|
| `app.acervatim.dev` | empreinte de la clé **debug** |

```powershell
keytool -list -v -alias androiddebugkey -keystore $env:USERPROFILE\.android\debug.keystore -storepass android -keypass android
```

Le `google_server_client_id` (client **Web**) ne change pas : c'est toujours lui l'`audience` du
token vérifiée par le backend. Rien à modifier côté serveur.

## Points d'attention

- **Ne pas suffixer la release.** Le package `app.acervatim` est l'identité de l'app publiée ; le
  changer rendrait la mise à jour Play Store impossible.
- Les deux installations consomment le même compte utilisateur si elles visent le même backend :
  la sync tirera les mêmes collections dans deux bases locales distinctes.
- L'APK debug n'est ni minifié ni optimisé (~50 Mo) : il ne mesure pas la taille livrée.

## Voir aussi

- [Publication Play Store](publication-play-store.md) — la variante release et sa signature.
- [Authentification Google](auth-google.md) — les clients OAuth et les SHA-1 à déclarer.
