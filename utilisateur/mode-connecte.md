---
title: Mode connecté
layout: default
parent: Utilisateur
nav_order: 4
---

# Le mode connecté (optionnel)

Acervatim fonctionne **intégralement sans compte ni réseau**. Au-dessus de ce socle hors-ligne,
deux niveaux optionnels s'ajoutent — vous pouvez les ignorer complètement et continuer à utiliser
l'application en local.

## Les trois modes

| Mode | Ce qu'il apporte |
|---|---|
| **Hors-ligne** (par défaut) | Tout l'essentiel, sans compte ni réseau : saisie manuelle, scan local, recherche, wishlist, import/export, sauvegarde ZIP. |
| **Connecté** | **Enrichissement** des fiches (titre, auteur·rice·s, pochette, année, synopsis…) depuis des bases spécialisées (voir [D'où viennent les informations](sources-de-donnees.md)), en utilisant **vos propres jetons d'accès** aux sources (*bring your own token*). Pas de synchronisation. |
| **Premium** | Enrichissement avec les **jetons Acervatim** — aucune clé personnelle à créer. Dans un second temps : **synchronisation cloud** de vos collections, d'abord comme sauvegarde, puis pour la consultation en ligne. |

## Le mode connecté en pratique

- **Enrichissement au scan ou à la recherche** : scanner un disque ou un manga et récupérer
  automatiquement ses métadonnées depuis des bases spécialisées, avec les jetons que vous avez
  renseignés dans **Réglages → Comptes & sources**.
- **Compte** : export de vos données (portabilité) et suppression de compte en un geste.

## Ce qui reste toujours vrai

- **Offline-first** : consultation, édition, recherche locale, import/export fonctionnent **sans
  réseau**, connecté ou non.
- **Rien en arrière-plan** : le serveur n'est sollicité que sur une **action explicite** de votre
  part (scan, recherche). Jamais de requête silencieuse.
- **Privacy by design** : aucune donnée personnelle directe n'est stockée côté serveur — pas de nom,
  pas de photo, et votre adresse e-mail Google est ignorée. Détails : [politique de
  confidentialité](../legal/confidentialite.md).

## Comment ça marche (en bref)

1. Vous vous connectez avec **Google Sign-In**.
2. L'application échange ce login contre un **jeton de session** stocké chiffré sur l'appareil.
3. Vos actions connectées (scan, recherche) interrogent les bases externes avec **vos jetons
   d'accès** — ou avec les jetons Acervatim si vous êtes premium.

> Pour le détail technique (architecture, API, sécurité), voir l'espace développeur :
> [Le serveur Acervatim](../developpeur/backend.md).
