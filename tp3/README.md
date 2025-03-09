# TP 3 Hardening d'OS

# Part 1 : a bit of exploration

### 1 : /proc

Afficher l'état complet de la ram
```
cat /proc/meminfo
```

Afficher le nombre de coeurs que votre cpu a
```
cat /proc/cpuinfo | grep "cpu cores"
```

Afficher le nombre de processus lancés
```
cat /proc/stat | grep "proc_running"
```

Afficher la ligne de commande utilisée pour lancer le kernel actuel
```
cat /proc/1/cmdline
```

Afficher la liste des connexions TCP actuelles
```
cat /proc/net/tcp
```

Afficher la valeur actuelle de la swappiness
```
cat /proc/sys/vm/swappinsess
```

### 2. /sys

Afficher la liste des périphériques de type bloc reconnus par l'OS
```
ls /sys/blocks
```

Afficher la liste de smodules kernels qui sont actuellement en cours d'utilisation
```
ls /sys/module
```

Afficher la liste des cartes réseau
```
ls /sys/class/net
```

## Part 2 : Gotta get chrooty

### 1.Play manually

Créer le dossier /srv/get_chrooted
```
mkdir srv
mkdir srv/get_chrooted
```

Essayer de chroot à l'intérieur en lançant un shell
```
chroot srv/get_chrooted/
"chroot : failed to run command '/bin/bash' : No such file or directory"
```
Déplacez le nécessaire pour pouvoir lancer un shell chrooté
```
mkdir /srv/get_chrooted/bin
mkdir /srv/get_chrooted/lib
mkdir /srv/get_chrooted/lib64

cp /bin/bash /srv/get_chrooted/bin/
cp -r /lib/* /srv/get_chrooted/lib/
cp -r /lib64/* /srv/get_chrooted/lib64/

sudo chroot /srv/get_chrooted/
#ça marche ! :)
```
### 2. SSH, old friend

Créez un user imsad
```
sudo adduser imsad
sudo passwd imsad
#ajout du mdp
```

Modifiez la configuration du serveur SSH
```
#Dans le fichier /etc/ssh/sshd_config :

Match User imsad
        ChrootDirectory /srv/get_ch
rooted

#Pour renforcer l'illusion qu'il est dans un environnement classique, on peut copier différentes commandes comme ls, et créer un répertoire pour cet user
```
Connexion ssh :
Créer une clé avec ssh-keygen, et l'ajouter dans le dossier .ssh du répertoir home du user immsad dans le chroot.
ajouter à la conf ssh :
```
PubKeyAuthentication yes
```

Redémarrer le service ssh et essayer une connexion.
Une fois le user connecté, il ne sais pas qu'il est dans un chroot car j'ai ajouté différentts dossier pour répliquer un système classique (/usr/bin /etc /lib /lib64 ...)

## Part 3. Cgroups

### 1.Explore

Afficher la liste des controllers CGroups dispos sur le système
```
cat /proc/cgroups
```

Afficher la quantité de mémoire max que l'on est autorisé à utiliser dans notre session utilisateur
```
cat /sys/fs/cgroup/user.slice/memory.max
```

Afficher le nom de tout les CGroups créés
```
ls /sys/fs/groups/ | grep '^d'
```

### 2.Do it


Nouveau CGroup
```
mkdir /sys/fs/cgroup/meow
cd /sys/fs/cgroup/meow
#activer les controllers au subtree_control parent
echo "+cpu +cpuset +memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
#ajout des controllers au subtree_control de meow
echo "+cpu +cpuset +memory" | sudo tee cgroup.subtree_control
```
créer un nouveau sous-cgroup
```
mkdir task1
```

Vérifiation que les controllers sont bien hérités :
```
echo "+cpuset +memory +cpu" | sudo tee task1/cgroup.subtree_control
cat task1/cgroup.subtree_control
#le fichier contient bien cpuset, cpu et memory. Les controllers ont bien été hérités
```

Mettez en place une limitation de la RAM
```
echo "150M" task1/memory.max
```
Prouvez que la limite est effective
```
stress-ng --vm 2 --vm-bytes 4G --timeout 60s
#résultat top
MiB Mem :   3658.7 total,    121.7 free,   36
#la mémoire est donc bien remplie
```

Ajouter votre shell bash au CGroup task1
```
echo $$ | sudo tee /sys/fs/cgroup/meow/task1/cgroup.proc

#lancement du stress
stress-ng --vm 1 --vm-bytes   50161  4480
stress-ng --vm 1 --vm-bytes   50162  1596
stress-ng --vm 1 --vm-bytes   50163 151236
grep stress                   50553  2048

#le processus ne dépasse jamais la limite sans être kill
```

Appliquer les restrictions cpu
```
#pwd : /sys/fs/cgroup/meow
echo "1" | sudo tee task1/cpu.weight
echo "2" | sudo tee task2/cpu.weight

#le htop montre que l'un des processus ne semble pas être limité alors que l'autre beaucoup plus.
```


## 3.systemd

### A.One-Shot

Lancer un serveur web python sous forme de service temporaire 
```
sudo systemd-run -u pyth_serv_test python3 -m http.server 8888

 systemctl status pyth_serv_test.service
● pyth_serv_test.service - /bin/python3 -m h>
     Loaded: loaded (/run/systemd/transient/>
  Transient: yes
     Active: active (running) since Thu 2025>
   Main PID: 63243 (python3)
      Tasks: 1 (limit: 23155)
     Memory: 9.7M
        CPU: 96ms
     CGroup: /system.slice/pyth_serv_test.se>
             └─63243 /bin/python3 -m http.se>

Feb 27 13:22:33 rocky systemd[1]: Started /b>

```
Appliquer à la volée des restrictions
```
sudo systemd-run -p MemoryMax=234M -u pyth_serv_test -m http.server 8888

#vérification que la mémoire est bien limitée

systemctl status pyth_serv_test
● pyth_serv_test.service - /bin/python3 -m h>
     Loaded: loaded (/run/systemd/transient/>
  Transient: yes
     Active: active (running) since Thu 2025>
   Main PID: 63385 (python3)
      Tasks: 1 (limit: 23155)
     Memory: 9.2M (max: 234.0M available: 22>
        CPU: 101ms
     CGroup: /system.slice/pyth_serv_test.se>
             └─63385 /bin/python3 -m http.se>

Feb 27 13:35:41 rocky systemd[1]: Started /b>
lines 1-12/12 (END)

#Memory : 9.2M (max:234 .....) prouve que la mémoire est bien limitée pour le service que je viens de créer.
```
### B.Real service

Créer un service web.service
```
touch /etc/systemd/system/web.service

#contenu du fichier :

[Unit]
Description=Simple NGINX Web Server
After=network.target

[Service]
ExecStart=/usr/sbin/nginx -g 'daemon off;'
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Restart=always
Type=forking
PIDFile=/run/nginx.pid

[Install]
WantedBy=multi-user.target
EOF

```
Test de visualisation de la page :
```
[vic@rocky ~]$ curl http://127.0.0.1
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hello, it works ! &#129497;&#8205;&#9794;</title>
</head>
<body>
    <h1>Hello, it works ! &#129497;&#8205;&#9794;</h1>
</body>
</html>

```
Modification du fichier pour ajouter les restrictions liées au CGroups
```
vim /etc/systemd/system/nginx-webserver.service


MemoryHigh=321M
IOReadBandwidthMax=/dev/sda 1048576
IOWriteBandwidthMax=/dev/sda 1048576
CPUQuota=50%
```

Vérification de la présence des restrictions

```
ls /sys/fs/cgroups/system.slice/web.service

bon en fait j'ai pas réussi à start mon service nginx mais ça devrait donner les fichiers suivants :

/sys/fs/cgroup/system.slice/web.service/memory.max 
#avec la valeur pour 321M

/sys/fs/cgroup/system.slice/web.service/io.max
8:0 rbps=1048576 wbps=1048576

/sys/fs/cgroup/system.slice/web.service/cpu.max

50000 100000


## Partie IV

## 1.Explore
Namespaces du bash actuel :
```
echo $$
ls /proc/1303/ns

cgroup  pid                user
ipc     pid_for_children   uts
mnt     time
net     time_for_children

```

Namespaces du PID 1 :
```
ls /proc/1/ns

cgroup  pid                user
ipc     pid_for_children   uts
mnt     time
net     time_for_children
```
All namespaces
```4026531834 time        6  1263 vic  /usr/lib/
4026531835 cgroup      6  1263 vic  /usr/lib/
4026531836 pid         6  1263 vic  /usr/lib/
4026531837 user        6  1263 vic  /usr/lib/
4026531838 uts         6  1263 vic  /usr/lib/
4026531839 ipc         6  1263 vic  /usr/lib/
4026531840 net         6  1263 vic  /usr/lib/
4026531841 mnt         6  1263 vic  /usr/lib/
```
## 2.Create

Créer un nouveau namespace
```
sudo unshare --net --fork bash
[root@rocky vic]# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

#je ne vois en effet plus ma carte réseau

lsns -t net

4026532283 net       3 12458 root unassigned      unshare

#Vérification de la présence dans /proc

ls -al /proc/$$/ns/net
lrwxrwxrwx. 1 root root 0 Mar  9 22:32 /proc/12459/ns/net -> 'net:[4026532283]'
```
Création d'un nouveau namespace pid
```
[vic@rocky ~]$ sudo unshare --pid --mount-proc --fork bash
[root@rocky vic]# ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 22:36 pts/0    00:00:00 bash
root          20       1  0 22:36 pts/0    00:00:00 ps -ef

#Vérification avec lsns
lsns -t pid
        NS TYPE NPROCS PID USER COMMAND
4026532284 pid       2   1 root bash

#Check de la présence dans /proc

ls -al /proc/$$/ns/pid
lrwxrwxrwx. 1 root root 0 Mar  9 22:36 /proc/1/ns/pid -> 'pid:[4026532284]'

```

## 3.AND MY CONTAINERS

Installation de docker sur la machine
cf https://docs.rockylinux.org/gemstones/containers/docker/
```
sudo systemctl status docker
● docker.service - Docker Application Contai>
     Loaded: loaded (/usr/lib/systemd/system>
     Active: active (running)
```
Prouvez que :
```
#Le processus sleep est bien lancé:
ps -ef | grep sleep

root       18321   18300  0 22:48 ?        00:00:00 sleep 9999
vic        18390   18189  0 22:53 pts/0    00:00:00 grep --color=auto sleep

#Le processus sleep est bien isolé à l'aide des namespaces

ls /proc/18321/ns/

#Ce dossier existe et est rempli, mais celui du deuxième processus lancé par vic n'existe pas :(

#Trouver le CGroup
cat /proc/18321/cgroup
0::/system.slice/docker-f79cc02ca4033c7052828eb4864707ce909f83df2ceb9d625527bf1c0f7fab33.scope

## C.CGroups

Lancer un conteneur restreint :
```
docker run -d --memory=456m debian sleep 9999
```
CGroup ?
```
ls /proc/19019/cgroup
0::/system.slice/docker-e3e309c9d47ce1a9412a0
7e57f04bcc0786c84a358b3d0bab1f7138e906cdffe.scope

cat /sys/fs/cgroup/system.slic
e/docker-e3e309c9d47ce1a9412a07e57f04bcc0786c84a358b3d0bab1f7138e906cdffe.scope/memory.max 478150656
#soit 456 MB
```
# Part V : Harden me baby

#WorkInProgress
