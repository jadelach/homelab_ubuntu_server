# Laboratoire 01 — Pi-hole sur Ubuntu Server (Docker)

**Projet :** Transformation d'une tour Acer (4 Go RAM DDR3, 1 To de stockage, processeur quad-core AMD) en serveur maison
**Objectif de ce labo :** Installer Pi-hole en conteneur Docker pour bloquer les publicités et les trackers au niveau DNS pour tout le réseau local

---

## Contexte

Avec seulement 4 Go de RAM, chaque service doit être léger et bien isolé. Ce premier laboratoire pose les fondations (OS minimal + Docker) qui serviront aussi aux prochains services (VPN WireGuard, etc.).

## Prérequis

- Tour Acer avec accès physique ou clavier/écran pour l'installation
- Clé USB (8 Go+) pour créer le média d'installation
- Accès à mon routeur pour configurer une IP statique/réservation DHCP

## Étape 1 — Installer Ubuntu Server 24.04 LTS (minimal)

1. J'ai téléchargé l'image ISO d'Ubuntu Server 24.04 : https://cdimage.ubuntu.com/ubuntu-mini-iso/releases/24.04.4/release/
2. J'ai créé une clé USB bootable avec Rufus.
3. J'ai démarré sur la clé USB et lancé l'installateur.
4. **Important** : au choix du profil de base, j'ai sélectionné **« Minimized »** plutôt que « Standard » — cela réduit le nombre de paquets installés et donc l'empreinte RAM au repos.
5. J'ai configuré une IP statique dès l'installation (une réservation qu'on peut aussi faire via netplan après coup) — un serveur doit toujours avoir la même adresse.
6. J'ai coché uniquement « OpenSSH server » dans les options, sans rien installer de superflu (pas de snap).

**Contrainte physique :** mon routeur était beaucoup trop loin de l'emplacement prévu pour la tour, et je ne pouvais pas le déplacer. J'ai donc acheté un prolongateur avec un port Ethernet, que j'ai relié à un switch (pour pouvoir brancher d'autres appareils filaires plus tard), puis j'ai connecté ma tour à ce switch.

## Étape 2 — Mises à jour et préparation du système

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nano htop
```

Après la mise à jour, j'ai choisi UTF-8 comme encodage pour la console, puis l'option Latin1/Latin5 - Western Europe and Romanian, et enfin la disposition de clavier English (US).

`htop` me permet de surveiller ma consommation RAM en temps réel — utile pour valider que je reste dans mes 4 Go.

## Étape 3 — Installer Docker et Docker Compose

J'ai suivi le guide officiel : https://docs.docker.com/engine/install/ubuntu/

```bash
sudo systemctl status docker
docker --version
docker compose version
```

### Problème rencontré : SSH ne fonctionnait pas

Je n'arrivais pas à me connecter en SSH à cause de mon routeur TP-Link. J'ai d'abord vérifié la configuration SSH directement sur le serveur :

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

Le SSH ne fonctionnant toujours pas, j'ai tenté d'installer Cockpit comme interface d'administration alternative :

```bash
. /etc/os-release
sudo apt install -t ${VERSION_CODENAME}-backports cockpit
sudo systemctl enable --now cockpit.socket
```

Cela n'a pas réglé le problème non plus. En creusant, voici ce que j'ai observé :

1. Le TP-Link a l'adresse IP `10.0.0.210`.
2. L'adresse lien-local (ether/link) de mon serveur Ubuntu correspondait à l'adresse IPv6 du TP-Link.
3. Mon serveur n'avait pas d'adresse IPv4 visible avec `ip a`, même si je l'avais configuré en statique avec `10.0.0.210`.
4. Le serveur avait pourtant accès à Internet — j'ai pu télécharger Docker, Cockpit et faire mes mises à jour sans problème.
5. Le serveur était bien branché sur un port Ethernet du TP-Link.
6. Sur le routeur, j'avais réservé l'adresse `10.0.0.210` à l'équipement qui partageait la même adresse IPv6 que le TP-Link.
7. Je ne connaissais pas la vraie adresse MAC de mon serveur Ubuntu et je n'arrivais pas à la trouver dans l'interface du routeur — je ne savais même pas si elle y apparaissait.

J'ai fini par corriger la configuration réseau directement via netplan, avec une adresse statique différente (`10.0.0.65`) :

```yaml
network:
  ethernets:
    enp0s10:
      addresses:
        - 10.0.0.65/24
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
      # Ports DNS indispensables
      - "53:53/tcp"
      - "53:53/udp"
      # Interface Web (changer 80 par 8080 si le port 80 est déjà utilisé)
      - "80:80/tcp"
      - "443:443/tcp"
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

Enregistrer et quitter (`Ctrl+O`, Entrée, puis `Ctrl+X`).

2. Rediriger `/etc/resolv.conf` vers le résolveur statique du système, pour ne pas perdre la connexion Internet locale :

```bash
sudo rm /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

Vérification — le port 53 doit être libre (la commande ne doit rien retourner) :

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
3. (Optionnel mais recommandé) Je garde un DNS secondaire externe (ex. : 1.1.1.1) en cas de panne du Pi-hole.

## Résultats attendus

- Pi-hole actif, bloquant les pubs/trackers pour tout le réseau
- Consommation RAM totale (OS + Docker + Pi-hole) sous ~800 Mo, laissant de la marge pour le prochain service
- Structure `~/homelab/` prête à accueillir le prochain `docker-compose.yml` (WireGuard)

## Prochaine étape (Labo 02)

Installer WireGuard (VPN) en conteneur, configuré pour utiliser Pi-hole comme résolveur DNS — accès distant sécurisé et blocage de pubs même hors réseau local.

---

*Labo réalisé dans le cadre du projet de transformation d'une tour Acer en serveur maison personnel.*
