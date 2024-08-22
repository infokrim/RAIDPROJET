# Mise en place d'un RAID 1 sur un serveur Proxmox

Cette documentation explique comment mettre en place un RAID 1 sur une machine virtuelle (un serveur) Proxmox, en partant d'un volume système existant de 30 Go, en ajoutant deux nouveaux disques de 15 Go chacun, puis en transférant les données du disque d'origine vers le nouveau volume en RAID 1. L'idée derrière l'ajout de disques de 15 Go au lieu de 30 Go est de maximiser l'efficacité du stockage, car les données présentes sur le disque de 30 Go n'occupent qu'un peu moins de 10 Go d'espace.

## 1. Configuration initiale du serveur

La capture d'écran suivante montre la configuration matérielle initiale du serveur virtualisé sur Proxmox:

![Configuration matérielle de la VM](1_Idee_ajout.png)

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

### 3.2 Partitionnement des disques

1. Connectez-vous à la machine virtuelle via SSH ou via la console Proxmox.
2. Utilisez `fdisk` pour partitionner les deux nouveaux disques de 15 Go de manière identique à celle du disque de 30 Go.

   ```bash
   fdisk /dev/sdX
