# Guide d'utilisation de l'interface web

Ce guide s'adresse à celles et ceux qui utilisent CookCLI depuis un navigateur,
sur ordinateur, tablette ou téléphone, pour trouver une recette, cuisiner,
faire les courses et planifier les repas. Aucune connaissance technique n'est
nécessaire.

L'adresse du site vous est donnée par la personne qui a installé le serveur
(par exemple `http://localhost:9080` ou `https://cuisine.example.com`).

## Sommaire

1. [Se repérer](#1-se-repérer)
2. [Trouver une recette](#2-trouver-une-recette)
3. [Lire une recette et l'adapter](#3-lire-une-recette-et-ladapter)
4. [Cuisiner avec le mode cuisine](#4-cuisiner-avec-le-mode-cuisine)
5. [Faire les courses](#5-faire-les-courses)
6. [Tenir le garde-manger](#6-tenir-le-garde-manger)
7. [Ranger les ingrédients par rayon](#7-ranger-les-ingrédients-par-rayon)
8. [Planifier les repas de la semaine](#8-planifier-les-repas-de-la-semaine)
9. [Écrire ou modifier une recette](#9-écrire-ou-modifier-une-recette)
10. [Réglages](#10-réglages)
11. [Raccourcis clavier](#11-raccourcis-clavier)
12. [Questions fréquentes](#12-questions-fréquentes)

---

## 1. Se repérer

En haut de chaque page :

- **Recettes** : votre collection.
- **Liste de Courses** : ce qu'il reste à acheter.
- **Garde-Manger** : ce que vous avez déjà chez vous.
- **Préférences** (icône d'engrenage) : langue et affichage.
- **La barre de recherche**.
- **Le bouton soleil / lune** : thème clair ou sombre.
- **Se connecter** : si le site demande un compte (voir ci-dessous).

Sur téléphone, ces liens sont regroupés derrière le bouton de menu.

### Avez-vous besoin d'un compte ?

Cela dépend de la façon dont le site a été installé :

- **Sans comptes** : tout le monde peut tout faire, rien à faire de spécial.
- **Avec comptes** : tout le monde peut lire les recettes, mais pour modifier
  quoi que ce soit, il faut **se connecter**. Ce que vous pouvez faire dépend
  du rôle que l'administrateur vous a donné :

  | Rôle | Vous pouvez |
  |---|---|
  | Lecteur | Lire les recettes |
  | Courses | … et gérer la liste de courses, le garde-manger et les rayons |
  | Éditeur | … et écrire, modifier et supprimer des recettes et des menus |
  | Administrateur | … et tout le reste |

Si une action vous est refusée, un bandeau l'explique, avec un lien pour vous
connecter si vous ne l'êtes pas. La connexion dure 30 jours sur un même
appareil. Pour un mot de passe oublié, adressez-vous à l'administrateur : il
n'y a pas de réinitialisation par e-mail.

---

## 2. Trouver une recette

![Liste des recettes](screenshots/recipe-list.png)

- **Parcourir** : la page Recettes montre les dossiers et les recettes sous
  forme de cartes. Cliquez sur un dossier pour y entrer ; le chemin en haut
  (fil d'Ariane) permet de remonter.
- **Trier** : par nom, date de création ou date de modification ; le bouton à
  côté inverse l'ordre.
- **Rechercher** : tapez dans la barre de recherche ; les résultats
  apparaissent au fil de la frappe. Appuyez sur Entrée pour voir tous les
  résultats. La recherche porte sur les titres, les ingrédients, les étapes et
  les étiquettes.
- **Se laisser surprendre** : le bouton **Recette au hasard** ouvre une
  recette tirée au sort dans le dossier affiché.
- **Menu du Jour** : si un planning de repas prévoit quelque chose pour
  aujourd'hui, un bandeau l'annonce en haut de la page Recettes.

![Recherche](screenshots/search.png)

---

## 3. Lire une recette et l'adapter

![Une recette](screenshots/recipe-detail.png)

Une recette présente :

- son titre, sa photo, sa description et ses informations (temps de
  préparation et de cuisson, étiquettes…) ;
- la liste des **ingrédients** (les ingrédients facultatifs sont signalés) et
  des **ustensiles** ;
- les **étapes**, avec les ingrédients, ustensiles et minuteurs mis en
  évidence dans le texte.

Une recette peut en utiliser une autre (une sauce, une pâte…) : son nom est
un lien vers elle.

**Changer les quantités** : utilisez les boutons **+** et **−** à côté du
nombre de portions (ou du facteur multiplicateur). Toutes les quantités sont
recalculées. Un rappel indique que la recette a été écrite pour un autre
nombre de portions : les temps de cuisson et la taille des plats peuvent
devoir être adaptés. Cliquez sur ce rappel pour revenir aux quantités
d'origine.

**Lancer un minuteur** : cliquez sur une durée dans une étape.

**Imprimer** : utilisez l'impression du navigateur, ou la touche `p`.

---

## 4. Cuisiner avec le mode cuisine

Sur une recette, cliquez sur le bouton **Cook** (icône de lecture ▶, ou touche `c`).

- La recette s'affiche **une étape à la fois**, en grand, avec en tête de
  chaque partie la liste des ingrédients à préparer (« mise en place »).
- **Passer à l'étape suivante** : glissez le doigt vers le haut (vers le bas
  pour revenir), ou utilisez les flèches du clavier.
- **Les minuteurs** de l'étape se lancent d'un clic.
- **L'écran reste allumé** pendant le mode cuisine, quand le navigateur le
  permet.
- **Quitter** : le bouton de fermeture, ou la touche Échap.

---

## 5. Faire les courses

![Liste de courses](screenshots/shopping-list.png)

### Remplir la liste

- Sur une recette : **Ajouter à la liste de courses**. Les quantités ajoutées
  sont celles du nombre de portions affiché : réglez-le avant.
- Sur un menu ou un planning : **Tout ajouter à la liste de courses**, ou
  seulement certains jours (voir § 8).

### Utiliser la liste

La page Liste de Courses :

- **additionne** les ingrédients identiques venant de plusieurs recettes ;
- les **range par rayon** du magasin (voir § 7) ;
- met à part ce que vous avez déjà, sous **Dans votre garde-manger** (voir
  § 6) ;
- rappelle en haut les recettes sélectionnées.

En magasin :

- **Cochez** chaque article dans votre panier.
- **La liste est partagée et se met à jour toute seule** : si quelqu'un coche
  un article sur son téléphone, vous le voyez aussitôt sur le vôtre.
- **Retirer une recette** : la croix à côté de son nom.
- **Copier** la liste, avec ou sans quantités et rayons, pour l'envoyer par
  message.
- **Tout effacer** une fois les courses terminées.

Un ingrédient qui n'est rangé dans aucun rayon peut être classé directement
depuis la liste.

---

## 6. Tenir le garde-manger

![Garde-manger](screenshots/pantry.png)

Le garde-manger est la liste de ce que vous avez chez vous. Tout ce qui s'y
trouve en quantité suffisante est **retiré de la liste de courses**.

- Les articles sont rangés par **emplacement** : frigo, congélateur, placard,
  épices…
- **Ajouter un article** : nom, emplacement, et si vous voulez :
  - la **quantité** (par exemple `2` litres ou `500` grammes) ;
  - un **seuil « bientôt épuisé »** : en dessous, l'article est signalé ;
  - la **date d'achat** et la **date de péremption**.
- **Modifier** ou **supprimer** un article en cliquant dessus.
- **Filtres** : **Tout**, **Bientôt épuisé**, **En rupture de stock**,
  **Expire bientôt**.
- Les dates de péremption s'affichent en clair : périmé, périme aujourd'hui,
  périme dans N jours.

Une quantité à `0` signifie « épuisé » : l'article reste alors sur la liste de
courses. Le mot `unlim` signifie « toujours disponible » (le thym du jardin,
l'eau…).

L'onglet **Texte** montre le fichier du garde-manger tel quel, pour les
personnes à l'aise avec son format.

---

## 7. Ranger les ingrédients par rayon

La page **Rayons** (accessible depuis la liste de courses ou les préférences)
décrit l'organisation de votre magasin, pour que la liste de courses suive
l'ordre de vos allées.

- **Onglet Rayons** : créez, renommez, supprimez et réordonnez les rayons ;
  ajoutez-y des ingrédients ou déplacez-les d'un rayon à l'autre (sélection
  multiple possible).
- **Regrouper** : si vos recettes écrivent tantôt « oignon », tantôt
  « oignons », regroupez-les sous un seul nom : la liste n'en affichera plus
  qu'un.
- **Onglet Sans rayon** : les ingrédients de vos recettes qui ne sont encore
  dans aucun rayon, à classer.
- **Onglet Texte** : le fichier tel quel.

---

## 8. Planifier les repas de la semaine

![Menu](screenshots/menu.png)

Un **menu** regroupe plusieurs recettes (un repas de fête, par exemple). Un
**planning de repas** les répartit dans un calendrier, jour par jour et repas
par repas.

### Créer un planning

Sur la page Recettes : **Nouveau Planning de repas**. Choisissez :

- le nom ;
- le premier jour (aujourd'hui, cette semaine, la semaine prochaine…) ;
- la durée (une ou deux semaines, ou un nombre de jours) ;
- les repas à prévoir (petit-déjeuner, déjeuner, dîner, en-cas) ;
- le nombre de portions.

### Remplir le calendrier

Le planning s'affiche comme un calendrier ; le jour courant est mis en
évidence et les jours passés sont grisés.

- **+** à côté d'un repas : choisissez une recette dans la fenêtre de
  recherche.
- **⋯** à côté d'un plat : **Déplacer vers…**, **Copier vers…** ou
  **Retirer**.
- **Glisser-déposer** un plat vers un autre repas pour le déplacer ; en
  maintenant Alt, Ctrl ou ⌘, pour le copier.

Si quelqu'un a modifié le planning entre-temps, votre modification est
refusée et la page vous invite à recharger : rien n'est écrasé.

### Faire les courses du planning

Cochez les jours voulus (raccourcis **Tous**, **7 prochains jours**,
**Aucun**), puis **Ajouter les jours à la liste de courses**. Ou ajoutez tout
le planning d'un coup avec **Tout ajouter à la liste de courses**.

---

## 9. Écrire ou modifier une recette

*Réservé au rôle Éditeur quand le site a des comptes.*

### Créer

Sur la page Recettes : **Nouvelle Recette** (ou **Nouveau Menu**). Donnez un
nom ; vous pouvez le placer dans un dossier en écrivant `Desserts/Tarte aux
pommes`. L'éditeur s'ouvre ensuite.

### L'éditeur

Sur une recette : **Modifier** (ou touche `e`).

Les recettes s'écrivent dans un format texte simple, le **Cooklang** :

```text
Faire fondre le @beurre{30%g} dans une #poêle{}.
Ajouter les @oignons{2}(émincés) et cuire ~{10%minutes}.
```

- `@` marque un ingrédient, `{…}` sa quantité, `%` sépare le nombre de
  l'unité, `(…)` sa préparation.
- `#` marque un ustensile, `~` un minuteur.
- Une ligne vide sépare deux étapes.

Pas besoin de tout retenir :

- **La barre d'outils** au-dessus du texte insère un ingrédient, un ustensile,
  un minuteur, une section, une note ou une référence à une autre recette.
- **L'éditeur propose des suggestions** au fil de la frappe (ingrédients déjà
  utilisés…) et **souligne les erreurs**.
- La [référence des formats](reference-formats-fichiers.md#1-recettes-cook)
  détaille toute la syntaxe.

Les autres actions de l'éditeur :

- **Enregistrer** ; l'état s'affiche (enregistrement en cours, enregistré,
  échec).
- **Photo** : ajouter ou changer la photo de la recette ou d'une étape. Les
  photos de téléphone sont réduites et **débarrassées de leur position GPS**.
  Sur la page d'une recette, l'icône d'appareil photo d'une étape ouvre
  directement ce choix.
- **Renommer** ou déplacer la recette.
- **Supprimer** la recette (une confirmation est demandée).

Si vous quittez la page sans enregistrer, le navigateur vous demande
confirmation.

---

## 10. Réglages

La page **Préférences** permet de :

- choisir la **langue** de l'interface (10 langues, dont le français) ; par
  défaut, c'est celle du navigateur ;
- choisir les **fonctionnalités affichées** dans la barre de navigation
  (masquer le garde-manger si vous ne l'utilisez pas, par exemple) ;
- voir où sont rangés les fichiers et la version installée ;
- pour l'administrateur : relier le site à un compte cook.md pour synchroniser
  les recettes.

Le **thème** clair ou sombre se change avec le bouton soleil / lune de
l'en-tête, ou la touche `t`.

---

## 11. Raccourcis clavier

Appuyez sur `?` pour afficher l'aide des raccourcis.

| Touche | Effet |
|---|---|
| `/` | Aller à la recherche |
| `g` puis `h` | Recettes |
| `g` puis `s` | Liste de courses |
| `g` puis `p` | Garde-manger |
| `g` puis `x` | Préférences |
| `t` | Thème clair / sombre |
| `Échap` | Fermer une fenêtre |

Sur une recette :

| Touche | Effet |
|---|---|
| `c` | Mode cuisine |
| `e` | Modifier |
| `a` | Ajouter à la liste de courses |
| `p` | Imprimer |
| `+` / `-` | Augmenter / diminuer les quantités |
| `]` / `[` | Augmenter / diminuer de 0,5 |

Dans le mode cuisine : flèches pour changer d'étape, `Échap` pour quitter.

---

## 12. Questions fréquentes

**Je ne vois pas le bouton Modifier (ou Ajouter à la liste de courses).**
Vous n'êtes pas connecté, ou votre rôle ne le permet pas. Connectez-vous, ou
demandez un autre rôle à l'administrateur.

**Un ingrédient n'apparaît pas dans ma liste de courses.**
Il est sans doute dans votre garde-manger en quantité suffisante : regardez la
partie « Dans votre garde-manger ». Mettez sa quantité à `0` s'il est épuisé.

**Le même ingrédient apparaît deux fois dans la liste.**
Il est écrit différemment dans deux recettes (« oignon » et « oignons »).
Regroupez les deux noms sur la page Rayons.

**Le stock « bientôt épuisé » ne s'affiche pas.**
La quantité et le seuil doivent avoir la même unité : `500 g` et `200 g`, pas
`1 kg` et `200 g`.

**Ma photo d'iPhone est refusée.**
Le format HEIC n'est pas lu par le serveur. Envoyez-la depuis Safari (qui la
convertit) ou réglez l'appareil photo sur « Le plus compatible »
(Réglages › Appareil photo › Formats).

**La photo d'une étape s'affiche sur la mauvaise étape.**
Une photo est liée au numéro de l'étape : si vous avez inséré une étape avant
elle, elle se retrouve décalée. Changez-la depuis l'éditeur.

**Ma modification du planning a été refusée.**
Quelqu'un d'autre l'a modifié pendant ce temps. Rechargez la page et
recommencez.
