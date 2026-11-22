# Référence des formats de fichiers

CookCLI ne stocke rien dans une base de données : tout est dans des fichiers
texte. Ce document décrit chacun d'eux, ce que CookCLI en accepte vraiment et
les pièges connus.

| Fichier | Rôle | Emplacement |
|---|---|---|
| [`*.cook`](#1-recettes-cook) | Une recette | Dossier de recettes, sous-dossiers compris |
| [`*.menu`](#2-menus-et-plans-de-repas-menu) | Un menu ou un plan de repas | Idem |
| [Images](#3-images) | Photo de titre ou d'étape | À côté de la recette |
| [`aisle.conf`](#4-rayons-aisleconf) | Rayons du magasin | `./config/` ou dossier global |
| [`pantry.conf`](#5-garde-manger-pantryconf) | Garde-manger | `./config/` ou dossier global |
| [`.shopping-list`](#6-liste-de-courses-shopping-list) | Ce qu'on veut acheter | Racine du dossier de recettes |
| [`.shopping-checked`](#7-articles-cochés-shopping-checked) | Ce qu'on a déjà acheté | Racine du dossier de recettes |
| [`users.toml` et fichiers de session](#8-fichiers-du-serveur-web) | Comptes du serveur web | Dossier global |

**Dossier global** : `$COOK_CONFIG_DIR` s'il est défini, sinon
`~/.config/cook/` (Linux), `~/Library/Application Support/cook/` (macOS),
`%APPDATA%\cook\config\` (Windows). Pour `aisle.conf` et `pantry.conf`,
`./config/` dans le dossier de recettes est prioritaire.

---

## 1. Recettes (`.cook`)

Les recettes suivent la [spécification Cooklang](https://cooklang.org/docs/spec/).
Exemple complet :

```cooklang
---
title: Risotto alla Milanese
servings: 4
tags: italien, riz
prep time: 10 min
cook time: 35 min
description: Un risotto crémeux au safran.
---

= Préparer la base =

Émincer l'@oignon{1}(finement). Chauffer le @bouillon de légumes{1%l}
dans une #casserole{} et le garder frémissant.

> Un bouillon maison fait toute la différence.

= Cuire le riz =

Faire fondre le @beurre{30%g}, ajouter l'oignon et cuire ~{5%minutes}.
Ajouter le @riz arborio{320%g} -- remuer sans cesse
Servir avec du @?parmesan{50%g} et une @./Sauces/Gremolata{}.
```

### Syntaxe acceptée

| Élément | Syntaxe | Exemple |
|---|---|---|
| Ingrédient (un mot) | `@nom` | `@sel` |
| Ingrédient (plusieurs mots ou quantité) | `@nom{quantité%unité}` | `@farine{250%g}`, `@œufs{3}`, `@huile d'olive{}` |
| Préparation de l'ingrédient | `(…)` juste après | `@ail{2%gousses}(écrasé)` |
| Ingrédient optionnel | `@?nom{…}` | `@?ciboulette` |
| Ustensile | `#nom{}` | `#poêle{}`, `#four` |
| Minuteur | `~nom{durée%unité}` ou `~{durée%unité}` | `~cuisson{20%minutes}` |
| Référence à une autre recette | `@./chemin{quantité}` | `@./Sauces/Béchamel{}`, `@./Pâte{2%servings}` |
| Section | `= Nom =` (ou `== Nom ==`) | `= Garniture =` |
| Note | ligne commençant par `>` | `> Se conserve 3 jours.` |
| Commentaire de ligne | `--` jusqu'à la fin de la ligne | `-- à vérifier` |
| Commentaire de bloc | `[- … -]` | `[- variante sans gluten -]` |
| Échappement | `\` devant un caractère | `\@` |
| Métadonnées | Bloc YAML entre `---` en tête de fichier | voir ci-dessous |

Les étapes sont séparées par une ligne vide.

### Ce que CookCLI n'accepte pas

CookCLI construit tous ses parseurs **sans aucune extension Cooklang**
(`Extensions::empty()`, `crates/core/src/parser.rs`). Ces syntaxes, acceptées
par d'autres outils Cooklang, restent du texte ici :

| Syntaxe | Effet dans CookCLI |
|---|---|
| Plage de valeurs `@riz{2-3%tasses}` | Pas une plage : `2-3` reste du texte |
| Fractions Unicode `@lait{½%l}` | Non reconnues ; écrire `1/2` |
| Unité sans `%` `@farine{250 g}` | Pas d'unité séparée ; écrire `250%g` |
| Référence `@&nom` à un ingrédient précédent | Non reconnue |
| Alias `@ingrédient\|alias{}` | Non reconnu |
| Mode par métadonnée `>> [mode]: …` | Non reconnu |

Autres points :

- **Les métadonnées `>> clé: valeur`** sont dépréciées : utilisez un bloc YAML.
  `cook doctor validate` les signale.
- **`--` et `[-` ouvrent un commentaire partout** sur la ligne, notes `>`
  comprises. Pour écrire « -- » dans le texte, échappez-le : `\--`.
- **Mise à l'échelle** : les quantités sont multipliées par le facteur. Une
  quantité fixe s'écrit avec `=` : `@sel{=1%pincée}` ne change pas.

### Métadonnées utilisées par CookCLI

Toute clé est permise ; celles-ci ont un effet :

| Clé | Effet |
|---|---|
| `title` | Titre affiché (sinon le nom du fichier) |
| `servings` | Nombre de portions de base, pour la mise à l'échelle par portions |
| `description` | Description sur la page et dans les flux |
| `tags` | Étiquettes affichées |
| `prep time`, `cook time` | Temps affichés |
| `image` | Image de titre (remplace `Recette.jpg`) |
| `date` | Date dans les flux et pour le tri (sinon la date du fichier) |
| `author` (ou `source.author`) | Auteur dans les flux |
| `source` | Source de la recette |

### Validation

```bash
cook doctor validate            # signale erreurs et avertissements
cook doctor validate --strict   # code de sortie 1 en cas de problème (CI)
cook recipe "Risotto.cook"      # affiche la recette telle qu'elle est comprise
```

---

## 2. Menus et plans de repas (`.menu`)

Un `.menu` utilise **la même syntaxe qu'une recette**. Il sert à regrouper des
références de recettes ; les ingrédients écrits directement sont permis.

```cooklang
---
servings: 2
---

== Samedi (2026-03-07) ==

Petit-déjeuner: \
- @flocons d'avoine{1%tasse} avec du @lait{1/2%tasse}

Déjeuner: \
- @./Côtelettes d'agneau{}

-- garder les restes pour des sandwichs

Dîner: \
- @./Pizza napolitaine{2%servings}

= 2026-03-08 Dîner

- @./Soupe{}
```

Règles :

- **C'est un plan de repas** dès que des sections portent une date
  `AAAA-MM-JJ` sur **au moins deux jours**. Il s'affiche alors en calendrier ;
  sinon c'est un menu ordinaire.
- Un texte après la date nomme le repas : `= 2026-03-08 Dîner`.
- Une ligne `Repas: \` ouvre un repas ; les lignes `- …` qui suivent en sont
  les plats. Le `\` en fin de ligne garde le titre et la liste dans la même
  étape.
- `@./Recette{2%servings}` demande la recette pour 2 portions ; `{}` garde son
  nombre de portions d'origine ; `{2}` multiplie par 2.
- Une section sans date est affichée sous « Hors de ce plan ».
- Le calendrier de l'interface web modifie seulement les lignes concernées et
  garde les commentaires.

---

## 3. Images

| Fichier | Rôle |
|---|---|
| `Recette.jpg` (ou `.jpeg`, `.png`, `.webp`) | Image de titre de `Recette.cook` |
| `Recette.S.N.jpg` | Photo de l'étape N de la section S (comptées à partir de 1) |
| `Recette.N.jpg` | Photo de l'étape N, comptée sur toute la recette (format ancien, toujours lu) |

L'interface web réenregistre toute image envoyée en JPEG, redressée, réduite à
2048 px et **sans ses métadonnées (y compris la position GPS)**. La limite
d'envoi est de 10 Mo. HEIC et AVIF ne sont pas lus par le serveur.

Une photo d'étape est liée à la **position** de l'étape : insérer une étape
avant elle la décale sur une autre étape.

---

## 4. Rayons (`aisle.conf`)

Trie la liste de courses par rayon. Format propre à Cooklang (ce n'est **pas**
du TOML) :

```text
// Rayons de mon supermarché
[fruits et légumes]
pommes
avocat | avocats
oignon | oignons

[crèmerie]
lait
beurre
```

| Élément | Syntaxe |
|---|---|
| Rayon | `[nom du rayon]` |
| Ingrédient | Un nom par ligne |
| Synonymes | `nom principal \| autre nom \| …` : la liste de courses les regroupe sous le premier nom |
| Commentaire | `//` **uniquement** |

Piège : **`#` et `--` ne sont pas des commentaires** dans ce fichier. Une ligne
`# Légumes` devient un ingrédient nommé « # Légumes ». `cook doctor aisle` et
la page Rayons signalent ces lignes.

Outils :

```bash
cook doctor aisle                  # ingrédients des recettes absents des rayons
cook doctor aisle --show-recipes   # … avec les recettes qui les utilisent
```

---

## 5. Garde-manger (`pantry.conf`)

Inventaire de ce qu'on a déjà. **Format TOML**. Un article présent est retiré
de la liste de courses.

```toml
# Mon garde-manger
[frigo]
lait = { quantity = "2%l", low = "500%ml", expire = "2026-12-01" }
"œufs" = { quantity = "12", low = "6", bought = "2026-11-20" }
beurre = "250%g"

[placard]
"riz arborio" = { quantity = "1%kg" }
farine = { quantity = "2%kg", low = "500%g" }
"tomates en boîte" = { quantity = "0" }

[jardin]
thym = { quantity = "unlim" }
```

| Attribut | Rôle |
|---|---|
| `quantity` | Quantité en stock, au format `nombre%unité` |
| `low` | Seuil de stock bas, dans la même unité |
| `bought` | Date d'achat |
| `expire` | Date de péremption |

Règles :

- **Forme courte** : `beurre = "250%g"` équivaut à
  `beurre = { quantity = "250%g" }`.
- **`unlim` ou `unlimited`** : toujours disponible, sans quantité mesurable.
- **`"0"`** : épuisé. L'ingrédient reste sur la liste de courses.
- **Dates** : `AAAA-MM-JJ` de préférence. Sont aussi lus, dans cet ordre :
  `JJ.MM.AAAA`, `JJ/MM/AAAA`, `MM/JJ/AAAA`, `AAAA.MM.JJ`, `JJ-MM-AAAA`. Une date
  comme `01/02/2025` est donc lue comme le 1er février.
- **Les noms avec espaces, accents ou autres caractères spéciaux** se mettent
  entre guillemets (`"riz arborio"`, `"œufs"`) : en TOML, une clé sans
  guillemets n'admet que `A-Z a-z 0-9 _ -`.
- **Le stock bas ne se calcule que si les unités sont identiques** : `g` avec
  `g`, pas `kg` avec `g`.
- Une quantité illisible (autre que `unlim`) est ignorée, avec un
  avertissement.
- Un article placé avant le premier `[section]` ne peut avoir qu'une quantité.
- CookCLI modifie ce fichier avec `toml_edit` : **commentaires, lignes vides et
  ordre des clés sont conservés.**

```bash
cook pantry list           # tout le garde-manger
cook pantry depleted       # épuisé ou sous le seuil
cook pantry expiring       # bientôt périmé
cook pantry recipes        # recettes réalisables avec le stock
```

---

## 6. Liste de courses (`.shopping-list`)

Écrit par l'interface web, et partagé avec les applications Cooklang : le
format doit rester exactement celui-ci. **Il ne contient pas d'ingrédients
calculés**, seulement ce qui a été demandé ; la liste à acheter est recalculée
à chaque affichage.

```text
./Risotto.cook{2}
./Steak.cook
  ./Sides/Riz
  ? ./Sauces/Chimichurri{2}
  ? parmesan{50%g}
./Plans/Semaine 1.menu
  ./Soupe.cook
sel
lait{1%l}
```

| Ligne | Signification |
|---|---|
| `./chemin` | Une recette ou un menu, chemin relatif au dossier de recettes |
| `{n}` après le chemin | Multiplicateur (`{2}`, `{0.5}`) ; absent = 1 |
| Ligne indentée sous une recette | Sous-recette référencée à inclure. Sans ligne indentée, **toutes** les sous-recettes sont incluses |
| Ligne indentée sous un menu | Recette du menu |
| `? ./chemin{n}` (indenté) | Recette optionnelle choisie |
| `? nom{quantité}` (indenté) | Ingrédient optionnel choisi, avec la quantité finale à acheter (non multipliée) |
| Ligne sans `./` | Article libre, avec éventuellement `{quantité}` |

L'ancien fichier `.shopping_list.txt` (colonnes séparées par des
tabulations) est converti automatiquement au premier chargement, puis renommé.

---

## 7. Articles cochés (`.shopping-checked`)

Journal **en ajout seul** de ce qui a été coché pendant les courses :

```text
+ farine
+ lait
- lait
+ beurre
```

- `+ nom` coche un article, `- nom` le décoche ; la dernière ligne pour un nom
  l'emporte. Ici : farine et beurre cochés, lait non coché.
- La comparaison des noms ignore la casse.
- Une ligne illisible (par exemple tronquée) est ignorée.
- L'action « compacter » de l'interface réécrit le journal en ne gardant que
  l'état utile ; « vider la liste » le vide.

---

## 8. Fichiers du serveur web

Tous dans le dossier global (ou à l'emplacement donné par `--users-file` /
`COOK_USERS_FILE` pour `users.toml`). **Ils ne doivent jamais être dans le
dossier de recettes** : le serveur refuse de démarrer dans ce cas.

| Fichier | Contenu | Sensible |
|---|---|---|
| `users.toml` | Comptes : hash Argon2 et rôle de chacun | Oui |
| `auth-secret` | Clé de signature des cookies de session (droits `0600`) | **Très** : permet de fabriquer une session |
| `revoked-sessions` | Sessions déconnectées avant leur expiration | Non |
| `session.json` | Connexion au compte cook.md (`cook login`) | Oui |
| `sync.db` | Base SQLite de l'état de synchronisation cook.md | Non |

Format de `users.toml` :

```toml
[users]
alice = "$argon2id$v=19$m=19456,t=2,p=1$…"          # hash seul : rôle admin
bob   = { hash = "$argon2id$…", role = "editor" }   # reader, shopper, editor, admin
```

Gérez-le avec `cook server user add | passwd | role | remove | list`. Pour
l'éditer à la main, `cook server hash-password` produit un hash. Les
commentaires sont conservés et les modifications prises en compte sans
redémarrer le serveur.

Voir [authentification et autorisations](authentification-serveur.md).
