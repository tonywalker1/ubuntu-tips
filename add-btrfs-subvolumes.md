# Add Btrfs subvolumes to a new installation of Ubuntu (or Debian)

## Warning: Never blindly follow directions from the Internet. That includes me! Make sure you read and understand each step...

First we need to mount the (possibly encrypted volume). Everything that follows will need to be adjusted for your specific system and use-case. 
```
sudo cryptsetup luksOpen /dev/nvme0n1p3 stuff
sudo mkdir /mnt/root /mnt/target
sudo mount -o noatime /dev/mapper/stuff /mnt/root/
```

The above commands created two mount points: one for the Btrfs tree (/mnt/root) and one to test mount our subvolumes (/mnt/target). With the Btrfs tree mounted, let's look at it...
```
sudo btrfs subvolume list /mnt/root
# ID 256 gen 25 top level 5 path @
# ID 257 gen 17 top level 5 path @home

```

```
cat /mnt/root/@/etc/fstab
# /dev/mapper/nvme0n1p3_crypt /               btrfs   defaults,subvol=@ 0       1
# UUID=947fb6b5-8b1d-4f34-be95-b3d369b52eab /boot           ext4    defaults        0       2
# UUID=B308-F41D  /boot/efi       vfat    umask=0077      0       1
# /dev/mapper/nvme0n1p3_crypt /home           btrfs   defaults,subvol=@home 0       2
```

Now, let's mount the @ and @home subvolumes that Ubuntu created for us...
```
sudo mount -o noatime,subvol=@ /dev/mapper/stuff /mnt/target/
sudo mount -o noatime,subvol=@home /dev/mapper/stuff /mnt/target/home/
```
Now, we are ready to do some real work.

Think *carefully* about which subvolumes you want to create. The following is what I usually do...
```
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
```

Before moving files into our new subvolumes, let's make using the wildcard easier...
```
shopt -s dotglob
```

On Btrfs, it is considered good practice to run use tmpfs with an SSD. There is some debate about that, but let's roll with this for now...
```
sudo rm -rf /mnt/root/@/tmp/*
sudo mount  -t tmpfs -o noatime,nosuid,nodev,size=4G none /mnt/target/tmp/
```

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
