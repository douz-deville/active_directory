# Phase 2 : Service de Fichiers et Permissions Network / NTFS

Ce module couvre la création d'un dossier partagé centralisé pour le département RH et l'application fine des autorisations réseau (SMB) et système de fichiers (NTFS) selon la méthode AGDLP.

### Objectifs :
* Créer le dossier local `C:\Partages\Donnees_RH` et le partager sous le nom caché `RH$`.
* Configurer le partage SMB avec un accès `Tout le monde = Contrôle Total`.
* Désactiver l'héritage NTFS et restreindre les droits au groupe `DL_Partage_RH_RW`.
* **Résultat attendu :** Julie (`j.rh`) a un accès complet en modification, tandis que Marc (`m.tech`) reçoit un message *"Accès refusé"*.


### Règle d'or des partages Active Directory :
* **Permissions de Partage (SMB) :** Ouvertes au maximum (`Tout le monde = Contrôle Total`).
* **Permissions NTFS (Sécurité locale) :** Verrouillées précisément avec les groupes de sécurité locaux de domaine (`DL_*`).


## 01. Création du dossier et du partage SMB

```powershell
# 1. Création du dossier local sur le serveur (-Force crée les dossiers parents si inexistants)
New-Item -Path "C:\Partages\Donnees_RH" -ItemType Directory -Force

# 2. Création du partage réseau SMB (Partage caché RH$)
New-SmbShare -Name "RH$" `
             -Path "C:\Partages\Donnees_RH" `
             -FullAccess "Everyone" `
             -Description "Dossier confidentiel de l'équipe RH"
```

## 02. Configuration des permissions NTFS (Méthode AGDLP)

Par défaut, un nouveau dossier hérite des permissions du disque C:. Il faut donc rompre l'héritage, réinitialiser les accès système de base et accorder les droits exclusifs au groupe DL_Partage_RH_RW.

```powershell
# Définition du chemin
$Path = "C:\Partages\Donnees_RH"

# 1. Récupération de l'ACL actuelle
$Acl = Get-Acl -Path$Path

# 2. Désactivation de l'héritage ($true = Protéger l'ACL, $false = Supprimer les règles héritées actuelles)
$Acl.SetAccessRuleProtection($true,$false)

# 3. Réinitialisation des permissions système de base (Administrateurs & SYSTEM)
$AdminRule = New-Object System.Security.AccessControl.FileSystemAccessRule("BUILTIN\Administrators", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$SystemRule = New-Object System.Security.AccessControl.FileSystemAccessRule("NT AUTHORITY\SYSTEM", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")

$Acl.SetAccessRule($AdminRule)
$Acl.SetAccessRule($SystemRule)

# 4. Ajout du groupe Domaine Local avec droits de Modification (Read, Write, Delete)
$GroupIdentity = "xyz\DL_Partage_RH_RW" # Remplacez 'xyz' par le nom NetBIOS de votre domaine
$Rights = "Modify"                      # "Modify" inclut Lecture, Écriture et Suppression
$Inheritance = "ContainerInherit, ObjectInherit"
$Propagation = "None"
$Type = "Allow"

$RHAccessRule = New-Object System.Security.AccessControl.FileSystemAccessRule($GroupIdentity, $Rights,$Inheritance, $Propagation,$Type)
$Acl.AddAccessRule($RHAccessRule)

# 5. Application de la nouvelle ACL sur le dossier
Set-Acl -Path $Path -AclObject$Acl
```

## 03. Vérification des accès

Pour valider le fonctionnement depuis la VM client Windows 11 :

    1. Test avec Julie Rh (j.rh) :

        - Connexion sur la VM client avec le compte j.rh.

        - Ouvrir l'explorateur de fichiers et saisir : \\DC1\RH$ (remplacer DC1 par le nom de votre serveur).

        - Résultat : Le dossier s'ouvre. Julie peut créer, modifier et supprimer des fichiers.

    2. Test avec Marc Tech (m.tech) :

        - Connexion sur la VM client avec le compte m.tech.

        - Saisir : \\DC1\RH$.

        - Résultat : Un message "Accès refusé" (Windows Network Error) apparaît.