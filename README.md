# 🔍 Lab Détection d'Intrusion — Kali Linux + Snort IDS + Ubuntu Server

<div align="center">

**Simulation d'attaque SSH · Détection IDS temps réel · Analyse nmap**

![Kali Linux](https://img.shields.io/badge/Kali_Linux-Attaquant-268BEE?style=for-the-badge&logo=kali-linux&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_Server-Victime%20%2F%20IDS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Snort](https://img.shields.io/badge/Snort-IDS%20%2F%20IPS-CC0000?style=for-the-badge)
![nmap](https://img.shields.io/badge/nmap-Reconnaissance-4A90D9?style=for-the-badge)
![UFW](https://img.shields.io/badge/UFW-Firewall-E74C3C?style=for-the-badge)
![UTM](https://img.shields.io/badge/UTM-Virtualisation-9B59B6?style=for-the-badge)

</div>

---

## 🎯 Résumé du projet

Ce home lab simule un scénario d'**intrusion réseau réaliste** entre deux machines virtualisées sous **UTM** (hyperviseur Apple Silicon / x64). Il met en œuvre une chaîne complète attaque → détection :

1. 🔴 **Kali Linux** mène une reconnaissance réseau (nmap) puis une attaque SSH ciblée
2. 🔵 **Ubuntu Server** héberge **Snort**, un IDS (Intrusion Detection System) open source qui analyse le trafic en temps réel et génère des alertes

> 💼 **Compétences démontrées** : reconnaissance réseau (nmap), attaque par force brute SSH, déploiement et configuration d'un IDS (Snort), écriture de règles de détection personnalisées, analyse d'alertes de sécurité, blocage réseau par pare-feu (UFW), méthodologie Red Team / Blue Team.

---

## 🧱 Stack technique

| Rôle | Machine | OS | Outils |
|---|---|---|---|
| 🔴 Attaquant | VM Kali Linux | Kali Rolling | nmap · Hydra |
| 🔵 Victime + IDS | VM Ubuntu Server | Ubuntu 22.04 LTS | Snort · OpenSSH · UFW · Fail2ban |
| 🖥️ Hyperviseur | Hôte | UTM (QEMU) | Réseau interne isolé |

---

## 🏗️ Architecture réseau

```
┌──────────────────────────────────────────────────────────┐
│                    UTM (Hôte)                            │
│                                                          │
│   ┌─────────────────────┐   ┌──────────────────────────┐ │
│   │   🔴 Kali Linux     │   │   🔵 Ubuntu Server        │ │
│   │   Attaquant         │   │   Victime + IDS          │ │
│   │                     │   │                          │ │
│   │  192.168.64.3       │   │  192.168.64.4            │ │
│   │                     │   │                          │ │
│   │  ┌───────────────┐  │   │  ┌────────────────────┐  │ │
│   │  │     nmap      │  │   │  │    Snort IDS       │  │ │
│   │  │     Hydra     │  │   │  │  (écoute eth0)     │  │ │
│   │  └───────────────┘  │   │  ├────────────────────┤  │ │
│   │                     │   │  │  OpenSSH (port 22) │  │ │
│   └─────────────────────┘   │  └────────────────────┘  │ │
│                             └──────────────────────────┘ │
│                                                          │
│         Réseau interne UTM : 192.168.128.20/24           │
└──────────────────────────────────────────────────────────┘

Flux d'attaque :
  Kali ──[nmap scan]──────────► Ubuntu (reconnaissance)
  Kali ──[SSH brute force]────► UFW (1ère ligne de défense)
                                    │ si non bloqué
                                    ▼
                              Snort détecte
                              & génère alertes
                                    │
                                    ▼
                          /var/log/snort/alert
                                    │
                                    ▼
                         Fail2ban bannit l'IP

```

---

## 🚀 Installation & configuration

### Étape 1 — Création des VMs sous UTM

**VM Kali Linux (Attaquant 🔴)**

```
UTM → Nouveau → Virtualiser
RAM       : 2 Go minimum
Stockage  : 30 Go
Réseau    : Mode "Réseau partagé"
```

**VM Ubuntu Server (Victime 🔵)**

```
UTM → Nouveau → Virtualiser
RAM       : 1 Go minimum
Stockage  : 20 Go
Réseau    : Mode "Réseau partagé"
```

> 💡 Sous UTM, le mode **"Réseau partagé"** (équivalent de VirtualBox Internal Network) isole totalement les VMs du réseau physique de l'hôte. Parfait pour un lab offensif.

---

### Étape 2 — Installation & configuration de Snort (Ubuntu)

#### 2.1 Installation

```bash
sudo apt update && sudo apt install -y snort
```

#### 2.2 Vérifier la configuration de base

```bash
# Fichier de configuration principal
sudo nano /etc/snort/snort.conf
```

#### 2.3 Écrire les règles de détection personnalisées

```bash
sudo nano /etc/snort/rules/local.rules
```

**Règle 1 — Détection brute force SSH (multiples tentatives de connexion)**

```
# Alerte si plus de 5 tentatives SSH en 60 secondes depuis la même source
alert tcp any any -> $HOME_NET 22 (msg:"ALERTE SSH Brute Force détecté"; \
  flags:S; threshold:type threshold, track by_src, count 5, seconds 60; \
  sid:1000001; rev:1;)
```

**Règle 2 — Détection connexion SSH suspecte**

```
# Alerte sur toute nouvelle connexion SSH entrante
alert tcp any any -> $HOME_NET 22 (msg:"ALERTE SSH Connexion entrante suspecte"; \
  flags:S; sid:1000002; rev:1;)
```

**Règle 3 — Détection scan de ports nmap**

```
# Alerte sur scan SYN nmap (nombreuses connexions SYN sans ACK)
alert tcp any any -> $HOME_NET any (msg:"ALERTE Scan de ports détecté"; \
  flags:S; threshold:type threshold, track by_src, count 20, seconds 5; \
  sid:1000003; rev:1;)
```

#### 2.4 Tester la validité des règles

```bash
sudo snort -T -c /etc/snort/snort.conf -i enp0s1
# Résultat attendu : "Snort successfully validated the configuration!"
```

#### 2.5 Lancer Snort en mode détection

```bash
# Mode alerte console (affichage immédiat)
sudo snort -q -A console -i enp0s1 -c /etc/snort/snort.conf
```

---

### Étape 3 — Installation OpenSSH sur Ubuntu (cible)

```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh

# Vérifier que SSH écoute sur le port 22
ss -tlnp | grep :22
```

---

## 🎬 Scénarios d'attaque & détection

---

### 🔍 Scénario 1 — Reconnaissance réseau avec nmap

**Depuis Kali — Scan des services et versions :**

```bash
# Scan services + scripts de détection par défaut
nmap -sV -sC  -p 22 192.168.64.4
```

**Résultat attendu :**

#capture nmap-scan-result.png


**Alertes Snort générées (terminal Ubuntu) :**

#capture snort-alert-console.png

> 🔵 **Analyse Blue Team** : Snort identifie immédiatement l'origine du scan (`192.168.64.3`) et le service ciblé (port 22). L'horodatage précis permet une corrélation temporelle avec d'autres logs système.

---

### ⚔️ Scénario 2 — Brute Force SSH & détection Snort

#### Phase Red Team — Attaque depuis Kali

**Générer une wordlist :**

```bash
# Wordlist
cat > wordlist.txt << EOF
admin
root
password
123456
ubuntu
toor
kali
letmein
EOF
```

**Lancer l'attaque avec Hydra :**

```bash
hydra -l ubuntu -P wordlist.txt ssh://192.168.64.4 -t 4 -V -f
```

---

#### Phase Blue Team — Détection par Snort

**Alertes générées en temps réel :**

#capture Hydra-bruteforce.png


---

#### Réponse Blue Team — Bannissement automatique avec Fail2ban

```bash
sudo apt install fail2ban -y
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled  = true
port     = ssh
maxretry = 3
bantime  = 3600
findtime = 600
logpath  = /var/log/auth.log
```

```bash
sudo systemctl enable fail2ban && sudo systemctl start fail2ban

# Vérifier le bannissement de l'IP attaquante
sudo fail2ban-client status sshd
```

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     3
|  `- Journal matches:   _SYSTEM_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   192.168.64.3   ✅
```

---

#### Réponse Blue Team — Blocage SSH par pare-feu (UFW)

UFW (Uncomplicated Firewall) constitue la **première ligne de défense** au niveau réseau : il bloque les connexions en amont, avant même que Snort ou Fail2ban n'entrent en jeu.

**Activation et configuration de base :**

```bash
# Activer UFW
sudo ufw enable

# Vérifier le statut
sudo ufw status verbose
```

**Règles de protection SSH :**

```bash

# Limiter SSH globalement
sudo ufw limit ssh

# Bloquer explicitement l'IP attaquante après détection Snort
sudo ufw deny from 192.168.64.3 to any port 22

# Refuser tout autre trafic entrant par défaut
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

**Vérification des règles actives :**

```bash
sudo ufw status numbered
```

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     LIMIT IN    Anywhere
[ 2] 22/tcp                     DENY IN     192.168.64.3     ← IP Kali bloquée ✅
[ 3] 22/tcp (v6)                 LIMIT IN    Anywhere (v6)
```

**Tester le blocage depuis Kali :**

```bash
# Depuis Kali — tentative SSH après blocage UFW
ssh targetuser@192.168.64.4
# → ssh: connect to host 192.168.64.4 port 22: Connection refused  ✅
```

> 🔵 **Analyse Blue Team** : la combinaison **UFW + Snort + Fail2ban** forme une défense en profondeur (*defense in depth*) : UFW bloque au niveau réseau, Snort détecte et alerte, Fail2ban bannit automatiquement après analyse des logs. Chaque couche est indépendante — si l'une est contournée, les autres restent actives.

---

## 📊 Résultats & observations

| Étape | Action | Résultat |
|---|---|---|
| Reconnaissance | `nmap -sV -sC` sur la cible | Port 22 SSH détecté, version OpenSSH identifiée |
| Détection scan | Règle Snort sid:1000003 | Alerte générée en < 1 seconde |
| Brute Force | Hydra, 8 tentatives SSH | 8 alertes Snort sid:1000001 enregistrées |
| Réponse Fail2ban | Bannissement automatique | IP `192.168.64.3` bannie après 3 échecs |
| Réponse UFW | Blocage pare-feu niveau réseau | Connexions SSH refusées avant d'atteindre le service |

---

## 🔧 Commandes de référence

### Snort

```bash
# Tester la configuration
sudo snort -T -c /etc/snort/snort.conf -i enp0s1

# Lancer en mode console (alertes immédiates)
sudo snort -A console -q -c /etc/snort/snort.conf -i enp0s1

# Lancer en mode fichier
sudo snort -A fast -q -c /etc/snort/snort.conf -i enp0s1 -l /var/log/snort/

# Lire les alertes enregistrées
sudo cat /var/log/snort/alert

# Voir les alertes en temps réel
sudo tail -f /var/log/snort/alert
```

### nmap

```bash
# Scan services + scripts (utilisé dans ce lab)
nmap -sV -sC -p 22 192.168.64.4

# Scan SYN furtif
sudo nmap -sS 192.168.64.4

# Détection OS
sudo nmap -O 192.168.64.4
```

### Fail2ban

```bash
sudo fail2ban-client status sshd                          # Statut jail SSH
sudo fail2ban-client set sshd unbanip 192.168.64.3        # Débannir
sudo tail -f /var/log/fail2ban.log                        # Logs en temps réel
```

### UFW

```bash
sudo ufw enable                                           # Activer le pare-feu
sudo ufw status numbered                                  # Voir les règles actives
sudo ufw limit ssh                                        # Rate limiting SSH
sudo ufw deny from 192.168.64.3 to any port 22            # Bloquer une IP sur SSH
sudo ufw delete 2                                         # Supprimer la règle n°2
sudo ufw default deny incoming                            # Refuser tout trafic entrant
sudo ufw logging on                                       # Activer les logs UFW
sudo tail -f /var/log/ufw.log                             # Voir les logs en temps réel
```

---

## 🔒 Ce que ce lab enseigne

| Compétence | Côté |
|---|---|
| Cartographier un réseau et identifier des services exposés | 🔴 Red Team |
| Mener une attaque par force brute SSH | 🔴 Red Team |
| Déployer et configurer un IDS (Snort) | 🔵 Blue Team |
| Écrire des règles de détection personnalisées | 🔵 Blue Team |
| Bloquer des intrusions SSH avec UFW (pare-feu réseau) | 🔵 Blue Team |
| Mettre en place une réponse automatisée (Fail2ban) | 🔵 Blue Team |
| Défense en profondeur : UFW + Snort + Fail2ban | 🔵 Blue Team |

---

## 📚 Références

- [Documentation Snort](https://www.snort.org/documents)
- [Snort Rules Writing Guide](https://docs.snort.org/rules/)
- [nmap Reference Guide](https://nmap.org/book/man.html)
- [Hydra — GitHub](https://github.com/vanhauser-thc/thc-hydra)
- [Fail2ban Documentation](https://github.com/fail2ban/fail2ban/wiki)
- [UFW Documentation — Ubuntu](https://help.ubuntu.com/community/UFW)

---

<div align="center">

*Projet personnel · Home Lab sécurité offensive & défensive · 2026*

⚠️ *Réalisé dans un environnement isolé à des fins éducatives uniquement.*

</div>
