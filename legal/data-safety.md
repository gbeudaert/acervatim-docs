---
title: Data Safety (Google Play)
layout: default
parent: Légal
nav_order: 4
---

# Sécurité des données — déclaration Google Play

Déclaration de la section **« Sécurité des données »** de la fiche Google Play, alignée sur la
[politique de confidentialité](confidentialite.md). À reporter telle quelle dans la Play Console lors
de la publication.

## Méthodes de sécurité déclarées

- ✅ Les données sont **chiffrées en transit** (HTTPS/TLS).
- ✅ L'utilisateur peut **demander la suppression** de ses données (suppression de compte en cascade).
- ✅ Les pratiques de sécurité ont été examinées en interne.

## Tableau de correspondance

| Type de données Google Play | Collecté ? | Partagé ? | Optionnel ? | Finalité | Chiffré en transit | Suppressible |
|---|---|---|---|---|---|---|
| Nom | ❌ | — | — | — | — | — |
| Adresse e-mail | ❌ *(reçue mais immédiatement ignorée, jamais stockée)* | — | — | — | ✅ | — |
| Photo de profil | ❌ | — | — | — | — | — |
| Identifiant utilisateur (`sub` Google) | ✅ *(haché)* | ❌ | ❌ (requis) | Authentification | ✅ | ✅ |
| Identifiant interne (UUID) | ✅ | ❌ | ❌ (requis) | Identification dans l'app | ✅ | ✅ |
| Activité dans l'app (collections, items) | ✅ | ❌ | ❌ (cœur fonctionnel) | Stockage personnel | ✅ | ✅ |
| Achats dans l'app (token d'achat) | ✅ | ❌ | ✅ (si premium) | Vérification abonnement | ✅ | ✅ |
| Jetons OAuth tiers | ✅ *(chiffrés)* | ❌ | ✅ (si service connecté) | Appel API en votre nom | ✅ | ✅ |
| Informations financières | ❌ *(gérées par Google Play)* | — | — | — | — | — |
| Localisation | ❌ | — | — | — | — | — |
| Contacts, calendrier, SMS, photos | ❌ | — | — | — | — | — |
| Identifiants publicitaires | ❌ | — | — | — | — | — |

> Source : annexe « Sécurité des données » de la [politique de confidentialité](confidentialite.md).
> Toute évolution des données traitées doit mettre à jour **les deux** documents en même temps.
