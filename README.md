# Active-Directory-Build-Break-Secure

## Description
Ce projet illustre une approche "Purple Team" complète d'un environnement Active Directory. L'objectif était de concevoir un domaine d'entreprise fonctionnel, d'y introduire des vulnérabilités de configuration courantes, de les exploiter en condition réelle, puis d'appliquer les remédiations de sécurité adaptées (Hardening).

**Compétences mises en œuvre :** Administration Active Directory, Pentest Interne, Analyse de chemins d'attaque, Remédiation via GPO.

---

## 1. Architecture (Build)
L'environnement a été virtualisé et isolé dans un réseau local (NAT Network).

* **Contrôleur de Domaine (DC01) :** Windows Server 2022 (IP: `192.168.x.x`) - Rôles AD DS, DNS.
* **Client 1 (CL01) :** Windows 10 Pro (IP: `192.168.x.x`)
* **Client 2 (CL02) :** Windows 10 Pro (IP: `192.168.x.x`)
* **Machine Attaquante :** Kali Linux (IP: `192.168.x.x`)

**Vulnérabilités intentionnelles configurées :**
* Protocoles de diffusion LLMNR/NBT-NS actifs.
* Compte de service avec Service Principal Name (SPN) et mot de passe faible.
* Mots de passe administrateurs locaux identiques sur les postes clients.

---

## 2. Scénario d'Attaque (Break)

### A. Empoisonnement LLMNR avec Responder
* **Outil :** `Responder`
* **Action :** Écoute du réseau et interception d'une requête NetBIOS erronée d'un client. 
* **Résultat :** Capture d'un hash de type NetNTLMv2 et récupération du mot de passe en clair via `Hashcat`.

### B. Énumération du domaine avec BloodHound
* **Outils :** `Bloodhound.py`, `Neo4j`
* **Action :** Collecte des données de l'AD (utilisateurs, groupes, GPO, ACL) avec un compte utilisateur standard fraîchement compromis.
* **Résultat :** Identification du compte de service vulnérable et des chemins d'élévation de privilèges.

### C. Kerberoasting
* **Outils :** `Impacket (GetUserSPNs.py)`, `Hashcat`
* **Action :** Requête de tickets TGS pour le compte de service identifié, extraction du hash et attaque par dictionnaire hors-ligne.
* **Résultat :** Compromission du compte de service.

### D. Élévation de privilèges et Post-Exploitation
* **Outils :** `CrackMapExec` / `NetExec`, `Mimikatz`
* **Action :** Utilisation des credentials compromis pour du mouvement latéral (Pass-The-Hash) et dump de la mémoire LSASS.
* **Résultat :** Obtention des droits d'Administrateur du Domaine (Domain Admin).

---

## 3. Remédiation et Hardening (Secure)

Pour sécuriser l'infrastructure contre les attaques démontrées ci-dessus, les actions suivantes ont été implémentées via **GPO** :

| Vulnérabilité Exploitée | Solution de Remédiation (Hardening) |
| :--- | :--- |
| **Empoisonnement LLMNR/NBT-NS** | Création d'une GPO désactivant LLMNR (Configuration ordinateur > Modèles d'administration > Réseau > Client DNS) et NBT-NS via script PowerShell. |
| **Kerberoasting** | Application d'une politique de mot de passe strict (GPO) pour les comptes de service (min. 30 caractères aléatoires). Mise en place des gMSA (Group Managed Service Accounts) si possible. |
| **Mouvement Latéral / PTH** | Déploiement de **Microsoft LAPS** pour randomiser et faire tourner les mots de passe de l'administrateur local sur `CL01` et `CL02`. |
| **Dump LSASS (Mimikatz)** | Activation de **Credential Guard** et de la protection LSA via GPO ou modification du registre (`RunAsPPL`). |
