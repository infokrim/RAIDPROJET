Voici l'état de mes disques obtenus à l'aide des commandes suivantes :

```
lsblk
cat /proc/mdstat
blkid
```

![1_etat](https://github.com/user-attachments/assets/27b45d91-f050-4e4d-83fc-f4a500921669)
![2_etat](https://github.com/user-attachments/assets/10672a9c-2a54-4f69-8ce1-077fb165ae8e)



- **Commande lsblk**

**lsblk** affiche la hiérarchie des périphériques de blocs et leurs partitions.
Vous voyez ici que les disques sda et sdb ont chacun trois partitions (sda1, sda2, sda3, sdb1, sdb2, sdb3) qui sont toutes liées à des ensembles RAID (md127, md126, md125).
Le disque sdc est utilisé pour le système de fichiers principal (/) et la partition d'échange (swap).

- **Commande cat /proc/mdstat**

Cette commande montre l'état des ensembles RAID actifs sur votre système.
Les ensembles md125, md126, et md127 sont des ensembles RAID 1 (raid1) actifs qui utilisent les partitions des disques sda et sdb.
[UU] indique que toutes les copies du RAID sont synchronisées et fonctionnent correctement.

- **Commande blkid**

blkid est utilisé pour afficher les informations sur les périphériques de blocs, y compris leurs UUID et types de systèmes de fichiers.
Les sorties montrent les UUIDs des partitions et leur type (linux_raid_member pour les partitions RAID et ext4 pour les systèmes de fichiers normaux).
Les partitions RAID sont clairement identifiées, et vous pouvez voir que md127, md126, et md125 ont des systèmes de fichiers ext4 ou swap.

Mon but est de revenir à l'état suivant :   

![Cap2_lsblk](https://github.com/user-attachments/assets/148abcb7-8c55-4518-bf33-a83d006cea81)

# Réinitialisation des disques `sda` et `sdb` à l'état initial avant RAID 1

## 1. Arrêter les ensembles RAID
Arrêtez tous les ensembles RAID actifs (`md127`, `md126`, `md125`) pour les disques `sda` et `sdb` :

```bash
mdadm --stop /dev/md127
mdadm --stop /dev/md126
mdadm --stop /dev/md125
```
**`mdadm`** : C'est l'outil de gestion des ensembles RAID sous Linux.    
**`--stop`** : Cette option indique à mdadm d'arrêter un ensemble RAID. Lorsque vous arrêtez un ensemble RAID, celui-ci devient inactif, et toutes les opérations sur les partitions qui composent l'ensemble sont suspendues.   
**`/dev/md126`**: Il s'agit du nom de l'ensemble RAID que vous souhaitez arrêter. Les ensembles RAID sont généralement nommés sous la forme /dev/mdX, où X est un numéro ou un identifiant unique pour l'ensemble RAID.   

## 2. Supprimer les superblocs RAID

```bash
mdadm --zero-superblock /dev/sda1
mdadm --zero-superblock /dev/sda2
mdadm --zero-superblock /dev/sda3
mdadm --zero-superblock /dev/sdb1
mdadm --zero-superblock /dev/sdb2
mdadm --zero-superblock /dev/sdb3
```
**`mdadm`** : C'est l'outil de gestion des ensembles RAID sous Linux.
**`--zero-superblock`** : Cette option indique à mdadm de supprimer (mettre à zéro) le superbloc RAID sur le périphérique spécifié.
**`/dev/sda3`** : Il s'agit de la partition ou du disque sur lequel vous souhaitez supprimer le superbloc RAID. Dans cet exemple, c'est la troisième partition du disque /dev/sda.

Que fait cette commande ?

Superbloc RAID : Un superbloc RAID est une zone de métadonnées située sur chaque partition ou disque membre d'un ensemble RAID. Il contient des informations critiques sur l'ensemble RAID, telles que la configuration du RAID, les membres de l'ensemble, les identifiants uniques, etc.    

Suppression du superbloc : La commande mdadm --zero-superblock supprime ces métadonnées en "zéro" le superbloc, c'est-à-dire en effaçant toutes les informations RAID sur le périphérique spécifié. Après l'exécution de cette commande, la partition /dev/sda3 ne sera plus reconnue comme faisant partie d'un ensemble RAID.   

Réutilisation de la partition : Une fois le superbloc supprimé, la partition /dev/sda3 peut être réutilisée pour d'autres fins. Elle peut être formatée, ajoutée à un autre ensemble RAID, ou utilisée pour stocker des données non RAID. Le fait de supprimer le superbloc est une étape essentielle si vous souhaitez retirer définitivement une partition d'un ensemble RAID et l'utiliser pour autre chose.   


## 3. Vérifier que les superblocs sont supprimés

```
cat /proc/mdstat
```

Sortie de la commande :

```
root@SRV-DEB:/# cat /proc/mdstat
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10]
unused devices: <none>
```

**Explication** :

- Personalities :
Cette ligne affiche les types de RAID (ou "personalities") que votre noyau Linux peut actuellement gérer. 

- unused devices: <none> :
Cette ligne indique qu'il n'y a actuellement aucun périphérique RAID en attente d'utilisation ou non utilisé. Tous les périphériques de disques qui pourraient faire partie d'un ensemble RAID sont soit déjà intégrés dans un RAID, soit totalement libres (c'est-à-dire, ils n'ont pas de superbloc RAID ou ne sont pas en attente de réintégration dans un RAID).

Capture d'écran des 3 premières étapes :

![3_supraid](https://github.com/user-attachments/assets/ca09cf68-c116-4f22-9153-6909ef0de4c5)


