# CLAUDE.md - CS 1.6 Browser Edition

## Context

Counter-Strike 1.6 jouable dans le navigateur web, utilisant Xash3D-FWGS compile en WebAssembly via Emscripten. Usage personnel + multijoueur avec des amis.

- **Repo:** https://github.com/sekaijinlimited/cs16-browser
- **Base:** Fork de [modesage/cs1.6-browser](https://github.com/modesage/cs1.6-browser), fortement modifie

## Architecture

### Client (browser)

Le client est un `index.html` qui :
1. Telecharge `valve.zip` (415 MB) contenant les assets du jeu (maps, sons, modeles)
2. Cache le zip dans IndexedDB (pas de retelechargemet aux visites suivantes)
3. Extrait les fichiers dans le filesystem virtuel Emscripten (`/rodir/`)
4. Charge les extras.pk3 (valve + cstrike)
5. Initialise le moteur Xash3D-FWGS en WebAssembly

**Fichiers WASM (depuis npm packages, MIT) :**

| Package | Version | Fichiers |
|---------|---------|----------|
| `xash3d-fwgs` | 1.2.2 | `raw.js`, `xash.wasm`, `filesystem_stdio.wasm`, `libref_webgl2.wasm` |
| `cs16-client` | 0.1.2 | `dlls/cs_emscripten_wasm32.wasm`, `cl_dlls/client_emscripten_wasm32.wasm`, `cl_dlls/menu_emscripten_wasm32.wasm`, `cstrike_extras.pk3` |

**Points techniques importants :**
- Le `locateFile` utilise des `includes()` pour matcher les chemins flexiblement (l'engine dlopen avec des chemins absolus comme `/rodir/filesystem_stdio.wasm`)
- Le renderer est `webgl2` (argument `-ref webgl2`)
- `libvgui_support.wasm` est mappe sur le menu WASM
- Les assets du jeu viennent de SteamCMD app 90 (HLDS, telechargeable anonymement)
- F2 toggle les logs engine

### Serveur multijoueur (VPS yuna)

- **VPS:** yuna@72.61.124.223 (Ubuntu 24.04, 8GB RAM, 2 cores)
- **Docker image:** `yohimik/cs-web-server` (serveur dedie CS 1.6 + client web integre + bridge WebRTC)
- **Port 27016:** Client web integre + WebSocket/WebRTC bridge
- **Port 27015:** Serveur de jeu UDP (interne au container)

```yaml
# ~/cs16/docker-compose.yml
services:
  cs-server:
    image: yohimik/cs-web-server
    ports:
      - "27016:27016"
    volumes:
      - ./valve.zip:/opt/valve.zip
      - ./valve.zip:/xashds/public/valve.zip:ro
    restart: unless-stopped
```

**Le client integre (port 27016) utilise WebRTC** pour le networking, pas du WebSocket pur. Le WebSocket sur `/websocket` sert uniquement de signaling pour WebRTC.

### Nginx (port 80)

Sert notre client custom sur `http://72.61.124.223`.

Config: `/etc/nginx/sites-available/cs16`

## Key Paths

### Local (musubi-2)

| Path | Description |
|------|-------------|
| `/Users/weck0/projects/cs16/index.html` | Client browser principal |
| `/Users/weck0/projects/cs16/valve.zip` | Assets du jeu (gitignore) |
| `/Users/weck0/projects/cs16/get_cs_assets/` | Scripts SteamCMD |

### VPS (yuna)

| Path | Description |
|------|-------------|
| `~/cs16/docker-compose.yml` | Config Docker serveur CS 1.6 |
| `~/cs16/valve.zip` | Assets du jeu |
| `/var/www/cs16/` | Client web servi par Nginx |
| `/etc/nginx/sites-available/cs16` | Config Nginx |

## Commandes utiles

```bash
# Serveur CS 1.6
ssh yuna "cd ~/cs16 && sudo docker compose up -d"     # demarrer
ssh yuna "cd ~/cs16 && sudo docker compose down"       # arreter
ssh yuna "sudo docker logs cs16-cs-server-1 --tail 20" # logs

# Deployer le client sur le VPS
rsync -avz index.html raw.js *.wasm *.pk3 cl_dlls/ dlls/ yuna:/var/www/cs16/

# Git (utilise l'alias SSH sekaijin pour le bon compte GitHub)
# Remote: git@github.sekaijin:sekaijinlimited/cs16-browser.git
```

## TODO

- [ ] Configurer un domaine + SSL (HTTPS/WSS) avec Certbot
- [ ] Tester le multijoueur via le client integre (port 27016)
- [ ] Optimiser le chargement (compression gzip, split assets)
