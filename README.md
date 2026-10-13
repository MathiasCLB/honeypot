# Honeypot Cowrie : de la VM locale au déploiement sur Azure

**À savoir :** Ce projet est lié suivant (Mini SIEM) grâce à l'extraction et à l'analyse des logs provenant du honeypot.

**Pourquoi ?**
J'ai choisi ce projet pour pouvoir apprendre plusieurs compétences simultanément. De l'installation d'un machine virtuelle locale à une utilisation plus poussée sur Azure et jusqu'à l'exploitation et l'analyse des logs grâce au projet suivant, vous trouverez dans ce document comment j'ai procédé pour ce projet. 

Ce dépôt documente la mise en place d'un honeypot **Cowrie** (SSH/Telnet à basse interaction), d'abord dans une machine virtuelle Kali Linux locale pour apprendre, puis sur une machine virtuelle Azure exposée à Internet.


| | |
|---|---|
| **Outil** | [Cowrie](https://github.com/cowrie/cowrie), honeypot SSH/Telnet à basse interaction |
| **Environnement local** | VirtualBox sur Windows 11, VM Kali Linux |
| **Environnement cloud** | Azure for Students, VM Ubuntu Server 24.04 LTS |
| **Langages / outils** | Linux, Python, SSH, PowerShell |

 
## Sommaire
 
1. [Avertissements](#1-avertissements)
2. [Architecture](#2-architecture)
3. [Partie 1 : Cowrie sur une machine virtuelle Kali Linux locale](#3-partie-1--cowrie-sur-une-vm-kali-linux-locale)
4. [Partie 2 : déploiement sur Azure](#4-partie-2--déploiement-sur-azure)
5. [Coûts](#5-coûts)
6. [Ce que j'ai appris](#6-ce-que-jai-appris)
7. [Références](#7-références)

---

## 1. Avertissements

- Projet à but **éducatif**. Un honeypot attire volontairement des attaquants : ne l'installe jamais sur une machine qui contient des données personnelles, et jamais sur ton réseau domestique sans isolation.
- Cowrie est un honeypot à **basse interaction** : il simule un shell, l'attaquant n'obtient pas un vrai système. Le risque est réduit, mais pas nul.
- Le vrai service SSH d'administration doit toujours être déplacé sur un autre port et restreint à ton adresse IP avant d'exposer Cowrie sur le port 22.
- Ne publie jamais dans ce dépôt : adresses IP de tes machines, clés SSH (`*.pem`, clés privées), identifiants, noms d'utilisateur personnels.

## 2. Architecture

## 3. Partie 1 : Cowrie sur une machine virtuelle Kali Linux locale

### 3.1 Prérequis

- Windows 11 avec [VirtualBox](https://www.virtualbox.org/) installé
- Image de Kali Linux (voir 3.2)
- Au moins 2 Go de RAM et 25 Go de disque disponibles pour la VM

### 3.2 Créer la machine virtuelle Kali Linux

1. Aller sur [kali.org/get-kali](https://www.kali.org/get-kali/) puis sur **Installer**. Enfin, télécharger **Installer**.
2. Dans VirtualBox, cliquer sur **Nouvelle** :
  - Nom : `honeypot-kali-linux`, OS Ubuntu LTS
  - Mémoire: 2 Go minimum, même si 1 Go suffit (comme la taille de la machine virtuelle utilisée sur Azure)
  - Processeurs : 1
  - Disque : 25 Go
    
3. Important : dans l'assistant, cocher « Skip Unattended Installation » pour que l'installateur vous demande le nom d'utilisateur et le mot de passe.

4. Pendant l'installation, choisir le clavier français, créer un utilisateur non administrateur (par exemple honeypot) et noter sur un papier l'utilisateur ainsi que le mot de passe.

### 3.3 Réseau : NAT et redirection de ports

La machine virtuelle reste en mode **NAT** (isolée du réseau local). Pour atteindre Cowrie depuis Windows, nous allons ajouter des règles dans **Paramètres de la machine virtuelle > Réseau > Adaptateur 1 > Avancé > Redirection de ports** :

| Nom | Protocole | Port hôte | Port invité |
|---|---|---|---|
| Cowrie | TCP | 2222 | 2222 |
| SSH-admin (optionnel) | TCP | 2200 | 22 |

### 3.4 Préparer le système

Une fois le système installé :

Mettre le clavier AZERTY :
```bash
sudo dpkg-reconfigure keyboard-configuration
```

Mise à jour et dépendances :
 
```bash
sudo apt update
```
 
```bash
sudo apt install -y git python3-venv python3-pip libssl-dev libffi-dev build-essential libpython3-dev
```

### 3.5 Installer Cowrie

Cowrie ne doit **jamais** tourner en `root`. On l'installe donc sur un utilisateur, dans un environnement virtuel Python : 

Ouvrir un terminal (clic droit sur le bureau -> ouvrir un terminal).
Importer Cowrie depuis GitHub :
```bash
git clone https://github.com/cowrie/cowrie.git
```
Aller dans le dossier cowrie :
```bash
cd cowrie/
```
Créer l'environnement virtuel :
```bash
python3 -m venv cowrie-venv
```
Activer l'environnement virtuel :
```bash
source cowrie-venv/bin/activate
```

```bash
pip install --upgrade pip
```

```bash
pip3 install -r requirements.txt
```
Installer Cowrie :
```bash
pip install cowrie
```

```bash
tree bin
```

Trouver Cowrie :
```bash
ls /cowrie-venv/bin/
```

```bash
cp /home/kali/Desktop/cowrie/cowrie-venv/bin/cowrie /home/kali/desktop/cowrie/bin/
```

```bash
cp /home/kali/Desktop/cowrie/cowrie-venv/bin/playlog /home/kali/desktop/cowrie/bin
```

```bash
tree bin
```
Nous devons créer un fichier de configuration :
```bash
cp etc/cowrie.cfg.dist etc/cowrie.cfg
```

```bash
export PYTHONPATH=*(pwd)/src
```
Installer le projet en mode éditable (ne pas oublier le point à la fin) :
```bash
pip install -e .
```

##3.6 Démarrer et tester

Depuis le dossier `cowrie`, avec l'environnement virtuel activé :
Démarrer cowrie :
```bash
bin/cowrie start
```
Vérifier que cowrie fonctionne bien :
```bash
bin/cowrie status
```
Ouvrir un nouveau terminal :
Rediriger le port 22 vers le port 2222 :
```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
```

Revenir sur le premier terminal et vérifier les logs et la connexion avec SSH : 
```bash
tail -f var/log/cowrie/cowrie.log
```
Vous devriez voir **Ready to accept SSH Connections** sur la dernière ligne.
Toutes les commandes que les attaquants utilisent seront affichées ici.

Vous pouvez tester la connexion en utilisant une deuxième machine virtuelle et en utilisant la commande suivant :

```bash
sudo ssh root@ip-machine-honeypot -p 22
```

ou depuis PowerShell sur Windows :
```powershell
Test-NetConnection -ComputerName localhost -Port 2222
```
 
```powershell
ssh -p 2222 root@ip-machine-honeypot
```
Accepter avec `yes` et entrer n'importe quel mot de passe. Nous arrivons dans un faux shell. 

**METTRE CAPTURE ECRAN FAUX SHELL**

### 3.7 Test d'attaque avec Hydra
Hydra étant déjà installé sur Kali Linux, directement dans la machine virtuelle : 
```bash
echo -e "root\nadmin\ntest" > users.txt
```
 
```bash
echo -e "123456\npassword\nadmin\nroot" > passwords.txt
```
 
```bash
hydra -L users.txt -P passwords.txt ssh://127.0.0.1:2222
```

Toutes les commandes entrées apparaissent sur le premier terminal. 

---

## 4. Partie 2 : déploiement sur Azure

### 4.1 Compte Azure for Students

**Pourquoi Azure for Students ?** Grâce à l'offre étudiante d'Azure, nous pouvons bénéficier d'un crédit de 100$ et d'une grande variété de service gratuit.
1. Crée le compte sur [azure.microsoft.com/free/students](https://azure.microsoft.com/free/students) (et pas sur la page d'essai générale) avec ton adresse e-mail d'étudiant : aucune carte bancaire n'est demandée.


**EN CONSTRUCTION**


**CREATING**
