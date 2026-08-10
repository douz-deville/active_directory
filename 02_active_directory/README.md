# Contexte : L'entreprise "TechCorp"

L'entreprise TechCorp compte deux départements : Informatique (IT) et Ressources Humaines (RH) avec des accès à un dossier partagé et des politiques de sécurité.

## 01 Création de la structure d'OU

Une OU principale : TechCorp.

À l'intérieur de TechCorp, 3 sous-OU : Utilisateurs, Groupes, et Ordinateurs.

Dans Utilisateurs, deux sous-OU : IT et RH

```powershell
New-ADOrganizationalUnit -Name 'TechCorp' -Path 'DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'Utilisateurs' -Path 'OU=TechCorp,DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'Groupes' -Path 'OU=TechCorp,DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'Ordinateurs' -Path 'OU=TechCorp,DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'IT' -Path 'OU=Utilisateurs,OU=TechCorp,DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'RH' -Path 'OU=Utilisateurs,OU=TechCorp,DC=xyz,DC=com'
```

## 02 Création des utilisateurs

Dans Utilisateurs > IT : Marc Tech (m.tech).

Dans Utilisateurs > RH : Julie Rh (j.rh).

```powershell
$Password = ConvertTo-SecureString 'Passw0rd' -AsPlainText -Force

New-ADUser -Name 'Marc Tech' -GivenName 'Marc' -Surname 'Tech' -SamAccountName 'm.tech' -UserPrincipalName 'm.tech@xyz.com' -Path 'OU=IT,OU=Utilisateurs,OU=TechCorp,DC=xyz,DC=com' -AccountPassword $Password -Enabled $true -ChangePasswordAtLogon $false

New-ADUser -Name 'Julie Rh' -GivenName 'Julie' -Surname 'Rh' -SamAccountName 'j.rh' -UserPrincipalName 'j.rh@xyz.com' -Path 'OU=RH,OU=Utilisateurs,OU=TechCorp,DC=xyz,DC=com' -AccountPassword $Password -Enabled $true -ChangePasswordAtLogon $false
```

#### NB: PowerShell exige que le mot de passe d'un utilisateur soit transmis sous forme de chaîne sécurisée (SecureString)

## 03 Création des groupes

Dans Groupes :

Un Groupe Global nommé GG_RH (membres : j.rh).

Un Groupe Global nommé GG_IT (membres : m.tech).

Un Groupe Local de Domaine nommé DL_Partage_RH_RW (Accès Lecture/Écriture au partage RH).

GG_RH comme membre de DL_Partage_RH_RW.

1. Creation des Groupes

```powershell
New-ADGroup -Name "GG_RH" -SamAccountName "GG_RH" -GroupCategory Security -GroupScope Global -DisplayName "GG_RH" -Path "OU=Groupes,OU=TechCorp,DC=xyz,DC=com" -Description "Membres de l'équipe RH"

New-ADGroup -Name "GG_IT" -SamAccountName "GG_IT" -GroupCategory Security -GroupScope Global -DisplayName "GG_IT" -Path "OU=Groupes,OU=TechCorp,DC=xyz,DC=com" -Description "Membres de l'équipe IT"

New-ADGroup -Name "DL_Partage_RH_RW" -SamAccountName "DL_Partage_RH_RW" -GroupCategory Security -GroupScope DomainLocal -DisplayName "DL_Partage_RH_RW" -Path "OU=Groupes,OU=TechCorp,DC=xyz,DC=com" -Description "Accès Lecture et Écriture au partage RH"
```

2. Ajout des Membres

```powershell
Add-ADGroupMember -Identity "GG_RH" -Members "j.rh"

Add-ADGroupMember -Identity "GG_IT" -Members "m.tech"

Add-ADGroupMember -Identity "DL_Partage_RH_RW" -Members "GG_RH"
```

