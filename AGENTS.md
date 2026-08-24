<!-- bmad:context -->
<!-- Vérifié le 2026-08-22 contre 9556f38. Géré par bmad-project-context ; les éditions
     à l'intérieur de ce bloc sont remplacées au refresh. Garder hors marqueurs
     ce qui doit survivre. -->

## acervatim-docs

Documentation centrale d'Acervatim : site Jekyll (just-the-docs) publié sur GitHub Pages,
plus `agent/` — l'index de la base de code destiné aux agents IA, exclu du site. Aucun
code applicatif ici ; l'app Android est dans `../acervatim-app`, l'API dans
`../acervatim-backend`.

## Policy

- Ne nommer aucun matériel ni hébergeur (Raspberry Pi, NAS, AWS…) dans une page : l'offre
  se décrit en trois modes, sans cible d'hébergement.
- Toute page non destinée au public (spec de travail, sprint, audit) s'ajoute à `exclude:`
  de `_config.yml` **dans le même commit** — sinon Pages la publie au push suivant.
- La politique de confidentialité canonique est `legal/confidentialite.md` ; ne pas en
  créer de copie ailleurs.

## Where things are

- Point d'entrée agent : `agent/INDEX.md` — lookup par intention dans
  `agent/transverse/ou-trouver.md`, règles transverses dans `agent/conventions.md`.
- Site public : `utilisateur/` · `developpeur/` · `legal/`.

## Conventions that differ from defaults

- `agent/` est un index dérivé, jamais une source : s'il contredit le code, le code gagne
  et c'est `agent/` qu'on corrige.

<!-- /bmad:context -->
