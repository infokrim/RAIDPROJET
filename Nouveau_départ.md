# État des Disques après Réinitialisation

## Disques et Partitions

Après avoir arrêté les ensembles RAID et supprimé les superblocs RAID, voici l'état actuel des disques `sda` et `sdb` :

```bash
root@SRV-DEB:/# lsblk
NAME      MAJ:MIN RM    SIZE RO TYPE  MOUNTPOINTS
sda         8:0    0     15G  0 disk
├─sda1      8:1    0   13,5G  0 part
│ └─md127   9:127  0   13,5G  0 raid1
├─sda2      8:2    0      1G  0 part
│ └─md126   9:126  0 1023,9M  0 raid1
└─sda3      8:3    0    511M  0 part
  └─md125   9:125  0  510,9M  0 raid1
sdb         8:16   0     15G  0 disk
├─sdb1      8:17   0   13,5G  0 part
│ └─md127   9:127  0   13,5G  0 raid1
├─sdb2      8:18   0      1G  0 part
│ └─md126   9:126  0 1023,9M  0 raid1
└─sdb3      8:19   0    511M  0 part
  └─md125   9:125  0  510,9M  0 raid1
sdc         8:32   0     32G  0 disk
├─sdc1      8:33   0     31G  0 part  /
├─sdc2      8:34   0      1K  0 part
└─sdc5      8:37   0    975M  0 part  [SWAP]
sr0        11:0    1    3,7G  0 rom
