---
title: Publication Play Store
layout: default
parent: Développeur
nav_order: 6
---

# Publication Play Store

Processus de publication de l'application Android `app.acervatim` sur Google Play : modèle de
distribution, prérequis, cohérence légale, versioning et rappels de sécurité.

## Modèle de distribution

- Application **gratuite**, sans achat bloquant ni publicité.
- Publiée via **Google Play Console**, format **Android App Bundle (`.aab`)** signé.
- **Play App Signing** activé : Google gère la clé de signature de distribution ; la clé d'upload est
  conservée par l'éditeur (jamais commitée).

## Prérequis (une seule fois)

| Élément | Détail |
|---|---|
| Compte Google Play Console | Inscription développeur (frais unique), vérification d'identité. |
| **Politique de confidentialité publique** | URL HTTPS exigée dès qu'une permission sensible (`CAMERA`) ou une collecte existe → [politique de confidentialité](../legal/confidentialite.md). |
| **Data Safety** | Formulaire Play renseigné en cohérence avec la politique → [Data Safety](../legal/data-safety.md). |
| Adresse e-mail de contact | Affichée sur la fiche Store. |

## Cohérence légale

L'application **n'est plus 100 % hors-ligne** depuis l'ajout du mode connecté optionnel (Google
Sign-In) : la déclaration **Data Safety** et la **politique de confidentialité** doivent refléter
les données traitées en mode connecté. Le tableau de correspondance Google Play est tenu dans la
[politique de confidentialité § annexe](../legal/confidentialite.md).

## Versioning

- `versionName` en SemVer (ex. `0.1.0`).
- `versionCode` entier monotone, incrémenté à chaque upload (convention :
  `MAJOR * 10000 + MINOR * 100 + PATCH`).

## Sécurité — rappels

- Le **keystore** et `keystore.properties` ne sont **jamais commités** (gitignored) et sont
  sauvegardés hors dépôt. Leur perte empêche toute mise à jour ultérieure (hors reset Play App
  Signing).
- Les secrets (mots de passe keystore) restent locaux.

## Voir aussi

- [Authentification Google](auth-google.md) · [Politique de confidentialité](../legal/confidentialite.md) · [Data Safety](../legal/data-safety.md).
