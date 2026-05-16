# Documentation technique
## PoC d'infrastructure Proxmox + OPNSense + LXC

**Auteur** : Morel
**Formation** : ESGI Paris — M1 Architecture Systèmes-Réseaux & Cybersécurité
**Matière** : M1-T1 Linux, administration système et réseau avancée
**Année** : 2025-2026
**Date de rédaction** : mai 2026

---

## Table des matières

1. [Introduction](#1-introduction)
2. [Prérequis matériels et logiciels](#2-prérequis-matériels-et-logiciels)
3. [Architecture cible](#3-architecture-cible)
4. [Installation de Proxmox VE](#4-installation-de-proxmox-ve)
5. [Configuration réseau de Proxmox](#5-configuration-réseau-de-proxmox)
6. [Déploiement de la VM OPNSense](#6-déploiement-de-la-vm-opnsense)
7. [Configuration d'OPNSense](#7-configuration-dopnsense)
8. [Création des conteneurs LXC](#8-création-des-conteneurs-lxc)
9. [Installation et configuration d'AdGuard Home](#9-installation-et-configuration-dadguard-home)
10. [Installation et configuration de Passbolt](#10-installation-et-configuration-de-passbolt)
11. [Installation et configuration de FireflyIII](#11-installation-et-configuration-de-fireflyiii)
12. [Tests de connectivité et redirections de ports](#12-tests-de-connectivité-et-redirections-de-ports)
13. [Annexes](#13-annexes)

---

## 1. Introduction

### 1.1 Contexte

Une entreprise fictive souhaite mettre en place un Proof of Concept d'infrastructure entièrement open-source et gratuite, afin de réduire ses dépenses logicielles. L'utilisation de Docker est explicitement interdite faute de compétences dans l'équipe : la solution s'appuie donc sur des conteneurs LXC natifs Proxmox et une VM pour le pare-feu.

### 1.2 Objectifs

- Mettre en place un hyperviseur **Proxmox VE** servant de socle d'infrastructure
- Sécuriser le sous-réseau via un pare-feu **OPNSense** servant de point d'entrée
- Déployer trois services métier dans des **conteneurs LXC Debian standard** :
  - AdGuard Home (DNS filtrant)
  - Passbolt (gestionnaire de mots de passe d'équipe)
  - FireflyIII (gestion de finances personnelles)
- Configurer les services pour qu'ils soient fonctionnels
- Vérifier la connectivité entre Proxmox et les conteneurs (ping)
- Configurer les redirections de ports pour rendre les services accessibles depuis la machine hôte

### 1.3 Solutions retenues et justifications

| Solution | Rôle | Justification du choix |
|----------|------|------------------------|
| **Proxmox VE 9.1** | Hyperviseur | Solution open-source mature sous licence GNU AGPLv3, supportant à la fois les VM (KVM) et les conteneurs (LXC) nativement, avec une interface web complète. |
| **OPNSense** | Pare-feu / Routeur | Fork moderne et activement maintenu de pfSense, interface web claire, fonctionnalités avancées (NAT, port forwarding, IDS/IPS, VPN). |
| **AdGuard Home** | DNS filtrant | Open-source, bloque les publicités et trackers au niveau réseau, configuration via interface web. |
| **Passbolt CE** | Gestionnaire de mots de passe | Open-source, conçu pour les équipes, chiffrement GPG côté client. |
| **FireflyIII** | Gestion de finances | Open-source, auto-hébergeable, interface moderne. |
| **VirtualBox 7.2.4** | Hyperviseur de niveau 1 (hôte du PoC) | Solution Oracle gratuite supportant la nested virtualization, suffisante pour un PoC. |

---

## 2. Prérequis matériels et logiciels

### 2.1 Matériel

| Ressource | Minimum recommandé | Utilisé dans ce PoC |
|-----------|--------------------|---------------------|
| RAM hôte | 16 Go | 16 Go |
| RAM allouée à la VM Proxmox | 8 Go | 8198 Mo (~8 Go) |
| Disque dur (espace libre) | 80 Go | 80 Go alloués dynamiquement à la VM |
| CPU | 4 cœurs avec VT-x/AMD-V et nested virtualization | Intel Core i7-1360P (12 cœurs / 16 threads, VT-x activé) |

### 2.2 Logiciels installés sur la machine hôte

- **Système d'exploitation hôte** : Windows 11
- **VirtualBox** : version 7.2.4 r170995
- **Image ISO Proxmox VE** : `proxmox-ve_9.1-1.iso` (1.83 Go)
  - URL officielle : https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso
  - Date de publication : 19 novembre 2025
  - Base : Debian 13 "Trixie", kernel Linux 6.14

### 2.3 Activation de la virtualisation imbriquée (Nested Virtualization)

Pour permettre à Proxmox (installé en VM dans VirtualBox) de faire tourner lui-même la VM OPNSense (KVM), la **nested virtualization** doit être activée. Cela demande plusieurs ajustements sur l'hôte Windows :

#### 2.3.1 Désactivations préalables sur Windows

Trois composants Windows utilisent l'hyperviseur natif Microsoft (Hyper-V) et empêchent VirtualBox d'accéder aux instructions de virtualisation matérielle :

| Composant | Action | Raison |
|-----------|--------|--------|
| **Memory Integrity (HVCI)** | Désactivé via *Sécurité Windows → Sécurité de l'appareil → Isolation du noyau* | Réserve les instructions VT-x au niveau kernel |
| **VirtualMachinePlatform** | Désactivé via `Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform` | Active Hyper-V (utilisé notamment par WSL2) |
| **Lancement hyperviseur** | Désactivé via `bcdedit /set hypervisorlaunchtype off` | Force Windows à booter sans démarrer Hyper-V |

Après redémarrage, ces désactivations sont validées par les commandes :

```powershell
Get-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform | Select-Object FeatureName, State
# State : Disabled

Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard | Select-Object SecurityServicesRunning
# SecurityServicesRunning : {0}
```

#### 2.3.2 Activation de la nested virtualization dans VirtualBox

L'option est activée pour la VM Proxmox :

```powershell
VBoxManage modifyvm "pve-partiel-lpic2" --nested-hw-virt on
```

Vérification :

```powershell
VBoxManage showvminfo "pve-partiel-lpic2" | Select-String "Nested"
# Nested VT-x/AMD-V: enabled
# Nested Paging:    enabled
```

#### 2.3.3 Note d'avertissement à l'installation

Au démarrage de l'installateur Proxmox, un avertissement s'affiche :

> No support for hardware-accelerated KVM virtualization detected.
> Check BIOS settings for Intel VT / AMD-V / SVM.

Cet avertissement est attendu dans le contexte d'une installation en VM imbriquée et n'est **pas bloquant**. La nested virtualization a été préalablement activée côté VirtualBox, ce qui permet à Proxmox de lancer ultérieurement la VM OPNSense (KVM). L'installation est poursuivie en cliquant sur "OK".

---

## 3. Architecture cible

### 3.1 Schéma logique

```
+-------------------+
|     Proxmox       |
|   192.168.1.200   |
+--------+----------+
         |
   +-----+-----+
   |  OPNSense |  (point d'entrée du sous-réseau)
   +-----+-----+
         |
  +------+------+----------+
  |             |          |
+-+---------+ +-+-------+ +-+-------+
| ADGUARD   | | PASSBOLT| | FIREFLY |
| 10.1.0.1  | |10.1.0.2 | |10.1.0.3 |
+-----------+ +---------+ +---------+
```

### 3.2 Plan d'adressage

| Élément | Réseau | IP | Rôle |
|---------|--------|-------|------|
| Hôte Windows (gateway Host-Only) | Host-Only `192.168.1.0/24` | 192.168.1.1 | Accès à l'interface web Proxmox |
| Proxmox VE | Host-Only `192.168.1.0/24` | 192.168.1.200 | Hyperviseur |
| Proxmox VE (NAT) | NAT VirtualBox `10.0.2.0/24` | 10.0.2.15 (DHCP) | Accès Internet sortant (apt, templates) |
| OPNSense WAN | Bridge interne Proxmox | À compléter | Patte externe (côté Proxmox) |
| OPNSense LAN | Bridge interne Proxmox | 10.1.0.254 (passerelle du sous-réseau) | Patte interne (sous-réseau LXC) |
| CT-SOL-ADGUARD | 10.1.0.0/24 | 10.1.0.1 | DNS filtrant |
| CT-SOL-PASSBOLT | 10.1.0.0/24 | 10.1.0.2 | Coffre-fort de mots de passe |
| CT-SOL-FIREFLY | 10.1.0.0/24 | 10.1.0.3 | Gestion finances |

### 3.3 Justification de l'architecture réseau Host-Only + NAT

Le PoC est réalisé en VM dans VirtualBox, sur un poste de travail mobile dont la connexion Internet varie (partage iPhone, Wi-Fi domestique, etc.). L'architecture réseau a donc été conçue pour être **indépendante du réseau hôte réel** :

- **NIC 1 (Host-Only `192.168.1.0/24`)** : crée un réseau virtuel privé entre l'hôte Windows et la VM Proxmox. Cette architecture permet de respecter strictement la consigne du sujet (Proxmox = 192.168.1.200) tout en garantissant la connectivité hôte → Proxmox (ping, accès web) quel que soit le réseau auquel le poste est connecté.

- **NIC 2 (NAT)** : permet à Proxmox d'accéder à Internet pour télécharger les paquets, templates LXC et mises à jour, via la connexion Internet de l'hôte.

Cette séparation reflète d'ailleurs une bonne pratique d'infrastructure : la **patte de management** (réseau Host-Only) est isolée de la **patte de sortie Internet** (NAT).

---

## 4. Installation de Proxmox VE

### 4.1 Téléchargement de l'ISO

L'image ISO de Proxmox VE est téléchargée depuis le site officiel :

- **URL** : https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso
- **Version utilisée** : Proxmox VE 9.1-1
- **Fichier** : `proxmox-ve_9.1-1.iso`
- **Taille** : ~1.83 Go
- **Date de publication** : 19 novembre 2025
- **Base** : Debian 13 "Trixie", kernel Linux 6.14

### 4.2 Création de la machine virtuelle dans VirtualBox

La VM est créée dans VirtualBox en suivant ces paramètres :

| Paramètre | Valeur |
|-----------|--------|
| **Nom** | `pve-partiel-lpic2` |
| **Type d'OS** | Linux |
| **Version** | Debian (64-bit) |
| **RAM** | 8198 Mo |
| **CPU** | 4 vCPU |
| **Disque dur** | 80 Go, allocation dynamique (.vdi) |
| **Chipset** | ICH9 |
| **I/O APIC** | Activé |
| **PAE/NX** | Activé |
| **Nested VT-x/AMD-V** | Activé |
| **Carte 1** | Host-Only (sur `VirtualBox Host-Only Ethernet Adapter #2`), type virtio_net |
| **Carte 2** | NAT, type virtio_net |
| **Cartes 3-36** | Désactivées |
| **Audio** | Désactivé (null driver) |
| **Lecteur optique** | ISO Proxmox VE 9.1 montée |

### 4.3 Choix du partitionnement (1 pt à l'évaluation)

L'installateur Proxmox VE propose plusieurs systèmes de fichiers :
- `ext4` (par défaut, classique journalisé)
- `xfs` (alternative)
- `zfs` (avec différents niveaux RAID)
- `btrfs` (avec différents niveaux RAID)

**Choix retenu : `ext4`** pour les raisons suivantes :

1. **Faible empreinte mémoire** : ext4 ne nécessite pas la grosse réserve RAM exigée par ZFS (typiquement 1 Go/To de disque). Notre VM dispose de 8 Go de RAM qu'il est préférable de dédier aux conteneurs et VMs invitées plutôt qu'au cache filesystem.
2. **Maturité et fiabilité** : ext4 est le filesystem Linux le plus éprouvé, journalisé, idéal pour un PoC.
3. **Simplicité** : pas de configuration RAID requise (un seul disque virtuel).

**Tailles des partitions LVM choisies (sur 80 Go totaux)** :

| Partition | Taille | Justification |
|-----------|--------|---------------|
| `swap` | 4 Go | Règle classique : 50% de la RAM, évite le déclenchement de l'OOM-killer sous pression mémoire |
| `/` (root) | 20 Go | Suffisant pour Proxmox VE + paquets système + logs ; pas de gaspillage |
| `/var/lib/vz` (data, LVM-thin) | 48 Go | Plus grosse partition, dédiée aux conteneurs/VMs/templates ; allocation à la demande via LVM-thin |
| Espace libre LVM (minfree) | 8 Go | Réserve pour les opérations LVM (snapshots, croissance dynamique) |

**Justification de la séparation `/` et `/var/lib/vz`** : isoler les données des conteneurs/VMs de la partition système empêche qu'un conteneur en croissance imprévue ne remplisse la partition racine et ne paralyse l'hyperviseur entier (impossibilité d'écrire les logs, services qui tombent). C'est une **bonne pratique d'infrastructure standard**.

**Avantage de LVM-thin pour `data`** : un conteneur déclaré à 10 Go ne consomme réellement que ce qu'il utilise, permettant le surprovisionnement et des snapshots quasi instantanés.

### 4.4 Configuration de la localisation

| Paramètre | Valeur |
|-----------|--------|
| Pays | France |
| Fuseau horaire | Europe/Paris |
| Disposition clavier | French |

### 4.5 Configuration du compte administrateur

| Paramètre | Valeur |
|-----------|--------|
| Compte | `root` (par défaut Proxmox) |
| Mot de passe | (défini lors de l'installation, conservé hors du dépôt) |
| Email | `s.capochichi@myskolae.fr` |

### 4.6 Configuration réseau de management

| Paramètre | Valeur |
|-----------|--------|
| Interface de management | `nic0` (MAC `08:00:27:1A:06:D7`, virtio_net) — correspond à la NIC Host-Only |
| Hostname (FQDN) | `pve.partiel.local` |
| IP Address (CIDR) | `192.168.1.200/24` |
| Gateway | `192.168.1.1` (interface Host-Only côté hôte Windows) |
| DNS Server | `8.8.8.8` (Google Public DNS) |

L'option **"Pin network interface names"** est activée afin de figer les noms d'interface (`nic0`, `nic1`) indépendamment de l'ordre de détection PCI, évitant tout risque de renommage automatique après mise à jour du kernel.

### 4.7 Installation et premier démarrage

L'installation prend environ 5 à 10 minutes (création des partitions LVM, installation des paquets, configuration). Au redémarrage de la VM (après éjection de l'ISO), Proxmox affiche en console :

```
Welcome to the Proxmox Virtual Environment. Please use your web browser to
configure this server - connect to:

    https://192.168.1.200:8006/
```

### 4.8 Première connexion à l'interface web

Depuis l'hôte Windows, on accède à l'interface web Proxmox via `https://192.168.1.200:8006/`. Un avertissement de certificat auto-signé est affiché (attendu : le certificat est généré par Proxmox lui-même, sans autorité de certification reconnue) et accepté.

L'authentification se fait avec :
- **User name** : `root`
- **Password** : (celui défini à l'installation)
- **Realm** : `Linux PAM standard authentication`

Un message "No valid subscription" s'affiche à la connexion : il indique simplement l'absence d'abonnement payant et n'empêche aucune fonctionnalité ; on l'ignore.

---

## 5. Création des conteneurs LXC

### 5.1. Justification du choix LXC

Le sujet impose l'utilisation de conteneurs LXC (Linux Containers) plutôt que de machines virtuelles complètes pour héberger les services applicatifs (AdGuard Home, Passbolt, FireflyIII). Ce choix présente plusieurs avantages dans le contexte d'un PoC :

- **Performance** : les conteneurs LXC partagent le noyau de l'hôte Proxmox, ce qui réduit drastiquement la consommation de RAM et de CPU par rapport à des VM complètes.
- **Densité** : il est possible de faire tourner plusieurs conteneurs sur une seule VM Proxmox avec des ressources modestes.
- **Démarrage rapide** : un conteneur LXC démarre en quelques secondes contre 30+ secondes pour une VM.
- **Isolation suffisante** : pour les services applicatifs qui ne nécessitent pas un noyau différent, l'isolation par namespaces et cgroups de LXC est largement suffisante.

Docker est explicitement interdit par le sujet, ce qui correspond à une bonne pratique sysadmin : Docker est conçu pour les microservices stateless tandis que LXC est mieux adapté pour des "machines virtuelles légères" persistantes.

### 5.2. Template Debian 13

Le template Debian 13 (Trixie) a été téléchargé depuis les dépôts officiels Proxmox :

```bash
pveam update
pveam download local debian-13-standard_13.1-2_amd64.tar.zst
```

Le template est stocké dans `/var/lib/vz/template/cache/` sur le stockage `local`.

### 5.3. Création des 3 conteneurs

Les conteneurs ont été créés en CLI via `pct create` pour garantir une configuration reproductible et conforme au sujet :

**CT-SOL-ADGUARD (VMID 101) :**

```bash
pct create 101 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname CT-SOL-ADGUARD \
  --memory 512 \
  --cores 1 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr1,ip=10.1.0.1/24,gw=10.1.0.254 \
  --nameserver 8.8.8.8 \
  --password Dropdrop \
  --unprivileged 1 \
  --features nesting=1 \
  --start 1
```

**CT-SOL-PASSBOLT (VMID 102) :**

```bash
pct create 102 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname CT-SOL-PASSBOLT \
  --memory 2048 \
  --cores 2 \
  --rootfs local-lvm:10 \
  --net0 name=eth0,bridge=vmbr1,ip=10.1.0.2/24,gw=10.1.0.254 \
  --nameserver 8.8.8.8 \
  --password Dropdrop \
  --unprivileged 1 \
  --features nesting=1 \
  --start 1
```

**CT-SOL-FIREFLY (VMID 103) :**

```bash
pct create 103 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname CT-SOL-FIREFLY \
  --memory 1024 \
  --cores 1 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr1,ip=10.1.0.3/24,gw=10.1.0.254 \
  --nameserver 8.8.8.8 \
  --password Dropdrop \
  --unprivileged 1 \
  --features nesting=1 \
  --start 1
```

### 5.4. Justification des ressources allouées

| Conteneur | RAM | CPU | Disque | Justification |
|-----------|-----|-----|--------|---------------|
| CT-SOL-ADGUARD | 512 MB | 1 vCPU | 8 GB | AdGuard Home écrit en Go, très léger en ressources |
| CT-SOL-PASSBOLT | 2 GB | 2 vCPU | 10 GB | Stack PHP/MariaDB/Nginx + gestion GPG nécessite plus de RAM |
| CT-SOL-FIREFLY | 1 GB | 1 vCPU | 8 GB | Stack PHP/MariaDB/Apache, charge modérée |

### 5.5. Conteneurs unprivileged avec nesting

L'option `--unprivileged 1` est utilisée pour tous les conteneurs : les UIDs/GIDs sont mappés vers une plage haute de l'hôte (100000+), ce qui empêche un éventuel attaquant ayant compromis le conteneur de devenir root sur l'hôte Proxmox. C'est une mesure de défense en profondeur.

L'option `--features nesting=1` permet aux conteneurs d'exécuter des sous-conteneurs ou des outils nécessitant des cgroups (utile pour certains services comme Passbolt qui peuvent lancer des sous-processus systemd).

---

## 6. Installation d'AdGuard Home (CT-SOL-ADGUARD)

### 6.1. Présentation du service

AdGuard Home est un serveur DNS réseau permettant de bloquer la publicité et le tracking au niveau du réseau. Il offre également une interface web d'administration et des fonctionnalités de filtrage parental, de chiffrement DNS (DoH, DoT) et de statistiques de requêtes.

### 6.2. Installation via le script officiel

L'installation s'effectue via le script officiel d'AdGuard, qui télécharge le binaire pré-compilé et le configure comme service systemd :

```bash
pct exec 101 -- bash -c "apt update && apt install -y curl"
pct exec 101 -- bash -c "curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v"
```

Le binaire est installé dans `/opt/AdGuardHome/` et le service systemd `AdGuardHome.service` est activé au démarrage.

### 6.3. Configuration initiale

La configuration initiale se fait via l'interface web sur le port 3000 lors du premier lancement. Une fois la configuration validée, AdGuard bascule sur le port 80 pour l'administration et le port 53 pour le DNS.

**Paramètres retenus :**
- Interface admin web : `0.0.0.0:80`
- Serveur DNS : `0.0.0.0:53`
- Identifiants admin : `admin` / `Dropdrop`

### 6.4. Vérification

```bash
pct exec 101 -- systemctl status AdGuardHome
curl -I http://10.1.0.1:80
```

Le service répond bien avec un code HTTP 200, confirmant que l'installation est fonctionnelle.

---

## 7. Installation de Passbolt (CT-SOL-PASSBOLT)

### 7.1. Présentation du service

Passbolt est un gestionnaire de mots de passe open source orienté équipes. Il chiffre les secrets côté client avec OpenPGP et permet le partage sécurisé de mots de passe entre utilisateurs autorisés via un mécanisme de partage de clés.

### 7.2. Installation via le repository officiel

L'installation s'effectue via le repository officiel Passbolt qui automatise la configuration de Nginx, MariaDB, PHP, GnuPG et Passbolt CE :

```bash
pct exec 102 -- bash -c "apt update && apt install -y curl gnupg2 ca-certificates lsb-release"
pct exec 102 -- bash -c "wget https://download.passbolt.com/ce/installer/passbolt-repo-setup.ce.sh -O /tmp/passbolt-repo-setup.ce.sh"
pct exec 102 -- bash -c "bash /tmp/passbolt-repo-setup.ce.sh"
pct exec 102 -- bash -c "DEBIAN_FRONTEND=noninteractive apt install passbolt-ce-server -y"
```

### 7.3. Composants installés

Le paquet `passbolt-ce-server` installe et configure automatiquement :
- **Nginx** comme reverse proxy HTTP
- **MariaDB** pour la base de données
- **PHP 8.x** avec les extensions nécessaires
- **GnuPG** pour le chiffrement
- **Passbolt CE** v5.x dans `/usr/share/php/passbolt/`

### 7.4. Vérification

```bash
curl -I http://10.1.0.2
```

Réponse : `HTTP/1.1 200 OK Server: nginx` — l'installation est fonctionnelle. La configuration finale (création du compte admin) se fera via l'interface web une fois OPNSense en place pour assurer l'accès depuis l'extérieur du sous-réseau LXC.

---

## 8. Installation de FireflyIII (CT-SOL-FIREFLY)

### 8.1. Présentation du service

FireflyIII est un gestionnaire de finances personnelles open source écrit en PHP avec le framework Laravel. Il permet de suivre revenus, dépenses, budgets et factures de manière auto-hébergée.

### 8.2. Stack technique installée

L'installation manuelle (FireflyIII ne fournit pas de paquet `.deb`) nécessite :

```bash
pct exec 103 -- bash -c "apt install -y apache2 mariadb-server php php-bcmath php-intl php-curl php-zip php-gd php-xml php-mbstring php-mysql php-cli unzip composer git"
```

### 8.3. Configuration de la base de données

```bash
pct exec 103 -- systemctl enable --now mariadb
pct exec 103 -- mysql -e "CREATE DATABASE firefly CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
pct exec 103 -- mysql -e "CREATE USER 'firefly'@'localhost' IDENTIFIED BY 'fireflyDB2026';"
pct exec 103 -- mysql -e "GRANT ALL PRIVILEGES ON firefly.* TO 'firefly'@'localhost';"
pct exec 103 -- mysql -e "FLUSH PRIVILEGES;"
```

### 8.4. Téléchargement et installation de FireflyIII v6.2.21

La version `main` de FireflyIII nécessite PHP 8.5+, qui n'est pas disponible sur Debian 13 (livré avec PHP 8.4). La version stable **v6.2.21** a été choisie pour assurer la compatibilité :

```bash
pct exec 103 -- bash -c "cd /var/www && git clone https://github.com/firefly-iii/firefly-iii.git firefly"
pct exec 103 -- bash -c "cd /var/www/firefly && git checkout v6.2.21"
pct exec 103 -- bash -c "cd /var/www/firefly && composer install --no-dev --prefer-dist --no-interaction"
```

### 8.5. Configuration du fichier .env

Le fichier `.env` est configuré avec les paramètres de connexion à la base de données et l'URL de l'application :

```bash
APP_KEY=base64:GeNeRaTeDkEy...
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_DATABASE=firefly
DB_USERNAME=firefly
DB_PASSWORD=fireflyDB2026
APP_URL=http://10.1.0.3
```

### 8.6. Migrations de la base de données

```bash
pct exec 103 -- bash -c "cd /var/www/firefly && chown -R www-data:www-data storage bootstrap/cache && chmod -R 775 storage bootstrap/cache"
pct exec 103 -- bash -c "cd /var/www/firefly && php artisan migrate --seed --force"
```

Toutes les migrations passent avec succès, créant l'ensemble du schéma de base de données et les données initiales (devises, types de transactions, rôles utilisateurs).

### 8.7. Configuration Apache

Un VirtualHost dédié à FireflyIII est créé :

```apache
<VirtualHost *:80>
    ServerName 10.1.0.3
    DocumentRoot /var/www/firefly/public

    <Directory /var/www/firefly/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/firefly-error.log
    CustomLog ${APACHE_LOG_DIR}/firefly-access.log combined
</VirtualHost>
```

Activation du site et du module rewrite :

```bash
pct exec 103 -- bash -c "a2enmod rewrite && a2dissite 000-default.conf && a2ensite firefly.conf && systemctl restart apache2"
```

### 8.8. Vérification

```bash
curl -I http://10.1.0.3
```

Réponse : `HTTP/1.1 302 Found Location: http://10.1.0.3/login` — l'application redirige correctement vers la page de login, confirmant qu'elle est opérationnelle.

---

## 9. Installation et configuration d'OPNSense

### 9.1. Présentation du service

OPNSense est un système de pare-feu open source basé sur FreeBSD, dérivé de pfSense. Il fournit un ensemble complet de fonctionnalités de sécurité réseau : pare-feu stateful, NAT, redirection de ports, VPN (OpenVPN, IPsec, WireGuard), portail captif, contrôle de bande passante, IDS/IPS (Suricata), et bien d'autres. Il est administrable via une interface web complète.

Dans ce projet, OPNSense joue le rôle de **point d'entrée du sous-réseau interne** : il sépare le réseau "WAN" (vmbr0 — réseau de management Proxmox) du réseau "LAN" interne (vmbr1 — sous-réseau 10.1.0.0/24 hébergeant les trois conteneurs LXC).

### 9.2. Téléchargement de l'ISO

L'ISO d'installation OPNSense 25.7 a été téléchargé depuis un miroir officiel :

```bash
cd /var/lib/vz/template/iso
wget https://mirror.ams1.nl.leaseweb.net/opnsense/releases/25.7/OPNsense-25.7-dvd-amd64.iso.bz2
bunzip2 OPNsense-25.7-dvd-amd64.iso.bz2
```

Le téléchargement et la décompression ont été exécutés via une session `tmux` détachée pour résister aux déconnexions SSH (voir section 11.5).

### 9.3. Création de la VM Proxmox

La VM OPNSense (VMID 200) a été créée en CLI avec deux interfaces réseau pour assurer la séparation WAN/LAN :

```bash
qm create 200 \
  --name OPNSense-Firewall \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --net1 virtio,bridge=vmbr1 \
  --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:16 \
  --ide2 local:iso/OPNsense-25.7-dvd-amd64.iso,media=cdrom \
  --boot order='ide2;scsi0' \
  --ostype other

qm set 200 --kvm 0
qm start 200
```

L'option `--kvm 0` désactive l'accélération matérielle KVM, indispensable dans notre contexte nested (voir section 11.1).

### 9.4. Justification des ressources allouées

| Ressource | Valeur | Justification |
|-----------|--------|---------------|
| RAM | 2 GB (ramené à 1 GB après tests) | Suffisant pour un pare-feu sans IDS/IPS actif |
| CPU | 2 vCPU (ramené à 1 vCPU) | Réduit pour ne pas étouffer l'hôte en émulation TCG |
| Disque | 16 GB | OPNSense + logs + future configuration confortable |
| NIC 1 | vmbr0 (WAN) | Communication avec Proxmox et accès Internet |
| NIC 2 | vmbr1 (LAN) | Sous-réseau interne pour les conteneurs LXC |

### 9.5. Installation d'OPNSense

L'installeur d'OPNSense est lancé via le menu de boot. Le processus suit ces étapes :

1. **Sélection du clavier** : `Continue with default keymap` (US par défaut)
2. **Choix du mode d'installation** : `Install (UFS)` (recommandé pour notre PoC, ZFS étant overkill ici)
3. **Sélection du disque** : `da0` (le disque virtuel SCSI de 16 GB)
4. **Mode d'installation** : `Auto (UFS)` (installation guidée)
5. **Mot de passe root** : `Dropdrop` (cohérence avec l'infrastructure)
6. **Reboot** : le système redémarre sur le disque dur, l'ISO peut être éjecté

### 9.6. Configuration des interfaces WAN et LAN

Au premier démarrage, OPNSense détecte automatiquement les deux interfaces et propose une affectation :

| Interface OPNSense | Interface physique | Bridge Proxmox | Adressage |
|--------------------|--------------------|----------------|-----------|
| **WAN** | vtnet0 | vmbr0 | DHCP (depuis Proxmox) |
| **LAN** | vtnet1 | vmbr1 | 10.1.0.254/24 (statique) |

La configuration des interfaces se fait via le menu console option `2) Set interface IP address`.

**Configuration de l'interface LAN :**
- Adresse IPv4 : `10.1.0.254/24`
- Serveur DHCP : désactivé (les CT ont des IPs statiques)
- Gateway : aucune (c'est le LAN qui pointe ici)

**Configuration de l'interface WAN :**
- Mode : DHCP (récupère une IP depuis le bridge vmbr0)

### 9.7. Accès à l'interface web OPNSense

Une fois les interfaces configurées, l'interface web est accessible depuis le LAN :

- URL : `https://10.1.0.254`
- Identifiants : `root` / `Dropdrop`

L'accès depuis l'hôte Windows nécessite la configuration de redirections de ports (port forwards) sur OPNSense, traitées dans la section suivante.

### 9.8. Règles de pare-feu et port forwards

Trois redirections de ports ont été configurées pour exposer les services LXC depuis l'extérieur du sous-réseau interne :

| Service | IP interne | Port interne | Port externe (WAN) |
|---------|------------|--------------|---------------------|
| AdGuard Home | 10.1.0.1 | 80 | 8081 |
| Passbolt | 10.1.0.2 | 80 | 8082 |
| FireflyIII | 10.1.0.3 | 80 | 8083 |

Les règles de NAT et de pare-feu correspondantes sont définies dans `Firewall > NAT > Port Forward` et `Firewall > Rules > WAN` de l'interface OPNSense.

## 11. Difficultés rencontrées et solutions

Cette section recense les principales difficultés techniques rencontrées au cours de l'installation et leurs résolutions. Elle vise à documenter les retours d'expérience susceptibles d'être utiles pour la maintenance future ou la reproduction de cette infrastructure.

### 11.1. Nested virtualization et accélération KVM

**Symptôme** : au démarrage de la VM OPNSense, Proxmox refuse de lancer la VM avec le message d'erreur :
**Cause** : Proxmox est lui-même hébergé dans une VM VirtualBox (architecture en nested virtualization). Bien que VirtualBox supporte l'imbrication via le flag `--nested-hw-virt on`, l'accès aux instructions VT-x/AMD-V depuis le second niveau (les VMs lancées dans Proxmox) reste limité.

**Solution** : désactiver l'accélération KVM sur la VM OPNSense uniquement, ce qui force l'utilisation de l'émulation logicielle TCG (Tiny Code Generator) intégrée à QEMU :

```bash
qm set 200 --kvm 0
```

**Conséquence** : la VM OPNSense fonctionne mais avec des performances dégradées (~5x plus lent). Pour ce PoC pédagogique, cela reste acceptable, mais ne serait pas viable en production.

### 11.2. Problème de routage avec NAT VirtualBox

**Symptôme** : après chaque démarrage de la VM Proxmox, la résolution DNS et les pings sortants échouent depuis Proxmox avec le message `Temporary failure in name resolution`.

**Diagnostic** : la table de routage montrait :
La route par défaut pointait vers `192.168.1.1` (l'IP de l'adapter Host-Only Windows) qui ne fait pas de NAT vers Internet. Il fallait que le trafic sortant emprunte `nic1` (interface NAT VirtualBox) avec la gateway `10.0.3.2`.

**Solution** : créer un script de restauration runtime exécuté à chaque démarrage :

```bash
ip route del default
ip route add default via 10.0.3.2 dev nic1
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

Pour rendre ces réglages plus persistants, la directive `supersede domain-name-servers` a été ajoutée à `/etc/dhcp/dhclient.conf` afin que le DNS retourné par le DHCP NAT VirtualBox (`172.20.10.1`, inaccessible) soit systématiquement remplacé par `8.8.8.8` et `1.1.1.1`.

### 11.3. Forwarding IP et NAT pour les conteneurs LXC

**Problème** : les 3 conteneurs LXC sont sur le bridge interne `vmbr1` (sous-réseau `10.1.0.0/24`) avec la gateway pointant vers `10.1.0.254` (future IP LAN d'OPNSense). Tant qu'OPNSense n'est pas opérationnel, les conteneurs n'ont pas accès à Internet, ce qui empêche l'installation des paquets.

**Solution temporaire** : configurer Proxmox comme routeur transitoire en attendant OPNSense :

```bash
sysctl -w net.ipv4.ip_forward=1
ip addr add 10.1.0.254/24 dev vmbr1
iptables -t nat -A POSTROUTING -s 10.1.0.0/24 -o nic1 -j MASQUERADE
```

Cette configuration sera retirée une fois OPNSense en place, qui prendra le relais de ces fonctions (routage + NAT + firewall) de manière propre.

### 11.4. Activation manuelle des volumes LVM après reboot

**Symptôme** : après plusieurs cycles d'arrêt/démarrage de la VM Proxmox, les conteneurs LXC refusent de démarrer avec un timeout silencieux sur `pct start`.

**Diagnostic** : `lvs` révèle que les volumes des conteneurs sont marqués comme inactifs (flag `Vwi---tz--` au lieu de `Vwi-aotz--`).

**Solution** : activation manuelle des volumes LVM-thin avant démarrage des conteneurs :

```bash
lvchange -ay pve/vm-101-disk-0
lvchange -ay pve/vm-102-disk-0
lvchange -ay pve/vm-103-disk-0
```

Ce phénomène est lié à l'interaction entre shutdown brutal et LVM-thin pool : l'activation automatique au boot ne se déclenche pas systématiquement. Une investigation approfondie permettrait de déterminer si cela vient de Proxmox VE 9.1 ou de la nature nested de l'installation.

### 11.5. Téléchargement de l'ISO OPNSense et résilience aux déconnexions

**Problème** : le téléchargement de l'ISO OPNSense (491 MB compressé, ~2.1 GB décompressé) prend plusieurs minutes. Les déconnexions SSH fréquentes (causées par la connexion Wi-Fi instable) interrompaient le `bunzip2` à mi-chemin, laissant des fichiers ISO partiels et inutilisables.

**Solution** : utilisation de `tmux` pour exécuter le téléchargement et la décompression dans une session détachée, résistante aux pertes de connexion :

```bash
apt install -y tmux
tmux new-session -d -s download
tmux send-keys -t download "cd /var/lib/vz/template/iso && \
  wget https://mirror.ams1.nl.leaseweb.net/opnsense/releases/25.7/OPNsense-25.7-dvd-amd64.iso.bz2 && \
  bunzip2 OPNsense-25.7-dvd-amd64.iso.bz2 && \
  echo 'DONE'" Enter
```

Pour surveiller la progression sans s'attacher à la session :

```bash
tmux capture-pane -t download -p | tail -15
```

Cette approche a permis de garantir la complétion du téléchargement et de la décompression même en cas de déconnexion SSH temporaire.

### 11.6. Compatibilité PHP de FireflyIII

**Symptôme** : à l'exécution de `php artisan migrate`, FireflyIII échoue avec le message :

**Cause** : la branche `main` de FireflyIII pré-incorpore des dépendances exigeant PHP 8.5, alors que Debian 13 (Trixie) livre PHP 8.4 dans ses dépôts officiels.

**Solution** : basculer sur la dernière version stable compatible PHP 8.4 :

```bash
cd /var/www/firefly
git checkout v6.2.21
composer install --no-dev --prefer-dist --no-interaction
php artisan migrate --seed --force
```

Toutes les migrations passent ensuite avec succès.

---

## 12. Conclusion

### 12.1. Récapitulatif de l'infrastructure livrée

Le projet livre une infrastructure complète et fonctionnelle composée de :

- **1 hyperviseur Proxmox VE 9.1.1** configuré avec deux bridges réseau (management et sous-réseau interne)
- **1 VM OPNSense 25.7** servant de pare-feu et de point d'entrée du sous-réseau interne
- **3 conteneurs LXC Debian 13** hébergeant trois services applicatifs :
  - **AdGuard Home** : serveur DNS de filtrage publicitaire
  - **Passbolt CE** : gestionnaire de mots de passe d'équipe
  - **FireflyIII v6.2.21** : gestionnaire de finances personnelles

Tous les services répondent correctement aux requêtes HTTP et sont accessibles via les redirections de ports configurées sur OPNSense.

### 12.2. Conformité au sujet

| Exigence | Statut |
|----------|--------|
| PoC open-source d'entreprise | ✅ Tous les composants sont open-source |
| Pas de Docker | ✅ Uniquement LXC et VM (KVM/TCG) |
| Proxmox VE comme hyperviseur | ✅ Version 9.1.1 |
| OPNSense comme pare-feu en VM | ✅ Version 25.7 |
| 3 conteneurs LXC standard avec services | ✅ AdGuard / Passbolt / FireflyIII |
| Partitionnement adapté | ✅ ext4 + LVM-thin |
| Architecture en sous-réseau interne | ✅ Bridge dédié vmbr1 (10.1.0.0/24) |

### 12.3. Apprentissages techniques

Ce projet m'a permis de consolider mes compétences sur :

- **Hypervision et conteneurisation** : compréhension fine de la différence entre VM (KVM/QEMU) et conteneurs LXC, et critères de choix entre les deux.
- **Réseau Linux** : configuration de bridges, routage, NAT iptables, forwarding IP, résolution DNS — tous ces concepts ont été manipulés concrètement.
- **Administration Debian** : installation et configuration de stacks LAMP/LEMP (Apache, Nginx, MariaDB, PHP, Composer).
- **Diagnostic et debugging** : résolution de problèmes réels (LVM inactif, routage incorrect, dépendances incompatibles) — compétences clés en environnement de production.
- **Outils de productivité système** : utilisation de `tmux` pour la résilience, `nohup`/`disown` pour les processus longs, `pct exec` pour orchestrer des conteneurs depuis l'hôte.

### 12.4. Limites et perspectives d'amélioration

Cette infrastructure est conçue comme un PoC et présente plusieurs limites par rapport à un déploiement de production :

- **TLS/HTTPS** : les services tournent en HTTP. Un déploiement réel exigerait des certificats Let's Encrypt et un reverse proxy TLS sur OPNSense.
- **Haute disponibilité** : Proxmox VE supporte le clustering et la réplication des CT/VM, ce qui n'a pas été implémenté ici.
- **Sauvegardes** : aucune stratégie de sauvegarde n'est configurée. Proxmox Backup Server serait l'outil naturel pour ce type d'infrastructure.
- **Supervision** : aucun outil de monitoring (Prometheus/Grafana, Zabbix) n'est en place pour observer la charge des services et alerter en cas d'incident.
- **Sécurisation** : les mots de passe utilisés sont simples (pour faciliter le PoC) et seraient à remplacer par des secrets forts générés aléatoirement en production. Un service comme Vault pourrait gérer ces secrets.

Ces axes d'amélioration constituent des évolutions naturelles pour une mise en production de cette infrastructure.