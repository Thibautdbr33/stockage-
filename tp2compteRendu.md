# TP2 : Modest Storage SAN - Compte Rendu

## Partie 0 : Prérequis

- Avoir **GNS3** installé
- Avoir **VBox** installé
- Disposer des **ISOs** nécessaires (switch, serveurs)

---

## Partie I : Setup GNS

### 1. Tableau d'adressage

| Node            | `mgmt-lan`     | `sto1-lan`   | `sto2-lan`   |
| --------------- | -------------- | ------------ | ------------ |
| `sto1.tp2.b3`   | x              | `10.3.1.1`   | `10.3.2.1`   |
| `sto2.tp2.b3`   | x              | `10.3.1.2`   | `10.3.2.2`   |
| `chunk1.tp2.b3` | `10.3.250.11`  | `10.3.1.101` | `10.3.2.101` |
| `chunk2.tp2.b3` | `10.3.250.12`  | `10.3.1.102` | `10.3.2.102` |
| `chunk3.tp2.b3` | `10.3.250.13`  | `10.3.1.103` | `10.3.2.103` |
| `master.tp2.b3` | `10.3.250.1`   | x            | x            |
| `web.tp2.b3`    | `10.3.250.101` | x            | x            |

### 2. Setup réseau et GNS3

#### Clonez votre machine Linux

1. Créez une machine Linux et avoir cette config j'ai choisi **Debian 12** (désolé):

![sorry](https://media1.tenor.com/m/GJcRHVW2LwYAAAAd/whatever-duh.gif)

   - Un utilisateur disposant des droits sudo.
   - SSH installé et activé.
   - Firewall configuré avec le port SSH ouvert.
   - Une carte NAT pour l'accès à Internet.
2. Clonez cette machine **6 fois** pour obtenir 7 VMs :
   - `master`, `web`, `chunk1`, `chunk2`, `chunk3`, `sto1`, et `sto2`.

![again](https://media1.tenor.com/m/Uo8tYyr9lVUAAAAC/again-and-again-and-again-arun-maini.gif)


#### Ajouter et configurer les cartes réseau

1. Importez les VMs dans GNS3.
2. Ajoutez les cartes réseaux et connectez-les selon le tableau d'adressage.
3. Configurez les interfaces avec `ip` ou via `/etc/network/interfaces`.

#### Configurer `master.tp2.b3` comme bastion SSH

1. Ajoutez une carte réseau **Host-Only** supplémentaire à `master.tp2.b3`.
2. Configurez SSH ProxyJump :
   ```bash
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

   Host sto1
       HostName 10.3.1.1
       User user1
       ProxyJump chunk1

   Host sto2
       HostName 10.3.1.2
       User user1
       ProxyJump chunk1
   ```
3. Copiez la clé publique sur toutes les machines :
   ```bash
   ssh-copy-id user1@<machine_name>
   ```

#### Configurer les noms d'hôtes et les fichiers `hosts`

1. Définissez les noms d'hôtes :
   ```bash
   sudo hostnamectl set-hostname <hostname>
   ```
2. Modifiez `/etc/hosts` :
   ```plaintext
   127.0.0.1       localhost
   10.3.2.1        sto1.tp2.b3 sto1
   10.3.2.2        sto2.tp2.b3 sto2
   10.3.1.101      chunk1.tp2.b3 chunk1
   10.3.1.102      chunk2.tp2.b3 chunk2
   10.3.1.103      chunk3.tp2.b3 chunk3
   10.3.250.1      master.tp2.b3 master
   10.3.250.101    web.tp2.b3 web
   ```
3. Vérifiez avec `ping <hostname>`.

![easy](https://media1.tenor.com/m/7Gz6Wttb-xAAAAAd/its-gonna-be-so-easy-frank-sinatra.gif)

---

## Partie II : SAN Network

### 1. Storage Machines

#### A. Disques et RAID

1. Ajoutez **6 disques** de 2 Go à `sto1` et `sto2`.
2. Configurez 3 RAID1 ici nommés `messi`, `neymar` et `suarez` :

![easy](https://media1.tenor.com/m/GYw66pbJHHsAAAAd/messichad-messi.gif)

   ```bash
   sudo mdadm --create /dev/md/messi --level=1 --raid-devices=2 /dev/sdb /dev/sdc
   sudo mdadm --create /dev/md/neymar --level=1 --raid-devices=2 /dev/sdd /dev/sde
   sudo mdadm --create /dev/md/suarez --level=1 --raid-devices=2 /dev/sdf /dev/sdg
   ```
3. Sauvegardez la configuration dans `/etc/mdadm/mdadm.conf` :
   ```bash
   sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
   ```
4. Mettez à jour l'initramfs :
   ```bash
   sudo update-initramfs -u
   ```
5. Vérifiez les RAID avec `lsblk` :
   ```plaintext
   NAME    MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
   sda       8:0    0    8G  0 disk
   md0       9:0    0    2G  0 raid1
   ...
   ```

#### B. Configuration des Targets iSCSI

1. Installez `targetcli-fb` (-fb pour Debian car targetcli pas reconnu sinon)  :
   ```bash
   sudo apt install -y targetcli-fb
   ```
2. Activez le service :
   ```bash
   sudo systemctl enable --now targetclid.service
   ```
3. Configurez les backstores et targets iSCSI :
   ```bash
   sudo targetcli
   /backstores/fileio create messi /dev/md/messi
   /iscsi create iqn.2024-12.tp2.b3:data-chunk1
   /iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/luns create /backstores/fileio/messi
   saveconfig
   exit
   ```
4. Répétez pour `neymar` et `suarez`.
5. Vérifiez avec `sudo targetcli ls`.

---

### 2. Chunk Machines

#### A. Configuration iSCSI

1. Installez les outils iSCSI :
   ```bash
   sudo apt install -y open-iscsi
   ```
2. Modifiez l'InitiatorName dans `/etc/iscsi/initiatorname.iscsi` :
   ```plaintext
   InitiatorName=iqn.2024-12.tp2.b3:chunk1-initiator
   ```
3. Découvrez les targets iSCSI :
   ```bash
   sudo iscsiadm -m discoverydb -t st -p 10.3.1.1:3260 --discover
   ```
4. Connectez-vous aux targets :
   ```bash
   sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk1 --login
   ```

#### B. Multipathing

1. Installez `multipath-tools` :

   ```bash
   sudo apt install -y device-mapper-multipath
   ```

2. Configurez `/etc/multipath.conf` :

3. Copiez le fichier d’exemple fourni dans le dossier de documentation :

   ```bash
   sudo cp /usr/share/doc/device-mapper-multipath/examples/multipath.conf /etc/multipath.conf
   ```

4. Modifiez la section `defaults` pour configurer les options suivantes :

5. Assurez-vous que le fichier `/etc/multipath.conf` contient les paramètres suivants :

   ```plaintext
   defaults {
       user_friendly_names yes
       find_multipaths yes
       path_grouping_policy failover
       features "1 queue_if_no_path"
       no_path_retry 100
       polling_interval 2
       fast_io_fail_tmo 3
   }
   ```

6. Rechargez la configuration multipath et redémarrez le service :
   ```bash
   sudo systemctl restart multipath-tools.service
   sudo multipath -r
   ```

7. Activez le service multipath :
   ```bash
   sudo systemctl enable --now multipath-tools.service
   ```

8. Vérifiez le multipathing :
   ```bash
   sudo multipath -ll
   ```

**Résultat attendu :**
```plaintext
mpatha (36001405245a1502fa7244cba904620eb) dm-2 LIO-ORG,chunk2
size=511M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdc 8:32 active ready running
```

---

### 3. Formatage et Montage

1. Formatez les volumes en `xfs` :
   ```bash
   sudo mkfs.xfs /dev/mapper/mpatha
   ```
2. Montez les volumes :
   ```bash
   sudo mkdir -p /mnt/data_chunk1
   sudo mount /dev/mapper/mpatha /mnt/data_chunk1
   ```
3. Ajoutez une unité systemd pour montage automatique.

---

### 4. Tests

1. Simulez une coupure réseau en coupant un lien.
2. Observez la bascule avec `multipath -ll` avant et après.

**Avant la panne :**

```plaintext
mpatha (36001405245a1502fa7244cba904620eb) dm-2 LIO-ORG,chunk2
size=511M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdc 8:32 active ready running
```

**Après la panne :**

```plaintext
mpatha (36001405245a1502fa7244cba904620eb) dm-2 LIO-ORG,chunk2
size=511M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=10 status=failed
  `- 5:0:0:0 sdc 8:32 failed faulty
```

---

### 5. Réplication

Répliquez le setup de `chunk1` sur `chunk2` et `chunk3`.

1. Configurez les targets et initiators iSCSI.
2. Configurez le multipathing.
3. Formatez et montez les volumes sur `/mnt/data_chunk1` et `/mnt/data_chunk2`.

---

**Preuve du setup**

- Sur `chunk1.tp2.b3`, `chunk2.tp2.b3` et `chunk3.tp2.b3`
- Les commandes suivantes avec leurs résultats :

**Chunk1 :**

```bash
$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda           8:0    0    8G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    7G  0 part
  ├─rl-root 253:0    0  6.2G  0 lvm   /
  └─rl-swap 253:1    0  820M  0 lvm   [SWAP]
sdb           8:16   0  511M  0 disk
└─mpathb    253:4    0  511M  0 mpath
  └─mpathb1 253:5    0  503M  0 part  /mnt/data_chunk2
sdc           8:32   0  511M  0 disk
└─mpatha    253:2    0  511M  0 mpath
  └─mpatha1 253:3    0  503M  0 part  /mnt/data_chunk1

$ df -h
Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/mpatha1  439M  149M  291M  34% /mnt/data_chunk1
/dev/mapper/mpathb1  439M  206M  234M  47% /mnt/data_chunk2

$ iscsiadm -m session -P 3
Target: iqn.2024-12.tp2.b3:data-chunk1
    Current Portal: 10.3.1.1:3260,1
    Persistent Portal: 10.3.1.1:3260,1
    Initiator Name: iqn.2024-12.tp2.b3:chunk1-initiator

$ multipath -ll
mpatha (36001405245a1502fa7244cba904620eb) dm-2 LIO-ORG,chunk1
size=511M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdc 8:32 active ready running
```

**Chunk2 :**

```bash
$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda           8:0    0    8G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    7G  0 part
  ├─rl-root 253:0    0  6.2G  0 lvm   /
  └─rl-swap 253:1    0  820M  0 lvm   [SWAP]
sdb           8:16   0  511M  0 disk
└─mpathb    253:4    0  511M  0 mpath
  └─mpathb1 253:5    0  503M  0 part  /mnt/data_chunk2
sdc           8:32   0  511M  0 disk
└─mpatha    253:2    0  511M  0 mpath
  └─mpatha1 253:3    0  503M  0 part  /mnt/data_chunk1

$ df -h
Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/mpatha1  439M  149M  291M  34% /mnt/data_chunk1
/dev/mapper/mpathb1  439M  206M  234M  47% /mnt/data_chunk2

$ iscsiadm -m session -P 3
Target: iqn.2024-12.tp2.b3:data-chunk2
    Current Portal: 10.3.1.2:3260,1
    Persistent Portal: 10.3.1.2:3260,1
    Initiator Name: iqn.2024-12.tp2.b3:chunk2-initiator

$ multipath -ll
mpatha (36001405245a1502fa7244cba904620eb) dm-2 LIO-ORG,chunk2
size=511M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdc 8:32 active ready running
```

**Chunk3 :**

```bash
$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda           8:0    0    8G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    7G  0 part
  ├─rl-root 253:0    0  6.2G  0 lvm   /
  └─rl-swap 253:1    0  820M  0 lvm   [SWAP]
sdb           8:16   0  511M  0 disk
└─mpathb    253:4    0  511M  0 mpath
  └─mpathb1 253:5    0  503M  0 part  /mnt/data_chunk2
sdc           8:32   0  511M  0 disk
└─mpatha    253:2    0  511M  0 mpath
  └─mpatha1 253:3    0  503M  0 part  /mnt/data_chunk1

$ df -h
Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/mpatha1  439M  149M  291M  34% /mnt/data_chunk1
/dev/mapper/mpathb1  439M  206M  234M  47% /mnt/data_chunk2

$ iscsiadm -m session -P 3
Target: iqn.2024-12.tp2.b3:data-chunk3
    Current Portal: 10.3.1.3:3260,1
    Persistent Portal: 10.3.1.3:3260,1
    Initiator Name: iqn.2024-12.tp2.b3:chunk3-initiator

$ multipath -ll
mpatha (36001405245a1502fa7244cba904620eb) dm-2 LIO-ORG,chunk3
size=511M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdc 8:32 active ready running
```
![easy](https://media1.tenor.com/m/qXkUtlnTNI0AAAAd/rain.gif)


# Partie III : Système de Fichiers Distribué

## Introduction

Nous allons étendre notre infrastructure existante (SAN) en ajoutant un système de fichiers distribé avec **MooseFS**. Ce dernier ajoute un niveau supplémentaire de redondance, en répliquant les fichiers entre plusieurs serveurs chunk. Voici la structure finale :

- **Moose Master Server** : `master.tp2.b3`
  - Contient les métadonnées et expose les partitions réseau.
  - Fournit une WebUI pour la gestion (accessible sur `http://<master>:9425`).
- **Moose Chunk Servers** : `chunk1.tp2.b3`, `chunk2.tp2.b3`, `chunk3.tp2.b3`
  - Gèrent le stockage des données répliquées.
  - Proposent deux volumes : `/mnt/data_chunk1` et `/mnt/data_chunk2`.
- **Web Server** : `web.tp2.b3`
  - Consomme les partitions distribuées, à l'aide d'un serveur **NGINX**.

---

## 1. Moose Master Server

### Installation et Configuration

> ***Sur ******`master.tp2.b3`****** uniquement.***

1. **Installer MooseFS Master et la WebUI**

   Ajoutez le dépôt et installez les paquets :

   ```bash
   wget -O /etc/yum.repos.d/MooseFS-3-el9.repo https://ppa.moosefs.com/MooseFS-3-el9.repo
   sudo dnf install moosefs-master moosefs-cgi -y
   ```

2. **Démarrer les services**

   Activez et lancez les services requis :

   ```bash
   sudo systemctl enable --now moosefs-master
   sudo systemctl enable --now moosefs-cgi
   ```

3. **Configurer le firewall**

   Ouvrez les ports requis pour MooseFS :

   ```bash
   sudo ufw allow 80/tcp
   sudo ufw allow 9425/tcp
   sudo ufw allow 9419/tcp
   sudo ufw allow 9420/tcp
   sudo ufw allow 9421/tcp
   sudo ufw reload
   ```

4. **Accéder à la WebUI**

   La WebUI est disponible sur :

   ```
   http://<IP_MASTER>:9425
   ```

   Elle permet de superviser les chunks et le stockage.

---

## 2. Moose Chunk Servers

> ***Sur ******`chunk1.tp2.b3`******, ******`chunk2.tp2.b3`****** et ******`chunk3.tp2.b3`******.***

### Installation et Configuration

1. **Installer MooseFS Chunk Server**

   Ajoutez le dépôt et installez les paquets :

   ```bash
   wget -O /etc/yum.repos.d/MooseFS-3-el9.repo https://ppa.moosefs.com/MooseFS-3-el9.repo
   sudo dnf install moosefs-chunkserver -y
   ```

2. **Configurer le Chunk Server**

   Modifiez le fichier `/etc/mfs/mfschunkserver.cfg` pour indiquer l'adresse du serveur master :

   ```bash
   MASTER_HOST = master.tp2.b3
   ```

3. **Préparer les partitions**

   Attribuez les partitions à l'utilisateur `mfs` :

   ```bash
   sudo chown -R mfs:mfs /mnt/data_chunk1
   sudo chown -R mfs:mfs /mnt/data_chunk2
   ```

   Ajoutez les partitions dans `/etc/mfs/mfshdd.cfg` :

   ```bash
   /mnt/data_chunk1
   /mnt/data_chunk2
   ```

4. **Démarrer le service Chunk Server**

   Activez et lancez le service :

   ```bash
   sudo systemctl enable --now moosefs-chunkserver
   ```

5. **Vérification**

   Sur la WebUI de `master.tp2.b3`, les trois noeuds Chunk doivent apparaître comme actifs, avec les volumes disponibles.

---

## 3. Machine Consommatrice : Web Server

> ***Sur ******`web.tp2.b3`****** uniquement.***

### Installation et Configuration

1. **Installer MooseFS Client**

   Ajoutez le dépôt et installez les paquets :

   ```bash
   wget -O /etc/yum.repos.d/MooseFS-3-el9.repo https://ppa.moosefs.com/MooseFS-3-el9.repo
   sudo dnf install moosefs-client -y
   ```

2. **Monter la partition MooseFS**

   Créez un point de montage et montez la partition :

   ```bash
   sudo mkdir -p /mnt/www
   sudo mfsmount /mnt/www -H master.tp2.b3
   ```

   Vérifiez avec `lsblk` et assurez-vous que la partition est bien montée.

3. **Installer NGINX**

   Installez le serveur web et ouvrez le port 80 :

   ```bash
   sudo dnf install nginx -y
   sudo ufw allow 80/tcp
   ```

4. **Configurer le serveur web**

   Créez un fichier `index.html` dans `/mnt/www` :

   ```bash
   echo "Hello, MooseFS!" | sudo tee /mnt/www/index.html
   ```

   Créez une configuration minimale pour NGINX :

   ```bash
   sudo nano /etc/nginx/conf.d/default.conf
   ```

   ```nginx
   server {
       listen 80;
       server_name _;
       root /mnt/www;
       index index.html;
   }
   ```

5. **Démarrer et tester le service**

   Lancez le service et vérifiez :

   ```bash
   sudo systemctl enable --now nginx
   curl http://localhost
   ```

   On voit le message : `Hello, MooseFS!`

   Pour prouver que le site est actif depuis une autre machine, utilisez la commande suivante :

```bash
curl http://web.tp2.b3
```

Vous on voit aussi la réponse :

```html
Hello, MooseFS!
```
![easy](https://media1.tenor.com/m/nB37dEyQ_H4AAAAC/hey-cat.gif)






