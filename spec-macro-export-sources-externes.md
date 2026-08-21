# Spec macro — Export de la collection vers les sources externes

> **Document interne** (exclu du site publié, comme `sprints-conformite-offre.md`).
> Créé le 2026-08-20. Sprints d'exécution : `acervatim-backend/docs/interne/sprints/sync-externe/`
> et `acervatim-app/docs/interne/sprints/sync-externe/` (répertoires **gitignorés**).

## 1. Objectif produit

Pousser sa collection Acervatim **vers** les services externes :

| Type | Destination | Correspondance naturelle |
|---|---|---|
| Vinyles | Discogs | Collection (possédés) et Wantlist (désirés) |
| Mangas | MyAnimeList | Liste manga de l'utilisateur |

## 2. Le renversement à mesurer

Jusqu'ici, Acervatim **lit** les sources externes : recherche, enrichissement, jaquettes. Les jetons
OAuth servent exclusivement à consulter. Cette évolution les fait **écrire dans le compte réel de
l'utilisateur**, chez un tiers.

Ça change trois choses :

- **La réversibilité.** Une erreur de lecture affiche une mauvaise donnée ; une erreur d'écriture
  modifie la collection Discogs d'une personne — parfois constituée sur des années. Les garde-fous
  ne sont pas optionnels.
- **Le risque du jeton.** Un jeton compromis permettait jusqu'ici de lire ; il permettrait désormais
  de **détruire** la collection distante. La justification du chiffrement AES-256-GCM des tokens
  (déjà en place) devient encore plus structurante.
- **Le rythme.** Une lecture, c'est une requête. Un export, c'est des centaines de requêtes d'un
  coup, sous quota tiers.

## 3. État réel du code (audit du 2026-08-20)

### Les jetons actuels ne sont pas des jetons d'écriture

- **MAL** : `mal.adapter.ts` construit l'URL d'autorisation **sans paramètre `scope`**
  (`response_type`, `client_id`, `code_challenge`, `state`, `redirect_uri` — rien d'autre). Les deux
  adapters stockent `scopes: []`.
  ⇒ **À vérifier en premier** : le jeton actuel permet-il `PATCH /v2/manga/{id}/my_list_status` ? Si
  non, il faut demander `write:users`, donc **refaire consentir tous les utilisateurs déjà
  connectés**. Ce n'est pas un détail technique : c'est un parcours utilisateur à prévoir.
- **Discogs** : OAuth 1.0a n'a pas de scopes — le jeton d'accès porte les droits du compte, écriture
  comprise. L'écriture devrait donc fonctionner sans reconsentement. À confirmer par un appel réel.

### Discogs a besoin du `username`, qui est une donnée identifiante

Les endpoints de collection Discogs sont nominatifs :
`/users/{username}/collection/folders/{id}/releases/{release_id}`. Le code ne récupère jamais ce
`username` (aucun appel à `/oauth/identity`).

**Tension à arbitrer** : un pseudo Discogs est une donnée personnelle — souvent le même pseudo que
partout ailleurs, publiquement associé à une collection. L'invariant du projet est « aucune donnée
perso directe en base ». Trois issues :

1. **Ne pas le stocker** : appeler `/oauth/identity` à chaque export. Une requête de plus, aucune
   donnée conservée. *Recommandé* — le coût est négligeable au regard d'un export de plusieurs
   centaines d'appels.
2. Le stocker **chiffré** dans `oauth_credentials` (déjà chiffré, déjà lié au provider).
3. Le stocker en clair — **exclu**, contradictoire avec l'invariant publié.

### Seuls les éléments déjà identifiés sont exportables

Pour pousser un vinyle vers Discogs, il faut un `release_id` Discogs. L'app le détient
(`sources[]` avec `provider: 'discogs'`), **mais seulement pour les disques ajoutés via
enrichissement**. Un disque saisi à la main, ou trouvé sans correspondance, n'en a pas.

Même chose côté manga : il faut l'identifiant MAL de la série.

⇒ **La couverture de l'export sera partielle, et c'est structurel.** L'UI doit l'annoncer avant de
lancer (« 142 disques sur 380 peuvent être exportés »), pas le découvrir après. Ne pas essayer de
résoudre l'identifiant à la volée pendant l'export : cela multiplierait les appels et introduirait
des rapprochements approximatifs dans une opération d'écriture — le pire endroit pour ça.

## 4. Les décisions à trancher

### Sens unique, pas synchronisation bidirectionnelle

Le besoin exprimé est « pousser ». **Recommandation : s'y tenir strictement.** Une synchronisation
bidirectionnelle demanderait une résolution de conflits entre deux modèles de données qui ne se
recouvrent pas (voir plus bas), pour un bénéfice non demandé. Un export unidirectionnel est
compréhensible, testable et sûr.

### Ne jamais supprimer à distance

**Recommandation forte : l'export est additif.** Supprimer un disque dans Acervatim ne doit **pas**
le retirer de Discogs.

Raison : un bug de synchronisation, une base locale réinitialisée, une réinstallation mal restaurée
— et l'export vide la collection Discogs de l'utilisateur. Le rapport bénéfice/risque est
catastrophique. Si un besoin de retrait apparaît, il devra passer par une action explicite,
unitaire, confirmée — jamais par la propagation automatique d'un état local.

### Idempotence : ne pas créer de doublons

Relancer un export ne doit pas ajouter deux fois les mêmes disques. Piège : sur Discogs, posséder
**deux exemplaires** du même pressage est légitime et représenté par deux *instances*. « Déjà
présent » n'est donc pas déductible du seul `release_id` — il faut mémoriser ce qu'Acervatim a
poussé (l'`instance_id` renvoyé à l'ajout) pour distinguer « je l'ai déjà exporté » de « l'utilisateur
en a deux exemplaires ».

### MAL : posséder n'est pas avoir lu

MAL modélise une **progression de lecture** (`num_volumes_read`, `status: reading/completed/…`).
Acervatim modélise une **possession** par tome. Ce n'est pas la même chose : on peut posséder 12
tomes et n'en avoir lu aucun.

Écrire `num_volumes_read = nombre de tomes possédés` serait un mensonge dans le compte de
l'utilisateur. À trancher explicitement — options : ne remplir que le statut de liste sans la
progression, demander à l'utilisateur ce qu'il veut y voir, ou renoncer à mapper la progression.
Décider **avant** de coder : c'est une question de sens, pas d'implémentation.

## 5. Contraintes opérationnelles

- **Quota Discogs : 60 requêtes/minute.** Exporter 500 disques = ~9 minutes d'appels soutenus. Cela
  ne peut pas vivre dans un cycle requête/réponse : l'export doit être un **job asynchrone**, sur le
  patron déjà éprouvé de `edition-import` (BullMQ, `202 { jobId }`, endpoint de suivi, progression).
- **Échecs partiels.** Sur 300 éléments, certains échoueront (identifiant obsolète, release
  supprimée, erreur transitoire). Un export n'est pas « réussi » ou « échoué » : il produit un
  **rapport**. Le modèle doit le prévoir dès le départ.
- **Reprise.** Un job interrompu doit pouvoir reprendre sans re-pousser ce qui l'a déjà été — ce que
  garantit le suivi d'idempotence ci-dessus.

## 6. Palier

Fonctionnalité **Premium** : elle consomme du serveur (job long, appels sortants soutenus) et
s'inscrit dans la règle actée — *ce qui a un coût serveur réel est premium*
(cf. [sprints-conformite-offre.md](sprints-conformite-offre.md)). Elle exige de toute façon un jeton
personnel connecté (BYOT), les jetons Acervatim n'ayant aucun sens ici : on écrit dans le compte de
l'utilisateur, avec **son** identité.

## 7. Découpage

```
Backend                                     App
───────                                     ───
SE1 fondations d'écriture
    (scopes, identité, job, garde-fous)
        │
        ├──► SE2 export Discogs (vinyles) ──► AE1 UI export Discogs
        │
        └──► SE3 export MAL (mangas) ───────► AE2 UI export MAL
```

SE1 est un prérequis dur des deux autres. SE2/SE3 sont indépendants entre eux — Discogs d'abord
(mapping plus naturel, pas de reconsentement attendu, question de sens moins épineuse que MAL).

## 8. Points ouverts

- Le jeton MAL actuel autorise-t-il l'écriture ? (bloquant SE3 — et détermine s'il faut refaire
  consentir les utilisateurs déjà connectés)
- Où stocker — ou ne pas stocker — le `username` Discogs ?
- MAL : que faire de la progression de lecture ?
- Un retrait distant explicite est-il souhaité un jour, ou l'export reste-t-il définitivement
  additif ?

## 9. Documents liés

| Document | Contenu |
|---|---|
| [agent/transverse/sources-externes.md](agent/transverse/sources-externes.md) | Résolution des jetons, sources et adapters |
| [agent/backend/scan-asynchrone-bullmq.md](agent/backend/scan-asynchrone-bullmq.md) | Passerelle BullMQ — le patron de job à réutiliser |
| [sprints-conformite-offre.md](sprints-conformite-offre.md) | Modèle d'offre et rattachement au Premium |
