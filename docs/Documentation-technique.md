# Documentation technique
## PoC d'infrastructure Proxmox + OPNSense + LXC

**Auteur** : Morel
**Formation** : ESGI Paris — M1 Architecture Systèmes-Réseaux & Cybersécurité
**Matière** : M1-T1 Linux, administration système et réseau avancée
**Année** : 2025-2026

---

## Table des matières

1. [Introduction](#1-introduction)
2. [Prérequis](#2-prérequis)
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

L'entreprise XYZ souhaite mettre en place un Proof of Concept d'infrastructure entièrement open-source et gratuite, afin de réduire ses dépenses logicielles. L'utilisation de Docker est interdite faute de compétences dans l'équipe.

### 1.2 Objectifs

- Mettre en place un hyperviseur Proxmox VE
- Sécuriser le sous-réseau via un pare-feu OPNSense
- Déployer trois services métier dans des conteneurs LXC Debian
- Garantir l'accès aux services depuis la machine hôte via redirections de ports

### 1.3 Solutions retenues

| Solution | Rôle | Justification |
|----------|------|---------------|
| **Proxmox VE** | Hyperviseur | Solution open-source mature, supporte VM (KVM) et conteneurs (LXC) nativement |
| **OPNSense** | Pare-feu / Routeur | Fork moderne de pfSense, interface web claire, fonctionnalités avancées |
| **AdGuard Home** | DNS filtrant | Bloque pubs et trackers au niveau réseau |
| **Passbolt** | Gestionnaire de mots de passe | Open-source, conçu pour les équipes |
| **FireflyIII** | Gestion de finances | Open-source, auto-hébergeable |

---

## 2. Prérequis

### 2.1 Matériel

| Ressource | Minimum | Utilisé dans ce PoC |
|-----------|---------|---------------------|
| RAM | 8 Go | À compléter |
| Disque dur | 80 Go | À compléter |
| CPU | 2 cores avec VT-x/AMD-V | À compléter |

### 2.2 Logiciels nécessaires sur la machine hôte

- VirtualBox (version à compléter)
- Image ISO Proxmox VE (version à compléter)

### 2.3 Activation de la virtualisation imbriquée

*(à remplir : justification + capture d'écran VirtualBox)*

---

## 3. Architecture cible

```
+-------------------+
|     Proxmox       |
| @IP 192.168.1.200 |
+--------+----------+
         |
   +-----+-----+
   |  OPNSense |
   +-----+-----+
         |
  +------+------+----------+
  |             |          |
+-+---------+ +-+-------+ +-+-------+
| ADGUARD   | | PASSBOLT| | FIREFLY |
|10.1.0.1   | |10.1.0.2 | |10.1.0.3 |
+-----------+ +---------+ +---------+
```

### 3.1 Plan d'adressage

| Élément | Réseau | IP | Rôle |
|---------|--------|----|----|
| Proxmox | LAN principal | 192.168.1.200 | Hyperviseur |
| OPNSense WAN | LAN principal | À compléter | Patte externe |
| OPNSense LAN | Sous-réseau | À compléter | Patte interne (passerelle) |
| CT-SOL-ADGUARD | Sous-réseau | 10.1.0.1 | DNS |
| CT-SOL-PASSBOLT | Sous-réseau | 10.1.0.2 | Coffre-fort |
| CT-SOL-FIREFLY | Sous-réseau | 10.1.0.3 | Finances |

---

## 4. Installation de Proxmox VE

*(à remplir au fur et à mesure : téléchargement ISO, création VM VirtualBox, partitionnement, installation pas à pas avec captures d'écran)*

### 4.1 Téléchargement de l'ISO
### 4.2 Création de la machine virtuelle dans VirtualBox
### 4.3 Choix du partitionnement (1 pt à l'évaluation)
### 4.4 Installation pas à pas

---

## 5. Configuration réseau de Proxmox

### 5.1 Configuration de l'interface principale (vmbr0)
### 5.2 Création du bridge interne (vmbr1) pour le sous-réseau
### 5.3 Vérification (`/etc/network/interfaces`)

---

## 6. Déploiement de la VM OPNSense

### 6.1 Téléchargement de l'image OPNSense
### 6.2 Création de la VM dans Proxmox
### 6.3 Installation d'OPNSense

---

## 7. Configuration d'OPNSense

### 7.1 Configuration des interfaces (WAN/LAN)
### 7.2 Règles de pare-feu
### 7.3 Configuration NAT et redirections de ports
### 7.4 Export de la configuration

---

## 8. Création des conteneurs LXC

### 8.1 Téléchargement du template Debian standard
### 8.2 Paramètres communs aux trois conteneurs
### 8.3 Création de CT-SOL-ADGUARD
### 8.4 Création de CT-SOL-PASSBOLT
### 8.5 Création de CT-SOL-FIREFLY

---

## 9. Installation et configuration d'AdGuard Home

### 9.1 Installation
### 9.2 Configuration de base
### 9.3 Vérification du service

---

## 10. Installation et configuration de Passbolt

### 10.1 Prérequis (MariaDB, Nginx, PHP, GPG)
### 10.2 Installation
### 10.3 Configuration HTTPS
### 10.4 Création du compte administrateur

---

## 11. Installation et configuration de FireflyIII

### 11.1 Prérequis (PHP, base de données, serveur web)
### 11.2 Installation
### 11.3 Configuration initiale

---

## 12. Tests de connectivité et redirections de ports

### 12.1 Ping depuis Proxmox vers les conteneurs
### 12.2 Accès aux services depuis la machine hôte
### 12.3 Tableau récapitulatif des redirections de ports

| Service | Port externe (hôte) | Port interne (conteneur) | URL d'accès |
|---------|---------------------|---------------------------|-------------|
| AdGuard Home | À compléter | À compléter | À compléter |
| Passbolt | À compléter | À compléter | À compléter |
| FireflyIII | À compléter | À compléter | À compléter |

---

## 13. Annexes

### 13.1 Commandes utiles
### 13.2 Dépannage
### 13.3 Références
