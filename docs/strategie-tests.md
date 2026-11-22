# Stratégie de tests

CookCLI a deux familles de tests automatisés : des **tests Rust** (environ
1 500, unitaires et d'intégration) et des **tests de bout en bout Playwright**
(un peu moins de 300, dans 34 fichiers) qui pilotent un vrai navigateur.

Voir aussi : [guide développeur](guide-developpeur-serveur.md).

## 1. Vue d'ensemble

| Niveau | Emplacement | Outil | Ce qu'il vérifie |
|---|---|---|---|
| Unitaire | Modules `#[cfg(test)]` dans `src/` et `crates/` ; `crates/core/src/pantry/tests.rs`, `crates/core/src/shopping_list/tests.rs` | `cargo test` | Fonctions isolées : parseurs, calculs, middlewares, signature des sessions… |
| Intégration CLI | `tests/*.rs` (49 fichiers) | `cargo test`, `assert_cmd`, `predicates` | Le binaire `cook` lancé comme un utilisateur le ferait |
| Instantanés | `tests/snapshots/` | `insta` | Sorties complètes comparées à une référence enregistrée |
| API du serveur | `tests/*_api_test.rs`, `server_*_test.rs`… | `reqwest`, `tokio-tungstenite` | Un vrai `cook server` lancé sur un port libre, interrogé en HTTP ou WebSocket |
| Bout en bout | `tests/e2e/*.spec.ts` | Playwright (Chromium) | L'interface web dans un navigateur |
| Accessibilité | `tests/e2e/accessibility.spec.ts` | `@axe-core/playwright` | WCAG 2.0 AA |
| Cohérence | `tests/locales_test.rs`, tests de `src/web/api_docs.rs` | `cargo test` | Traductions complètes, documentation de l'API à jour |

## 2. Lancer les tests

### Rust

```bash
cargo test                          # tout (s'arrête au premier binaire en échec)
cargo test --no-fail-fast           # tout, en montrant tous les échecs
cargo test --test pantry_api_test   # un seul fichier de tests/
cargo test nom_du_test              # tests dont le nom contient ce texte
cargo test -- --nocapture           # avec la sortie des tests
cargo test --no-default-features    # sans sync, import, lsp, self-update (comme en CI)
```

Les tests Rust compilent le binaire : les fichiers front doivent exister
(`npm ci && npm run build-css && npm run build-js`).

### Playwright

```bash
npm test                     # tous les tests, Chromium
npm run test:ui              # mode interactif
npm run test:headed          # navigateur visible
npm run test:debug           # pas à pas
npx playwright test tests/e2e/pantry.spec.ts      # un seul fichier
npx playwright test --workers=1                    # sans parallélisme
npm run test:report          # rapport HTML du dernier passage
```

Playwright démarre lui-même `cook server ./seed --port 9080`. En local il
compile d'abord CSS, JS et binaire, et réutilise un serveur déjà lancé sur le
port 9080.

### Avant de proposer une modification

```bash
cargo fmt
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --no-fail-fast
npm test        # si l'interface ou ce qu'elle affiche a changé
```

## 3. Ce qui tourne en CI

Le workflow `.github/workflows/test.yml` s'exécute sur chaque push et pull
request vers `main`.

| Job | Système | Étapes |
|---|---|---|
| Test | Linux | `cargo fmt --check`, `cargo clippy -D warnings`, `cargo test --workspace`, puis `cargo test --no-default-features` ; les écarts d'instantanés sont publiés en artefact |
| UI Tests | Linux | Build debug, Playwright Chromium avec **un seul worker** et 2 nouvelles tentatives ; rapport et vidéos des échecs en artefact |
| Test on macOS / Windows | macOS, Windows | `cargo test --workspace` |

Autres workflows : `docker.yml` (construction de l'image), `deb.yaml` et
`rpm.yaml` (paquets), `release.yaml` (publication), `workflow-lint.yml`
(actionlint et zizmor sur les fichiers de `.github/`).

Pour vérifier les workflows en local :

```bash
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12
docker run --rm -v "$PWD:/repo" -w /repo ghcr.io/zizmorcore/zizmor:1.30.1 --min-severity=low .github/workflows
```

## 4. Écrire un test Rust

### Un test qui lance `cook`

**Tout test qui lance `cook`, et en particulier `cook server`, doit isoler sa
configuration** avec `common::with_isolated_config` (`tests/common/mod.rs`).
Cette fonction pointe `COOK_CONFIG_DIR` vers un dossier du `TempDir` du test.

Ce n'est pas une question de propreté. `cook server` lit `session.json` au
démarrage et, s'il en trouve un, synchronise le dossier servi avec le
`sync.db` global. Sans isolation, un `cargo test` sur la machine d'une personne
qui a fait `cook login` peut **envoyer les recettes de test sur son vrai compte
cook.md**, ou propager des suppressions. Changer `HOME` ou `XDG_CONFIG_HOME` ne
suffit pas (Windows les ignore). `tests/config_dir_env_test.rs` garde ce
comportement.

Aides disponibles dans `tests/common/mod.rs` :

| Fonction | Rôle |
|---|---|
| `with_isolated_config` | Isole la configuration d'une commande |
| `setup_test_recipes` | Crée un dossier temporaire avec des recettes |
| `create_test_menu`, `create_test_reports` | Ajoutent un menu ou des modèles de rapport |
| `create_test_context` | Construit un `Context` pour appeler une commande en Rust |
| `run_command_with_args` | Lance une commande et récupère sa sortie |

### Un test d'API

Modèle : `tests/search_page_test.rs`. Le test réserve un port libre, lance
`cook server` sur un dossier temporaire, l'interroge avec `reqwest`, et un
`ServerGuard` tue le serveur à la fin, même si le test échoue. Marquez le
fichier `#![cfg(feature = "server")]`.

### Les instantanés

```bash
cargo test                       # un écart fait échouer le test
cargo insta review               # examiner et accepter les nouvelles sorties
INSTA_UPDATE=always cargo test   # tout accepter (à relire ensuite avec git diff)
```

### Les chemins

**Construisez un chemin attendu avec le séparateur de la plateforme**
(`Path::join`, ou `.replace('/', std::path::MAIN_SEPARATOR_STR)` comme dans
`tests/doctor_characterization_test.rs`). CookCLI affiche des chemins natifs :
un `"config/pantry.conf"` écrit en dur échoue seulement sur Windows, donc
seulement en CI. Les URL et chemins d'API, que le code normalise en `/`, font
exception.

## 5. Écrire un test Playwright

```typescript
import { test, expect } from '@playwright/test';
import { TestHelpers } from '../fixtures/test-helpers';

test.describe('Ma fonctionnalité', () => {
  let helpers: TestHelpers;

  test.beforeEach(async ({ page }) => {
    helpers = new TestHelpers(page);
    await helpers.navigateTo('/');
  });

  test('affiche quelque chose', async ({ page }) => {
    await expect(page.getByRole('heading', { name: 'Recettes' })).toBeVisible();
  });
});
```

- Préférez les sélecteurs sémantiques (`getByRole`, `getByLabel`) et les
  attributs `data-*` aux classes CSS.
- `tests/fixtures/test-helpers.ts` fournit `TestHelpers`, `RecipePage` et
  `ShoppingListPage`.
- **Un test qui simule des cases cochées par la réponse de
  `POST /api/shopping_list` doit aussi intercepter
  `GET /api/shopping_list/checked`** : le flux d'événements de la page relit
  les cases cochées à son ouverture.
- **Quand une modification touche ce que plusieurs pages partagent** (URL,
  templates, badges), lancez **tous** les tests Playwright : d'autres fichiers
  vérifient aussi ces éléments.

## 6. Pièges connus

| Symptôme | Cause | Solution |
|---|---|---|
| `seed/config/pantry.conf` ou `aisle.conf` modifiés après les tests | Les tests du garde-manger et des rayons écrivent dans `./seed` | `git status`, puis `git checkout seed/config/` |
| Échec aléatoire d'un test de liste de courses ou de garde-manger en local | Ces tests partagent les fichiers de `./seed` et tournent en parallèle | Relancer avec `--workers=1` avant de soupçonner le code |
| Deux tests de permissions de `cookcli-core` échouent | Ils vérifient des droits de fichiers, que root ignore | Normal en conteneur root ; leur message le dit |
| `cargo test` s'arrête au premier échec | Comportement par défaut | `--no-fail-fast` |
| Un test passe en local mais échoue sur Windows | Chemin écrit avec `/` | Voir § 4, les chemins |
| `locales_test` échoue après une fusion | Deux branches ont ajouté la même clé de traduction | Supprimer le doublon dans chaque langue |
| Les tests de deux copies du dépôt se marchent dessus | Même `CARGO_TARGET_DIR` : l'une remplace le binaire de l'autre | Un dossier `target` par copie |
| Build refusé : `output.css` ou `editor.bundle.js` manquant | Fichiers générés, absents d'un clone neuf | `npm ci && npm run build-css && npm run build-js` |

## 7. Couverture par fonctionnalité

| Fonctionnalité | Tests Rust | Tests Playwright |
|---|---|---|
| Lecture et formats de recette | `recipe_test`, `output_formats_test`, `document_formats_test`, `schema_output_test`, `fractions_test`, `snapshot_test` | `recipe-display`, `recipe-scaling`, `reference-scale` |
| Liste de courses | `shopping_list_*_test` (9 fichiers) | `shopping-list`, `-live`, `-copy`, `-remove`, `-pantry` |
| Garde-manger | `pantry_test`, `pantry_api_test`, tests de `cookcli-core` | `pantry`, `pantry-file` |
| Rayons | `aisles_api_test`, `doctor_characterization_test` | `aisles` |
| Menus et plans | `menu_api_test`, `menu_in_menu_test`, `new_menu_test`, `plan_edit_test`, `plan_shopping_test` | `meal-plan`, `menu-editor`, `menu-scale`, `plan-editing`, `plan-shopping` |
| Éditeur | `recipe_file_api_test`, `recipe_rename_api_test`, `recipe_image_api_test`, `step_images_test`, `lsp_sessions_test` | `editor*` (6 fichiers), `new-file-names` |
| Recherche | `search_page_test` | `search`, `static-file-search` |
| Sécurité | `server_auth_test`, `cors_test`, `path_traversal_test`, `shopping_list_path_traversal_test`, `server_symlinks_test`, `config_dir_env_test` | `sign-in-notice` |
| Site statique et flux | `build`, `build_reproducible_test`, `server_feed_test` | — |
| Divers | `cli_integration_test`, `completions_test`, `report_test`, `random_recipe_test`, `locales_test`, `api_docs_md_test` | `navigation`, `preferences`, `cooking-mode`, `unsaved-changes`, `api-docs`, `accessibility`, `performance` |
