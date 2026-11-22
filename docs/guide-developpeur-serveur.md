# Guide développeur : ajouter une fonctionnalité au serveur web

Ce guide explique comment ajouter une page, une route d'API ou une commande à
CookCLI en respectant les règles que le code impose. Beaucoup de ces règles
sont vérifiées par des tests : les ignorer fait échouer `cargo test` ou la CI.

Lectures utiles avant de commencer :
[pile technique](stack-technique-serveur.md) ·
[authentification](authentification-serveur.md) ·
[cartographie des pages](cartographie-pages-web.md) ·
[stratégie de tests](strategie-tests.md) ·
`CLAUDE.md` et `CONTRIBUTING.md` à la racine (en anglais).

## 1. Préparer l'environnement

```bash
npm ci                       # dépendances front (Tailwind, esbuild, CodeMirror, Playwright)
npm run build-css            # produit static/css/output.css
npm run build-js             # produit static/js/editor.bundle.js
cargo build                  # build.rs refuse de compiler sans les deux fichiers ci-dessus
cargo run -- server ./seed   # serveur sur http://localhost:9080 avec les recettes d'exemple
```

Pendant le développement, dans deux autres terminaux :

```bash
npm run watch-css
npm run watch-js
```

| Je modifie… | Il faut… |
|---|---|
| Du CSS ou du JS dans `static/` | Rafraîchir le navigateur (en build debug, `static/` est lu sur le disque) |
| Un template `templates/*.html` | **Recompiler et redémarrer** : Askama compile les templates dans le binaire |
| Du Rust | Recompiler et redémarrer |
| Un fichier `locales/*.ftl` | Recompiler et redémarrer |

## 2. Où mettre le code

| Couche | Emplacement | Contient |
|---|---|---|
| Logique métier | `crates/core/src/` (`cookcli-core`) | Ce qui sert à la fois à la CLI et au serveur : recherche, liste de courses, garde-manger, rayons, doctor… |
| Formats de sortie | `crates/format/src/` | Formateurs publiés (markdown, latex, typst, schema.org…) |
| Commande CLI | `src/<commande>.rs` + `src/args.rs` + `src/main.rs` | Arguments clap et appel à `cookcli-core` |
| Route d'API | `src/server/handlers/<domaine>.rs`, enregistrée dans `api()` de `src/server/mod.rs` | Handler Axum qui renvoie du JSON |
| Page HTML | handler dans `src/server/ui.rs` (route dans `ui()`), structure dans `src/web/templates.rs`, gabarit dans `templates/` | Rendu Askama |
| Construction des données d'une page | `src/web/builders.rs` | Partagée par le serveur et le site statique (`cook build web`) |
| Droits | `src/server/auth/middleware.rs` | Tables `WRITES`, `PROTECTED_READS`, `RECIPE_READS` |
| Traductions | `locales/<langue>/*.ftl` | 10 langues |
| Script de page | `<script>` dans le template, ou `static/js/` ; module bundlé dans `static/js/src/` | JS natif, sans framework |

**Règle** : si la logique peut servir à la CLI, mettez-la dans `cookcli-core`
et faites du handler une fine couche HTTP. C'est déjà le cas pour la
recherche : la page `/search`, la liste déroulante et `cook search` appellent
la même fonction et trouvent donc les mêmes recettes.

## 3. Ajouter une route d'API

### 3.1 Écrire le handler

Modèle tiré de `src/server/handlers/pantry.rs` (`add_item`) :

```rust
use super::common::{json_error, ApiError};
use crate::server::{activity, AppState};
use crate::web::viewer::Viewer;
use axum::{extract::State, http::StatusCode, response::IntoResponse, Extension, Json};
use std::sync::Arc;

pub async fn add_item(
    State(state): State<Arc<AppState>>,
    Extension(viewer): Extension<Viewer>,      // obligatoire pour une écriture
    Json(item): Json<AddPantryItem>,
) -> Result<impl IntoResponse, ApiError> {
    let item = item
        .trimmed()
        .map_err(|e| (StatusCode::BAD_REQUEST, json_error(e)))?;   // jamais e.to_string() à la main

    // … la modification, de préférence via cookcli-core …

    // Une fois la modification réussie, et seulement alors :
    activity::record(
        &viewer,
        format_args!("added {} to the pantry", activity::quoted(&item.name)),
    );

    Ok(Json(ApiResponse { success: true, message: "…".into() }))
}
```

Les règles :

| Règle | Pourquoi |
|---|---|
| **Toute erreur envoyée au client passe par `json_error`** (API) ou `ui::error_page` (page). Jamais `e.to_string()` dans une réponse. | Ces fonctions passent le message dans `private_paths::hide`, qui retire les chemins absolus. Le chemin du dossier de recettes ne doit jamais apparaître dans une réponse. |
| **Un handler qui modifie quelque chose prend `Extension<Viewer>` et appelle `activity::record`** après le succès. | Le journal d'activité dit qui a fait quoi. |
| **Le texte venant de la requête est cité avec `activity::quoted` ou `activity::file`.** | Un saut de ligne dans un nom ne peut pas fabriquer une fausse ligne de journal. |
| **Un handler `GET` ne modifie jamais rien.** | Les lectures ne sont pas protégées par la vérification d'origine ni par les droits d'écriture. |
| **Un chemin reçu passe par `handlers::common::is_recipe_request`** (recettes) ou `cookcli_core::find::is_safe_relative_path`. | Il doit rester dans le dossier de recettes, ne viser aucun fichier caché (`.git`, `.env`…) et, pour une recette, finir par `.cook` ou `.menu`. |
| **Un fichier du dossier de recettes est servi par `/api/static`** (`server/static_files.rs`), jamais par un nouveau `ServeDir`. | `/api/static` ne sert que des images et refuse les noms commençant par `.`. |
| **Une écriture de fichier passe par `fs_atomic`** (fichier temporaire puis renommage). | Un arrêt brutal ne laisse pas un fichier à moitié écrit. |
| **Deux écritures concurrentes sur le même fichier sont sérialisées** par un `tokio::sync::Mutex` dans `AppState` (voir `checked_log_lock`, `aisle_lock`). | `flock` ne protège pas deux tâches d'un même processus. |
| **Rien de ce dont le serveur a besoin n'est sous `#[cfg(feature = "sync")]`.** | L'image Docker est compilée sans `sync` (`--no-default-features --features server,lsp`). |

### 3.2 Enregistrer la route

1. Exportez le handler dans `src/server/handlers/mod.rs`.
2. Ajoutez la route dans `api()` de `src/server/mod.rs` :

   ```rust
   .route("/pantry/add", post(handlers::add_pantry_item))
   ```

### 3.3 Déclarer les droits

Dans `src/server/auth/middleware.rs`, ajoutez l'écriture à la table `WRITES`
avec la capacité qu'elle mérite :

```rust
const WRITES: &[(&str, Capability)] = &[
    ("/api/shopping_list", Capability::EditLists),   // shopper
    ("/api/pantry", Capability::EditLists),
    ("/api/recipes", Capability::EditRecipes),       // editor
    // …
];
```

- Une écriture **absente** de `WRITES` exige `Administer` (admin) : un oubli
  ne se voit qu'aux utilisateurs non admin, qui reçoivent 403.
- Un préfixe couvre toutes les routes en dessous : `/api/pantry` couvre
  `/api/pantry/add`.
- Une lecture qui doit être protégée va dans `PROTECTED_READS`.
- Une lecture que les visiteurs de `--recipes-only` doivent pouvoir faire va
  dans `RECIPE_READS`. Sinon ils sont invités à se connecter.

Côté template, cachez les contrôles que l'utilisateur ne peut pas utiliser
avec `viewer.can_edit_lists()`, `viewer.can_edit_recipes()` ou
`viewer.can_admin()`. Ce n'est qu'un confort : la vraie protection est le
middleware.

### 3.4 Documenter la route

Ajoutez une entrée dans `src/web/api_docs.rs` (page `/api-docs`), avec des
exemples **vérifiés sur un serveur réel**. Le test
`every_router_path_is_documented` échoue si une route de `api()` n'est pas
documentée, `no_documented_path_is_stale` si une entrée désigne une route
disparue. Puis régénérez `docs/api.md` :

```bash
UPDATE_API_DOCS=1 cargo test --test api_docs_md_test
```

Attention : le test ne compare que les chemins, pas les méthodes HTTP. Une
méthode fausse dans la documentation n'est pas détectée.

## 4. Ajouter une page

1. **Structure du template** dans `src/web/templates.rs`, sur le modèle de
   `SearchTemplate` :

   ```rust
   #[cfg(feature = "server")]
   #[derive(Template)]
   #[template(path = "ma_page.html")]
   pub struct MaPageTemplate {
       pub active: String,             // entrée de navigation mise en évidence
       pub tr: Tr,                     // traductions
       pub prefix: String,             // --url-prefix, à mettre devant chaque lien
       pub static_mode: bool,          // false dans le serveur
       pub repo_url: Option<String>,
       pub features: FeatureFlags,     // fonctionnalités affichées
       pub viewer: Viewer,             // qui regarde, pour cacher les contrôles
       // … vos données …
   }
   ```

2. **Gabarit** `templates/ma_page.html` :

   ```html
   {% extends "base.html" %}
   {% block title %}{{ tr.t("ma-page-titre") }} - Cook{% endblock %}
   {% block content %}
     <h1 class="text-display font-bold text-text">{{ tr.t("ma-page-titre") }}</h1>
     <a href="{{ prefix }}/recipe/…">…</a>
   {% endblock %}
   ```

3. **Handler** dans `src/server/ui.rs`, avec les extensions
   `Extension<LanguageIdentifier>`, `Extension<FeatureFlags>` et
   `Extension<Viewer>`, puis la route dans `ui()`.
4. **Droits** : une page en lecture n'a rien à déclarer, sauf si elle doit
   être réservée (`PROTECTED_READS`) ou visible en `--recipes-only`
   (`RECIPE_READS`).

Règles propres aux pages :

- **Tous les liens commencent par `{{ prefix }}`**, sinon la page casse avec
  `--url-prefix`.
- **Le JS qui appelle l'API utilise aussi le préfixe** :
  `fetch('{{ prefix }}/api/…')`.
- **Ne passez jamais `cond && node`** à `replaceChildren` ou `append` : le DOM
  écrit `false` en texte. Filtrez d'abord, comme `el()` / `fill()` dans
  `templates/aisles.html`.
- **Données saisies non envoyées** : marquez le formulaire ou le champ avec
  `data-unsaved`, ou la page avec `data-dirty`, ou utilisez
  `CookUnsaved.watch(fn)`. Une page qui navigue seule après un enregistrement
  utilise `CookUnsaved.leave(url)` ou `CookUnsaved.reload()`, pas `location`.
- **Styles** : utilisez les classes de `static/css/components.css` (`.card`,
  `.btn`, `.btn-primary`, `.btn-danger`, `.icon-btn`, `.row`, `.select`…) et
  les jetons de couleur de `static/css/input.css`. Un seul `.btn-primary` par
  vue.
- **Si la page doit aussi exister dans le site statique** (`cook build web`),
  construisez ses données dans `src/web/builders.rs` et tenez compte de
  `static_mode` (pas d'API, pas d'écriture).

## 5. Ajouter des textes traduits

Tout texte visible passe par Fluent :

```ftl
# locales/en-US/common.ftl
ma-page-titre = My page
recettes-trouvees = { $count ->
    [one] { $count } recipe
   *[other] { $count } recipes
}
```

```html
{{ tr.t("ma-page-titre") }}
{{ tr.tn("recettes-trouvees", count) }}
```

- **Ajoutez chaque nouveau message aux 10 langues** : de-DE, el-GR, en-US,
  es-ES, eu-ES, fr-FR, it-IT, ja-JP, nl-NL, sv-SE. `tests/locales_test.rs`
  échoue sinon.
- **Un même identifiant défini deux fois** fait planter toutes les pages de
  cette langue. Deux branches qui ajoutent la même clé passent leurs tests
  séparément : **relancez le test après chaque fusion**.
- Une accolade littérale s'écrit `{"{"}`.
- Askama passe les champs par référence : les helpers prennent `&usize` ou
  `&f64`.

## 6. Ajouter une commande CLI

1. Variante dans l'enum `Command` de `src/args.rs`, avec une aide détaillée
   et des exemples (le texte `///` devient l'aide de `cook <commande> --help`).
2. Module `src/<commande>.rs` avec une structure `XxxArgs` (clap) et
   `pub fn run(ctx: &Context, args: XxxArgs) -> Result<()>`.
3. Cas dans le `match` de `src/main.rs`.
4. La logique dans `cookcli-core`.
5. Une page `docs/<commande>.md` et une ligne dans `docs/README.md`.

Les complétions shell (`cook completions`) sont générées depuis clap : rien à
faire de plus.

## 7. Contraintes à ne pas casser

- **Parseur sans extensions** : tous les parseurs Cooklang sont construits
  avec `Extensions::empty()`. N'en activez aucune sans changement de la
  spécification.
- **Coloration de l'éditeur** (`static/js/src/cooklang-mode.js`) : elle doit
  suivre les mêmes règles que le parseur (`--` et `[-` ouvrent un commentaire
  partout, `\` échappe un caractère).
- **Chemins de références** : ils gardent leur `./` initial. Pour comparer deux
  références, retirez `./` et l'extension.
- **`pantry.conf`** se modifie uniquement par `cookcli-core` (`toml_edit`),
  pour garder les commentaires.
- **`.shopping-list` et `.shopping-checked`** sont partagés avec les
  applications Cooklang : leur format ne doit pas changer.

## 8. Avant de proposer la modification

```bash
cargo fmt
cargo clippy                    # aucun avertissement accepté
cargo test --no-fail-fast       # tous les binaires de test
npm test                        # Playwright, si l'interface a changé
```

Liste de vérification d'une nouvelle route ou page :

- [ ] Logique dans `cookcli-core` si elle peut servir à la CLI.
- [ ] Écriture déclarée dans `WRITES` avec la bonne capacité.
- [ ] `Extension<Viewer>` et `activity::record` sur chaque modification.
- [ ] Erreurs via `json_error` ou `error_page`.
- [ ] Chemins vérifiés (`is_recipe_request` / `is_safe_relative_path`).
- [ ] Liens et `fetch` préfixés par `{{ prefix }}`.
- [ ] Contrôles cachés selon `viewer` ; ajouté à `RECIPE_READS` si les
      visiteurs `--recipes-only` doivent y accéder.
- [ ] Messages ajoutés aux 10 langues.
- [ ] Route documentée dans `api_docs.rs` et `docs/api.md` régénéré.
- [ ] Tests Rust (API) et Playwright (interface) ajoutés.
- [ ] Rien d'indispensable sous `#[cfg(feature = "sync")]`.
- [ ] Message de commit sémantique : `feat:`, `fix:`, `docs:`, `chore:`.
