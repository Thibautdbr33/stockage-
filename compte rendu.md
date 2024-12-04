# TP1 : Single-machine storage Réponse

# 0. Prérequis

➜ **une VM  Debian 12 <span style="color: red;">sans interface graphique</span>**

![problem](https://media.tenor.com/LoNa2zOMxoAAAAAM/its-very-important-it-matters.gif)

# I. Fundamentals

🌞 **Lister tous les périphériques de stockage branchés à la VM**

- `lsblk -d`

🌞 **Lister toutes les partitions des périphériques de stockage**

- `lsblk -f`

🌞 **Effectuer un test SMART sur le disque**

- Installer l'outil smartmontools
- Testez la santé du disque principal /dev/sda : `smartctl -H /dev/sda`

- Affichez des informations détaillées sur le disque : `sudo smartctl -a /dev/sda`

- Malheureusement , les commandes ci-dessus ne fonctionnent pas sur les disques virtuel de virtual box

![problem](https://media1.tenor.com/m/VpALCb0oXrQAAAAd/oss117-alerte-rouge-en-afrique-noire.gif)

🌞 **Espace disque : afficher l'espace disque restant sur la partition `/`**

- `df -h /`

🌞 **Inodes**

- afficher la quantité d'inodes restants: `df -i /`

🌞 **Latence disque**

- Installer ioping 

- utilisez `ioping` pour déterminer la latence disque :  `ioping -c 10 -d 1G /dev/sda1`

🌞 **Déterminer la taille du cache *filesystem***

- `free -h`

# II. Partitioning

Ici on utilise un des disques supplémentaires branchés à la VM : `sdb`.


🌞 **Ajouter `sdb` comme Physical Volume LVM**

- Créer un volume physique (PV) sur le disque sdb : `sudo pvcreate /dev/sdb`

🌞 **Créer un Volume Group LVM nommé `storage`**

- `sudo vgcreate storage /dev/sdb`

🌞 **Créer un Logical Volume LVM**

- dans le VG `storage`
- nommé `smol_data`
- sa taille : 2G
- La commande est : `sudo lvcreate -L 2G -n smol_data storage`

🌞 **Créer un deuxième Logical Volume LVM**

- dans le VG `storage`
- nommé `big_data`
- sa taille : tout le reste du VG
- La commande est : `sudo lvcreate -l 100%FREE -n big_data storage`

🌞 **Créez un système de fichiers sur les deux LVs**

 

`sudo mkfs.ext4 /dev/storage/smol_data`

   `sudo mkfs.ext4 /dev/storage/big_data`

🌞 **Montez la partition `smol_data`**

- Création du point de montage : `sudo mkdir -p /mnt/lvm_storage/smol`
- Montage des volumes : `sudo mount /dev/storage/smol_data /mnt/lvm_storage/smol`

🌞 **Montez la partition `big_data`**

- Création du point de montage : `sudo mkdir -p /mnt/lvm_storage/big`
- Montage des volumes : `sudo mount /dev/storage/big_data /mnt/lvm_storage/big`

🌞 **Configurer un *automount***

- Il n'était pas possible de créer l'automount avec un Debian 12 avec interface de bureau sans raisons apparentes donc j'ai refait le TP avec une machine en sans interface graphique.

- Ajoutez dans /etc/fstab les lignes suivantes :
- `/dev/mapper/storage-smol_data /mnt/lvm_storage/smol ext4 defaults 0 0`
- `/dev/mapper/storage-big_data /mnt/lvm_storage/big ext4 defaults 0 0`


# III. RAID

## 1. Simple RAID

🌞 **Mettre en place un RAID 5**

- avec TROIS disques parmis les disques supplémentaires branchés à la VM : `sdc` `sdd` et `sde`
- en utilisant `mdadm`
- La commande est : `sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdc /dev/sdd /dev/sde`

🌞 **Prouvez que le RAID5 est en place**:

- `mdadm --detail /dev/md0`

🌞 **Rendre la configuration automatique au boot de la machine**

- `sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf`

- `sudo update-initramfs -u` 

🌞 **Créez un système de fichiers sur la partition proposé par le RAID**

- Formatez le RAID en ext4 : `sudo mkfs.ext4 /dev/md0`

🌞 **Monter la partition sur `/mnt/raid_storage`**

- Création du point de montage : `sudo mkdir -p /mnt/raid_storage`
- Montage des volumes : `sudo mount /dev/md0 /mnt/raid_storage`
- Vérification que la partition est bien monté : `df -h /mnt/raid_storage`
🌞 **Prouvez que...**

- la partition est bien montée : `lsblk`
- il y a bien l'espace disponible attendue sur la partition : `df -h /mnt/raid_storage`
- vous pouvez lire et écrire sur la partition : `sudo touch /mnt/raid_storage/testfile`, `ls /mnt/raid_storage` 

> *Alors combien de Go dispo avec un RAID5 sur des disques de 10G ?* `20 Go disponibles`

🌞 **Mini benchmark**

- Teste de la vitesse d’écriture sur /mnt/raid_storage : `dd if=/dev/zero of=/mnt/raid_storage/testfile bs=1G count=1 oflag=direct`
- Comparaison avec la partition LVM montée sur /mnt/lvm_storage/smol : `dd if=/dev/zero of=/mnt/lvm_storage/smol/testfile bs=1G count=1 oflag=direct`

## 2. Break it

🌞 **Simule une panne**

- Marquez /dev/sdd comme défaillant : `sudo mdadm /dev/md0 --fail /dev/sdd`
- Retirez le disque défaillant du RAID : `sudo mdadm /dev/md0 --remove /dev/sdd`

🌞 **Montre l'état du RAID dégradé**

- avec une commande, on doit voir que le RAID est dégradé : `cat /proc/mdstat`
- On peut afficher les détails avec `sudo mdadm --detail /dev/md0`

🌞 **Remonte le disque dur**

- On Réintégre le disque dans le RAID : `sudo mdadm /dev/md0 --add /dev/sdd`

## 3. Spare disk

On ajoute le disque /dev/sdf au RAID en tant que disque spare :
- `sudo mdadm /dev/md0 --add /dev/sdf` 

🌞 **Ajoutez un disque encore inutilisé au RAID5 comme disque de *spare***

- On vérifie ensuite que le disque spare est bien ajouté : `sudo mdadm --detail /dev/md0`

🌞 **Simuler une panne**

- Marquez un disque actif comme défaillant (par exemple, /dev/sdd) : `sudo mdadm /dev/md0 --fail /dev/sdd`
- Vérifier que le disque spare a remplacé le disque défaillant : `cat /proc/mdstat`

🌞 **Remonter le disque débranché**

- Réintégrer le disque défaillant dans le RAID : `sudo mdadm /dev/md0 --add /dev/sdd`

## 4. Grow

🌞 **Ajoutez un disque encore inutilisé au RAID5 comme disque de *spare***

- Ajout d'un deuxième disque spare (par exemple, /dev/sdg) : `sudo mdadm /dev/md0 --add /dev/sdg`
- Vérifications que les deux disques spare sont bien ajoutés : `sudo mdadm --detail /dev/md0`

🌞 **Grow !**

- Convertion d'un disque spare en disque actif : `sudo mdadm --grow /dev/md0 --raid-devices=4`

🌞 **Prouvez que le RAID5 propose désormais 4 disques actifs**

- alors combien d'espace sur un RAID5 de 4 disques ? `30 Go disponible`

🌞 **Euuuh wait a sec... `/mnt/raid_storage` ???**

- Agrandissement du système de fichiers pour utiliser l’espace supplémentaire : `sudo resize2fs /dev/md0`

- Vérification que le système de fichiers a bien été agrandi : `df -h /mnt/raid_storage`

# IV. NFS

Enfin, on clôt le TP avec un **partage réseau simple à setup et relativement efficace : NFS.** Il est assez répandu dans le monde opensource pour des charges faibles.

C'est **un modèle de client/serveur** : on installe un serveur NFS sur une machine, on le configure pour indiquer quel(s) dossier(s) on veut partager sur le réseau, et d'autres machines peuvent s'y connecter pour accéder à ces dossiers.

🌞 **Installer un serveur NFS**
- Installation du server NFS : `sudo apt-get install nfs-server`

- Ajouter les lignes suivantes dans le fichier de conf `/etc/exports` pour partager les points de montages :
`/mnt/raid_storage *(rw,sync)`, `/mnt/lvm_storage *(rw,sync)`

- Application de la configuration : `sudo exportfs -arv`

    - a : Exporte tous les systèmes de fichiers.
    - r : Réexporte tous les répertoires.
    - v : Affiche les informations des exports.

- Vérification des exports: `sudo exportfs -v`

- Lancement du service NFS : `sudo systemctl start nfs-server`

- Il faut maintenant Configurer le firwall pour le partage NFS ne soit actif que sur mon réseau (192.168.248.0):

    - `sudo ufw allow from 192.168.248.0/24 to any port 2049 proto tcp`

    - `sudo ufw allow from 192.168.248.0/24 to any port 2049 proto udp`

![problem](https://media1.tenor.com/m/XC8c8MZFXVQAAAAd/we-have-a-problem-olsen.gif)

- j'ai n'ai pas réussi à monter les partages NFS sur mon client car il manque aussi le port 111 aussi utilisé par NFS:

   - `sudo ufw allow from 192.168.248.0/24 to any port 111 proto tcp`

   - `sudo ufw allow from 192.168.248.0/24 to any port 111 proto udp`

- Vérification du statut du firewall : `sudo ufw status`

🌞 **Configurer le client NFS**

- installer un client NFS pour se connecter au serveur : `sudo apt install nfs-common`

- Créer les points de montage pour le partage NFS :
  - `sudo mkdir -p /mnt/raid_storage`

  - `sudo mkdir -p /mnt/lvm_storage`

- Montez le partage NFS pour /mnt/raid_storage et /mnt/lvm_storage (en utilisant l'ip du serveur NFS):

     - `sudo mount 192.168.248.10:/mnt/raid_storage /mnt/raid_storage`

     - `sudo mount 192.168.248.10:/mnt/raid_storage /mnt/lvm_storage`

- Vérifier que les partages sont bien montés : 
    - `df -h`
    - `ls /mnt/raid_storage`
    - `ls /mnt/lvm_storage`

- Rendre persistant les montages des disques en les lignes ci-dessous dans /etc/fstab:

    - `192.168.248.10:/mnt/raid_storage /mnt/raid_storage nfs defaults 0 0`

    - `192.168.248.10:/mnt/lvm_storage /mnt/lvm_storage nfs defaults 0 0`

- Recharger les montages : `sudo mount -a`

🌞 **Benchmarkz**

- faites un test de vitesse d'écriture sur la partition `mdadm` montée en NFS : `/mnt/raid_storage`:
    
    - `dd if=/dev/zero of=/mnt/raid_storage/testfile bs=1G count=1 oflag=direct`

- faites un test de vitesse d'écriture sur la partition LVM montée en NFS : `/mnt/lvm_storage`

    - `dd if=/dev/zero of=/mnt/lvm_storage/testfile bs=1G count=1 oflag=direct`


![bobercurva](https://media.tenor.com/fF4sTbrZvnsAAAAM/bober-kurwa.gif)

