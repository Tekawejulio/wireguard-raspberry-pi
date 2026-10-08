# Serveur VPN personnel (WireGuard) sur Raspberry Pi 5

Serveur VPN **WireGuard** déployé avec **wg-easy** dans **Docker** sur un Raspberry Pi 5, pour accéder
à mon réseau domestique (Pi-hole, Jellyfin, Netdata…) de façon sécurisée.
L'accès depuis l'extérieur est assuré par **Tailscale**, car je n'ai pas accès à la configuration de la box Internet.

> Projet réalisé dans le cadre de ma formation en réseaux et systèmes (BAC IT, ISIPS Charleroi).

## Objectifs

- Comprendre le fonctionnement d'un VPN (tunnel chiffré, clés publiques/privées, NAT).
- Déployer un serveur WireGuard avec une interface de gestion simple.
- Diagnostiquer et résoudre les problèmes réels rencontrés (noyau, pare-feu, Docker).

## Architecture

```
Téléphone ──(tunnel WireGuard, UDP 51820)──► Raspberry Pi 5 (wg-easy, Docker) ──► réseau local
                                              192.168.129.200        │
                                                                     ├─ Pi-hole
                                                                     ├─ Jellyfin
                                                                     └─ Netdata

Accès hors de la maison : Téléphone ──(Tailscale)──► Raspberry Pi 5
```

| Élément | Détail |
|---|---|
| Matériel | Raspberry Pi 5 (4 Go), Raspberry Pi OS Lite 64 bits (Debian Trixie) |
| VPN | WireGuard via wg-easy v14 (Docker Compose) |
| Réseau VPN | 10.8.0.0/24 |
| Port | 51820/UDP (tunnel), 51821/TCP (interface web, local uniquement) |
| Accès externe | Tailscale |

## Installation

1. **Créer le dossier du projet** : `mkdir -p ~/wireguard && cd ~/wireguard`
2. **Générer le hash du mot de passe** (choisir un vrai mot de passe, long et unique) :
   ```bash
   docker run --rm ghcr.io/wg-easy/wg-easy:14 wgpw 'MON_MOT_DE_PASSE'
   ```
3. **Copier `docker-compose.yml`** et y coller le hash, en **doublant chaque `$` en `$$`** :
   ```bash
   sed -i '/PASSWORD_HASH/s/\$/$$/g' docker-compose.yml
   ```
4. **Vérifier puis lancer** :
   ```bash
   docker compose config -q && echo "Fichier OK"
   docker compose up -d
   docker ps --filter name=wg-easy      # doit afficher (healthy)
   ```
5. **Interface web** : `http://IP_DU_PI:51821`, créer un client, scanner le QR code avec l'app WireGuard.
6. **Vérifier le tunnel** :
   ```bash
   docker exec wg-easy wg show          # "latest handshake" + "transfer" des deux côtés
   ```

## Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| Conteneur en boucle de redémarrage (`Restarting (1)`) | Le noyau 6.18 du Pi n'a plus les modules `iptables` *legacy* ; l'image n'utilisait que ceux-là (`Table does not exist`) | Utiliser `iptables-nft` via `WG_POST_UP` / `WG_POST_DOWN` |
| Tunnel établi, mais le serveur ne renvoie presque rien (`transfer: 27 KiB received, 664 B sent`) | Règle NAT sur `wlan0` : dans le conteneur, la sortie s'appelle `eth0` | Remplacer `-o wlan0` par `-o eth0`, puis `docker compose up -d --force-recreate` |
| Mot de passe refusé / variables vides | Les `$` du hash bcrypt sont lus comme des variables par Docker Compose | Doubler chaque `$` en `$$` |
| Configuration incompatible | `wg-easy` sans version prend la v15, qui a changé de fonctionnement | Fixer l'image : `wg-easy:14` |
| Modification du compose sans effet | Un conteneur en cours ne relit pas le fichier | `docker compose up -d --force-recreate` |

## Résultat

Tunnel validé depuis un téléphone sur le réseau local : handshake établi et trafic dans les deux sens
(`2,19 MiB reçus / 20,54 MiB envoyés` lors du test de navigation).

## Limite et accès à distance avec Tailscale

Pour utiliser wg-easy depuis Internet, il faut rediriger le port **UDP 51820** sur la box vers le Pi, ce qui n'est
pas possible ici (pas d'accès aux réglages de la box). J'ai donc installé **Tailscale** (basé sur WireGuard),
qui fonctionne sans redirection de port :

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```

Depuis la 4G, avec Tailscale activé sur le téléphone, j'accède à Pi-hole, Jellyfin et Netdata via l'adresse Tailscale du Pi.

## Sécurité

- L'interface web (51821) n'est **jamais** exposée sur Internet.
- Le dossier `wg-data/` (clés des clients) et le hash du mot de passe ne sont **jamais** publiés (voir `.gitignore`).
- Seul le port 51820/UDP serait ouvert pour un accès externe via wg-easy.

## Captures d'écran

| Capture | Fichier |
|---|---|
| Interface wg-easy avec un client créé | `screenshots/wg-easy-dashboard.png` |
| Conteneur `healthy` (`docker ps`) | `screenshots/docker-ps-healthy.png` |
| Tunnel actif (`wg show`, handshake et transfer) | `screenshots/wg-show-handshake.png` |
| App WireGuard connectée sur le téléphone | `screenshots/telephone-wireguard.png` |
| Accès à Jellyfin/Netdata en 4G via Tailscale | `screenshots/acces-4g-tailscale.png` |

## Pistes d'amélioration

- Ouvrir le port 51820/UDP sur la box et utiliser un nom DNS dynamique (DuckDNS) pour l'accès externe via wg-easy.
- Utiliser Pi-hole comme DNS des clients VPN (`WG_DEFAULT_DNS`).
- Brancher le Pi en Ethernet (il est actuellement en Wi-Fi).
