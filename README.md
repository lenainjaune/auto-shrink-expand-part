RO voir git






































# auto-shrink-expand-part
Shrink / Expand a partition automagically without GUI application

TODO : prendre en compte la possibilité de déplacer (restauration d'un disque source plus petit que le disque cible avec partitions system+swap au final on aura system+swap+libre alors qu'on voudrait system+libre+swap pour pouvoir agrandir system dans l'espace libre, soit déplacer la partition swap à la fin qui souvent est une partition logique dans une partition étendue) : voir [ici](https://unix.stackexchange.com/questions/584078/move-partition-only-using-cli-tools-at-same-disk)

# NTFS Mount problem
En attendant de le mettre au bon endroit.

Si "The disk contains an unclean file system" ([source](https://sourceforge.net/p/clonezilla/discussion/Help/thread/a0203604/))

```sudo ntfsfix -d $PART```

Si "hibernated, refuses to mount" ([source](https://unix.stackexchange.com/questions/337152/cant-mount-windows-partition-in-linux-hibernated-refuses-to-mount/347319#347319))

```sudo mount -t ntfs-3g -o remove_hiberfile $DEV $MOUNT_PATH```

# The problem
Clonezilla restore and surely others can give problem of size when we want to restore. So if I backed up a 100 GB partition which only 20 GB used, I can not restore it on a 40 GB partition. The Linux **auto-shrink.sh** script corrects this before the backup operation. Reversely when the partition is backed up we can expand the partition to fill all the available free space with the Linux **auto-expand.sh** script.

# Linux
TODO : LVM parts (by default on Ubuntu)

TODO : partitions Ext2, Ext3, Crypté, ZFS, and so 

TODO : add EN comments (translate from FR to EN)

TODO : convert into a script and protect from mistakes

TODO : be sure to extract the good data for the sector size (sec_size) as fdisk returns many info related (physical, logical, 1x ...)

## Shrink avec GParted avec gestion de trou
TODO : automatiser

Situation : un vHD de 20 GB (system+swap) dont le système est Debian Bullseye formaté en Ext4 et qui est occupé à 5G/20GB

=> depuis le LiveISO GParted (gparted-live-1.3.1-1-i686.iso)
```
BBBbbbbbbESSSSe   initial : vHD de 20 GB (system+swap) dont system occupé à 5G/20GB

BBB______ESSSSe   réduire system à 10GB => trou entre partitions

BBBEEEEEEESSSSe   étendre partition étendue qui contient la partition de SWAP pour ne pas laisser de trou

BBBESSSSeeeeeee   déplacer partition de SWAP au début de la partition étendue (*)

BBBESSSSe______   réduire partition étendue à la taille de la partition de SWAP => trou à la fin
```
(*) déplacer la partition de SWAP affiche un avertissement comme quoi le système ne sera plus bootable (un swap bootable ???)

## Auto-shrink
Auto shrink an Ext4 Linux partition (like GParted do).
Based on : https://serverfault.com/questions/796758/shrink-partition-to-exactly-fit-the-underlying-filesystem-size/1024871#1024871

TODO : exclude all swap file before shrink

```sh
# The future auto-shrink.sh
PRT=/dev/sda3

# partition -> périphérique
# Avoir en tête que le lien entre une partition et son périphérique n'est 
#  pas forcément aussi simple que /dev/sda1 => /dev/sda ; 
#  penser aux partitions QCOW2 (ex : /dev/nbd0p1 qui est une partition 
#  du périphérique /dev/nbd0) et si la racine n'est pas /dev/ (je n'ai 
#  pas d'exemple pour étayer, mais ça me semble possible avec chroot)
#
# Explications :
#  $( bloc1 )$( bloc2 ) : concaténation de 2 blocs de sous commandes
#   $( bloc1 ) : trouve la racine (ici : /dev/)
#   $( bloc2 ) : trouve le périphérique suivi de | et du pon 
#                ici : /dev/sda|3
#    lsblk ... : -n (pas de ligne de titre), -t (affichage arborescent 
#                => liens de parentalité), -o (affiche juste le nom)
#    tac : inverse l'arborescence (le parent est après l'enfant)
#    awk ... : si ligne fini par partition cherchée (voir dessous) 
#              le prochain parent (ce qui est constitué que de chiffres 
#              et de lettres) sera le device cherché ; de plus on 
#              mémorise le numéro d'ordre de partition (pon) qu'on 
#              précède de | pour délimiter (voir remarque dessous)
#     $( ... ) : [dans le awk] extrait le nom de la partition derrière 
#                le dernier slash / ; donc on cherche ce qui termine par
#                un NON slash, 1 ou plusieurs fois (sda, etc.)
#
# Remarque : les numéros de partition dans le nom peuvent de pas 
#            correspondre avec le numéro d'ordre de la partition à 
#            traiter (partition ordered number) ; j'ai déjà vu une 
#            arborescence qui donne /dev/sda et ses partitions /dev/sda1
#            et /dev/sda5 (mais pas les intermédiaires) => pour traiter 
#            sda5 il faut considérer le numéro d'ordre de partition 2 
#            (c'est la 2ème partition)
#
# Visuellement si je recherche le périphérique de la partition sda3 :
# root@CZ-BKP:~# lsblk -n -t -o NAME | tac
# sr0
# └─sda4
# ├─sda3      <- searched part and NR = pon = 3
# ├─sda2
# ├─sda1
# sda         <- parent device and NR - pon = 6 - 3 = 3  => part n°3
# loop0
#
root@CLONEZILLA:~# root_dn_pon=$(\
 echo $PRT | grep -Eo ^/.+/ )'|'$( lsblk -nto NAME | tac | \
 awk '/'$( \
  echo $PRT | grep -Eo [^/]+$\
 )'$/ { ok = 1 ; pon = NR ; next } \
 ok && /^[a-z0-9]+$/ { print $0 "|" ( NR - pon ) ; exit }' )
root@CLONEZILLA:~# dn=$( echo $root_dn_pon | cut -d '|' -f 2 )
root@CLONEZILLA:~# dev=$( echo $root_dn_pon | cut -d '|' -f 1 )$dn
root@CLONEZILLA:~# pon=$( echo $root_dn_pon | cut -d '|' -f 3 )
# => /dev/sda sda 3

root@CLONEZILLA:~# fdisk -l $dev | grep -E "^(Sector size|$PRT)"
Sector size (logical/physical): 512 bytes / 512 bytes
Device     Boot    Start       End  Sectors  Size Id Type
/dev/sda3       40960000  80023551 39063552 18,6G 83 Linux
# => taille d'un secteur : 512 octets
# => début de la partition 3 (Debian Buster) : 40960000

# https://unix.stackexchange.com/questions/2668/finding-the-sector-size-of-a-partition
root@CLONEZILLA:~# sec_size=$( cat /sys/block/$dn/queue/hw_sector_size )
# => 512

root@CLONEZILLA:~# prt_start=$( partx -g -o START $PRT )
# => 40960000

root@CLONEZILLA:~# mount $PRT /mnt/ && df $PRT && umount $PRT
Filesystem 1K-blocks Used Available Use% Mounted on
/dev/sda3 19091540 3982736 14115952 23% /mnt
# => 14 GB disponible (100 - 23 = 77%)

# Vérification obligatoire avant réduction (demandé par resize2fs)
root@CLONEZILLA:~# e2fsck -f -y -v -C 0 $PRT
e2fsck 1.46.2 (28-Feb-2021)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information

122549 inodes used (10.02%, out of 1222992)
1114 non-contiguous files (0.9%)
97 non-contiguous directories (0.1%)
# of inodes with ind/dind/tind blocks: 0/0/0
Extent depth histogram: 111954/240
1105231 blocks used (22.64%, out of 4882432)
0 bad blocks
1 large file

99680 regular files
12029 directories
8 character device files
0 block device files
0 fifos
29 links
10823 symbolic links (10339 fast symbolic links)
0 sockets
------------
122569 files

# Réduire le système de fichiers au plus petit
root@CLONEZILLA:~# resize2fs -M $PRT
resize2fs 1.46.2 (28-Feb-2021)
Resizing the filesystem on /dev/sda3 to 1089991 (4k) blocks.
The filesystem on /dev/sda3 is now 1089991 (4k) blocks long.

root@CLONEZILLA:~# mount $PRT /mnt/ && df $PRT && umount $PRT
Filesystem 1K-blocks Used Available Use% Mounted on
/dev/sda3 4158788 3970592 0 100% /mnt
# => 100% utilisé, plus d'espace libre

# Calculs de réduction de la partition pour coller au système de fichiers
root@CLONEZILLA:~# dumpfs=$( dumpe2fs -h $PRT 2> /dev/null )
root@CLONEZILLA:~# block_count=$(\
 echo -e "$dumpfs" | grep "^Block count:" | cut -d ':' -f 2 | tr -d ' '\
 )
root@CLONEZILLA:~# block_size=$(\
 echo -e "$dumpfs" | grep "^Block size:" | cut -d ':' -f 2 | tr -d ' '\
 )
root@CLONEZILLA:~# prt_mini_size=$(\
 echo "( $block_count * $block_size ) / $sec_size + $prt_start + 10" \
 | bc )

# En NON interactif : réduire la partition au minimum
# https://stackoverflow.com/questions/52509644/how-do-i-get-over-parted-confirmation-request-in-a-script
root@CLONEZILLA:~# parted $dev ---pretend-input-tty << EOF
unit s
resizepart
$pon
$prt_mini_size
Yes
quit
EOF
# => OK en interactif, je n'ai rien cassé et le disque est rempli à 100%
# :D

# A priori nécessaire (je n'ai plus le message mais il y avait 
#  un warning de GParted concernant le montage et des blocs (FAT ?) )
root@CLONEZILLA:~# e2fsck -f -y -v -C 0 $PRT                                  
e2fsck 1.46.2 (28-Feb-2021)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure                                           
Pass 3: Checking directory connectivity                                        
Pass 4: Checking reference counts
Pass 5: Checking group summary information
                                                                               
      122549 inodes used (43.91%, out of 279072)
        1482 non-contiguous files (1.2%)
          97 non-contiguous directories (0.1%)
             # of inodes with ind/dind/tind blocks: 0/0/0
             Extent depth histogram: 111940/254
     1042942 blocks used (95.68%, out of 1089991)
           0 bad blocks
           1 large file

       99680 regular files
       12029 directories
           8 character device files
           0 block device files
           0 fifos
          29 links
       10823 symbolic links (10339 fast symbolic links)
           0 sockets
------------
      122569 files
# => OK plus de warning au montage !
```
## Auto-expand
Auto expand an Ext4 Linux partition (like GParted do).

```sh
# The future auto-expand.sh
PRT=/dev/sda3

root@CLONEZILLA:~# root_dn_pon=$(\
 echo $PRT | grep -Eo ^/.+/ )'|'$( lsblk -nto NAME | tac | \
 awk '/'$(\
  echo $PRT | grep -Eo [^/]+$\
 )'$/ { ok = 1 ; pon = NR ; next } \
 ok && /^[a-z0-9]+$/ { print $0 "|" ( NR - pon ) ; exit }' )
root@CLONEZILLA:~# dn=$( echo $root_dn_pon | cut -d '|' -f 2 )
root@CLONEZILLA:~# dev=$( echo $root_dn_pon | cut -d '|' -f 1 )$dn
root@CLONEZILLA:~# pon=$( echo $root_dn_pon | cut -d '|' -f 3 )
# => /dev/sda sda 3

root@CLONEZILLA:~# mount $PRT /mnt/ && df $PRT && umount $PRT
Sys. de fichiers blocs de 1K Utilisé Disponible Uti% Monté sur
/dev/sda3            4158788 3970832          0 100% /mnt

root@CLONEZILLA:~# fdisk -l $dev
Disk /dev/sda: 60 GiB, 64424509440 bytes, 125829120 sectors
Disk model: QEMU HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x3a867e24

Device     Boot    Start       End  Sectors  Size Id Type
/dev/sda1  *        2048    206847   204800  100M  7 HPFS/NTFS/exFAT
/dev/sda2         206848  40959999 40753152 19,4G  7 HPFS/NTFS/exFAT
/dev/sda3       40960000  49679938  8719939  4,2G 83 Linux
/dev/sda4       80023552 125829119 45805568 21,8G 83 Linux
# => partition sda4 (n+1) débute en 80023552
# => partition n termine au maximum en 80023552 - 1 = 80023551

# Augmenter partition au maxi dans espace disponible qui lui fait suite
#  partx -u met à jour la partition modifiée
# http://karelzak.blogspot.com/2015/05/resize-by-sfdisk.html
root@CLONEZILLA:~# echo ", +" | sfdisk --force -N $pon $dev
root@CLONEZILLA:~# partx -u $PRT

# Vérification obligatoire après expansion (demandé par resize2fs)
root@CLONEZILLA:~# e2fsck -f -y -v -C 0 $PRT
e2fsck 1.46.2 (28-Feb-2021)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure                                           
Pass 3: Checking directory connectivity                                        
Pass 4: Checking reference counts
Pass 5: Checking group summary information
                                                                               
      122562 inodes used (43.92%, out of 279072)
        1479 non-contiguous files (1.2%)
          97 non-contiguous directories (0.1%)
             # of inodes with ind/dind/tind blocks: 0/0/0
             Extent depth histogram: 111952/255
     1043002 blocks used (95.69%, out of 1089991)
           0 bad blocks
           1 large file

       99693 regular files
       12029 directories
           8 character device files
           0 block device files
           0 fifos
          29 links
       10823 symbolic links (10339 fast symbolic links)
           0 sockets
------------
      122582 files
      
# Etendre le système de fichiers au maxi pour coller à la partition
root@CLONEZILLA:~# resize2fs -p $PRT
resize2fs 1.46.2 (28-Feb-2021)
Resizing the filesystem on /dev/sda3 to 4882944 (4k) blocks.
The filesystem on /dev/sda3 is now 4882432 (4k) blocks long.
# => à l'air d'être nickel :DDD

root@CLONEZILLA:~# mount $PRT /mnt/ && df $PRT && umount $PRT
Sys. de fichiers blocs de 1K Utilisé Disponible Uti% Monté sur
/dev/sda3           19091540 3983032   14115668  23% /mnt
# => OK l'espace est à nouveau disponible

# => bizarrement on ne retombe pas sur l'état du début :
# root@CLONEZILLA:~# mount $PRT /mnt/ && df $PRT && umount $PRT
# Filesystem 1K-blocks Used Available Use% Mounted on
# /dev/sda3 19091540 3982736 14115952 23% /mnt
```
