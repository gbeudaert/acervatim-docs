---
title: D'où viennent les informations
layout: default
parent: Utilisateur
nav_order: 3
---

# D'où viennent les informations de vos collections

Acervatim fonctionne **hors-ligne par défaut** : vos collections vivent sur votre appareil, sans
compte ni réseau. Quand vous le souhaitez (et que vous êtes connecté), l'application peut
**enrichir** un élément — récupérer automatiquement sa pochette, son synopsis, ses auteurs, sa date,
etc. — en interrogeant des **bases de données spécialisées** reconnues.

Cette page explique **quelles sources** sont utilisées et **ce que ça implique pour votre vie privée**.

## Les sources par type de collection

| Type | Source(s) | Ce qu'on y trouve |
|---|---|---|
| **Vinyles / disques** | [Discogs](https://www.discogs.com) | Album, artiste(s), année, pochette, label — recherche par texte **ou par code-barres** (EAN). |
| **Mangas** | [MyAnimeList](https://myanimelist.net) + [Bibliothèque nationale de France (BnF)](https://catalogue.bnf.fr) | L'**œuvre** (synopsis, démographie, note, auteur) côté MyAnimeList ; l'**édition française** (éditeur, tome, ISBN) côté BnF — recherche **par ISBN**. |
| **Films** | [TMDB](https://www.themoviedb.org) | Titre, affiche, année, synopsis. |

> L'enrichissement est **optionnel** et **à votre initiative** : l'application reste pleinement
> utilisable sans jamais interroger ces sources.

## Scanner un manga : retrouver la série et tous ses tomes

Pour les mangas, vous pouvez **scanner le code-barres (ISBN)** d'un tome. Acervatim :

1. identifie l'ouvrage auprès de la **BnF** (catalogue officiel des éditions françaises) ;
2. retrouve la **série** correspondante et ses informations sur **MyAnimeList** ;
3. vous propose la série pour **valider les tomes** d'un coup.

**Les éditions spéciales sont reconnues.** Si vous possédez par exemple *L'Attaque des Titans* en
**édition Colossale** (un grand format qui regroupe plusieurs tomes de l'édition classique), le scan
le détecte : Acervatim sait qu'il s'agit de la Colossale et **à quels tomes de l'édition standard**
ce volume correspond. Vous ne mélangez donc pas une édition collector avec l'édition classique.

> Selon les ouvrages, l'information peut être partielle (séries en cours de publication, petits
> éditeurs, catalogage incomplet). Dans ce cas, l'application pré-remplit ce qu'elle a trouvé et vous
> complétez le reste — rien n'est jamais imposé.

## Et votre vie privée ?

Acervatim est conçu **privacy by design**. Concernant ces sources de données :

- Les recherches portent sur **l'objet** (un disque, un livre), **jamais sur vous**. Aucun nom, email
  ni identifiant personnel n'est transmis à Discogs, MyAnimeList, la BnF ou TMDB.
- Ces appels passent par **notre serveur**, qui sert d'intermédiaire ; les clés d'accès aux sources
  restent côté serveur et ne transitent jamais par votre appareil.
- Vous restez **maître de vos données** : ce qui est récupéré sert uniquement à pré-remplir une fiche,
  que vous pouvez modifier ou ignorer.

Pour le détail du traitement des données personnelles, voir la [politique de
confidentialité](../legal/confidentialite.md).

## Pour les développeurs

Le fonctionnement technique (d'où vient chaque champ, le « pivot » ISBN → BnF → MyAnimeList, le
format des réponses) est documenté côté développeur :
[Sources de données & pivot ISBN](../developpeur/sources-et-pivot-isbn.md).
