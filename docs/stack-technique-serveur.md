# Pile technique du serveur web (`cook server`)

Le serveur est un **monolithe Rust** qui sert des pages HTML générées côté
serveur. Il n'utilise **aucune base de données** : tout l'état vit dans des
fichiers texte du dossier de recettes. Les ressources front (CSS, JS) sont
**compilées dans le binaire**. Le code serveur fait environ 24 000 lignes
(`src/server/` et `src/web/`) et n'est compilé qu'avec la feature Cargo
`server`.

Voir aussi : [cartographie des fonctionnalités](cartographie-fonctionnalites.md).

## 1. Le socle HTTP

| Couche | Technologie | Rôle |
|---|---|---|
| Runtime asynchrone | **Tokio 1** (feature `full`) | Boucle d'événements, `TcpListener`, `Mutex` async, arrêt propre |
| Framework web | **Axum 0.8** (feature `ws`) | Routeur, extracteurs, état partagé `AppState` en `Arc`, WebSocket, SSE (événements poussés par le serveur) |
| Middlewares | **Tower 0.5** et **tower-http 0.7** | `ServeDir`, `CorsLayer`, `NormalizePath`, `SetResponseHeader`, logs (`trace`) |
| Sérialisation | **serde**, `serde_json`, `serde_yaml`, `toml`, `toml_edit` | API JSON ; modification de `pantry.conf` en gardant ses commentaires |
| Logs | **tracing** et `tracing-subscriber` | Filtrage par `RUST_LOG` |

Les middlewares s'empilent comme suit dans `src/server/mod.rs`, de
l'extérieur vers l'intérieur :

```
NormalizePath (supprime le « / » final)
 └ host_guard        → refuse un Host inconnu (protection contre le DNS rebinding)
   └ CorsLayer       → seulement si --cors-origin est passé
     └ write_guard   → refuse une écriture venant d'une autre origine (anti-CSRF)
       └ language_middleware / features_middleware → langue et options de la page
         └ DefaultBodyLimit (1 Mo ; une limite propre pour l'upload d'image)
           └ auth::middleware → session, rôle et capacité requise pour chaque route
             └ Routeur : /api/*  +  pages HTML  +  /static  +  /api/static
```

Ce sont presque tous des middlewares maison écrits avec
`axum::middleware::from_fn_with_state`. La logique de sécurité est donc dans
le projet, pas dans une bibliothèque. À elle seule, `cors.rs` fait environ
1 200 lignes.

## 2. Le rendu des pages

| Élément | Technologie |
|---|---|
| Templates | **Askama 0.12** : templates Jinja-like **compilés en Rust**, vérifiés au typage. Une modification de template exige donc de recompiler. Ils sont dans `templates/`, et leurs structures dans `src/web/templates.rs` |
| Traductions | **Fluent** (`fluent`, `fluent-templates`, `unic-langid`, `accept-language`) : 10 langues ; la langue est choisie par l'en-tête `Accept-Language` |
| Ressources statiques | **rust-embed 8** : le dossier `static/` est intégré au binaire en release, et lu sur le disque en debug |
| Images | **image 0.25** (jpeg, png, webp) : traitement de l'image de titre d'une recette au moment de l'upload |
| Dates | **chrono** (formats localisés) |

## 3. Le front-end

| Élément | Technologie |
|---|---|
| CSS | **Tailwind CSS 4** (CLI) ; jetons de couleur dans `input.css`, composants dans `components.css` |
| JavaScript | **JS natif sans framework** (pas de React, Vue ni htmx). Scripts autonomes : `search.js`, `cooking-mode.js`, `timers.js`, `keyboard-shortcuts.js`, `unsaved.js`, `sign-in-notice.js` |
| Éditeur de recettes | **CodeMirror 6** (`@codemirror/*`, `@lezer/highlight`) avec un mode Cooklang maison, l'autocomplétion et le lint |
| Empaquetage JS | **esbuild** : un seul bundle IIFE (`editor.bundle.js`), qui regroupe l'éditeur, la barre d'outils, le sélecteur de recette et le planificateur |

## 4. Le temps réel

| Besoin | Mécanisme |
|---|---|
| Liste de courses mise à jour en direct | **SSE** (`axum::response::sse` avec keep-alive) sur `/api/shopping_list/events` |
| Détecter qu'un fichier a été modifié hors du serveur | **notify 8** et `notify-debouncer-full` (surveillance du système de fichiers), pour `.shopping-list` et pour `users.toml` |
| Autocomplétion dans l'éditeur | **WebSocket** `/api/ws/lsp` relié à un sous-processus `cook lsp` (`tower-lsp`, `cooklang-language-server`). Le nombre de sessions est limité, sinon le serveur répond 503 |

## 5. Les données et leur persistance

- **Pas de SGBD.** Les recettes (`.cook`, `.menu`), `aisle.conf`,
  `pantry.conf`, `.shopping-list` et `.shopping-checked` sont de simples
  fichiers texte.
- **Écritures atomiques** (`fs_atomic.rs`, fichier temporaire puis renommage).
- **Concurrence** : `flock` côté système (via `libc` sous Linux) et des
  `tokio::sync::Mutex` dans le processus. Le `flock` ne protège pas deux tâches
  d'un même processus l'une de l'autre, d'où ce double verrou.
- La seule base est **SQLite**, en version intégrée au binaire, et elle ne sert
  qu'à l'état de synchronisation cook.md (`sync.db`, feature `sync`).
- Le métier Cooklang est délégué aux crates de l'écosystème : `cooklang`
  (parseur), `cooklang-find` (découverte des fichiers), `cooklang-reports`,
  ainsi qu'au crate interne `cookcli-core`.

## 6. La sécurité

| Sujet | Technologie et choix |
|---|---|
| Mots de passe | **Argon2** (`argon2`) ; compilé en `opt-level=3` même en debug pour que les tests restent rapides |
| Sessions | **Cookie signé en HMAC-SHA256** (`hmac`, `sha2`) au format `user:expiry:id:mac`, sans session stockée côté serveur, avec une liste de révocation |
| Autorisations | 4 rôles, et une capacité requise par route, décidée en un seul endroit (`required_capability`) |
| Origines | Vérification du Host, garde anti-CSRF sur les écritures, CORS opt-in |
| Fuites d'information | `private_paths::hide` masque les chemins absolus dans les erreurs ; `/api/static` ne sert que des images |

## 7. La synchronisation et les clients HTTP sortants

- **reqwest 0.12** avec **rustls** (pas d'OpenSSL) pour la synchronisation,
  l'import et la mise à jour.
- **cooklang-sync-client** : synchronisation avec cook.md, connexion par
  device-code, exécutée en tâche de fond dans le serveur.

## 8. Build, déploiement et tests

- **Build** : `build.rs` bloque la compilation si le CSS ou le JS ne sont pas
  compilés. Le Makefile enchaîne npm puis cargo. En release : LTO complet,
  `codegen-units=1`, binaire strippé.
- **Déploiement** : Docker (avec un profil `docker-dev` en LTO thin), paquets
  .deb et .rpm, binaires compilés pour la glibc 2.28 (image manylinux), options
  `--url-prefix` et `--proxy-strips-prefix` pour fonctionner derrière un
  reverse proxy.
- **Tests** : tests d'intégration Rust avec `assert_cmd`, `insta` et
  `tokio-tungstenite` ; tests de bout en bout **Playwright** avec **axe-core**
  pour l'accessibilité.

## Ce qu'il faut en retenir

**Points forts**

- Un seul binaire autonome et sans dépendance d'exécution : facile à héberger
  sur un NAS ou dans Home Assistant.
- Les fichiers restent la source de vérité : on peut les modifier à la main ou
  les versionner avec git.
- Templates et routes sont vérifiés à la compilation.
- La sécurité est pensée pour un serveur exposé sur un réseau local :
  rebinding, CSRF, rôles.

**Points de vigilance**

- Toute modification d'un template impose une recompilation et un redémarrage,
  ce qui ralentit les itérations sur l'interface.
- Askama 0.12 n'est plus la dernière version ; la migration change l'API.
- La sécurité repose sur beaucoup de code maison (CORS, auth) plutôt que sur
  des crates éprouvées : c'est puissant, mais le coût de maintenance est élevé.
- Le stockage en fichiers, avec verrous et watchers, convient bien à un foyer,
  mais montera mal en charge si de nombreux utilisateurs écrivent en même
  temps.
