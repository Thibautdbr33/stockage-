# TP2 : Modest Storage SAN Compte Rendu
# Partie 0 : Prérequis 
- Avoir GNS3 
- Avoir VBox 
- Les isos nécessaires (switch, serveurs)

# Partie I : Setup GNS

## 1. Tableau d'adressage

| Node            | `mgmt-lan`     | `sto1-lan`   | `sto2-lan`   |
|-----------------|----------------|--------------|--------------|
| `sto1.tp2.b3`   | x              | `10.3.1.1`   | `10.3.2.1`   |
| `sto2.tp2.b3`   | x              | `10.3.1.2`   | `10.3.2.2`   |
| `chunk1.tp2.b3` | `10.3.250.11`  | `10.3.1.101` | `10.3.2.101` |
| `chunk2.tp2.b3` | `10.3.250.12`  | `10.3.1.102` | `10.3.2.102` |
| `chunk3.tp2.b3` | `10.3.250.13`  | `10.3.1.103` | `10.3.2.103` |
| `master.tp2.b3` | `10.3.250.1`   | x            | x            |
| `web.tp2.b3`    | `10.3.250.101` | x            | x            |

## 2. Setup réseau et GNS3

➜ **Clonez votre machine Linux autant de fois que nécessaire**
  - Créer une machine linux (j'ai choisi Debian 12 désolé) avec un utilisteur qui à les droits suoders, le service ssh installé et activé, firewall installé et activé avec le ssh allowed et une carte NAT pour avoir internet sur toutes les machines 
  - cloné la machine 6 fois pour avoir 7 machines (master, web, chunk1, chunk2, chunk3, sto1, sto2)
  - Avoir la VM GNS3 dans VBox 
  - Importer l'image du switch et lui mettre sa licence

- **Configurer les interfaces réseau**
  - Importer les VM dans GNS3
  - Ajouter les cartes réseaux aux machines sur GNS3 et les connecter comme demandé dans le tableau d'adressage
  - Configurer les interfaces réseau sur chaque machine avec les adresses en fonction du tableau d'adressage 

- **ajoutez une carte host-only supplémentaire à la machine `master.tp2.b3`**
  - Pour l'utiliserez pour se connecter en SSH
  - on indique encore une carte de + dans GNS3 pour `master.tp2.b3`
  - ce sera notre bastion SSH : on se connectera à lui pour se connecter aux autres machines

➜ **Avoir une connexion SSH sur toutes les machines du Réseau**

  - Sur la machine physique, générer une parire de clé ssh `ssh-keygen -t rsa -b 4096 -C` 

  - Créer un fichier config là où est la clé ssh générée `C:\Users\thiba\.ssh` pour utiliser la machine `master.tp2.b3` comme bastion SSH grâce au Proxyjump (Utiliser master pour accéder aux autres machines du réseau et chunk1 pour accéder à sto1 et sto2) :
  ```
  Host master
    HostName 192.168.148.5
    User user1

Host chunk1
    HostName 10.3.250.11
    User user1
    ProxyJump master

Host chunk2
    HostName 10.3.250.12
    User user1
    ProxyJump master

Host chunk3
    HostName 10.3.250.13
    User user1
    ProxyJump master

Host web
    HostName 10.3.250.101
    User user1
    ProxyJump master

Host sto1
    HostName 10.3.1.1
    User user1
    ProxyJump chunk1

Host sto2
    HostName 10.3.1.2
    User user1
    ProxyJump chunk1
  ```

 - On copie la clé publique générée `C:\Users\thiba\.ssh\id_rsa.pub` dans le fichier `authorized_keys` de toutes les machines du réseau pour ne pas à avoir mettre le mot de passe à chaque fois qu'on se connecte.

➜ **Attribuez un hostname à chaque machine**

- avec une commande `hostnamectl set-hostname`

➜ **Remplir le fichier `hosts` de chaque machine**
  ```
  ```
- comme ça on utilisera parfois les noms des machines pour mettre en place la config
- On vérifie avec quelques pings que ça peut se joindre avec les noms

# II. SAN network

Dans cette partie, on va travailler autour du (modeste) réseau SAN : les machines de stockage (`sto1` et `sto2`) ainsi que les machines qui vont consommer ce stockage : les "chunks" (`chunk1`, `chunk2` et `chunk3`).

L'objectif est le suivant :

- **les machines de stockage**
  - ont plein plein de disques
  - ont du RAID configuré
  - propose les volumes RAID à travers le réseau comme *target iSCSI*
- **les machines "chunks"**
  - accèdent aux disque des machines de stockage
  - grâce à iSCSI
  - mise en place d'un *multipathing iSCSI*

Le *multipathing* permet de prévenir d'une éventuelle défaillance d'un lien réseau en proposant deux chemins d'accès réseau pour accéder à un disque donné en iSCSI.

> *En milieu pro, dans des environnements de grande taille, on voit aussi beaucoup de câbles fibre optique, et le protocole FC (Fiber Channel) qui remplace iSCSI. Il faut alors utiliser des switches spécialisés (la gamme 9000 chez Cisco : les switches MDS, spécifiques pour les SAN). Les concepts restent les mêmes : FC transporte du trafic SCSI.*

Pour setup tout ça, on va utiliser les tools standards de Open iSCSI.


## 1. Storage machines

> **Toutes les manips de cettes section sont à réaliser sur `sto1` et `sto2`.**

### A. Disks and RAID

On va au supermarché (celui de VBox, c'est gratuit) et on ajoute plein de disques aux machines, pour configurer du RAID, notre premier niveau de redondance.

➜ **Ajouter des disques aux machines `sto1` et `sto2`**

- combien ? de quelle taille ? en vrai faites vous plais
- à la fin on veut (au minimum) 3 volumes RAID sur chaque `sto`
- au plus simple
  - 6 disques, pour faire 3 RAID1
  - au minimum 1G par disque
- **la même taille pour tous les disques**

🌞 **Configurer des RAID**

- à la fin, on veut 3 volumes RAID sur `sto1` et 3 volumes RAID sur `sto2`
- vous pouvez faire le setup simple indiqué avant ou vous faire un peu plais et partir sur du RAID5 par exemple
- **il faut 3 volumes RAID prêts à l'emploi en fin de setup**
- vous utiliserez MDADM pour mettre en place les RAID

🌞 **Prouvez que vous avez 3 volumes RAID prêts à l'emploi**

- avec une commande `lsblk`


### B. iSCSI target

On va configurer nos deux machines `sto1` et `sto2` pour qu'elles exposent leurs volumes RAID sur le réseau, avec iSCSI.

On appelle *target iSCSI* un device exposé sur le réseau grâce au protocole iSCSI.

On appelle *iSCSI initiator* une machine qui va initier une connexion iSCSI pour accéder à un *target iSCSI*.

Sur les machines `sto1` et `sto2`, on va donc s'occuper pour le moment de les configurer pour qu'elles exposent les volumes RAID comme *target iSCSI*.

Pour ça, on va utiliser l'outil `target` dispo sous les OS Linux.

🌞 **Installer `target`**

- c'est le paquet `targetcli`

🌞 **Démarrer le service `target`**

- activez le aussi au démarrage de la machine

🌞 **Configurer les *targets iSCSI***

- il faut configurer :
  - 3 target iSCSI sur chaque machine : un pour exposer chacun des volumes RAID
  - une ACL sur chaque target (vous pouvez try de mettre une authent, mais je vous conseille de rester simple)
- chaque target doit respecter la convention de nommage pour son IQN :
  - `iqn.2024-12.tp2.b3:data-chunk1` pour le premier target iSCSI (destiné à être utilisé par la machine `chunk1.tp2.b3`)
  - `iqn.2024-12.tp2.b3:data-chunk2` pour le deuxième
  - et `iqn.2024-12.tp2.b3:data-chunk3`
- en utilisant `target-cli`
  - exemple pour créer un target

```bash
$ sudo targetcli

# on crée un objet fileio que target peut gérer à partir de notre volume RAID
/> /backstores/fileio create name=data-chunk1 file_or_dev=/dev/path/vers/RAID

# on crée un IQN : un identifiant iSCSI unique
/> /iscsi create iqn.2024-12.tp2.b3:data-chunk1

# on crée une ACL pour que notre initator puisse accéder à ce target iSCSI
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator

# on map un LUN de notre target iSCSI vers notre objet fileio (le volume RAID)
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/luns/ create /backstores/fileio/data-chunk1

/> saveconfig
/> exit
```

➜ **Vous devriez avoir un truc comme ça une fois en place, avec les trois volumes RAID exposés comme target :**

```bash
[it4@sto1 ~]$ sudo targetcli
targetcli shell version 2.1.57
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> ls
o- / ......................................................... [...]
  o- backstores .............................................. [...]
  | o- block .................................. [Storage Objects: 0]
  | o- fileio ................................. [Storage Objects: 3]
  | | o- chunk1 ... [/dev/md/chunk1 (511.0MiB) write-back activated]
  | | | o- alua ................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ....... [ALUA state: Active/optimized]
  | | o- chunk2 ... [/dev/md/chunk2 (511.0MiB) write-back activated]
  | | | o- alua ................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ....... [ALUA state: Active/optimized]
  | | o- chunk3 ... [/dev/md/chunk3 (511.0MiB) write-back activated]
  | |   o- alua ................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....... [ALUA state: Active/optimized]
  | o- pscsi .................................. [Storage Objects: 0]
  | o- ramdisk ................................ [Storage Objects: 0]
  o- iscsi ............................................ [Targets: 3]
  | o- iqn.2024-12.tp2.b3:data-chunk1 .................... [TPGs: 1]
  | | o- tpg1 ............................... [no-gen-acls, no-auth]
  | |   o- acls .......................................... [ACLs: 1]
  | |   | o- iqn.2024-12.tp2.b3:chunk1-initiator .. [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 ............... [lun0 fileio/chunk1 (rw)]
  | |   o- luns .......................................... [LUNs: 1]
  | |   | o- lun0 .. [fileio/chunk1 (/dev/md/chunk1) (default_tg_pt_gp)]
  | |   o- portals .................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ..................................... [OK]
  | o- iqn.2024-12.tp2.b3:data-chunk2 .................... [TPGs: 1]
  | | o- tpg1 ............................... [no-gen-acls, no-auth]
  | |   o- acls .......................................... [ACLs: 1]
  | |   | o- iqn.2024-12.tp2.b3:chunk2-initiator .. [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 ............... [lun0 fileio/chunk2 (rw)]
  | |   o- luns .......................................... [LUNs: 1]
  | |   | o- lun0 .. [fileio/chunk2 (/dev/md/chunk2) (default_tg_pt_gp)]
  | |   o- portals .................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ..................................... [OK]
  | o- iqn.2024-12.tp2.b3:data-chunk3 .................... [TPGs: 1]
  |   o- tpg1 ............................... [no-gen-acls, no-auth]
  |     o- acls .......................................... [ACLs: 1]
  |     | o- iqn.2024-12.tp2.b3:chunk3-initiator .. [Mapped LUNs: 1]
  |     |   o- mapped_lun0 ............... [lun0 fileio/chunk3 (rw)]
  |     o- luns .......................................... [LUNs: 1]
  |     | o- lun0 .. [fileio/chunk3 (/dev/md/chunk3) (default_tg_pt_gp)]
  |     o- portals .................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................... [OK]
  o- loopback ......................................... [Targets: 0]
```

## 2. Chunks machine

> **On passe sur juste `chunk1.tp2.b3` pour cette partie.** Vous répliquerez la conf que vous allez tester sur lui une fois que c'est ok sur les deux autres : `chunk2.tp2.b3` et `chunk3.tp2.b3`. Uniquement sur `chunk1.tp2.b3` pour le moment donc.

Ici, on va configurer `chunk1.tp2.b3` pour qu'il utilise un *iSCSI initiator* afin d'accéder aux volumes RAID proposés sur le réseau par `sto1` et `sto2`.

Y'a un tool très répandu sur les OS Linux pour utiliser iSCSI : `iscsiadm` et le démon `iscsid`.

Avec `iscsiadm` on peut :

- découvrir des target iSCSI en indiquant l'IP à laquelle on souhaite se connecter
  - on indiquera les IP de nos machines `sto1` et `sto2`
- configurer un *initiator*
  - pour se connecter et se *login* en iSCSI à nos targets

### A. Simple iSCSI

🌞 **Installer les tools iSCSI sur `chunk1.tp2.b3`**

- c'est le paquet `iscsi-initiator-utils`

🌞 **Configurer un iSCSI initiator**

- utilisez `iscsiadm` pour découvrir les *targets iSCSI* proposés par `sto1` et `sto2`
- faites vos recherches pour la conf, c'est une conf plutôt standard
- les commandes `iscsiadm` génèrent des fichiers dans `/var/lib/iscsi`
- découvrez et utilisez les targets destinés à cette machine, y'en a 4 :
  - `iqn.2024-12.tp2.b3:chunk1` sur
    - `sto1` donc : `10.3.1.1` et `10.3.1.2`
    - `sto2` donc : `10.3.2.1` et `10.3.2.2`
- n'oubliez pas d'activer dès le démarrage de la machine les services iSCSI
  - service `iscsi`
  - service `iscsid`
- je vous donne la première commande :

```bash
sudo iscsiadm -m discoverydb -t st -p 10.3.1.1:3260 --discover
```

> On peut par exemple aussi lister les hôtes découverts et utilisables avec `sudo iscsiadm -m node`.

🌞 **Modifier la configuration du démon iSCSI**

- le fichier `/etc/iscsi/iscsid.conf`
- modifier la ligne `node.session.timeo.replacement_timeout`
- mettez lui 0 pour valeur, ça donne : `node.session.timeo.replacement_timeout = 0`
- redémarrer les services `iscsi` et `iscsid`

> *Vivement recommandé d'aller voir ce que fait cette option de config !*

🌞 **Prouvez que la configuration est prête**

- avec `lsblk` on doit voir les volumes
  - on doit donc voir 4 volumes en iSCSI
  - y'en a que deux en réalité (un proposé par `sto1` et un proposé par `sto2`) mais on les utilise depuis deux IPs différents à chaque fois, donc on les voit en double
- une commande `iscsiadm -m session -P 3` aussi dans le compte-rendu : on y voit en détails les connexions iSCSI en cours

### B. Multipathing

> Toujouuurs sur `chunk1.tp2.b3`.

Le *multipathing* va terminer le setup iSCSI. Actuellement, notre machine `chunk1` voit 4 volumes, alors qu'il en existe que 2 en réalité.

On va configurer le *multipathing* pour que la machine `chunk1` ne voit que deux volumes, chacun ayant deux chemins d'accès possibles : **on ajoute un niveau de redondance pour tolérer les pannes réseau.**

🌞 **Installer les outils multipath sur `chunk1.tp2.b3`**

- c'est le paquet `device-mapper-multipath`

🌞 **Configurer le fichier `/etc/multipath.conf`**

- récupérez le fichier d'exemple `usr/share/doc/device-mapper-multipath/multipath.conf`
- modifier la section ``defaults` comme ceci :

```conf
defaults {
  user_friendly_names yes
  find_multipaths yes
  path_grouping_policy failover
  features "1 queue_if_no_path"
  no_path_retry 100
}
```

🌞 **Démarrer le service `multipathd`**

🌞 **Et euh c'est tout, il est smart enough**

- ptit `lsblk`
  - vous devriez avoir des volumes `mpatha` et `mpathb`
- on peut aussi `multipath -ll` pour avoir + d'infos sur l'état du *multipathing*

➜ **On aboutit sur un truc comme ça :**

```bash
[it4@chunk1 ~]$ sudo multipath -ll
mpatha (36001405245a1502fa7244cba904620eb) dm-2 LIO-ORG,chunk2
size=511M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdc 8:32 active ready running
mpathb (3600140502598561316949c98c14c64d7) dm-4 LIO-ORG,chunk2
size=511M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 6:0:0:0 sde 8:64 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 3:0:0:0 sdb 8:16 active ready running

[it4@chunk1 ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0    8G  0 disk  
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    7G  0 part  
  ├─rl-root 253:0    0  6.2G  0 lvm   /
  └─rl-swap 253:1    0  820M  0 lvm   [SWAP]
sdb           8:16   0  511M  0 disk  
└─mpathb    253:4    0  511M  0 mpath 
sdc           8:32   0  511M  0 disk  
└─mpatha    253:2    0  511M  0 mpath 
sdd           8:48   0  511M  0 disk  
└─mpatha    253:2    0  511M  0 mpath 
sde           8:64   0  511M  0 disk  
└─mpathb    253:4    0  511M  0 mpath 
```

## 3. Formatage et montage

🌞 **Créez une partition sur les devices `mpatha` et `mpathb`**

- avec `fdisk`

🌞 **Formatez en `xfs` les partitions**

- avec `mkfs`

🌞 **Point de montage `/mnt/data_chunk1`**

- créez-le avec `mkdir`
- utilisez une commande `mount` pour y monter `mpatha`
- écrivez un unité systemd de type `mount` pour avoir un montage automatique
- ajoutez un `After=network-online.target` pour pas qu'il démarre avant le réseau et ne bloque pas le boot

> *Impossible à faire proprement avec juste du `/etc/fstab` ce démarrage après le réseau !*

🌞 **Point de montage `/mnt/data_chunk2`**

- pareil, pour `mpathb`

Ca donne ça :

```bash
[it4@chunk1 ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0    8G  0 disk  
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    7G  0 part  
  ├─rl-root 253:0    0  6.2G  0 lvm   /
  └─rl-swap 253:1    0  820M  0 lvm   [SWAP]
sdb           8:16   0  511M  0 disk  
└─mpathb    253:4    0  511M  0 mpath 
  └─mpathb1 253:5    0  503M  0 part  
sdc           8:32   0  511M  0 disk  
└─mpatha    253:2    0  511M  0 mpath 
  └─mpatha1 253:3    0  503M  0 part  
sdd           8:48   0  511M  0 disk  
└─mpatha    253:2    0  511M  0 mpath 
  └─mpatha1 253:3    0  503M  0 part  
sde           8:64   0  511M  0 disk  
└─mpathb    253:4    0  511M  0 mpath 
  └─mpathb1 253:5    0  503M  0 part

[it4@chunk1 ~]$ df -h 
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                229M     0  229M   0% /dev/shm
tmpfs                 92M  2.5M   89M   3% /run
/dev/mapper/rl-root  6.2G  2.5G  3.7G  41% /
/dev/sda1            960M  280M  681M  30% /boot
tmpfs                 46M  4.0K   46M   1% /run/user/1000
/dev/mapper/mpatha1  439M  149M  291M  34% /mnt/data_chunk1
/dev/mapper/mpathb1  439M  206M  234M  47% /mnt/data_chunk2
```

## 4. Tests

On va simuler une coupure réseau, en coupant l'un des liens sur `chunk1.tp2.b3`.

On va suivre en temps réel la bascule iSCSI.

L'idée :

- on lance une écriture continue sur le volume monté en multipath avec un ptit boucle shell
  - `while true; do date >> /mnt/iscsidisk/date; done`
  - cette ligne fait le taff, elle écrit continuellement l'heure dans un fichier
  - comme ça on verra quand ça stoppe ou non les écritures
- on coupe une interface réseau
  - utilisez une commande `date` juste avant de couper, pour savoir l'heure précise à laquelle vous coupez
  - rallumez au bout de ~30 secondes/1 minute (pareil, en tapant `date` avant)
- on observe la bascule iSCSI
  - on peut regarder les logs en temps réel, et observer aussi la sortie de `multipath -ll` : `watch -n 0.5 multipath -ll`

### A. Simulation de panne

🌞 **Simuler une coupure réseau**

- couper un lien réseau
- surveiller les logs en temps réel
  - je veux quelques lignes de logs qui montre la bascule dans le compte-rendu
- surveiller `multipath -ll`
  - je veux bien un avant/après dans le compte-rendu

### B. Jouer avec les paramètres

Pour améliorer la rapidité de la bascule (attention si le réseau est instable, ça peut coûter cher), on peut ajouter un peu de conf et déterminer des temps plus courts pour les timeouts par exemple.

Je vous laisse aller lire le détails des options, je vous recommande de tester avec ces paramètres dans `/etc/multipath.conf` :

```conf
defaults {
 user_friendly_names yes
 find_multipaths yes
 path_grouping_policy failover
 no_path_retry 100
 features "1 queue_if_no_path"
 polling_interval 2
 max_polling_interval 8
 fast_io_fail_tmo 3
}
```

🌞 **Resimuler une panne**

- on devrait observer une bascule légèrement plus rapide

## 5. Replicate

➜ **Répliquer le setup de `chunk1` sur les machines `chunk2` et `chunk3`**

- chaque machine ne doit accéder qu'aux targets iSCSI qui lui sont destinés
  - chaque machine "chunk" a un IQN qui lui est dédié
  - la machine `chunk1`, on lui a donné accès qu'aux IQN `iqn.2024-12.tp2.b3:chunk1` (sur chaque IP)
  - il faut donc faire pareil la machine `chunk2` et sur `chunk3`
- même setup :
  - `iscsiadm` pour les 4 targets iSCSI
  - puis multipath pour en avoir plus que 2
  - vous formatez en `xfs` les deux volumes `mpatha` et `mpathb` et montez sur `/mnt/data_chunk1` et `/mnt/data_chunk2`

🌞 **Preuve du setup**

- sur `chunk2.tp2.b3` et sur `chunk3.tp2.b3`
- les commandes suivantes :

```bash
$ lsblk
$ df -h
$ iscsiadm -m session -P 3
$ multipath -ll
```

---

**On a un setup très solide, un ptit SAN modeste, mais solide !**

- **2 serveurs de stockage**
  - configuration RAID
  - qui sont exposés comme des targets iSCSI
- **3 serveurs qui accèdent en multipath à 2 volumes**
  - tolérant aux pannes réseau

**Et le tout est très facile à faire *scale* (ça se script/ansible très bien)** :

- agrandir les RAID
- ajouter des RAID/targets
- ajouter un initiator

Let's go, on enchaîne sur un setup de système de fichiers distribué sur ces 3 serveurs "chunk" (donc le nom va prendre sens).
