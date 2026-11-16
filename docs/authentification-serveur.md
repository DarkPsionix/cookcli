# Authentification et autorisations du serveur web (`cook server`)

L'authentification est **intégrée au serveur et désactivée par défaut**. Elle
s'active dès qu'il existe un fichier d'utilisateurs. Le serveur repose sur des
**mots de passe hachés en Argon2** et sur un **cookie signé** : il n'utilise
ni base de données, ni table de sessions, ni fournisseur externe (pas d'OAuth
ni de SSO). Le code est dans `src/server/auth/`.

Voir aussi : [pile technique du serveur](stack-technique-serveur.md) ·
[cartographie des fonctionnalités](cartographie-fonctionnalites.md).

## 1. Les comptes : le fichier `users.toml`

Les comptes se gèrent **uniquement en ligne de commande, sur la machine du
serveur**. Le navigateur ne sait que se connecter et se déconnecter : aucune
page web ne permet de créer un compte.

```bash
cook server user add alice --role editor   # demande le mot de passe sans l'afficher
```

Les autres commandes :

| Commande | Effet |
|---|---|
| `cook server user passwd` | Change le mot de passe |
| `cook server user role` | Change le rôle |
| `cook server user remove` | Supprime le compte |
| `cook server user list` | Liste les comptes |
| `cook server hash-password` | Produit un hash pour éditer le fichier à la main |

Le fichier contient un hash Argon2 (format PHC) pour chaque utilisateur :

```toml
[users]
alice = "$argon2id$v=19$m=19456,t=2,p=1$…"        # hash seul = rôle admin
bob   = { hash = "$argon2id$…", role = "editor" }
```

- **Emplacement**, par ordre de priorité : `--users-file`, puis
  `COOK_USERS_FILE`, puis `users.toml` dans le dossier de config global.
- **Le serveur refuse de démarrer** si ce fichier ou la clé de session se
  trouvent dans le dossier de recettes. Ce dossier est servi par
  `/api/static`, ce qui exposerait les hashes et la clé (`refuse_inside`,
  `src/server/auth/mod.rs`).
- **Rechargement à chaud** : un watcher `notify` surveille le dossier du
  fichier. Un ajout, un changement de rôle ou une suppression s'applique sans
  redémarrer le serveur.

## 2. La connexion (`POST /login`)

1. L'utilisateur envoie son nom et son mot de passe par le formulaire de
   `/login`.
2. Le serveur vérifie le mot de passe avec **Argon2id** dans un thread séparé,
   pour ne pas bloquer le serveur.
3. Si c'est correct, il pose le cookie de session et redirige vers la page
   demandée. Ce paramètre `next` est nettoyé, ce qui empêche une redirection
   vers un autre site.
4. La connexion est inscrite dans le journal d'activité. Un échec est
   journalisé **sans le nom saisi**, parce que les gens tapent parfois leur mot
   de passe dans ce champ par erreur.

## 3. La session : un cookie signé, sans état côté serveur

```
cook_session=<user>:<expiry>:<id>:<mac>
```

- **`mac`** : une signature **HMAC-SHA256** calculée sur l'utilisateur,
  l'expiration, un identifiant aléatoire de 16 octets **et le hash actuel de
  son mot de passe**.
- **Clé de signature** : 32 octets aléatoires stockés dans le fichier
  `auth-secret` du dossier de config. Le fichier est créé au premier
  démarrage, avec les droits `0600` sous Unix.
- **Durée** : 30 jours.
- **Attributs du cookie** : `HttpOnly` (illisible par JavaScript),
  `SameSite=Lax`, et `Secure` quand le reverse proxy signale du HTTPS
  (`X-Forwarded-Proto`). Le `Path` suit `--url-prefix`.

Le serveur ne garde aucune table de sessions. À chaque requête, le middleware
recalcule la signature et vérifie l'expiration. Comme le hash du mot de passe
fait partie de la signature, une révocation se produit dans ces cas :

| Événement | Effet |
|---|---|
| Changement de mot de passe | Toutes les sessions de l'utilisateur deviennent invalides |
| Suppression de l'utilisateur | Idem |
| Suppression du fichier `auth-secret` | **Tout le monde** est déconnecté |
| Changement de rôle | Sans déconnexion ; appliqué dès la requête suivante |

## 4. La déconnexion (`POST /logout`)

Effacer le cookie dans le navigateur ne suffirait pas : une copie du cookie,
prise sur un ordinateur partagé ou dans un log de proxy, resterait valable
30 jours. La déconnexion **inscrit donc l'identifiant de la session dans une
liste de révocation persistée sur disque** (`revoked.rs`), puis efface le
cookie. La liste survit aux redémarrages et se purge d'elle-même après
30 jours.

## 5. Protections contre les attaques

| Menace | Contre-mesure |
|---|---|
| Vol de la base de mots de passe | Hachage Argon2id avec sel aléatoire |
| Deviner quels comptes existent (timing) | Un nom inconnu est vérifié contre un faux hash, pour que la réponse prenne autant de temps qu'un mauvais mot de passe |
| Saturer la mémoire par des tentatives en masse | Au plus **2 vérifications Argon2 simultanées** (environ 19 Mo chacune) ; mot de passe limité à 1 024 caractères |
| Falsifier un cookie | Signature HMAC-SHA256 avec une clé secrète |
| Vol d'un cookie par JavaScript (XSS) | `HttpOnly` |
| CSRF | `SameSite=Lax` et la garde `write_guard` sur l'origine des écritures |
| Redirection vers un site externe après connexion | `sanitize_next` |

## 6. Après l'authentification : les autorisations

Le middleware `auth::middleware` crée un `Viewer` pour chaque requête :
visiteur non connecté, ou utilisateur avec son rôle. Il compare ensuite ce
rôle à la capacité exigée par la route (`required_capability`,
`src/server/auth/middleware.rs`) et répond **401** à un visiteur non connecté,
**403** à un rôle insuffisant.

| Rôle | Lire les recettes | Liste de courses, garde-manger, rayons | Créer, modifier, supprimer des recettes | Sync cook.md |
|---|:-:|:-:|:-:|:-:|
| visiteur non connecté | ✅ | ❌ | ❌ | ❌ |
| `reader` | ✅ | ❌ | ❌ | ❌ |
| `shopper` | ✅ | ✅ | ❌ | ❌ |
| `editor` | ✅ | ✅ | ✅ | ❌ |
| `admin` | ✅ | ✅ | ✅ | ✅ |

- **Sans `users.toml`**, tout le monde peut tout faire : seules les
  protections réseau s'appliquent (écoute sur `localhost` par défaut,
  vérification de l'origine des écritures).
- **Avec `--recipes-only`** (ou `COOK_RECIPES_ONLY`), un visiteur non connecté
  peut seulement lire les recettes.
- Une nouvelle route d'écriture qui n'est pas listée dans la table `WRITES`
  exige automatiquement le rôle `admin`. Un oubli ne peut donc pas ouvrir une
  écriture à tout le monde.

## Limites à connaître

- **Rien ne limite le nombre de tentatives par IP ou par compte.** La limite
  de 2 vérifications simultanées protège la mémoire, pas contre une attaque
  par force brute lente. Si le serveur est exposé sur Internet, mettez un
  fail2ban ou une limite de débit au niveau du reverse proxy.
- **HTTPS n'est pas géré par le serveur.** Il faut un reverse proxy TLS ;
  sinon le mot de passe et le cookie circulent en clair sur le réseau.
- **Il n'y a ni 2FA, ni réinitialisation par e-mail, ni SSO.** Seul un
  administrateur ayant accès à la machine peut réinitialiser un mot de passe,
  avec `user passwd`.

À distinguer : `cook login` / `cook logout` ne servent pas à cette
authentification. Ces commandes connectent CookCLI à un **compte cook.md**
pour la synchronisation, par un flux device-code, et l'état est stocké dans
`session.json`. C'est un mécanisme indépendant.
