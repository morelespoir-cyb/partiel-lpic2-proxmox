# Mémo — Modifications système temporaires pour le partiel LPIC2

**Date** : 7 mai 2026
**Raison** : Activer la nested virtualization dans VirtualBox pour pouvoir faire tourner OPNSense (VM KVM) à l'intérieur de Proxmox VE.
**Durée prévue** : jusqu'au rendu du partiel (16 mai 2026), réactivation possible ensuite.

---

## Ce qui a été désactivé

### 1. Memory Integrity (HVCI / Core Isolation)
- **Avant** : Activé (`SecurityServicesRunning : {2}`)
- **Après** : Désactivé
- **Comment vérifier** :
  ```powershell
  Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard | Select-Object SecurityServicesRunning
  ```
- **Impact sécurité** : perte d'une couche de défense contre l'injection de code malveillant en kernel mode. Windows Defender + BitLocker + UAC restent actifs.

### 2. VirtualMachinePlatform (feature Windows)
- **Avant** : Enabled
- **Après** : Disabled
- **Comment vérifier** :
  ```powershell
  Get-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform | Select-Object FeatureName, State
  ```
- **Impact** : WSL2 ne fonctionne plus (passage automatique en WSL1, ou WSL totalement KO).

### 3. Hypervisor lancement automatique (au cas où)
- Commande exécutée : `bcdedit /set hypervisorlaunchtype off`
- **Pour vérifier** : `bcdedit /enum {current}` → chercher `hypervisorlaunchtype`

---

## Comment TOUT RÉACTIVER après le partiel

Lancer **PowerShell en administrateur**, puis :

```powershell
# 1. Réactiver VirtualMachinePlatform (pour WSL2)
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -All -NoRestart

# 2. Réactiver le lancement de l'hyperviseur Windows
bcdedit /set hypervisorlaunchtype auto

# 3. Pour Memory Integrity : Paramètres Windows → Sécurité Windows
#    → Sécurité de l'appareil → Isolation du noyau → Activer "Intégrité de la mémoire"

# 4. Redémarrer le PC
Restart-Computer
```

Après redémarrage, vérifier WSL2 :
```powershell
wsl --status
wsl --version
```

---

## Alternative Linux pendant la période de désactivation

Une VM Ubuntu Server sera créée dans VirtualBox pour remplacer WSL2.
À configurer après l'installation de Proxmox.
