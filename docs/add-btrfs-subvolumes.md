# Add Btrfs sub-volumes to a new installation of Ubuntu

You are here because you want to use Btrfs on Ubuntu, so I won't try to convince you that Btrfs is
worth using or list its features. Instead, I will provide a guide for a basic configuration that I
use on workstations and laptops. What you see below is the result of years of experience with Linux
and Btrfs. I don't have the time (yet) to discuss each little nuance of the configuration. However,
this setup is not optimized for simple snapshots. Instead, it is optimized to performance and
security.

Start by booting from a Live ISO and open a terminal.

Next we need to mount the (possibly encrypted volume). Everything that follows will need to be
adjusted for your specific system and use-case.

```
sudo cryptsetup luksOpen /dev/nvme0n1p3 stuff
sudo mkdir /mnt/root /mnt/target
sudo mount -o noatime /dev/mapper/stuff /mnt/root/
```

The above commands created two mount points: one for the Btrfs tree (/mnt/root) and one to test
mount our sub-volumes (/mnt/target). NOTE: The mount command **did NOT** use subvol=...!

With the Btrfs tree mounted, let's look at it...

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

Now, let's mount the @ and @home sub-volumes that Ubuntu created for us.
```
sudo mount -o noatime,subvol=@ /dev/mapper/stuff /mnt/target/
sudo mount -o noatime,subvol=@home /dev/mapper/stuff /mnt/target/home/
```

Now, we are ready to do some real work.

Think *carefully* about which sub-volumes you want to create. 

I create sub-volumes for two reasons:
1. Security and control. For example, some directories (e.g., ```/var/tmp```) should be mounted with 
   ```nosuid``` and ```nodev```. Creating a sub-volume allows me to control mount-time settings.   
2. Performance. Btrfs has one queue per sub-volume. By placing disk-intensive applications on a 
   separate sub-volume, other sub-volumes may have better access to the underlying storage. Of 
   course, benchmarking is necessary to determine if this is always true for a specific workload.
   Still, I have decided that it is a good rule-of-thumb for me.

The following is some version of what I often do...

```
cd /mnt/root
sudo btrfs subvolume create @snap
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

Before moving files into our new sub-volumes, let's make using the wildcard easier...

```
shopt -s dotglob
```

With a SSD, it is considered good practice to run use tmpfs for ```/tmp```.

```
sudo rm -rf /mnt/root/@/tmp/*
sudo mount  -t tmpfs -o noatime,nosuid,nodev,size=4G none /mnt/target/tmp/
```

Now, let setup our new sub-volumes...

```
sudo mv /mnt/root/@/snap/* /mnt/root/@snap/
sudo mount -o noatime,subvol=@snap /dev/mapper/stuff /mnt/target/snap

sudo mkdir /mnt/target/snapshots
sudo mount -o noatime,nosuid,nodev,noexec,subvol=@snapshots /dev/mapper/stuff /mnt/target/snapshots/

sudo mv /mnt/root/@/var/* /mnt/root/@var/
sudo mount -o noatime,nosuid,nodev,noexec,subvol=@var /dev/mapper/stuff /mnt/target/var

sudo mkdir /mnt/target/var/lib/containers
sudo mount -o noatime,subvol=@containers /dev/mapper/stuff /mnt/target/var/lib/containers/

sudo mv /mnt/root/@var/lib/dpkg/* /mnt/root/@dpkg/
sudo mount -o noatime,nosuid,nodev,subvol=@dpkg /dev/mapper/stuff /mnt/target/var/lib/dpkg

#
# Kubuntu installs Flatpak automatically, so do this instead...
#
    sudo mv /mnt/root/@var/lib/flatpak/* /mnt/root/@flatpak/
#
# For Ubuntu do this...
#
    sudo mkdir /mnt/target/var/lib/flatpak
#
sudo mount -o noatime,nosuid,nodev,subvol=@flatpak /dev/mapper/stuff /mnt/target/var/lib/flatpak

sudo mkdir /mnt/target/var/lib/libvirt
sudo mount -o noatime,nosuid,nodev,noexec,subvol=@libvirt /dev/mapper/stuff /mnt/target/var/lib/libvirt

sudo mkdir /mnt/target/var/lib/machines
sudo mount -o noatime,subvol=@machines /dev/mapper/stuff /mnt/target/var/lib/machines/

sudo mkdir /mnt/target/var/lib/portables
sudo mount -o noatime,subvol=@portables /dev/mapper/stuff /mnt/target/var/lib/portables/

sudo mv /mnt/root/@var/log/* /mnt/root/@log/
sudo mount -o noatime,nosuid,nodev,noexec,subvol=@log /dev/mapper/stuff /mnt/target/var/log

# This move may fail if /tmp is empty. This is OK and expected.
sudo mv /mnt/root/@var/tmp/* /mnt/root/@var_tmp/
sudo mount -o noatime,nosuid,nodev,subvol=@var_tmp /dev/mapper/stuff /mnt/target/var/tmp

sudo chmod 0700 /mnt/target/snapshots
sudo chmod 1777 /mnt/target/var/tmp
```

Ok, that was a lot of practice and repetition. Now, say you want to install PostgreSQL. Knowing that
it can be disk intensive, you might want to place the database(s) in a sub-volume. You should
now be able to do that!

One last thing, we need to edit fstab to add the new volumes...

```
sudo nano -w /mnt/target/etc/fstab
```

Here is the /etc/fstab for the above scenario...

```
/dev/mapper/nvme0n1p3_crypt /                    btrfs   noatime,subvol=@                               0  0
/dev/mapper/nvme0n1p3_crypt /home                btrfs   noatime,nosuid,nodev,subvol=@home              0  0
/dev/mapper/nvme0n1p3_crypt /snap                btrfs   noatime,nosuid,nodev,noexec,subvol=@snap       0  0
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

UUID=947fb6b5-8b1d-4f34-be95-b3d369b52eab /boot           ext4    defaults        0       2

UUID=B308-F41D  /boot/efi       vfat    umask=0077      0       1
```

Reboot.

