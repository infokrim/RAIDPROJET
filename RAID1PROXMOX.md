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


