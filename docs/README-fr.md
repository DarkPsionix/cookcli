# Documentation en français

Ces documents complètent la documentation officielle de CookCLI, rédigée en
anglais (une page par commande, listées dans [`README.md`](README.md)).

## Pour utiliser CookCLI

| Document | Pour qui | Contenu |
|---|---|---|
| [Guide d'utilisation de l'interface web](guide-utilisation-web.md) | Toute personne qui utilise le site | Trouver une recette, cuisiner, faire les courses, garde-manger, planning des repas, éditeur, raccourcis, questions fréquentes |
| [Référence des formats de fichiers](reference-formats-fichiers.md) | Qui écrit ou modifie les fichiers à la main | Syntaxe des recettes et des menus acceptée par CookCLI, images, `aisle.conf`, `pantry.conf`, liste de courses, fichiers du serveur |

## Pour installer et administrer le serveur

| Document | Contenu |
|---|---|
| [Guide de déploiement et de sécurisation](guide-deploiement-securisation.md) | Docker ou binaire, reverse proxy HTTPS (Caddy, nginx), comptes et rôles, limite des tentatives de connexion, journal, sauvegardes, mises à jour, liste de vérification |
| [Authentification et autorisations](authentification-serveur.md) | Fonctionnement des comptes, des sessions, de la déconnexion, des rôles et de leurs limites |

## Pour comprendre le projet

| Document | Contenu |
|---|---|
| [Cartographie des fonctionnalités](cartographie-fonctionnalites.md) | Toutes les commandes et fonctionnalités, CLI, serveur et site statique, avec le code correspondant |
| [Cartographie des pages du site web](cartographie-pages-web.md) | Chaque page : route, contenu, actions, droits, appels d'API |
| [Pile technique du serveur](stack-technique-serveur.md) | Frameworks et bibliothèques : Axum, Askama, Tailwind, CodeMirror, temps réel, stockage, sécurité, build |

## Pour développer

| Document | Contenu |
|---|---|
| [Guide développeur du serveur](guide-developpeur-serveur.md) | Ajouter une route, une page, des traductions ou une commande en respectant les règles du projet |
| [Stratégie de tests](strategie-tests.md) | Tests Rust et Playwright : les lancer, les écrire, la CI, les pièges connus, la couverture par fonctionnalité |
| [Synchroniser le fork](synchronisation-fork.md) | Récupérer les nouveautés de `cooklang/cookcli` sans perdre son travail |

## Par où commencer ?

- **Vous découvrez le projet** : la
  [cartographie des fonctionnalités](cartographie-fonctionnalites.md).
- **Vous voulez juste cuisiner** : le
  [guide d'utilisation](guide-utilisation-web.md).
- **Vous installez le serveur pour d'autres personnes** : le
  [guide de déploiement](guide-deploiement-securisation.md), jusqu'à sa liste
  de vérification.
- **Vous modifiez le code** : le
  [guide développeur](guide-developpeur-serveur.md), puis la
  [stratégie de tests](strategie-tests.md).

## Documentation officielle (en anglais)

- [`README.md`](README.md) : index des commandes
- [`server.md`](server.md) : toutes les options de `cook server`
- [`api.md`](api.md) : référence complète de l'API HTTP
- [`../README.md`](../README.md) : présentation et installation
- [`../CONTRIBUTING.md`](../CONTRIBUTING.md) : contribuer au projet
