# Pi-hole sur Ubuntu Server (Docker)

**Objectif de ce labo :** Installer Pi-hole en conteneur Docker pour bloquer les publicités et les trackers au niveau DNS pour tout le réseau local

---

## Contexte

Avec seulement 4 Go de RAM, chaque service doit être léger et bien isolé. Ce premier laboratoire pose les fondations qui serviront aussi pour les prochains services (VPN WireGuard, etc.).


## Étape 1 — Installer Ubuntu Server 24.04 LTS (minimal)

1. J'ai téléchargé l'image ISO d'Ubuntu Server 24.04 : https://cdimage.ubuntu.com/ubuntu-mini-iso/releases/24.04.4/release/
2. J'ai créé une clé USB bootable avec Rufus.
3. J'ai démarré sur la clé USB et lancé l'installateur.
4. Au choix du profil de base, j'ai sélectionné **« Minimized »** plutôt que « Standard ». Cela réduit le nombre de paquets installés et donc l'empreinte RAM au repos.
5. J'ai coché uniquement « OpenSSH server » dans les options.

**Contrainte physique :** mon routeur était beaucoup trop loin de l'emplacement prévu pour la tour, et je ne pouvais pas le déplacer. J'ai donc acheté un prolongateur avec un port Ethernet.

## Étape 2 — Mises à jour et préparation du système

```bash
sudo apt update && sudo apt upgrade -y
```

Après la mise à jour, j'ai choisi UTF-8 comme encodage pour la console, puis l'option Latin1/Latin5 - Western Europe and Romanian, et la disposition de clavier English (US).


## Étape 3 — Installer Docker et Docker Compose

J'ai suivi le guide officiel : https://docs.docker.com/engine/install/ubuntu/

```bash
sudo systemctl status docker
docker --version
docker compose version
```

### Problème rencontré : SSH ne fonctionnait pas

Je n'arrivais pas à me connecter en SSH à cause de mon prolongateur. J'ai d'abord vérifié la configuration SSH directement sur le serveur :

```bash
sudo nano /etc/ssh/sshd_config
```

Je me suis assuré que la ligne suivante n'était pas commentée et était réglée sur `yes` :

```
PasswordAuthentication yes
```

Puis j'ai redémarré le service :

```bash
sudo systemctl restart ssh
```

Cela n'a pas réglé le problème non plus. En creusant, voici ce que j'ai observé :

Ma configuration Netplan avait mis deux noms d'interfaces différents.

J'ai fini par corriger la configuration réseau directement via Netplan, avec une adresse statique :

```yaml
network:
  ethernets:
    enp0s10:
      addresses:
        - X.X.X.X/24
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses:
          - 127.0.0.1
          - 1.1.1.1
  version: 2
  renderer: networkd
```
```bash
sudo netplan apply
```

Une fois que SSH fonctionnait, j'ai installé Cockpit comme interface d'administration alternative où je peux voir ma consommation également :

```bash
. /etc/os-release
sudo apt install -t ${VERSION_CODENAME}-backports cockpit
sudo systemctl enable --now cockpit.socket
```

## Étape 4 — Créer la structure du projet

```bash
mkdir -p ~/homelab/pihole
cd ~/homelab/pihole
```

J'ai créé un fichier `docker-compose.yml` :

```yaml
# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
      - "4433:443/tcp"
    environment:
      TZ: 'America/Toronto'
      FTLCONF_webserver_api_password: 'VOTRE_MOT_DE_PASSE_SECURISE'
      FTLCONF_dns_listeningMode: 'ALL'
    volumes:
      - './etc-pihole:/etc/pihole'
    restart: unless-stopped
```

> **Note RAM :** Pi-hole consomme typiquement 50-100 Mo — largement dans mes moyens.

### Libérer le port 53

Pour que Pi-hole puisse utiliser le port 53, j'ai dû désactiver le DNS stub de systemd-resolved.

1. Ouvrir la configuration de systemd-resolved :

```bash
sudo nano /etc/systemd/resolved.conf
```

Sous `[Resolve]`, décommenter (ou ajouter) la ligne suivante et la passer à `no` :

```ini
[Resolve]
DNSStubListener=no
```

Enregistrer et quitter.

2. Rediriger `/etc/resolv.conf` vers le résolveur statique du système, pour ne pas perdre la connexion Internet locale :

```bash
sudo rm /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

Vérification — le port 53 doit être libre :

```bash
sudo ss -tulpn | grep :53
```

3. Relancer le déploiement du conteneur Pi-hole :

```bash
docker compose up -d
```

## Étape 5 — Lancer Pi-hole

```bash
docker compose up -d
```

Je vérifie que le conteneur tourne :

```bash
docker ps
docker logs pihole
```

## Étape 6 — Accéder à l'interface web

Depuis un navigateur sur le réseau local : `http://<IP-de-ma-tour>:8080/admin`

Je me connecte avec le mot de passe défini dans `VOTRE_MOT_DE_PASSE_SECURISE`.

## Étape 7 — Configurer mon réseau pour utiliser Pi-hole

1. Je vais dans les paramètres DHCP de mon routeur.
2. Je change le DNS primaire distribué aux appareils pour l'IP de ma tour Acer.
3. Je garde un DNS secondaire externe (ex. : 1.1.1.1) en cas de panne du Pi-hole.

## Résultats attendus

- Pi-hole actif, bloquant les pubs/trackers pour tout le réseau
- Consommation RAM totale (OS + Docker + Pi-hole) sous ~800 Mo, laissant de la marge pour le prochain service
- Structure `~/homelab/` prête à accueillir le prochain `docker-compose.yml` (WireGuard)

## Prochaine étape

Installer WireGuard (VPN) en conteneur, configuré pour utiliser Pi-hole comme résolveur DNS — accès distant sécurisé et blocage de pubs même hors réseau local.

---

*Labo réalisé dans le cadre du projet de transformation d'une tour Acer en serveur maison personnel.*
