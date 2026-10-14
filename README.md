# FR  

# Honeypot Cowrie : de la machine virtuelle locale au déploiement sur Azure

**À savoir :** Ce projet est lié au suivant (Mini SIEM), notamment grâce à l'extraction et à l'analyse des logs provenant du honeypot.

**Pourquoi ?**  

J'ai choisi ce projet pour pouvoir apprendre plusieurs compétences simultanément. De l'installation d'une machine virtuelle locale à une utilisation plus poussée sur Azure et jusqu'à l'exploitation et l'analyse des logs grâce au projet suivant, vous trouverez dans ce document comment j'ai procédé pour ce projet. Si vous suivez mes étapes, adaptez-les à votre cas.  

Ce dépôt documente la mise en place d'un honeypot **Cowrie** (SSH/Telnet à basse interaction), d'abord dans une machine virtuelle Kali Linux locale pour apprendre, puis sur une machine virtuelle Azure exposée à Internet.


| | |
|---|---|
| **Outil** | [Cowrie](https://github.com/cowrie/cowrie), honeypot SSH/Telnet à basse interaction |
| **Environnement local** | VirtualBox, Kali Linux |
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

## 2. Partie 1 : Cowrie sur une machine virtuelle Kali Linux locale

### 2.1 Prérequis

- Windows 11 avec [VirtualBox](https://www.virtualbox.org/) installé
- Image de Kali Linux (voir 3.2)
- Au moins 2 Go de RAM et 25 Go de disque disponibles pour la machine virtuelle

### 2.2 Créer la machine virtuelle Kali Linux

1. Allez sur [kali.org/get-kali](https://www.kali.org/get-kali/) puis sur **Installer**. Enfin, téléchargez **Installer**.
2. Dans VirtualBox, cliquez sur **Nouvelle** :
  - Nom : `honeypot-kali-linux`, OS Ubuntu LTS
  - Mémoire: 2 Go minimum, même si 1 Go suffit (comme la taille de la machine virtuelle utilisée sur Azure)
  - Processeurs : 1
  - Disque : 25 Go
    
3. Important : dans l'assistant, cocher « Skip Unattended Installation » pour que l'installateur vous demande le nom d'utilisateur et le mot de passe.

4. Pendant l'installation, choisissez le clavier français, créez un utilisateur non administrateur (par exemple honeypot) et notez sur un papier l'utilisateur ainsi que le mot de passe.

### 2.3 Réseau : NAT et redirection de ports

La machine virtuelle reste en mode **NAT** (isolée du réseau local). Pour atteindre Cowrie depuis Windows, nous allons ajouter des règles dans **Paramètres de la machine virtuelle > Réseau > Adaptateur 1 > Avancé > Redirection de ports** :

| Nom | Protocole | Port hôte | Port invité |
|---|---|---|---|
| Cowrie | TCP | 2222 | 2222 |
| SSH-admin (optionnel) | TCP | 2200 | 22 |

### 2.4 Préparer le système

Une fois le système installé :

Mettre le clavier en AZERTY :
```bash
sudo dpkg-reconfigure keyboard-configuration
```

Mises à jour et dépendances :
 
```bash
sudo apt update
```
 
```bash
sudo apt install -y git python3-venv python3-pip libssl-dev libffi-dev build-essential libpython3-dev
```

### 2.5 Installer Cowrie

Cowrie ne doit **jamais** tourner en `root`. On l'installe donc sur un utilisateur, dans un environnement virtuel Python : 

Ouvrez un terminal (clic droit sur le bureau -> ouvrir un terminal).
Importez Cowrie depuis GitHub :
```bash
git clone https://github.com/cowrie/cowrie.git
```
Allez dans le dossier cowrie :
```bash
cd cowrie/
```
Créez l'environnement virtuel :
```bash
python3 -m venv cowrie-venv
```
Activez l'environnement virtuel :
```bash
source cowrie-venv/bin/activate
```

```bash
pip install --upgrade pip
```

```bash
pip3 install -r requirements.txt
```
Installez Cowrie :
```bash
pip install cowrie
```

```bash
tree bin
```

Trouvez Cowrie :
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
Installez le projet en mode éditable (ne pas oublier le point à la fin) :
```bash
pip install -e .
```

## 2.6 Démarrer et tester

Depuis le dossier `cowrie`, avec l'environnement virtuel activé :
Démarrez cowrie :
```bash
bin/cowrie start
```
Vérifiez que cowrie fonctionne bien :
```bash
bin/cowrie status
```
Ouvrez un nouveau terminal :
Redirigez le port 22 vers le port 2222 :
```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
```

Revenez sur le premier terminal et vérifiez les logs et la connexion avec SSH : 
```bash
tail -f var/log/cowrie/cowrie.log
```
Vous devriez voir **Ready to accept SSH Connections** sur la dernière ligne.
Toutes les commandes que les attaquants utilisent seront affichées ici.

Vous pouvez tester la connexion en utilisant une deuxième machine virtuelle et en utilisant la commande suivante :

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

<img width="2559" height="1044" alt="image" src="https://github.com/user-attachments/assets/ac82d55c-6c75-46a0-89fa-540427597262" />

### 2.7 Test d'attaque avec Hydra
Hydra étant déjà installé sur Kali Linux, tapez directement dans la machine virtuelle : 
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

## 3. Partie 2 : déploiement sur Azure

### 3.1 Compte Azure for Students

**Pourquoi Azure for Students ?** Grâce à l'offre étudiante d'Azure, nous pouvons bénéficier d'un crédit de 100$ et d'une grande variété de services gratuits.
1. Créer le compte sur [azure.microsoft.com/free/students](https://azure.microsoft.com/free/students) (et pas sur la page d'essai générale) avec une adresse e-mail d'étudiant : aucune carte bancaire n'est demandée.

2. Dans le portail, taper **Education** dans la barre de recherche puis aller dans **Vue d'ensemble** pour voir les détails de l'offre étudiant et pour vérifier que le crédit de 100$ (ou environ) nous a bien été attribué. Sinon, il faudra attendre quelques jours.
Sachant que la machine virtuelle prend sur notre crédit (voir 4.2), nous pouvons créer une alerte de budget pour pouvoir mieux gérer notre crédit et ne pas tout attribuer à **une seule** machine virtuelle.
Tout ceci est adapté à l'abonnement et l'établissement car les régions de déploiement sont limitées. Dans mon cas, avec l'abonnement Azure for Students et du fait que je suis scolarisé à l'ESAIP, ma machine virtuelle se situe en Espagne et certaines tailles de machines virtuelles ne sont disponibles que dans certaines régions.

### 3.2 Créer la machine virtuelle

Portail Azure > **Machines virtuelles** > **Créer** > **Machine virtuelle 

| Champ | Valeur |
|---|---|
| Groupe de ressources | `CowrieHoneypot_group` (nouveau) |
| Nom | `honeypot-cowrie` |
| Région | une région **autorisée par la politique de l'abonnement** (dans le cas de l'abonnement étudiant choisir **Spain Central**(voir 3.9) |
| Image | Ubuntu Server 24.04 LTS, x64 Gen2 |
| Taille | `Standard_B2ts_v2` (environ 0,012 $/h) |
| Authentification | Clé publique SSH, nouvelle paire de clés |
| Nom d'utilisateur | `azureuser` |
| Ports entrants | SSH (22) |
| Disque | SSD standard, cocher « Supprimer avec la machine virtuelle » |
| Adresse IP publique | Nouvelle, **statique**, supprimée avec la VM |

Vous pouvez ensuite passer toutes les autres options.  

Au moment de créer, cliquer sur **Télécharger la clé privée et créer la ressource**. Sans ce fichier `.pem`, vous ne pourrez **jamais** vous connecter.

### 3.3 Se connecter en SSH
Copiez la clé dans `.ssh` (PowerShell)  
```powershell
Copy-Item "$HOME\Downloads\<NOM_DE_LA_CLE>.pem" "$HOME\.ssh\<NOM_DE_LA_CLE>.pem"
```
Vous pouvez également le faire depuis l'explorateur de fichiers.

Pour vous connecter : 
```powershell
ssh -i $HOME\.ssh\<NOM_DE_LA_CLE>.pem azureuser@<IP_PUBLIQUE>
```
Remplacez par l'IP publique de votre machine virtuelle sur Azure ainsi que le chemin où se trouve votre clé.  

### 3.4 Déplacer le vrai SSH sur le port 22222

Par défaut, Cowrie prend le port 22 car c'est celui que les bots scannent en priorité. Nous devons donc déplacer le vrai SSH d'administration sur le port 22222.  
Il faut cependant suivre un **ordre** : on ajoute le nouveau port, on teste, puis seulement on retire l'ancien.  


Sur Ubuntu 24.04, SSH est lancé par un socket systemd, et la ligne `Port` de `sshd_config` est ignorée. On vérifie donc :

```bash
systemctl is-active ssh.socket
```

Si la réponse est `active`, créez un fichier de surcharge qui écoute sur les deux ports, en IPv4 **et** en IPv6 :

```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
```
 
```bash
printf '[Socket]\nListenStream=\nListenStream=0.0.0.0:22\nListenStream=[::]:22\nListenStream=0.0.0.0:22222\nListenStream=[::]:22222\n' | sudo tee /etc/systemd/system/ssh.socket.d/override.conf
```
 
```bash
sudo systemctl daemon-reload
```
 
```bash
sudo systemctl restart ssh.socket
```
 
```bash
sudo ss -tlnp | grep -E ':22\b|:22222'
```

Lors de l'exécution de la dernière commande, si vous voyez `0.0.0.0:22` et `0.0.0.0:22222`, alors jusque là tout se passe parfaitement.  

On ouvre ensuite le port 22222 dans les paramètres réseau de la machine virtuelle directement sur le portail Azure (**Mise en réseau > Paramètres réseau > Ajouter une règle de port d'entrée**) :

| Champ | Valeur |
|---|---|
| Source | Adresses IP : **ton IP publique** (`Invoke-RestMethod https://api.ipify.org`) |
| Plage de ports de destination | `22222` |
| Protocole | TCP |
| Action | Autoriser |
| Priorité | un nombre libre (par exemple 1010) |
| Nom | `ssh-admin-22222` |

On ouvre un deuxième terminal PowerShell pour tester la connexion sur le nouveau port : 

```powershell
ssh -p 22222 -i $HOME\.ssh\<NOM_DE_LA_CLE>.pem azureuser@<IP_PUBLIQUE>
```

Toujours en remplaçant par la bonne adresse IP et le bon chemin de fichier.  

Une fois la connexion confirmée, on retire le port 22 du vrai SSH :
 
```bash
printf '[Socket]\nListenStream=\nListenStream=0.0.0.0:22222\nListenStream=[::]:22222\n' | sudo tee /etc/systemd/system/ssh.socket.d/override.conf
```
 
```bash
sudo systemctl daemon-reload
```
 
```bash
sudo systemctl restart ssh.socket
```

Seul le 22222 doit rester à l'écoute. Désormais, il faudra **toujours** se connecter avec `-p 22222`

### 3.5 Préparer la machine virtuelle

Cette machine virtuelle n'a que 1 Go de RAM : il va falloir créer un fichier d'échange pour éviter que pip ne soit tué par manque de mémoire pendant l'installation : 

```bash
sudo fallocate -l 2G /swapfile
```
 
```bash
sudo chmod 600 /swapfile
```
 
```bash
sudo mkswap /swapfile
```
 
```bash
sudo swapon /swapfile
```
 
```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
 
Une fois ceci fait, on installe les mises à jour et dépendances :
 
```bash
sudo apt-get update
```
 
```bash
sudo apt-get upgrade -y
```
 
```bash
sudo apt-get install -y python3-pip python3-venv libssl-dev libffi-dev build-essential libpython3-dev python3-minimal authbind
```
 
Puis on crée un utilisateur dédié car Cowrie **refuse** de démarrer en root :
 
```bash
sudo adduser --disabled-password --gecos "" cowrie
```
 
```bash
sudo su - cowrie
```

## 3.6 Installer Cowrie

La documentation actuelle de Cowrie installe le paquet avec `pip`, puis crée le dossier de configuration avec `cowrie init`. Ces commandes se lancent avec l'utilisateur `cowrie` :  

On crée tout d'abord un dossier `honeypot` dans lequel on va installer cowrie

```bash
mkdir ~/honeypot && cd ~/honeypot
```

 Puis on crée l'environnement virtuel python pour pouvoir exécuter cowrie :
 
```bash
python3 -m venv cowrie-env
```
 On active cet environnement virtuel :
 
```bash
source cowrie-env/bin/activate
```
 
```bash
python -m pip install --upgrade pip
```
Puis on installe cowrie :

```bash
python -m pip install cowrie
```

Et on crée le dossier de configuration :
 
```bash
cowrie init
```
On démarre cowrie :

```bash
cowrie start
```
 
Et on vérifie :
 
```bash
tail -n 20 var/log/cowrie/cowrie.log
```
 
```bash
ss -tlnp | grep 2222
```
 
On ouvre un autre terminal pour tester la connexion :
 
```bash
ssh -p 2222 root@127.0.0.1
```
 
### 3.7 Exposer Cowrie sur le port 22
 
Avec `authbind`, un utilisateur normal peut écouter sur un port inférieur à 1024. Depuis l'utilisateur azureuser :
 
```bash
sudo touch /etc/authbind/byport/22
```
 
```bash
sudo chown cowrie:cowrie /etc/authbind/byport/22
```
 
```bash
sudo chmod 770 /etc/authbind/byport/22
```
 
Avec l'utilisateur `cowrie`, on arrête Cowrie :

```bash
cowrie stop
```

Puis on édite `~/honeypot/etc/cowrie.cfg`. il faut chercher si une ligne `listen_endpoints` existe déjà sous la section `[ssh]` : si oui, changer sa valeur ; sinon, ajouter la section **une seule fois** (deux sections `[ssh]` provoquent une erreur) :

 
```ini
[ssh]
listen_endpoints = tcp:22:interface=0.0.0.0
```
 
```bash
AUTHBIND_ENABLED=yes cowrie start
```
 
```bash
ss -tlnp | grep ':22 '
```
 
Dans le groupe de sécurité réseau Azure, la règle du port 22 doit avoir comme source **Any** : c'est maintenant le seul port ouvert à tout Internet, et il n'est servi que par le honeypot.
 
Depuis Windows, le PC avait enregistré l'empreinte du vrai SSH pour cette IP. Supprimez-la, puis testez :
 
```powershell
ssh-keygen -R <IP_PUBLIQUE>
```
 
```powershell
ssh root@<IP_PUBLIQUE>
```
(Remplacer IP_PUBLIQUE par votre IP publique.)

### 3.8 Lancement automatique avec systemd
 
Sans service, un redémarrage de la VM arrête Cowrie sans prévenir. Arrêtez l'instance lancée à la main (`cowrie stop` avec l'utilisateur `cowrie`, puis `exit`), et créez le service depuis l'utilisateur azureuser. La commande `printf` écrase le fichier en entier, ce qui évite les doublons de contenu :
 
```bash
printf '[Unit]\nDescription=Cowrie SSH/Telnet honeypot\nAfter=network.target\n\n[Service]\nType=forking\nUser=cowrie\nGroup=cowrie\nWorkingDirectory=/home/cowrie/honeypot\nEnvironment=PATH=/home/cowrie/honeypot/cowrie-env/bin:/usr/local/bin:/usr/bin:/bin\nExecStart=/home/cowrie/honeypot/cowrie-env/bin/cowrie start\nExecStop=/home/cowrie/honeypot/cowrie-env/bin/cowrie stop\nPIDFile=/home/cowrie/honeypot/var/run/cowrie.pid\nAmbientCapabilities=CAP_NET_BIND_SERVICE\nRestart=on-failure\nRestartSec=5\n\n[Install]\nWantedBy=multi-user.target\n' | sudo tee /etc/systemd/system/cowrie.service
```
 
Le fichier obtenu :
 
```ini
[Unit]
Description=Cowrie SSH/Telnet honeypot
After=network.target
 
[Service]
Type=forking
User=cowrie
Group=cowrie
WorkingDirectory=/home/cowrie/honeypot
Environment=PATH=/home/cowrie/honeypot/cowrie-env/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=/home/cowrie/honeypot/cowrie-env/bin/cowrie start
ExecStop=/home/cowrie/honeypot/cowrie-env/bin/cowrie stop
PIDFile=/home/cowrie/honeypot/var/run/cowrie.pid
AmbientCapabilities=CAP_NET_BIND_SERVICE
Restart=on-failure
RestartSec=5
 
[Install]
WantedBy=multi-user.target
```
 
On vérifie le système, on active puis on démarre :
 
```bash
sudo systemd-analyze verify /etc/systemd/system/cowrie.service
```
 
```bash
sudo systemctl daemon-reload
```
 
```bash
sudo systemctl enable --now cowrie
```
 
```bash
systemctl status cowrie --no-pager
```
 
```bash
sudo ss -tlnp | grep ':22 '
```
 
Pour valider le démarrage automatique, redémarrez la VM (`sudo reboot`), reconnectez-vous sur le port 22222, et vérifiez que Cowrie est revenu seul.

### 3.9 Connexion au honeypot

Ouvrez un nouveau terminal PowerShell et exécutez :

```powershell
ssh root@IP_PUBLIQUE_AZURE
```

Remplacez IP_PUBLIQUE_AZURE par l'IP publique de votre machine virtuelle Azure.  
Vous arrivez sur un faux shell : 

<img width="2341" height="1278" alt="image" src="https://github.com/user-attachments/assets/ce745730-acd5-488e-90d8-94b846f70334" />


 ## 4. Coûts
 - Machine virtuelle Azure `Standard_B2ts_v2` : environ 0,012$/h, soit environ 9$ par mois en continu, payés grâce au crédit étudiant. Cependant, pour économiser des coûts supplémentaires, on éteint la machine virtuelle dès qu'on ne l'utilise plus.
 - Machine virtuelle locale : Gratuite grâce à VirtualBox et Kali Linux.

## 5. Ce que j'ai appris

- Qu'est-ce qu'un **"honeypot"**, comment l'installer et comment l'administrer.
- Pourquoi séparer le SSH d'administration du honeypot et comment le faire.
- Rappel des commandes Linux fondamentales et apprentissage de nouvelles commandes.
- Diagnostiquer ses erreurs, se documenter, demander de l'aide pour les réparer et apprendre de ces erreurs.

## 6. Références
 
- [Cowrie, dépôt officiel](https://github.com/cowrie/cowrie)
- [Documentation de Cowrie : installation](https://docs.cowrie.org/en/latest/INSTALL.html)
- [Azure for Students](https://azure.microsoft.com/free/students)
- [Kali Linux](https://www.kali.org/)

# 7. Licence
Ce projet est réalisé à but éducatif et est sous licence MIT - voir le fichier LICENSE pour plus de détails. 

## 8. Suite du projet au Mini SIEM !  

# EN  

# Cowrie Honeypot: from a local virtual machine to an Azure deployment
 
**Good to know:** This project is linked to the next one (Mini SIEM), notably through the extraction and analysis of the logs coming from the honeypot.
 
**Why?**  
 
I chose this project to learn several skills at the same time. From setting up a local virtual machine, to more advanced use on Azure, and up to exploiting and analysing the logs through the next project, this document describes how I went about this project. If you follow my steps, adapt them to your own situation.  
 
This repository documents the setup of a **Cowrie** honeypot (low-interaction SSH/Telnet), first in a local Kali Linux virtual machine to learn, then on an Azure virtual machine exposed to the Internet.
 
 
| | |
|---|---|
| **Tool** | [Cowrie](https://github.com/cowrie/cowrie), low-interaction SSH/Telnet honeypot |
| **Local environment** | VirtualBox, Kali Linux |
| **Cloud environment** | Azure for Students, Ubuntu Server 24.04 LTS VM |
| **Languages / tools** | Linux, Python, SSH, PowerShell |
 
 
## Contents
 
1. [Warnings](#1-warnings)
2. [Part 1: Cowrie on a local Kali Linux virtual machine](#2-part-1-cowrie-on-a-local-kali-linux-virtual-machine)
3. [Part 2: deployment on Azure](#3-part-2-deployment-on-azure)
4. [Costs](#4-costs)
5. [What I learned](#5-what-i-learned)
6. [References](#6-references)
---
 
## 1. Warnings
 
- This is an **educational** project. A honeypot deliberately attracts attackers: never install it on a machine that contains personal data, and never on your home network without isolation.
- Cowrie is a **low-interaction** honeypot: it simulates a shell, and the attacker does not get a real system. The risk is reduced, but not zero.
- The real SSH administration service must always be moved to another port and restricted to your IP address before exposing Cowrie on port 22.
- Never publish in this repository: the IP addresses of your machines, SSH keys (`*.pem`, private keys), credentials, personal usernames.
## 2. Part 1: Cowrie on a local Kali Linux virtual machine
 
### 2.1 Prerequisites
 
- Windows 11 with [VirtualBox](https://www.virtualbox.org/) installed
- A Kali Linux image (see 2.2)
- At least 2 GB of RAM and 25 GB of disk space available for the virtual machine
### 2.2 Create the Kali Linux virtual machine
 
1. Go to [kali.org/get-kali](https://www.kali.org/get-kali/) and select **Installer**, then download the **Installer** image.
2. In VirtualBox, click **New**:
  - Name: `honeypot-kali-linux`, OS: Ubuntu LTS
  - Memory: 2 GB minimum, although 1 GB is enough (like the size of the virtual machine used on Azure)
  - Processors: 1
  - Disk: 25 GB
    
3. Important: in the wizard, tick "Skip Unattended Installation" so that the installer asks you for the username and password.
4. During the installation, choose the French keyboard, create a non-administrator user (for example honeypot) and write down the username and password on paper.
### 2.3 Network: NAT and port forwarding
 
The virtual machine stays in **NAT** mode (isolated from the local network). To reach Cowrie from Windows, we are going to add rules in **Virtual machine settings > Network > Adapter 1 > Advanced > Port Forwarding**:
 
| Name | Protocol | Host port | Guest port |
|---|---|---|---|
| Cowrie | TCP | 2222 | 2222 |
| SSH-admin (optional) | TCP | 2200 | 22 |
 
### 2.4 Prepare the system
 
Once the system is installed:
 
Set the keyboard to AZERTY:
```bash
sudo dpkg-reconfigure keyboard-configuration
```
 
Updates and dependencies:
 
```bash
sudo apt update
```
 
```bash
sudo apt install -y git python3-venv python3-pip libssl-dev libffi-dev build-essential libpython3-dev
```
 
### 2.5 Install Cowrie
 
Cowrie must **never** run as `root`. We therefore install it under a regular user, in a Python virtual environment: 
 
Open a terminal (right-click on the desktop -> open a terminal).
Download Cowrie from GitHub:
```bash
git clone https://github.com/cowrie/cowrie.git
```
Go into the cowrie folder:
```bash
cd cowrie/
```
Create the virtual environment:
```bash
python3 -m venv cowrie-venv
```
Activate the virtual environment:
```bash
source cowrie-venv/bin/activate
```
 
```bash
pip install --upgrade pip
```
 
```bash
pip3 install -r requirements.txt
```
Install Cowrie:
```bash
pip install cowrie
```
 
```bash
tree bin
```
 
Find Cowrie:
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
We need to create a configuration file:
```bash
cp etc/cowrie.cfg.dist etc/cowrie.cfg
```
 
```bash
export PYTHONPATH=*(pwd)/src
```
Install the project in editable mode (don't forget the dot at the end):
```bash
pip install -e .
```
 
## 2.6 Start and test
 
From the `cowrie` folder, with the virtual environment activated:
Start cowrie:
```bash
bin/cowrie start
```
Check that cowrie is running properly:
```bash
bin/cowrie status
```
Open a new terminal:
Redirect port 22 to port 2222:
```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
```
 
Go back to the first terminal and check the logs and the SSH connection: 
```bash
tail -f var/log/cowrie/cowrie.log
```
You should see **Ready to accept SSH Connections** on the last line.
All the commands that attackers use will be displayed here.
 
You can test the connection using a second virtual machine with the following command:
 
```bash
sudo ssh root@honeypot-machine-ip -p 22
```
 
or from PowerShell on Windows:
```powershell
Test-NetConnection -ComputerName localhost -Port 2222
```
 
```powershell
ssh -p 2222 root@honeypot-machine-ip
```
Accept with `yes` and enter any password. We land in a fake shell. 
 
<img width="2559" height="1044" alt="image" src="https://github.com/user-attachments/assets/ac82d55c-6c75-46a0-89fa-540427597262" />
### 2.7 Attack test with Hydra
Since Hydra is already installed on Kali Linux, type directly in the virtual machine: 
```bash
echo -e "root\nadmin\ntest" > users.txt
```
 
```bash
echo -e "123456\npassword\nadmin\nroot" > passwords.txt
```
 
```bash
hydra -L users.txt -P passwords.txt ssh://127.0.0.1:2222
```
 
All the commands entered appear in the first terminal. 
 
---
 
## 3. Part 2: deployment on Azure
 
### 3.1 Azure for Students account
 
**Why Azure for Students?** Thanks to Azure's student offer, we can benefit from a $100 credit and a wide variety of free services.
1. Create the account at [azure.microsoft.com/free/students](https://azure.microsoft.com/free/students) (and not on the general trial page) with a student email address: no credit card is required.
2. In the portal, type **Education** in the search bar, then go to **Overview** to see the details of the student offer and to check that the credit of $100 (or thereabouts) has been granted to us. If not, we will have to wait a few days.
Since the virtual machine draws on our credit (see section 4), we can create a budget alert to manage our credit better and not allocate all of it to **a single** virtual machine.
All of this depends on the subscription and the institution, because the deployment regions are limited. In my case, with the Azure for Students subscription and because I study at ESAIP, my virtual machine is located in Spain, and some virtual machine sizes are only available in certain regions.
### 3.2 Create the virtual machine
 
Azure portal > **Virtual machines** > **Create** > **Azure virtual machine**
 
| Field | Value |
|---|---|
| Resource group | `CowrieHoneypot_group` (new) |
| Name | `honeypot-cowrie` |
| Region | a region **allowed by the subscription policy** (for the student subscription, choose **Spain Central**, see 3.1) |
| Image | Ubuntu Server 24.04 LTS, x64 Gen2 |
| Size | `Standard_B2ts_v2` (about $0.012/h) |
| Authentication | SSH public key, new key pair |
| Username | `azureuser` |
| Inbound ports | SSH (22) |
| Disk | Standard SSD, tick "Delete with VM" |
| Public IP address | New, **static**, deleted with the VM |
 
You can then skip all the other options.  
 
When creating the VM, click **Download private key and create resource**. Without this `.pem` file, you will **never** be able to connect.
 
### 3.3 Connect via SSH
Copy the key into `.ssh` (PowerShell)  
```powershell
Copy-Item "$HOME\Downloads\<KEY_NAME>.pem" "$HOME\.ssh\<KEY_NAME>.pem"
```
You can also do it from File Explorer.
 
To connect: 
```powershell
ssh -i $HOME\.ssh\<KEY_NAME>.pem azureuser@<PUBLIC_IP>
```
Replace with the public IP of your virtual machine on Azure, as well as the path where your key is located.  
 
### 3.4 Move the real SSH to port 22222
 
By default, Cowrie takes port 22 because that is the port bots scan first. We therefore need to move the real administration SSH to port 22222.  
There is, however, an **order** to follow: we add the new port, we test, and only then do we remove the old one.  
 
 
On Ubuntu 24.04, SSH is started by a systemd socket, and the `Port` line in `sshd_config` is ignored. So we check:
 
```bash
systemctl is-active ssh.socket
```
 
If the answer is `active`, create an override file that listens on both ports, on IPv4 **and** IPv6:
 
```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
```
 
```bash
printf '[Socket]\nListenStream=\nListenStream=0.0.0.0:22\nListenStream=[::]:22\nListenStream=0.0.0.0:22222\nListenStream=[::]:22222\n' | sudo tee /etc/systemd/system/ssh.socket.d/override.conf
```
 
```bash
sudo systemctl daemon-reload
```
 
```bash
sudo systemctl restart ssh.socket
```
 
```bash
sudo ss -tlnp | grep -E ':22\b|:22222'
```
 
When running the last command, if you see `0.0.0.0:22` and `0.0.0.0:22222`, then everything is going perfectly so far.  
 
We then open port 22222 in the network settings of the virtual machine, directly in the Azure portal (**Networking > Network settings > Add inbound port rule**):
 
| Field | Value |
|---|---|
| Source | IP addresses: **your public IP** (`Invoke-RestMethod https://api.ipify.org`) |
| Destination port ranges | `22222` |
| Protocol | TCP |
| Action | Allow |
| Priority | a free number (for example 1010) |
| Name | `ssh-admin-22222` |
 
We open a second PowerShell terminal to test the connection on the new port: 
 
```powershell
ssh -p 22222 -i $HOME\.ssh\<KEY_NAME>.pem azureuser@<PUBLIC_IP>
```
 
Again, replacing the IP address and the file path with the right ones.  
 
Once the connection is confirmed, we remove port 22 from the real SSH:
 
```bash
printf '[Socket]\nListenStream=\nListenStream=0.0.0.0:22222\nListenStream=[::]:22222\n' | sudo tee /etc/systemd/system/ssh.socket.d/override.conf
```
 
```bash
sudo systemctl daemon-reload
```
 
```bash
sudo systemctl restart ssh.socket
```
 
Only 22222 must remain listening. From now on, you must **always** connect with `-p 22222`.
 
### 3.5 Prepare the virtual machine
 
This virtual machine has only 1 GB of RAM: we need to create a swap file to prevent pip from being killed for lack of memory during the installation: 
 
```bash
sudo fallocate -l 2G /swapfile
```
 
```bash
sudo chmod 600 /swapfile
```
 
```bash
sudo mkswap /swapfile
```
 
```bash
sudo swapon /swapfile
```
 
```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
 
Once this is done, we install the updates and dependencies:
 
```bash
sudo apt-get update
```
 
```bash
sudo apt-get upgrade -y
```
 
```bash
sudo apt-get install -y python3-pip python3-venv libssl-dev libffi-dev build-essential libpython3-dev python3-minimal authbind
```
 
Then we create a dedicated user, because Cowrie **refuses** to start as root:
 
```bash
sudo adduser --disabled-password --gecos "" cowrie
```
 
```bash
sudo su - cowrie
```
 
## 3.6 Install Cowrie
 
The current Cowrie documentation installs the package with `pip`, then creates the configuration folder with `cowrie init`. These commands are run with the `cowrie` user:  
 
First, we create a `honeypot` folder in which we will install cowrie
 
```bash
mkdir ~/honeypot && cd ~/honeypot
```
 
 Then we create the Python virtual environment so that we can run cowrie:
 
```bash
python3 -m venv cowrie-env
```
 We activate this virtual environment:
 
```bash
source cowrie-env/bin/activate
```
 
```bash
python -m pip install --upgrade pip
```
Then we install cowrie:
 
```bash
python -m pip install cowrie
```
 
And we create the configuration folder:
 
```bash
cowrie init
```
We start cowrie:
 
```bash
cowrie start
```
 
And we check:
 
```bash
tail -n 20 var/log/cowrie/cowrie.log
```
 
```bash
ss -tlnp | grep 2222
```
 
We open another terminal to test the connection:
 
```bash
ssh -p 2222 root@127.0.0.1
```
 
### 3.7 Expose Cowrie on port 22
 
With `authbind`, a regular user can listen on a port below 1024. From the azureuser user:
 
```bash
sudo touch /etc/authbind/byport/22
```
 
```bash
sudo chown cowrie:cowrie /etc/authbind/byport/22
```
 
```bash
sudo chmod 770 /etc/authbind/byport/22
```
 
With the `cowrie` user, we stop Cowrie:
 
```bash
cowrie stop
```
 
Then we edit `~/honeypot/etc/cowrie.cfg`. We need to check whether a `listen_endpoints` line already exists under the `[ssh]` section: if so, change its value; otherwise, add the section **only once** (two `[ssh]` sections cause an error):
 
 
```ini
[ssh]
listen_endpoints = tcp:22:interface=0.0.0.0
```
 
```bash
AUTHBIND_ENABLED=yes cowrie start
```
 
```bash
ss -tlnp | grep ':22 '
```
 
In the Azure network security group, the port 22 rule must have **Any** as its source: it is now the only port open to the whole Internet, and it is served only by the honeypot.
 
From Windows, the PC had saved the fingerprint of the real SSH for this IP. Delete it, then test:
 
```powershell
ssh-keygen -R <PUBLIC_IP>
```
 
```powershell
ssh root@<PUBLIC_IP>
```
(Replace PUBLIC_IP with your public IP.)
 
### 3.8 Automatic start with systemd
 
Without a service, a reboot of the VM stops Cowrie without warning. Stop the instance started by hand (`cowrie stop` with the `cowrie` user, then `exit`), and create the service from the azureuser user. The `printf` command overwrites the whole file, which avoids duplicated content:
 
```bash
printf '[Unit]\nDescription=Cowrie SSH/Telnet honeypot\nAfter=network.target\n\n[Service]\nType=forking\nUser=cowrie\nGroup=cowrie\nWorkingDirectory=/home/cowrie/honeypot\nEnvironment=PATH=/home/cowrie/honeypot/cowrie-env/bin:/usr/local/bin:/usr/bin:/bin\nExecStart=/home/cowrie/honeypot/cowrie-env/bin/cowrie start\nExecStop=/home/cowrie/honeypot/cowrie-env/bin/cowrie stop\nPIDFile=/home/cowrie/honeypot/var/run/cowrie.pid\nAmbientCapabilities=CAP_NET_BIND_SERVICE\nRestart=on-failure\nRestartSec=5\n\n[Install]\nWantedBy=multi-user.target\n' | sudo tee /etc/systemd/system/cowrie.service
```
 
The resulting file:
 
```ini
[Unit]
Description=Cowrie SSH/Telnet honeypot
After=network.target
 
[Service]
Type=forking
User=cowrie
Group=cowrie
WorkingDirectory=/home/cowrie/honeypot
Environment=PATH=/home/cowrie/honeypot/cowrie-env/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=/home/cowrie/honeypot/cowrie-env/bin/cowrie start
ExecStop=/home/cowrie/honeypot/cowrie-env/bin/cowrie stop
PIDFile=/home/cowrie/honeypot/var/run/cowrie.pid
AmbientCapabilities=CAP_NET_BIND_SERVICE
Restart=on-failure
RestartSec=5
 
[Install]
WantedBy=multi-user.target
```
 
We check the unit file, enable it, then start it:
 
```bash
sudo systemd-analyze verify /etc/systemd/system/cowrie.service
```
 
```bash
sudo systemctl daemon-reload
```
 
```bash
sudo systemctl enable --now cowrie
```
 
```bash
systemctl status cowrie --no-pager
```
 
```bash
sudo ss -tlnp | grep ':22 '
```
 
To validate the automatic start, reboot the VM (`sudo reboot`), reconnect on port 22222, and check that Cowrie came back on its own.
 
### 3.9 Connect to the honeypot
 
Open a new PowerShell terminal and run:
 
```powershell
ssh root@AZURE_PUBLIC_IP
```
 
Replace AZURE_PUBLIC_IP with the public IP of your Azure virtual machine.  
You land on a fake shell: 
 
<img width="2341" height="1278" alt="image" src="https://github.com/user-attachments/assets/ce745730-acd5-488e-90d8-94b846f70334" />
 
 ## 4. Costs
 - Azure virtual machine `Standard_B2ts_v2`: about $0.012/h, i.e. about $9 per month if running continuously, paid with the student credit. However, to save additional costs, we shut the virtual machine down as soon as we no longer use it.
 - Local virtual machine: free thanks to VirtualBox and Kali Linux.

## 5. What I learned
 
- What a **"honeypot"** is, how to install it and how to administer it.
- Why to separate the administration SSH from the honeypot, and how to do it.
- A refresher on essential Linux commands and learning new ones.
- Diagnosing my own errors, researching, asking for help to fix them, and learning from these mistakes.

## 6. References
 
- [Cowrie, official repository](https://github.com/cowrie/cowrie)
- [Cowrie documentation: installation](https://docs.cowrie.org/en/latest/INSTALL.html)
- [Azure for Students](https://azure.microsoft.com/free/students)
- [Kali Linux](https://www.kali.org/)

# 7. License
This project is for educational purposes and is licensed under the MIT License - see the LICENSE file for more details. 
 
## 8. Next step of the project: the Mini SIEM!
