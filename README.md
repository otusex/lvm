# LVM snapshots,mirrors


Install xfsdump - and xfsrestore utilities to back up and restore files in an XFS file system.
lvm2 - Logical Volume Manager (LVM) is a device mapper framework that provides logical volume management for the Linux kernel.
```bash
yum install xfsdump lvm2 -y
```


The original partition:

```bash
lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─centos-root           253:0    0 37.5G  0 lvm  /
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    2G  0 disk 
sde                       8:64   0    2G  0 disk 

```


Generate fstab

cat << EOF > /etc/fstab
/dev/mapper/centos-root /                                   xfs     defaults        0 0
UUID=0356e691-d6fb-4f8b-a905-4230dbe62a32  /boot            xfs     defaults        0 0
EOF

Prepare a temporary root part for / partition:

```bash
pvcreate /dev/sdb
vgcreate tmp_root /dev/sdb
lvcreate -n lv_root -l +100%FREE /dev/tmp_root
```

Create a file system on it and mount it to transfer data there:

```bash
mkfs.xfs /dev/tmp_root/lv_root
mount /dev/tmp_root/lv_root /mnt
```


Backup copy all data from / partition to /mnt:

```bash
xfsdump -J - /dev/centos/root | xfsrestore -J - /mnt
```

Reconfigure grub to switch to a new one at startup /

```bash
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Update initrd:

```bash
dracut  --add="lvm"  --force /boot/initramfs-$(uname -r).img $(uname -r) -M
```


Change grub parameters:

```bash
sed -i"" -e "s#rd.lvm.lv=centos/root#rd.lvm.lv=tmp_root/lv_root#g" /boot/grub2/grub.cfg
```

```bash

Regenerate fstab

cat << EOF > /etc/fstab
/dev/mapper/tmp_root-lv_root /                              xfs     defaults        0 0
UUID=0356e691-d6fb-4f8b-a905-4230dbe62a32  /boot            xfs     defaults        0 0
EOF
```



Exit the chroot with the exit command. Next, reboot.
Rebooting successfully with a new root volume. You can verify this by looking at the output

```bash
lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─centos-root         253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
└─tmp_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

```


Now we need to change the size of the old VG and return root to it. To do this, delete the old LV size of 40G and create a new one for 5G:

```bash
lvremove /dev/centos/root
lvcreate -n new_root -L 5G centos
```

Repeat actions
```bash

mkfs.xfs /dev/centos/new_root
mount /dev/centos/new_root /mnt
xfsdump -J - /dev/tmp_root/lv_root | xfsrestore -J - /mnt
```






