# Guide de déploiement et de sécurisation

Ce guide décrit comment mettre `cook server` en service de façon durable :
installation, accès réseau, HTTPS, comptes, protection contre les tentatives de
connexion en masse et sauvegardes. Il se termine par une liste de vérification
à suivre avant d'exposer le serveur hors du réseau local.

Les options du serveur sont détaillées (en anglais) dans
[`server.md`](server.md). Le fonctionnement de la connexion est expliqué dans
[authentification et autorisations](authentification-serveur.md).

## 1. Choisir un scénario

| Scénario | Exposition | Ce qu'il faut |
|---|---|---|
| **A. Usage personnel sur un poste** | `localhost` uniquement | `cook server` suffit ; rien d'autre |
| **B. Foyer, réseau local de confiance** | Réseau local, accès par adresse IP | `--host` ; comptes conseillés si tout le monde ne doit pas modifier les recettes |
| **C. Réseau local avec un nom d'hôte** (`nas.local`) | Réseau local | `--host` et `--cors-origin http://nas.local:9080` |
| **D. Accès depuis Internet** | Public | Reverse proxy HTTPS, comptes **obligatoires**, limite de tentatives, sauvegardes |

Par défaut le serveur n'écoute que sur `localhost` et n'a pas de comptes :
**tous ceux qui peuvent le joindre peuvent tout modifier.** Dès que d'autres
personnes que vous y accèdent, créez des comptes (§ 4).

## 2. Installer

### Avec Docker (recommandé pour un serveur)

L'image officielle `ghcr.io/cooklang/cookcli` contient uniquement le serveur et
le serveur de langage de l'éditeur. Elle n'a ni la synchronisation cook.md, ni
l'import, ni l'auto-mise à jour. Elle tourne sous l'utilisateur non root
`1000:1000` et lance `cook server /recipes --host`.

```bash
mkdir -p cookcli/recipes cookcli/config
cd cookcli
```

`docker-compose.yml` :

```yaml
services:
  cookcli:
    image: ghcr.io/cooklang/cookcli:latest
    # Si votre UID n'est pas 1000 : user: "UID:GID" (voir `id -u` et `id -g`)
    # user: "1000:1000"
    ports:
      # 127.0.0.1 : seul le reverse proxy de cette machine peut le joindre.
      # Retirez-le pour un accès direct depuis le réseau local (scénarios B et C).
      - "127.0.0.1:9080:9080"
    volumes:
      - ./recipes:/recipes
      - ./config:/config
      # Heure locale dans le journal d'activité (sinon UTC) :
      - /etc/localtime:/etc/localtime:ro
    environment:
      COOK_CONFIG_DIR: /config
      # L'adresse que le navigateur affiche, https:// derrière un proxy TLS.
      COOK_CORS_ORIGIN: https://cook.example.com
    restart: unless-stopped
```

Points d'attention :

- **Créez `recipes/` et `config/` vous-même** avant le premier démarrage. Un
  dossier créé par Docker appartient à root et le conteneur, qui tourne en UID
  1000, ne peut pas y écrire. Le script de démarrage refuse un `/recipes` non
  modifiable et explique comment corriger.
- **`COOK_CONFIG_DIR: /config` est indispensable pour les comptes** :
  l'utilisateur du conteneur n'a pas de dossier personnel, donc sans lui il n'y
  a nulle part où garder `users.toml` et la clé des cookies.
- Si `/recipes` est vide au montage, vous partez d'une collection vide. Pour
  des exemples, lancez `docker compose run --rm cookcli cook seed /recipes`.

### Avec le binaire

Téléchargez le binaire de la [dernière version](https://github.com/cooklang/cookcli/releases)
ou installez-le avec Cargo (voir le [README](../README.md)). Exemple de
service systemd, sous un utilisateur dédié :

```ini
# /etc/systemd/system/cookcli.service
[Unit]
Description=CookCLI web server
After=network.target

[Service]
User=cook
Group=cook
Environment=COOK_CONFIG_DIR=/var/lib/cookcli/config
Environment=COOK_CORS_ORIGIN=https://cook.example.com
ExecStart=/usr/local/bin/cook server /var/lib/cookcli/recipes --port 9080
Restart=on-failure
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/cookcli

[Install]
WantedBy=multi-user.target
```

Sans `--host`, le serveur n'écoute que sur `localhost` : c'est ce qu'il faut
derrière un reverse proxy installé sur la même machine.

**Règle importante** : le dossier de configuration ne doit pas se trouver dans
le dossier de recettes. Le serveur refuse de démarrer dans ce cas, car ce
dossier est servi en partie par `/api/static` et les hashes et la clé de
session seraient exposés.

## 3. Mettre le serveur derrière un reverse proxy HTTPS

Le serveur ne gère pas TLS. **Sans HTTPS, les mots de passe et les cookies de
session circulent en clair.** Placez-le derrière un reverse proxy qui :

1. termine HTTPS ;
2. transmet l'en-tête `Host` d'origine ;
3. envoie `X-Forwarded-Proto: https`, pour que le cookie de session soit
   marqué `Secure`.

Et nommez l'adresse publique dans `--cors-origin` ou `COOK_CORS_ORIGIN`. Sans
cela, le serveur refuse de répondre à un nom d'hôte qu'il ne connaît pas
(protection contre le DNS rebinding).

### Caddy

Caddy obtient le certificat tout seul et transmet `Host` et
`X-Forwarded-Proto` par défaut.

```caddyfile
cook.example.com {
    reverse_proxy 127.0.0.1:9080
}
```

### nginx

```nginx
# Limite de tentatives de connexion : 5 par minute et par IP
limit_req_zone $binary_remote_addr zone=cook_login:10m rate=5r/m;

server {
    listen 443 ssl;
    http2 on;
    server_name cook.example.com;

    ssl_certificate     /etc/letsencrypt/live/cook.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cook.example.com/privkey.pem;

    # Les photos de téléphone dépassent la limite par défaut de nginx (1 Mo).
    client_max_body_size 20m;

    location = /login {
        limit_req zone=cook_login burst=5 nodelay;
        proxy_pass http://127.0.0.1:9080;
        include /etc/nginx/cook_proxy.conf;
    }

    location / {
        proxy_pass http://127.0.0.1:9080;
        include /etc/nginx/cook_proxy.conf;
    }
}
```

```nginx
# /etc/nginx/cook_proxy.conf
proxy_set_header Host              $host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;

# Mise à jour en direct de la liste de courses (Server-Sent Events)
proxy_buffering off;
proxy_read_timeout 1h;

# Autocomplétion de l'éditeur (WebSocket /api/ws/lsp)
proxy_http_version 1.1;
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

`$connection_upgrade` se définit une fois dans le bloc `http` :

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

### Servir sous un sous-chemin

Pour `https://example.com/cook/` : ajoutez `--url-prefix /cook`. Si le proxy
retire `/cook` avant de transmettre (Traefik `StripPrefix`, nginx
`proxy_pass http://127.0.0.1:9080/;` avec la barre finale, Home Assistant
ingress), ajoutez aussi `--proxy-strips-prefix`. Détails dans
[`server.md`](server.md#behind-a-proxy-that-strips-the-prefix).

### Ce qu'il ne faut pas faire

- **N'utilisez `--no-csrf-check` qu'en dernier recours.** Cette option retire
  la vérification de l'en-tête `Host` et de l'origine des écritures. Elle ne
  sert que pour un proxy qui réécrit `Host` d'une façon impossible à décrire
  avec `--cors-origin`.
- **N'utilisez pas `--cors-origin "*"`** sauf pour publier les recettes en
  lecture à n'importe quel site ; les écritures restent alors refusées aux
  autres origines.
- **Ne publiez pas le port 9080 directement sur Internet**, sans proxy TLS.

## 4. Créer les comptes et choisir les rôles

```bash
# Binaire
cook server user add alice --role admin
cook server user add bob --role editor

# Docker (avant ou pendant le fonctionnement du conteneur)
docker compose run --rm cookcli cook server user add alice --role admin
```

| Rôle | Peut |
|---|---|
| `reader` | Lire les recettes |
| `shopper` | … et gérer la liste de courses, le garde-manger, les rayons |
| `editor` | … et créer, modifier, supprimer recettes et menus |
| `admin` | … et lier le serveur à cook.md |

Conseils :

- Donnez à chacun **le rôle le plus faible** qui lui suffit. Un `shopper`
  suffit pour faire les courses.
- **Créez un compte par personne**, pas un compte partagé : le journal
  d'activité dit qui a fait quoi, et on peut retirer l'accès d'une seule
  personne.
- Pour un site public de recettes en lecture, ajoutez `--recipes-only` (ou
  `COOK_RECIPES_ONLY=1`) : les visiteurs non connectés ne voient que les
  recettes.
- Un compte ajouté ou modifié est pris en compte sans redémarrer.

Pour **déconnecter tout le monde** (par exemple après une fuite), supprimez
les fichiers `auth-secret` et `revoked-sessions` du dossier de configuration,
puis redémarrez. Pour une seule personne, changez son mot de passe
(`cook server user passwd`) ou supprimez son compte.

## 5. Se protéger contre les tentatives de connexion en masse

Le serveur ne limite pas le nombre de tentatives par IP ou par compte. Il
limite seulement à 2 le nombre de vérifications de mot de passe simultanées,
pour protéger sa mémoire. Pour un serveur public, ajoutez une protection en
amont.

**Au niveau du proxy** : la directive `limit_req` de l'exemple nginx ci-dessus
limite `POST /login` à 5 tentatives par minute et par IP. Avec Caddy, il faut
un module de limite de débit (par exemple `caddy-ratelimit`).

**Avec fail2ban** : un échec de connexion renvoie le statut **401** sur
`POST /login`. Le journal d'activité de CookCLI note `guest failed to sign in`
**sans l'adresse IP**, donc fail2ban doit lire les journaux du proxy :

```ini
# /etc/fail2ban/filter.d/cookcli.conf
[Definition]
failregex = ^<HOST> .* "POST /login(\?\S*)? HTTP/[0-9.]+" 401
```

```ini
# /etc/fail2ban/jail.d/cookcli.conf
[cookcli]
enabled  = true
filter   = cookcli
logpath  = /var/log/nginx/access.log
maxretry = 5
findtime = 10m
bantime  = 1h
```

Avec `--url-prefix /cook`, remplacez `/login` par `/cook/login` dans le filtre.

## 6. Journal d'activité et supervision

Le serveur écrit une ligne sur sa sortie standard pour chaque modification
faite par l'interface : qui, quoi, quand. On le lit avec `docker compose logs`
ou `journalctl -u cookcli`.

```text
2026-09-27 08:42:03 alice created recipe "Soups/Pea soup.cook"
2026-09-27 09:12:44 guest failed to sign in
```

- Les modifications faites directement sur les fichiers n'y figurent pas.
- Pour plus de détails, lancez avec `RUST_LOG=info` (ou `debug`).
- Une suite de `guest failed to sign in` signale une attaque par force brute.

## 7. Sauvegarder

Tout est dans des fichiers : la sauvegarde se résume à copier deux dossiers.

| Dossier | Contenu | Importance |
|---|---|---|
| **Dossier de recettes** | Recettes `.cook`, menus `.menu`, images, `.shopping-list`, `.shopping-checked`, et `config/` s'il existe (`aisle.conf`, `pantry.conf` locaux) | **Essentiel** |
| **Dossier de configuration** (`COOK_CONFIG_DIR`) | `users.toml`, `auth-secret`, `revoked-sessions`, `aisle.conf` et `pantry.conf` globaux, `session.json` et `sync.db` (cook.md) | Important ; **contient des secrets** |

Conseils :

- Le dossier de recettes est du texte : le **versionner avec git** donne
  l'historique de chaque recette en plus de la sauvegarde.
- Stockez la sauvegarde du dossier de configuration **chiffrée** :
  `auth-secret` permet de fabriquer des sessions valides.
- Une restauration sans `auth-secret` déconnecte simplement tout le monde ; une
  restauration sans `users.toml` retire tous les comptes et **rouvre le
  serveur à tous**.

## 8. Mettre à jour

- **Docker** : `docker compose pull && docker compose up -d`. Préférez une
  version précise (`ghcr.io/cooklang/cookcli:0.41.1`) à `latest` pour choisir
  le moment des mises à jour.
- **Binaire** : `cook update` (ou `cook update --check-only` pour vérifier).
- Lisez le [CHANGELOG](../CHANGELOG.md) avant une mise à jour : les options de
  sécurité y sont signalées quand elles changent.

## 9. Liste de vérification avant d'exposer le serveur

- [ ] Le serveur est derrière un reverse proxy **HTTPS** ; le port 9080 n'est
      pas joignable directement depuis Internet.
- [ ] Le proxy transmet `Host` et envoie `X-Forwarded-Proto: https`.
- [ ] L'adresse publique est nommée dans `COOK_CORS_ORIGIN` ou
      `--cors-origin`.
- [ ] `--no-csrf-check` n'est **pas** utilisé.
- [ ] Un fichier `users.toml` existe, avec un compte par personne et le rôle
      le plus faible possible.
- [ ] `--recipes-only` est activé si les visiteurs non connectés ne doivent
      voir que les recettes.
- [ ] Le dossier de configuration est **hors** du dossier de recettes et n'est
      lisible que par l'utilisateur du service.
- [ ] Les tentatives sur `POST /login` sont limitées (proxy ou fail2ban).
- [ ] Les deux dossiers sont sauvegardés, la configuration de façon chiffrée.
- [ ] Le journal d'activité est conservé et consulté.
- [ ] La version utilisée est connue, et les mises à jour suivies.
