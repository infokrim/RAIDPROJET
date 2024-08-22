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
