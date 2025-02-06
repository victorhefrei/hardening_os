# Compte rendu tp1
## Hardening OS

### Partie 1

Lister tous les utilisateurs sur la machine :
``` 
cat /etc/passwd
```
Lister tous les groupes d'utilisateurs :
``` 
cat /etc/group
```

Lister les groups auxquels un utilisateur appartient :
``` 
groups [username]
```
Lister les processus lancés par root :
```
 ps -aux | grep root
```
Lister les processus en cours d'exécution lancés par un utilisateur :
```
ps -U [username] -u [username] u
```
Déterminer le hash du mot de passe de root
```
cat /etc/shadow | grep root
```
Déterminer le hash du mot de passe de votre utilisateur
```
cat /etc/shadow | [username]
```

Déterminer l'algorithme utilisé pour le hash du mot de passe :
 -> regarder le début du hash.
$1$ = md5
$5$ = sha 256
$6$ = sha 512

Déterminer, pour l'utilisateur root : 
 - son shell par défault :
```
cat /etc/passwd | grep root | cut -d ':' -f 1,7
```
 - le chemin vers son répertoir personnel: 
```
 cat /etc/passwd | grep root | cut -d ':' -f 1,6
```

Déterminer, pour un utilisateur 
 - son shell par défault
```
cat /etc/passwd | grep [username] | cut -d ':' -f 1,7
```
 - le chemin vers son répertoir personnel
```
cat /etc/passwd | grep [username] | cut -d ':' -f 1,6
```

Afficher la ligne de configuration du fichier qui permet à votre utilisateur d'utiliser sudo : 
```
cat /etc/sudoers | grep [username]
```
Creer le user meow :
```
sudo adduser meow -g admins -M -s /bin/false
```

configuration du fichier sudoers (avec visudo) pour autoriser meow a executer ls, mat, less et more :
```
meow ALL=(vic) NOPASSWD: /usr/bin/ls, /usr/bin/cat, /usr/bin/less, /usr/bin/more
```
configuration du fichier sudoers pour le groupe admins
```
%sys ALL=(ALL) NOPASSWD: /usr/bin/dnf
```

conf pour mon user :
```
[username]  ALL=(ALL)   ALL
```

Comment se connecter en root : 
```
sudo -u [username] more /var/log/dnf.log
```

une fois dans le "more" :
```
!/bin/bash
```

On peut ensuite passer root.

configuration "safe" : 
laisser simplement l'accès à /bin/ls et /bin/cat qui permet d'éviter ce contournement.


### Partie 2
permissions fichiers/dossiers : 

liste utilisateurs :
```
ls -l /etc/passwd
-rw-r--r--
```
hashes des mots de passe utilisateurs :
```
ls -l /etc/shadow
----------
```
conf serveur openssh :
```
ls -l /etc/ssh/ssh_config
-rw-r--r--
```
répertoire personnel utilisateur root :
```
ls -l / #regarder le /root
dr-xr-x---
```
répertoire personnel de votre utilisateur :
```
ls -l $HOME/..
drwx------
```
programme ls :
```
ls -l /bin/ls
-rwxr-xr-x
```
programme systemctl :
```
ls -l /bin/systemctl
-rwxr-xr-x
```

Lister les programmes qui ont le bit suid activé : 
```
sudo find / -type f -perm -4000 -ls
```

Rendre le fichier de configuration du serveur OpenSSH immuable
```
sudo chmod 444 /etc/ssh/ssh_config
```
vérification :
```
ls -l /etc/ssh/ssh_config 
-r--r--r--
```

restreindre l'accès au fichier "dont_readme.txt"
```
chmod 600 /tmp/dont_readme.txt
```
meow ne peux pas lire le fichier tandis que root peux.

### partie 3

déterminer la liste des programmes qui écoutent sur un port tcp et udp :
```
ss -tupnla
```
On peux voir avec les commandes suivantes qu'il n'y a pas règles pour les ports précédemment détectés : 
```
sudo firewall-cmd --zone=public --query-port=68/tcp
sudo firewall-cmd --zone=public --query-port=22/tcp
sudo firewall-cmd --zone=public --query-port=323/udp
```
Fermer tout les ports à part ssh :
```
sudo firewall-cmd --zone=public --remove-service=dhcpv6-client
sudo firewall-cmd --zone=public --remove-service=cockpit
sudo firewall-cmd --reload
```

#partie 3 - 2 - 3 a finir

### Partie 4

Déterminer la liste des partitions du système :
```
lsblk
```
Pour la partition montée sur /, on peut voir dans le retour de la commande "lsblk" qu'il s'agit de la partition sda2/[hostname]-root pour mon os.

Déterminer les options de montage de la partition /tmp

```
mount | grep tmp
```

Changer les options de montage pour /tmp

```
sudo mount -t tmpfs noexec /tmp
```
Une fois qu'il est monté, j'ai copié la commande ls dans le /tmp que j'ai renommé "test", et vérifié qu'il était exécutable. J'ai ensuite tenté de l'exécuter mais je ne peux pas.

### Partie 5

Afficher tout les programmes en cours d'exécution :
```
ps aux
```
Afficher l'identifiant du processus openssh :
```
ps aux | grep sshd
```
Le PID du serveur ssh est 738.

Changer le port d'écoute du serveur OpenSSH :
Dans le fichier /etc/ssh/sshd_config
```
port=2341
```

Il est parfois utile de changer le port pour éviter que les attaquant sachent qu'un serveur ssh est disponible lorsqu'ils font un scan des ports classique.


Configuration d'une authentification par clé :
```
#Sur la machine locale
ssh-keygen

ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2341 username@server_ip

#fichier conf du serveur ssh /etc/ssh/sshd_config
PubKeyAuthentication yes
```

Désactiver la connexion par password :

```
#Fichier /etc/ssh/sshd_config

PasswordAuthentication no
```

Désactiver la connexion root :
```
#Fichier /etc/ssh/sshd_config
PermitRootLogin no
```

### Mise en place d'une connexion SSH via certificat :

Génération de la clé SSH du serveur  
```
ssh-keygen -t rsa -b 4096 -f /etc/ssh/ssh_host_rsa_key -N ""
```

Génération de la paire de clés du CA hôte  
```
ssh-keygen -t rsa -b 4096 -f ~/host_ca -N ""
```

Signature de la clé hôte avec le CA  
```
sudo ssh-keygen -s ~/host_ca -I rocky_tp -h -n localhost -V +52w /etc/ssh/ssh_host_rsa_key.pub
```

Déplacement et configuration du CA hôte sur le serveur  
```
sudo mkdir -p /etc/ssh/certs
sudo mv ~/host_ca.pub /etc/ssh/certs/
sudo chmod 644 /etc/ssh/certs/host_ca.pub
```

Ajout de la configuration du serveur SSH  
```
HostCertificate /etc/ssh/ssh_host_rsa_key-cert.pub
TrustedUserCAKeys /etc/ssh/certs/host_ca.pub
```

Redémarrage du service SSH  
```
sudo systemctl restart ssh
```

Copie de la clé publique du CA vers le client  
```
scp -P 2222 /etc/ssh/certs/host_ca.pub vic@localhost:/tmp/
```

Déplacement et permission de la clé CA sur le client  
```
mv /tmp/host_ca.pub ~/.ssh/
chmod 644 ~/.ssh/host_ca.pub
```

Ajout du CA aux hôtes connus du client  
```
echo "@cert-authority * $(cat ~/.ssh/host_ca.pub)" >> ~/.ssh/known_hosts
```

Test de la connexion SSH  
```
ssh -p 2222 vic@localhost
```



Idées pour sécuriser le serveur SSH :
- mise en place d'une mfa (multi factor authentication)
- mettre en place une whitelist pour les postes distants autorisés à se connecter au serveur
- limiter l'accès au système de fichiers pour les utilisateurs
- limiter les commandes disponibles pour les utilisateurs
- bloquer les tentatives de bruteforce


