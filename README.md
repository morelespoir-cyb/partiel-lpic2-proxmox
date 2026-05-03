# Partiel LPIC2 — PoC d'infrastructure Proxmox

Projet partiel de la matière **M1-T1 Linux, administration système et réseau avancée** (ESGI Paris, SI 4 RJ A, 2025-2026).

## Contexte

Mise en place d'une infrastructure entièrement open-source dans le but de réduire les dépenses d'une entreprise fictive. L'utilisation de Docker est interdite : on s'appuie sur des conteneurs LXC natifs Proxmox.

## Architecture

```
+-------------------+
|      Proxmox      |
|  192.168.1.200    |
+--------+----------+
         |
   +-----+-----+
   |  OPNSense |  (point d'entrée du sous-réseau)
   +-----+-----+
         |
  +------+------+------+
  |      |      |      |
+-+---+ +-+--+ +-+----+
|ADGUARD| |PASSBOLT| |FIREFLY|
|10.1.0.1| |10.1.0.2| |10.1.0.3|
+-------+ +--------+ +--------+
```

## Solutions déployées

| Conteneur | Solution | Rôle | IP |
|-----------|----------|------|-----|
| CT-SOL-ADGUARD | AdGuard Home | DNS filtrant réseau | 10.1.0.1 |
| CT-SOL-PASSBOLT | Passbolt | Gestionnaire de mots de passe en équipe | 10.1.0.2 |
| CT-SOL-FIREFLY | FireflyIII | Gestion de finances personnelles | 10.1.0.3 |

## Stack technique

- **Hyperviseur** : Proxmox VE 8.x
- **Pare-feu** : OPNSense (VM)
- **Conteneurs** : LXC sur templates Debian 12 standard
- **Hôte de virtualisation** : VirtualBox (nested virtualization)

## Structure du dépôt

```
partiel-lpic2-proxmox/
├── Proxmox/        # fichiers de configuration de l'hyperviseur
├── OPNSense/       # configuration du pare-feu
├── Adguard/        # configuration du conteneur AdGuard Home
├── Passbolt/       # configuration du conteneur Passbolt
├── Firefly/        # configuration du conteneur FireflyIII
├── docs/           # documentation technique (source markdown)
└── screenshots/    # captures d'écran utilisées dans la documentation
```

## Documentation technique

La documentation technique complète permettant de remonter l'infrastructure de zéro est disponible dans `docs/Documentation-technique.md` (export PDF dans le rendu final).

## Auteur

**Morel** — Étudiant M1 Architecture Systèmes-Réseaux & Cybersécurité, ESGI Paris.
