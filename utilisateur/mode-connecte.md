---
title: Mode connecté
layout: default
parent: Utilisateur
nav_order: 4
---

# Le mode connecté (optionnel)

Acervatim fonctionne **intégralement sans compte ni réseau**. Le **mode connecté** est une option :
en vous connectant avec Google, vous débloquez des services qui s'appuient sur le **serveur
Acervatim**. Vous pouvez l'ignorer complètement et continuer à utiliser l'application en local.

## Ce que le mode connecté apporte

| Service | À quoi ça sert |
|---|---|
| **Enrichissement au scan** | Scanner un disque ou un manga et récupérer automatiquement titre, auteur·rice·s, pochette, année… depuis des bases spécialisées (voir [D'où viennent les informations](sources-de-donnees.md)). |
| **Synchronisation** | Retrouver vos collections sur plusieurs appareils, sauvegardées côté serveur. |
| **Compte** | Export de vos données (portabilité) et suppression de compte en un geste. |
| **Premium** | Quotas étendus et fonctionnalités avancées (via Google Play ou code d'invitation). |

## Ce qui reste toujours vrai

- **Offline-first** : consultation, édition, recherche locale, import/export fonctionnent **sans
  réseau**, connecté ou non.
- **Rien en arrière-plan** : le serveur n'est sollicité que sur une **action explicite** de votre
  part (scan, recherche, synchronisation). Jamais de requête silencieuse.
- **Privacy by design** : aucune donnée personnelle directe n'est stockée côté serveur — pas de nom,
  pas de photo, et votre adresse e-mail Google est ignorée. Détails : [politique de
  confidentialité](../legal/confidentialite.md).

## Comment ça marche (en bref)

1. Vous vous connectez avec **Google Sign-In**.
2. L'application échange ce login contre un **jeton de session** stocké chiffré sur l'appareil.
3. Vos actions connectées (scan, sync) passent par le serveur, qui interroge les bases externes à
   votre place et garde les clés d'accès de son côté.

> Pour le détail technique (architecture, API, sécurité), voir l'espace développeur :
> [Le serveur Acervatim](../developpeur/backend.md).
