# Partage_De_Fichiers

# Procédure - Serveur de Fichiers Windows Server 2022

**Domaine :** `LabDT.corp`  
**Serveur :** `SRV1` - IP `192.168.1.20`  
**Client :** `CLIENT1` - Windows 11  


---

## Prérequis

- Windows Server 2022 avec le rôle AD DS déployé
- Poste client Windows 11 joint au domaine `LabDT.corp`
- DNS du client pointant vers le DC (`192.168.1.20`)

---

## Étape 1 - Installation du rôle Serveur de fichiers

```
Install-WindowsFeature -Name FS-FileServer -IncludeManagementTools
```

**Vérification :**
```
Get-WindowsFeature -Name FS-FileServer
```

---

## Étape 2 - Création du dossier racine

```
New-Item -Path "C:\Shares\Documents_Entreprise" -ItemType Directory
```

---

## Étape 3 - Création du partage réseau "Docs"

```
New-SmbShare -Name "Docs" -Path "C:\Shares\Documents_Entreprise"
```

**Permission de partage - Tout le monde en Contrôle total (la sécurité est gérée par NTFS) :**
```
Grant-SmbShareAccess -Name "Docs" -AccountName "Tout le monde" -AccessRight Full -Force
```

---

## Étape 4 - Création des sous-dossiers

```
New-Item -Path "C:\Shares\Documents_Entreprise\RH" -ItemType Directory
New-Item -Path "C:\Shares\Documents_Entreprise\Comptabilité" -ItemType Directory
New-Item -Path "C:\Shares\Documents_Entreprise\Direction" -ItemType Directory
```

---

## Étape 5 - Création des groupes AD et des utilisateurs de test

### Groupes de sécurité

```
New-ADGroup -Name "RH" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "Comptabilité" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "Direction" -GroupScope Global -GroupCategory Security
```

### Utilisateurs de test

```
New-ADUser -Name "user-rh" -AccountPassword (ConvertTo-SecureString "Azerty123!" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "user-compta" -AccountPassword (ConvertTo-SecureString "Azerty123!" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "user-direction" -AccountPassword (ConvertTo-SecureString "Azerty123!" -AsPlainText -Force) -Enabled $true
```

### Ajout des utilisateurs dans les groupes

```
Add-ADGroupMember -Identity "RH" -Members "user-rh"
Add-ADGroupMember -Identity "Comptabilité" -Members "user-compta"
Add-ADGroupMember -Identity "Direction" -Members "user-direction"
```

---

## Étape 6 - Configuration des permissions NTFS

### Matrice des permissions

| Dossier | Groupe | Permission NTFS |
|---|---|---|
| `Documents_Entreprise` | Utilisateurs du domaine | Lecture et exécution |
| `RH` | RH | Modification |
| `RH` | Direction | Modification |
| `Comptabilité` | Comptabilité | Modification |
| `Comptabilité` | Direction | Modification |
| `Direction` | Direction | Modification |

### Points d'attention

>  **Règle NTFS :** Un `Deny` explicite écrase toujours un `Allow`.  
> Vérifier l'absence de règles `Deny Write` héritées sur les sous-dossiers via **Propriétés → Sécurité → Avancé**.

>  **Héritage :** Les sous-dossiers héritent des permissions du dossier parent. Vérifier que `Utilisateurs du domaine` n'a pas l'écriture sur les sous-dossiers.

### Vérification des ACL via PowerShell

```
Get-Acl "C:\Shares\Documents_Entreprise\Comptabilité" | Format-List
```

---

## Étape 7 - Lister les partages sur le serveur

```
Get-SmbShare
```

**Résultat attendu :**

| Name | Path | Description |
|---|---|---|
| Docs | C:\Shares\Documents_Entreprise | |
| Documents_Entreprise | C:\Shares\Documents_Entreprise | |
| ADMIN$ | C:\WINDOWS | Administration à distance |
| C$ | C:\ | Partage par défaut |
| NETLOGON | C:\WINDOWS\SYSVOL\... | Partage de serveur d'accès |
| SYSVOL | C:\WINDOWS\SYSVOL\... | Partage de serveur d'accès |

---

## Étape 8 - Mappage du lecteur réseau sur le poste client

### Configuration du DNS client (prérequis)

```powershell
Set-DnsClientServerAddress -InterfaceIndex 13 -ServerAddresses "192.168.1.20"
```

**Vérification :**
```
Get-DnsClientServerAddress -InterfaceIndex 13
```

### Mappage du lecteur réseau Z:

```
New-PSDrive -Name "Z" -PSProvider FileSystem -Root "\\SRV1\Docs" -Persist
```

Le lecteur `Z:` apparaît dans l'explorateur sous **Emplacements réseau** → `Docs (\\SRV1) (Z:)`.

---

## Étape 9 - Tests d'accès

### Procédure de test

Pour chaque compte, se connecter sur le poste client `CLIENT1` et tenter de créer un fichier dans chaque sous-dossier via `Z:\`.

### Résultats des tests

| Compte | Dossier RH | Dossier Comptabilité | Dossier Direction |
|---|---|---|---|
| `LABDT\user-rh` | Écriture autorisée |  Accès refusé |  Accès refusé |
| `LABDT\user-compta` | Accès refusé |  Écriture autorisée |  Accès refusé |
| `LABDT\user-direction` |  Écriture autorisée |  Écriture autorisée |  Écriture autorisée |

---

