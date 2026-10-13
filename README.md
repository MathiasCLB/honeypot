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
1. Créer le compte sur [azure.microsoft.com/free/students](https://azure.microsoft.com/free/students) (et pas sur la page d'essai générale) avec une adresse e-mail d'étudiant : aucune carte bancaire n'est demandée.

2. Dans le portail, taper **Education** dans la barre de recherche puis aller dans **Vue d'ensemble** pour voir les détails de l'offre étudiant et pour vérifier que le crédit de 100$ (ou environ) nous a bien été attribué. Sinon, il faudra attendre quelques jours.
Sachant que la machine virtuelle prend sur notre crédit (voir 4.2), nous pouvons créer une alerte de budget pour pouvoir mieux gérer notre crédit et ne pas tout attribuer à **une seule** machine virtuelle.
Tout ceci est adapté à l'abonnement et l'établissement car les régions de déploiement sont limitées. Dans mon cas, avec l'abonnement Azure for Students et du fait que je suis scolarisé à l'ESAIP, ma machine virtuelle se situe en Espagne et certaines tailles de machines virtuelles ne sont disponibles que dans certaines régions.

### 4.2 Créer la machine virtuelle

Portail Azure > **Machines virtuelles** > **Créer** > **Machine virtuelle 

| Champ | Valeur |
|---|---|
| Groupe de ressources | `CowrieHoneypot_group` (nouveau) |
| Nom | `honeypot-cowrie` |
| Région | une région **autorisée par la politique de l'abonnement** (dans le cas de l'abonnement étudiant choisir **Spain Central**(voir 4.9) |
| Image | Ubuntu Server 24.04 LTS, x64 Gen2 |
| Taille | `Standard_B2ts_v2` (environ 0,012 $/h) |
| Authentification | Clé publique SSH, nouvelle paire de clés |
| Nom d'utilisateur | `azureuser` |
| Ports entrants | SSH (22) |
| Disque | SSD standard, cocher « Supprimer avec la machine virtuelle » |
| Adresse IP publique | Nouvelle, **statique**, supprimée avec la VM |

Vous pouvez ensuite passer toutes les autres options.  

Au moment de créer, cliquer sur **Télécharger la clé privée et créer la ressource**. Sans ce fichier `.pem`, vous ne pourrez **jamais** vous connecter.

### 4.3 Se connecter en SSH
Copiez la clé dans `.ssh` (PowerShell)  
```powershell
Copy-Item "$HOME\Downloads\<NOM_DE_LA_CLE>.pem" "$HOME\.ssh\<NOM_DE_LA_CLE>.pem"
``
Vous pouvez également le faire depuis l'explorateur de fichiers.

Pour vous connecter : 
```powershell
ssh -i $HOME\.ssh\<NOM_DE_LA_CLE>.pem azureuser@<IP_PUBLIQUE>
```
Remplacez par l'IP publique de votre machine virtuelle sur Azure ainsi que le chemin où se trouve votre clé.  

### 4.4 Déplacer le vrai SSH sur le port 22222

Par défaut, Cowrie prend le port 22 car c'est celui que les bots scannent en priorité. Nous devons donc déplacer le vrai SSH d'administration va donc sur le 22222.  
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

On ouvre un duexième terminal PowerShell pour tester la connexion sur le nouveau port : 

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

### 4.5 Préparer la machine virtuelle

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

## 4.6 Installer Cowrie

La documentation actuelle de Cowrie installe le paquet avec `pip`, puis crée le dossier de configuration avec `cowrie init`. Ces commandes se lancent avec l'utilisateur `cowrie` :  

On crée tour d'abord un dossier `honeypot` dans lequel on va installer cowrie

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
 
### 4.7 Exposer Cowrie sur le port 22
 
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

### 4.8 Lancement automatique avec systemd
 
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

 ## 5. Coûts
 - Machine virtuelle Azure `Standard_B2ts_v2` : environ 0,012$/h, soit environ 9$ par mois en continu, payés grâce au crédit étudiant. Cependant, pour économiser des coûts supplémentaires, on éteint la machine virtuelle dès qu'on ne l'utilise plus.
 - Machine virtuelle locale : Gratuite grâce à VirtualBox et Kali Linux.

## 6. Ce que j'ai appris

- Qu'est-ce qu'un **"honeypot"**, comment l'installer et comment l'administrer.
- Pourquoi séparer le SSH d'administration du honeypot et comment le faire.
- Rappel des commandes Linux fondamentales et apprentissage de nouvelles commandes.
- Diagnostiquer ses erreurs, se documenter, demander de l'aide pour les réparer et apprendre de ces erreurs.

## 7. Références
 
- [Cowrie, dépôt officiel](https://github.com/cowrie/cowrie)
- [Documentation de Cowrie : installation](https://docs.cowrie.org/en/latest/INSTALL.html)
- [Azure for Students](https://azure.microsoft.com/free/students)
- [Kali Linux](https://www.kali.org/)

## 8. Suite du projet au Mini SIEM !

