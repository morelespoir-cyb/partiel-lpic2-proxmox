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

## 5. Configuration réseau de Proxmox

*(à remplir au fur et à mesure)*

### 5.1 Configuration de l'interface principale (vmbr0)
### 5.2 Création du bridge interne (vmbr1) pour le sous-réseau
### 5.3 Vérification (`/etc/network/interfaces`)

---

## 6. Déploiement de la VM OPNSense

*(à remplir)*

### 6.1 Téléchargement de l'image OPNSense
### 6.2 Création de la VM dans Proxmox
### 6.3 Installation d'OPNSense

---

## 7. Configuration d'OPNSense

*(à remplir)*

### 7.1 Configuration des interfaces (WAN/LAN)
### 7.2 Règles de pare-feu
### 7.3 Configuration NAT et redirections de ports
### 7.4 Export de la configuration

---

## 8. Création des conteneurs LXC

*(à remplir)*

### 8.1 Téléchargement du template Debian standard
### 8.2 Paramètres communs aux trois conteneurs
### 8.3 Création de CT-SOL-ADGUARD
### 8.4 Création de CT-SOL-PASSBOLT
### 8.5 Création de CT-SOL-FIREFLY

---

## 9. Installation et configuration d'AdGuard Home

*(à remplir)*

### 9.1 Installation
### 9.2 Configuration de base
### 9.3 Vérification du service

---

## 10. Installation et configuration de Passbolt

*(à remplir)*

### 10.1 Prérequis (MariaDB, Nginx, PHP, GPG)
### 10.2 Installation
### 10.3 Configuration HTTPS
### 10.4 Création du compte administrateur

---

## 11. Installation et configuration de FireflyIII

*(à remplir)*

### 11.1 Prérequis (PHP, base de données, serveur web)
### 11.2 Installation
### 11.3 Configuration initiale

---

## 12. Tests de connectivité et redirections de ports

*(à remplir)*

### 12.1 Ping depuis Proxmox vers les conteneurs
### 12.2 Accès aux services depuis la machine hôte
### 12.3 Tableau récapitulatif des redirections de ports

| Service | Port externe (hôte) | Port interne (conteneur) | URL d'accès |
|---------|---------------------|--------------------------|-------------|
| AdGuard Home | À compléter | À compléter | À compléter |
| Passbolt | À compléter | À compléter | À compléter |
| FireflyIII | À compléter | À compléter | À compléter |

---

## 13. Annexes

### 13.1 Commandes utiles
### 13.2 Dépannage
### 13.3 Références

- Documentation officielle Proxmox VE : https://pve.proxmox.com/wiki/Main_Page
- Documentation OPNSense : https://docs.opnsense.org/
- AdGuard Home : https://github.com/AdguardTeam/AdGuardHome
- Passbolt CE : https://www.passbolt.com/
- FireflyIII : https://www.firefly-iii.org/
