sudo cryptsetup luksOpen /dev/nvme0n1p3 stuff
sudo mkdir /mnt/root /mnt/target
sudo mount -o noatime /dev/mapper/stuff /mnt/root/

sudo btrfs subvolume list /mnt/root
ID 256 gen 25 top level 5 path @
ID 257 gen 17 top level 5 path @home

cat /mnt/root/@/etc/fstab
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/nvme0n1p3_crypt /               btrfs   defaults,subvol=@ 0       1
# /boot was on /dev/nvme0n1p2 during installation
UUID=947fb6b5-8b1d-4f34-be95-b3d369b52eab /boot           ext4    defaults        0       2
# /boot/efi was on /dev/nvme0n1p1 during installation
UUID=B308-F41D  /boot/efi       vfat    umask=0077      0       1
/dev/mapper/nvme0n1p3_crypt /home           btrfs   defaults,subvol=@home 0       2

sudo mount -o noatime,subvol=@ /dev/mapper/stuff /mnt/target/
sudo mount -o noatime,subvol=@home /dev/mapper/stuff /mnt/target/home/

sudo btrfs subvolume create @snapshots
sudo btrfs subvolume create @var
sudo btrfs subvolume create @containers
sudo btrfs subvolume create @dpkg
sudo btrfs subvolume create @flatpak
sudo btrfs subvolume create @libvirt
sudo btrfs subvolume create @machines
sudo btrfs subvolume create @portables
sudo btrfs subvolume create @log
sudo btrfs subvolume create @var_tmp

shopt -s dotglob

sudo rm -rf /mnt/root/@/tmp/*
sudo mount  -t tmpfs -o noatime,nosuid,nodev,size=4G none /mnt/target/tmp/

sudo mkdir /mnt/target/snapshots
sudo mount -o noatime,nosuid,nodev,noexec,subvol=@snapshots /dev/mapper/stuff /mnt/target/snapshots/

sudo mv /mnt/root/@/var/* /mnt/root/@var/
sudo mount -o noatime,nosuid,nodev,noexec,subvol=@var /dev/mapper/stuff /mnt/target/var

sudo mkdir /mnt/target/var/lib/containers
sudo mount -o noatime,subvol=@containers /dev/mapper/stuff /mnt/target/var/lib/containers/

sudo mv /mnt/root/@var/lib/dpkg/* /mnt/root/@dpkg/
sudo mount -o noatime,nosuid,nodev,subvol=@dpkg /dev/mapper/stuff /mnt/target/var/lib/dpkg

sudo mkdir /mnt/target/var/lib/flatpak
sudo mount -o noatime,nosuid,nodev,subvol=@flatpak /dev/mapper/stuff /mnt/target/var/lib/flatpak

sudo mkdir /mnt/target/var/lib/libvirt
sudo mount -o noatime,nosuid,nodev,noexec,subvol=@libvirt /dev/mapper/stuff /mnt/target/var/lib/libvirt

sudo mkdir /mnt/target/var/lib/machines
sudo mount -o noatime,subvol=@machines /dev/mapper/stuff /mnt/target/var/lib/machines/

sudo mkdir /mnt/target/var/lib/portables
sudo mount -o noatime,subvol=@portables /dev/mapper/stuff /mnt/target/var/lib/portables/

sudo mv /mnt/root/@var/log/* /mnt/root/@log/
sudo mount -o noatime,nosuid,nodev,noexec,subvol=@log /dev/mapper/stuff /mnt/target/var/log

sudo mv /mnt/root/@var/tmp/* /mnt/root/@var_tmp/
sudo mount -o noatime,nosuid,nodev,subvol=@var_tmp /dev/mapper/stuff /mnt/target/var/tmp


sudo nano -w /mnt/target/etc/fstab

/dev/mapper/nvme0n1p3_crypt /                    btrfs   noatime,subvol=@                               0  0
/dev/mapper/nvme0n1p3_crypt /home                btrfs   noatime,nosuid,nodev,subvol=@home              0  0
/dev/mapper/nvme0n1p3_crypt /snapshots           btrfs   noatime,nosuid,nodev,noexec,subvol=@snapshots  0  0
tmpfs                       /tmp                 tmpfs   noatime,nosuid,nodev,size=4G                   0  0
/dev/mapper/nvme0n1p3_crypt /var                 btrfs   noatime,nosuid,nodev,noexec,subvol=@var        0  0
/dev/mapper/nvme0n1p3_crypt /var/lib/containers  btrfs   noatime,subvol=@containers                     0  0
/dev/mapper/nvme0n1p3_crypt /var/lib/dpkg        btrfs   noatime,nosuid,nodev,subvol=@dpkg              0  0
/dev/mapper/nvme0n1p3_crypt /var/lib/flatpak     btrfs   noatime,nosuid,nodev,subvol=@flatpak           0  0
/dev/mapper/nvme0n1p3_crypt /var/lib/libvirt     btrfs   noatime,nosuid,nodev,noexec,subvol=@libvirt    0  0
/dev/mapper/nvme0n1p3_crypt /var/lib/machines    btrfs   noatime,subvol=@machines                       0  0
/dev/mapper/nvme0n1p3_crypt /var/lib/portables   btrfs   noatime,subvol=@portables                      0  0
/dev/mapper/nvme0n1p3_crypt /var/log             btrfs   noatime,nosuid,nodev,noexec,subvol=@log        0  0
/dev/mapper/nvme0n1p3_crypt /var/tmp             btrfs   noatime,nosuid,nodev,subvol=@var_tmp           0  0

# /boot was on /dev/nvme0n1p2 during installation
UUID=947fb6b5-8b1d-4f34-be95-b3d369b52eab /boot           ext4    defaults        0       2

# /boot/efi was on /dev/nvme0n1p1 during installation
UUID=B308-F41D  /boot/efi       vfat    umask=0077      0       1
