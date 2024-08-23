# Mise en place d'un RAID 1 sur un serveur Debian sur Proxmox

Cette documentation explique comment mettre en place un RAID 1 sur un serveur Proxmox, en partant d'un volume système existant de 30 Go, en ajoutant deux nouveaux disques de 15 Go chacun, puis en transférant les données du disque d'origine vers le nouveau volume en RAID 1. L'idée derrière l'ajout de disques de 15 Go au lieu de 30 Go est de maximiser l'efficacité du stockage, car les données présentes sur le disque de 30 Go n'occupent qu'un peu moins de 10 Go d'espace.

## 1. Configuration initiale du serveur

La capture d'écran suivante montre la configuration matérielle initiale du serveur virtualisé sur Proxmox:

![Avantraid1](https://github.com/user-attachments/assets/b5b57f32-0d0a-4c53-8b09-7a720b7503b4)


### Détails de la configuration:

- **Mémoire**: 1 Go / 2 Go
- **Processeur**: 1 CPU (1 socket, 1 core)
- **Disques**:
  - Disque système actuel : `vm-753-disk-0` (30 Go)
  - Deux nouveaux disques ajoutés :
    - `vm-753-disk-3` (15 Go)
    - `vm-753-disk-4` (15 Go)
  
## 2. Objectif

L'objectif est de configurer un RAID 1 sur les deux nouveaux disques de 15 Go afin de remplacer le disque système existant de 30 Go tout en conservant les données existantes. Les disques de 15 Go ont été choisis car les données actuelles sur le disque de 30 Go occupent moins de 10 Go, permettant ainsi de faire des économies d'espace disque tout en assurant la redondance.

## 3. Étapes de mise en œuvre

### 3.1 Ajout des nouveaux disques

1. Dans l'interface de gestion Proxmox, sélectionnez la machine virtuelle souhaitée.
2. Ajoutez deux disques supplémentaires de 15 Go chacun. Ceux-ci apparaîtront comme `vm-753-disk-3` et `vm-753-disk-4` dans la configuration matérielle.


![Cap1_add2hdd](https://github.com/user-attachments/assets/f9316447-1ab2-4013-8c36-2de6e8366abb)

### 3.2 Vérification des disques dans le système

Une fois les nouveaux disques ajoutés à la machine virtuelle via Proxmox, il est nécessaire de vérifier qu'ils sont bien reconnus par le système d'exploitation de la VM.

1. **Vérification de la présence des disques** :
Connectez-vous à la machine virtuelle via SSH ou en utilisant la console Proxmox, puis utilisez la commande suivante pour lister tous les périphériques de bloc disponibles, c'est-à-dire les disques et leurs partitions :

  ```bash
  lsblk
  ```
La commande lsblk affiche une liste des périphériques de bloc (comme les disques durs) sur le système, montrant leur taille, leur type, et les points de montage s'ils sont montés. Cette commande est utile pour visualiser la structure des disques et des partitions sans avoir à les monter.

![Cap2_lsblk](https://github.com/user-attachments/assets/0efec1bd-897c-40b4-b669-ceaf25de4cfc)

Comme le montre cette capture, les nouveaux disques sda et sdb de 15 Go sont bien détectés par le système. Vous pouvez également voir le disque sdc de 32 Go, qui est le disque d'origine, ainsi que les partitions existantes.   

2. **Partitionnement des nouveaux disques** :

Après avoir confirmé que les disques sont reconnus, nous allons les partitionner en utilisant fdisk.

### 3.3 : Partitionnement des nouveaux disques
### 3.3 Recréer la même structure de partitionnement que `sdc` sur les disques `sda` et `sdb`

#### 3.3.1 Création des partitions sur le disque `sda`

1. **Créer la partition principale (`sda1`)** :
   - Lancez `fdisk` :
     ```bash
     fdisk /dev/sda
     ```
   - Tapez `n` pour créer une nouvelle partition.
   - Sélectionnez `p` pour créer une partition primaire.
   - Choisissez `1` pour le numéro de partition.
   - Appuyez sur `ENTER` pour accepter le secteur de début par défaut.
   - Entrez la taille souhaitée (par exemple, `+13G` pour correspondre à `sdc1`).
   
2. **Créer la partition étendue (`sda2`)** :
   - Dans `fdisk`, tapez `n` pour créer une nouvelle partition.
   - Sélectionnez `e` pour créer une partition étendue.
   - Choisissez `2` pour le numéro de partition.
   - Acceptez les valeurs par défaut pour le secteur de début.
   - Entrez `+975M` pour la taille de la partition
  
3. **Créer la partition logique pour le swap (`sda5`)** :
   - Dans `fdisk`, tapez `n` pour créer une nouvelle partition.
   - Sélectionnez `l` pour créer une partition logique à l'intérieur de la partition étendue.
   - Acceptez les valeurs par défaut pour le secteur de début.
   - .Acceptez les valeurs par défaut pour occuper tout l'espace restant.
   - Tapez `t`, choisissez `5`, puis entrez `82` pour définir la partition comme Linux swap.
   - Sauvegardez les modifications en tapant `w`.

Capture d'écran du partitionnement de sda :

![1a_partition](https://github.com/user-attachments/assets/eb1b8e76-a017-4d6b-b6e5-e17a0a46c18b)

Oups, j'ai oublié de mettre un type de partition swap a sda5. Pas de soucis, il suffit de retourner dans fdisk et faire les changements voulus comme ci-dessous :

![3_modifsda5](https://github.com/user-attachments/assets/a2a35a08-8404-4cea-9022-880501e4fe5b)

##### Changer le type de partition pour `sda5`

1. **Changer le type de partition pour `sda5`** :
   - Tapez `t` pour changer le type d'une partition.
   - Entrez `5` pour sélectionner la partition logique `sda5`.
   - Entrez `82` pour définir le type de partition sur "Linux swap / Solaris".

2. **Vérifier les modifications** :
   - Tapez `p` pour afficher la table de partitions et vérifier que `sda5` est bien configurée en type `82`.

3. **Écrire les modifications sur le disque** :
   - Tapez `w` pour écrire les modifications sur le disque et quitter `fdisk`.


#### 3.3.2 Répéter la même procédure pour le disque `sdb`

- Répétez les mêmes étapes pour le disque `sdb` afin de recréer les partitions `sdb1`, `sdb2`, et `sdb5`.

![4_patsdb](https://github.com/user-attachments/assets/792e91c2-066a-4cbf-84e6-1abf561ec650)

**Vérifier la structure de chaque disque** :
   - Utilisez la commande `fdisk -l` pour vérifier la nouvelle structure de partitions.

![5_fdisk_l](https://github.com/user-attachments/assets/f99e9eae-08bd-47c9-b252-ef3b55a8f5d5)

Comme vous le voyez, les disques sda et sdb ont maintenant la même structure de partitions que le disque sdc, ce qui est parfait pour la prochaine étape de la configuration du RAID 1.

### 4. Création du RAID 1

#### 4.1 Création du RAID 1 pour la partition principale

1. **Création du RAID 1** :
   - Utilisez `mdadm` pour créer le RAID 1 sur les partitions principales `sda1` et `sdb1` :
     ```bash
     mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1
     ```
2. **Vérification du RAID** :
   - Vérifiez que le RAID 1 est en cours de synchronisation en consultant l'état avec :
     ```bash
     cat /proc/mdstat
     ```
Voici la capture d'écran des deux étapes précédentes :

![6_raidmd0](https://github.com/user-attachments/assets/a43349d0-6bf3-4d90-8ef1-3180395b80cf)

#### Explication de la sortie des commandes `mdadm` et `cat /proc/mdstat`

1. **Commande `mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1`** :
   - `mdadm` vous informe que `sda1` et `sdb1` contiennent déjà des systèmes de fichiers `ext2/ext3/ext4` et que ces partitions pourraient ne pas être adaptées comme périphérique de démarrage à cause de la métadonnée RAID.
   - Après confirmation (`Y`), le RAID 1 (`/dev/md0`) est créé avec succès, utilisant `sda1` et `sdb1`.
   - Le message indique également que le RAID utilise la version 1.2 des métadonnées, qui est placée au début du disque.

2. **Commande `cat /proc/mdstat`** :
   - Cette commande affiche l'état actuel de votre ensemble RAID.
   - La sortie montre que le RAID 1 (`md0`) est actif, avec les deux disques `sda1` et `sdb1` synchronisés. Le processus de synchronisation est en cours, avec 48,4 % déjà terminé.
   - La vitesse de synchronisation est de 206300K/sec, et il reste environ 0,5 minute pour terminer la synchronisation.

3. **Deuxième commande `cat /proc/mdstat` (après synchronisation)** :
   - La deuxième exécution de `cat /proc/mdstat` montre que la synchronisation est maintenant terminée.
   - Le RAID 1 (`md0`) est actif avec les deux disques (`sda1` et `sdb1`) parfaitement synchronisés et fonctionnant normalement. Le statut `[UU]` indique que les deux disques sont en bon état et complètement opérationnels.

#### 4.2 Création du RAID 1 pour le swap

1. **Création du RAID 1 pour le swap** :
   - Utilisez `mdadm` pour créer un RAID 1 sur les partitions de swap `sda5` et `sdb5` :
     ```bash
     mdadm --create --verbose /dev/md1 --level=1 --raid-devices=2 /dev/sda5 /dev/sdb5
     ```

2. **Vérification du RAID pour le swap** :
   - Vérifiez que le RAID 1 pour le swap est également en cours de synchronisation :
     ```bash
     cat /proc/mdstat
     ```

Voici la capture d'écran des deux étapes précédentes :

![7_raidmd1](https://github.com/user-attachments/assets/99a98116-1f5d-433a-abe7-f67fb321255b)

#### 4.3 Configuration du fichier `/etc/mdadm/mdadm.conf`

1. **Ajouter la configuration du RAID** :
   - Ajoutez les détails de la configuration RAID à `/etc/mdadm/mdadm.conf` pour qu'il soit reconnu au démarrage :
     ```bash
     mdadm --detail --scan >> /etc/mdadm/mdadm.conf
     ```

2. **Mise à jour initramfs** :
   - Mettez à jour initramfs pour inclure les nouvelles configurations RAID :
     ```bash
     update-initramfs -u
     ```
La première commande ajoute la configuration RAID actuelle au fichier de configuration de mdadm, garantissant que le RAID sera reconnu lors du démarrage.
La seconde commande met à jour le système initial (initramfs) pour s'assurer que ces nouvelles configurations RAID sont prises en compte dès le démarrage suivant.

Voici la sortie indiquée :

 ```bash
root@SRV-DEB:/#  mdadm --detail --scan >> /etc/mdadm/mdadm.conf
root@SRV-DEB:/#  update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.1.0-23-amd64
root@SRV-DEB:/#
 ```
### 4. Formater les partitions RAID

#### 4.1 Formater la partition RAID principale

1. **Formater en ext4** :
   - Formatez la partition RAID principale (`/dev/md0`) en ext4 :
     ```bash
     mkfs.ext4 /dev/md0 -L "sysraid"
     ```

2. **Vérification du formatage** :
   - Vérifiez la partition pour s'assurer qu'elle est bien formatée en ext4 :
     ```bash
     blkid /dev/md0
     ```

#### 4.2 Configurer et activer le swap sur le RAID

1. **Formater la partition RAID swap (`/dev/md1`)** :
   - Configurez le swap sur la partition RAID :
     ```bash
     mkswap /dev/md1
     ```

2. **Activer le swap** :
   - Activez le swap immédiatement :
     ```bash
     swapon /dev/md1
     ```

3. **Vérification du swap** :
   - Vérifiez que le swap est actif :
     ```bash
     swapon --show
     ```
La capture d'écran suivante montre les étapes 4:

![8_formatage](https://github.com/user-attachments/assets/3039f310-43e8-40b7-99a6-043ed8d6c4b3)

4. **Vérification de la structure des disques** :
   - Vérifiez que le swap est actif :
     ```bash
     lsblk -f
     ```
![apres formatage](https://github.com/user-attachments/assets/fda3d552-d1fa-4585-89a0-3ab174626784)


### 5. Verrouillage du nom de l'ensemble RAID et configuration avec l'UUID

#### 5.1 Création de l'ensemble RAID avec un nom spécifique

Lors de la création de l'ensemble RAID, il est recommandé d'assigner un nom pour éviter les conflits de nommage au démarrage.

1. **Création du RAID 1 avec un nom spécifique** :
   - Utilisez la commande suivante pour créer le RAID 1 tout en assignant un nom :
     ```bash
     mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1 --name=sysraid
     ```

#### 5.2 Verrouillage du nom de l'ensemble RAID avec l'UUID

Pour assurer que le nom de l'ensemble RAID (`/dev/md0`) reste constant, nous devons l'associer à son UUID dans le fichier `/etc/mdadm/mdadm.conf`.

1. **Trouver l'UUID de l'ensemble RAID** :
   - Exécutez la commande suivante pour obtenir l'UUID de l'ensemble RAID :
     ```bash
     mdadm --detail --scan
     ```

2. **Ajouter l'UUID à `/etc/mdadm/mdadm.conf`** :
   - Ajoutez la sortie de la commande précédente dans le fichier `/etc/mdadm/mdadm.conf` :
     ```bash
     mdadm --detail --scan >> /etc/mdadm/mdadm.conf
     ```
     
En vérifiant le fichier conf (`cat /etc/mdadm/mdadm.conf`), j'ai constaté une erreur de doublon, j'ai donc modifié le fichier conf afin de marquer le md0 en trop en commentaire comme ci-dessous avec la commande :

```bash
nano /etc/mdadm/mdadm.conf:
 ```

![correctionconf](https://github.com/user-attachments/assets/9b0e09e7-4486-4dba-ac69-d7264f4c84b6)

3. **Mettre à jour initramfs** :
   - Mettez à jour initramfs pour s'assurer que les modifications sont prises en compte au démarrage :
     
     ```bash
     update-initramfs -u
     ```
Voici la sortie indiquant que la mise à jour s'est bien déroulée :

```bash
update-initramfs: Generating /boot/initrd.img-6.1.0-23-amd64
```

#### 5.3 Utilisation de l'UUID dans `/etc/fstab`

Pour garantir que les partitions RAID soient montées correctement au démarrage, utilisez l'UUID dans le fichier `/etc/fstab` au lieu du nom de périphérique (`/dev/md0`).

1. **Trouver l'UUID de la partition RAID** :
   - Utilisez `blkid` pour trouver l'UUID de la partition RAID principale (`/dev/md0`) :
     ```bash
     blkid /dev/md0
     ```

2. **Modifier `/etc/fstab` pour utiliser l'UUID** :
   - Remplacez la ligne correspondante dans `/etc/fstab` par l'UUID :
     ```bash
     UUID=79dcf1ad:b0244cd7:cf2aced4:f0e99d3b / ext4 nofail 0 1
     ```
En géneral, on met default, mais ici on met nofail pour éviter d'être bloqué au démarrage de la machine.

   - Faites de même pour la partition de swap, en utilisant l'UUID obtenu avec `blkid` pour `/dev/md1` :
     ```bash
     UUID=abcdef01-2345-6789-abcd-ef0123456789 none swap sw 0 0
     ```

Avec cette configuration, le système va vérifier l'association entre le nom de l'ensemble RAID et son UUID dans `/etc/mdadm/mdadm.conf`, garantissant ainsi un montage stable au démarrage.

### 6. Copie des données du disque d'origine vers le RAID

#### 6.1 Monter la partition RAID

1. **Créer un point de montage** :
   - Créez un répertoire temporaire pour monter la partition RAID :
     ```bash
     mkdir /mnt/md0
     ```

2. **Monter la partition RAID** :
   - Montez la partition RAID sur ce répertoire :
     ```bash
     mount /dev/md0 /mnt/md0
     ```

#### 6.2 Copier les données

1. **Utiliser `rsync` pour copier les données** :
   - Utilisez `rsync` pour copier les données de l'ancienne partition (`/dev/sdc1`) vers la nouvelle partition RAID (`/mnt/md0`) :
   - 
     ```bash
     rsync -aAXv / /mnt/md0 --exclude=/mnt/md0 --exclude=/proc --exclude=/tmp --exclude=/sys --exclude=/dev --exclude=/run
     ```

2. **Vérification de la copie** :
   - Une fois la copie terminée, vérifiez que toutes les données ont été correctement transférées en naviguant dans le répertoire `/mnt/md0`.

### 7. Vérification post-redémarrage

Après avoir redémarré le système, il est crucial de vérifier que toutes les configurations ont été appliquées correctement et que le RAID fonctionne comme prévu.

#### 7.1 Vérification des montages

1. **Vérifier que la partition root est montée depuis le RAID** :
   - Utilisez la commande suivante pour vérifier le point de montage de la partition root :
     ```bash
     mount | grep "on / type"
     ```
   - Vous devriez voir que la partition root est montée depuis `/dev/md0` ou son UUID, ce qui confirme que le RAID est utilisé comme partition principale.

2. **Vérifier l'état du swap** :
   - Vérifiez que la partition de swap est active :
     ```bash
     swapon --show
     ```
   - La sortie devrait indiquer que le swap est monté depuis `/dev/md1` ou son UUID, ce qui confirme que le RAID est utilisé pour le swap.

#### 7.2 Vérification de l'état du RAID

1. **Vérifier l'état du RAID** :
   - Utilisez la commande suivante pour vérifier l'état de tous les ensembles RAID :
     ```bash
     cat /proc/mdstat
     ```
   - Assurez-vous que tous les disques du RAID sont en état `UU`, ce qui signifie que les deux disques sont actifs et synchronisés. Cela garantit que le RAID 1 fonctionne correctement.

#### 7.3 (Optionnel) Réinstallation de GRUB

Si ce n'est pas déjà fait, vous pourriez vouloir réinstaller GRUB sur les disques RAID pour assurer un démarrage sécurisé.

1. **Réinstaller GRUB** :
   - Réinstallez GRUB sur les deux disques du RAID :
     ```bash
     grub-install /dev/sda
     grub-install /dev/sdb
     ```
   - Mettez à jour GRUB pour inclure les nouvelles configurations :
     ```bash
     update-grub
     ```

### 8. Conclusion et récapitulatif

Le RAID 1 a été configuré avec succès sur le serveur Debian hébergé sur Proxmox. Les données ont été copiées du disque d'origine vers le RAID, et le système a redémarré avec les partitions RAID configurées pour être utilisées comme partition root et pour le swap.

Les vérifications post-redémarrage ont confirmé que les montages sont corrects et que le RAID est en bon état. Cette documentation détaille chaque étape du processus, y compris la résolution des problèmes rencontrés et les vérifications finales.

### Capture d'écran finale :

- **Montage de la partition root depuis le RAID** : La sortie de `mount | grep "on / type"` doit montrer que `/` est monté depuis `/dev/md0`.
- **État du swap** : La sortie de `swapon --show` doit indiquer que le swap est actif sur `/dev/md1`.
- **État du RAID** : La sortie de `cat /proc/mdstat` doit montrer que le RAID est en état `UU`, indiquant que les disques sont synchronisés.

Ces captures d'écran servent de preuve que le système est correctement configuré et fonctionnel après la migration vers le RAID 1.

voici deux captures une avant de réinstaller grub et une après :

![12_verif](https://github.com/user-attachments/assets/9dd37234-1cc9-4edf-bfe4-43f7333f648c)

![13_verifok](https://github.com/user-attachments/assets/53b8f560-cb17-4dfb-b2f9-bf8c921738a1)

### 8. Conclusion et récapitulatif

Le RAID 1 a été configuré avec succès sur le serveur Debian hébergé sur Proxmox. Les données ont été copiées du disque d'origine vers le RAID, et le système a redémarré avec les partitions RAID configurées pour être utilisées comme partition root et pour le swap.

Les vérifications post-redémarrage ont confirmé que les montages sont corrects et que le RAID est en bon état. Cette documentation détaille chaque étape du processus, y compris la résolution des problèmes rencontrés et les vérifications finales.

### Capture d'écran finale :

- **Montage de la partition root depuis le RAID** : La sortie de `mount | grep "on / type"` montre que `/` est monté depuis `/dev/md0`.
- **État du swap** : La sortie de `swapon --show` indique que le swap est actif sur `/dev/md1`.
- **État du RAID** : La sortie de `cat /proc/mdstat` montre que le RAID est en état `UU`, indiquant que les disques sont synchronisés.

Les captures d'écran servent de preuve que le système est correctement configuré et fonctionnel après la migration vers le RAID 1.
