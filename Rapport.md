# Lab — Simulation d'Attaque Cybersécurité & Détection avec Wazuh

**Date :** 19 février 2026  
**Environnement :** VMware Workstation  
**Auteur :** Mamadou Aliou Barry

---

## Contexte et Objectifs

Ce lab a pour objectif de simuler une attaque réelle sur un environnement vulnérable afin de comprendre comment une entreprise peut être compromise, comment un SIEM comme Wazuh peut détecter ces intrusions, et comment remédier aux vulnérabilités identifiées.

**Objectifs pédagogiques :**
- Comprendre le cycle d'une attaque (Reconnaissance → Exploitation → Post-exploitation)
- Observer en temps réel la détection d'intrusions via Wazuh
- Appliquer des mesures correctives concrètes
- Sensibiliser aux bonnes pratiques de sécurité

---

## Architecture du Laboratoire

| Machine | IP | Rôle |
|---|---|---|
| Ubuntu Desktop | 10.0.2.10 | Wazuh Manager (Dashboard, Indexer, Manager) |
| Ubuntu Server | 10.0.2.128 | Cible vulnérable (Apache2, SSH, DVWA) |
| Kali Linux | 10.0.2.50 | Machine attaquante (Nmap, Hydra) |

**Réseau :** VMnet4 — Host-Only, sous-réseau `10.0.2.0/24`

---

## Installation et Configuration

### Wazuh Manager (Ubuntu Desktop)

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

Accès au dashboard : `https://10.0.2.10`

### Agent Wazuh (Ubuntu Server)

```bash
sudo apt install wazuh-agent=4.7.5-1 -y
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

### Configuration de la machine cible vulnérable

**Apache2 :**
```bash
sudo apt install apache2 -y
sudo systemctl status apache2
```

**Création d'un utilisateur avec mot de passe faible :**
```bash
sudo adduser aliou
# mot de passe défini : 123456
```

**Activation de l'authentification SSH par mot de passe :**
```bash
sudo nano /etc/ssh/sshd_config
# Modifier : PasswordAuthentication no → PasswordAuthentication yes
```

**Installation de DVWA :**
```bash
sudo apt install php php-mysqli php-gd libapache2-mod-php mysql-server git -y
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
sudo systemctl restart apache2
sudo mysql -u root
```

---

## Déroulement de l'Attaque

### Phase 1 — Reconnaissance (Nmap)

L'attaquant depuis Kali Linux effectue un scan de reconnaissance pour identifier les services exposés.

```bash
nmap -sV -A 10.0.2.128
```

**Résultats obtenus :**
- Port `22/TCP` — OpenSSH (Ubuntu) — **OUVERT**
- Port `80/TCP` — Apache httpd (Ubuntu) — **OUVERT**
- OS identifié : Linux
- Version d'Apache visible dans les headers HTTP

### Phase 2 — Brute Force SSH (Hydra)

Après avoir identifié le port SSH ouvert, l'attaquant lance une attaque par force brute.

```bash
hydra -l aliou -P /usr/share/wordlists/rockyou.txt ssh://10.0.2.128 -t 4
```

**Résultats :**
- Identifiants trouvés : `aliou` / `123456`
- Nombre de tentatives : ~3 586 100
- Durée : environ **2 minutes**

> Ce résultat démontre la dangerosité des mots de passe faibles combinés à l'authentification SSH par mot de passe.

### Phase 3 — Connexion SSH

Avec les identifiants récupérés, l'attaquant se connecte directement au serveur :

```bash
ssh aliou@10.0.2.128
# mot de passe : 123456
```

Preuve de succès : le prompt `aliou@srvubuntu` confirme l'accès complet au système.

---

## Détection par Wazuh

L'agent Wazuh déployé sur l'Ubuntu Server a détecté l'attaque en temps réel :

- **1763 alertes** générées sur la période
- **11 échecs d'authentification** remontés
- **Alertes critiques niveau 10** (brute force SSH) déclenchées dès le début de l'attaque Hydra
- Source identifiée : IP `10.0.2.50` (Kali Linux)
- Après la connexion réussie : alerte d'**authentification réussie après tentatives multiples** — signal classique d'intrusion

**MITRE ATT&CK détectées :**
- Valid Accounts
- Password Guessing
- SSH
- Account Manipulation

---

## Remédiation

### Sécurisation SSH

```bash
# Désactiver l'auth par mot de passe
PasswordAuthentication no   # dans /etc/ssh/sshd_config

# Installer Fail2ban
sudo apt install fail2ban -y
```

- Utiliser uniquement l'authentification par clés SSH
- Changer le port SSH par défaut (22 → port non standard)
- Limiter les tentatives de connexion avec Fail2ban

### Politique de Mots de Passe

- Imposer des mots de passe forts (minimum 12 caractères, majuscules, chiffres, caractères spéciaux)
- Activer l'authentification multi-facteurs (MFA)

### Sécurisation Apache

```apache
ServerTokens Prod
ServerSignature Off
```

- Activer HTTPS avec un certificat TLS/SSL
- Supprimer ou protéger DVWA en production

### Surveillance Continue

- Maintenir l'agent Wazuh actif et à jour sur tous les serveurs
- Configurer des alertes email/Slack pour les événements critiques
- Effectuer des audits de sécurité réguliers

---

## Conclusion

Ce lab démontre de manière concrète comment un attaquant peut compromettre un serveur en exploitant des vulnérabilités simples mais courantes : mot de passe faible et SSH mal configuré.

Wazuh s'est révélé efficace pour détecter l'attaque en temps réel, générant des alertes critiques dès le début du brute force. Cela illustre l'importance d'un SIEM dans toute infrastructure de sécurité.

Les trois corrections prioritaires qui auraient rendu cette attaque impossible :
1. Désactiver l'auth SSH par mot de passe → utiliser des clés SSH
2. Politique de mots de passe forts + Fail2ban
3. Masquer la version d'Apache + activer HTTPS

---

## Outils Utilisés

| Outil | Usage |
|---|---|
| Nmap | Scan de reconnaissance réseau |
| Hydra | Brute force SSH |
| Wazuh 4.7 | SIEM — détection d'intrusion |
| DVWA | Application web vulnérable (cible) |
| VMware Workstation | Virtualisation du lab |
