# Aarch64 Arch Linux Distro + LUKS + BTRFS + Yubikey

Adventures in getting everything to work together.

## What I wanted

I had recently ordered an NVME drive and the [NVME adapter](https://pine64.com/product/pinebook-pro-m-2-ngff-nvme-ssd-interface-adapter/) for my Pinebook Pro, and I wanted to move away from Manjaro for a number of reasons. I'm already a huge fan and user of ZFS, so I wanted to try out btrfs on the nvme for possible performance and snapshots, and to get a better sense of it as a filesystem. I wanted to encrypt the entire working drive and I wanted to decrypt it with my Yubikey.

## What I had

A 2020 [Pinebook Pro](https://www.pine64.org/pinebook-pro/), A [Yubikey 5C NFC](https://www.yubico.com/product/yubikey-5c-nfc/), A working Manjaro installation on the internal emmc (128GB), a spare emmc (64GB), and an [emmc/SD adapter](https://pine64.com/product/usb-adapter-for-emmc-module/). And some free time.

## What this is

A record of the final, working, and (hopefully) repeatable process. I took quite a few notes during the process and had many dead ends and resets, so this will likely still have a series of assumptions about my target audience as well as omissions and mistakes.

## NVME installation of aarch65 Arch w/ LUKS container, using btrfs

This was done from a bootable Aarch64 Arch Linux image on my spare 64GB emmc drive mounted into the SD card adapter and in the SD card slot of they system. The boot/working drive all the commands were run from was mounted as /dev/mmcblk1p1 (boot) and mmcblk1p2 (root).
All commands were run as `root`/UID0 from a `sudo -s` shell.

### Downloads

### Initializing the NVMe device

To start I made sure that the nvme drive itself was initialized and had no bootable sectors.

```bash
## reset nvme
# zero out intial sectors of nvme
dd if=/dev/zero of=/dev/nvme0n1 bs=1M count=32
```

### Create the GPT partitions on NVMe device

I use only GPT here, as I dont' know if anything else works, and this worked fine. I started the first partition at the 32MiB mark on the drive and going until 550MiB. I do this mostly out of habit, in case there needs to be an initial boot loader on the drive, but since I'm using Tow-Boot on SPI, this isn't necessary and could be started from the first sector.

The first partition will become our /boot eventually, with the 2nd partition being the remainder of the nvme device and eventually our root.

```bash
# create partitions
# gpt table
# primary partition starting at the 32MiB mark on drive, going to 550MiB 
# a second primary partition starting at 500MiB and taking the rest of the available disk
parted --script /dev/nvme0n1 mklabel gpt mkpart primary 32MiB 550MiB name 1 boot mkpart primary 550Mib 100% name 2 root
```

### Creation of filesystems, and LUKS container

I got much of the btrfs subvolume, remount, and swap set ups from [this post on nerdstuff.org](https://nerdstuff.org/posts/2020/2020-004_arch_linux_luks_btrfs_systemd-boot/).

```bash
# create an ext4 format filesystem on the boot partition
mkfs --type=ext4 -F /dev/nvme0n1p1
# Create a LUKS encyrption container for the rest of the drive
cryptsetup -y -v --use-urandom luksFormat /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 cryptroot
# create a btrfs filesystem inside the LUKS container
mkfs --type=btrfs -L ROOT -f /dev/mapper/cryptroot
# mount the btrfs filesystem first
mount /dev/mapper/cryptroot /mnt
# create root subvolume
btrfs sub create /mnt/@
# create swap subvolume
btrfs sub create /mnt/@swap
# create home subvolume
btrfs sub create /mnt/@home
# create package subvolume
btrfs sub create /mnt/@pkg
# create snapshots subvolume
btrfs sub create /mnt/@snapshots
# unmount now that we're done
umount /mnt
```

### Remount btrfs subvolumes & boot partition

Notes TBD

```bash
## remount all the newly created subvolumes for write
# noatime since we're on nvme, why waste writes
# compress-force=zstd:1 for nvme1 
# mount the root
mount -o noatime,compress-force=zstd:1,space_cache=v2,subvol=@ /dev/mapper/cryptroot /mnt
# create the requisit subdirectories for the rest of the mounts
mkdir -p /mnt/boot /mnt/home /mnt/var/cache/pacman/pkg /mnt/.snapshots /mnt/btrfs
# mount home
mount -o noatime,compress-force=zstd:1,space_cache=v2,subvol=@home /dev/mapper/cryptroot /mnt/home
# mount the pkg cache
mount -o noatime,compress-force=zstd:1,space_cache=v2,subvol=@pkg /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg
# mount the snapshots
mount -o noatime,compress-force=zstd:1,space_cache=v2,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
# mount the underlying btrfs
mount -o noatime,compress-force=zstd:1,space_cache=v2,subvolid=5 /dev/mapper/cryptroot /mnt/btrfs
# mount the boot volume
mount -o noatime /dev/nvme0n1p1 /mnt/boot
# create additional boot sub directories
mkdir -p /mnt/boot/extlinux
```

### Create some swap

Notes TBD

```bash
## create some swap
# Create the file
truncate -s 0 /mnt/btrfs/@swap/swapfile
# set type
chattr +C /mnt/btrfs/@swap/swapfile
# Fill the file to 4GB (1Mx4096)
dd if=/dev/zero of=/mnt/btrfs/@swap/swapfile bs=1M count=4096 status=progress
# set the permission
chmod 600 /mnt/btrfs/@swap/swapfile
# make it a swap file
mkswap /mnt/btrfs/@swap/swapfile
# enable the swap
swapon /mnt/btrfs/@swap/swapfile
```

### Install Arch (finally!)

Notes TBD

```bash
## install Arch
# Extract the aarch65 arch linux distro to the mounted root
bsdtar -xvpf ./ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
# copy the existing extlinux.conf so we have a working copy to modifiy
cp -Rp /boot/extlinux/extlinux.conf /mnt/boot/extlinux/
# add some additional packages 
pacstrap -P /mnt libdrm-pinebookpro pinebookpro-audio ap6256-firmware mtd-utils wget libfido2 zram-generator sudo nvme-cli dialog parted wpa_supplicant btrfs-progs
# create an initial fstab
genfstab -U /mnt >> /mnt/etc/fstab

```

#### Chroot actions
 
```bash
arch-chroot /mnt
```

##### INSIDE Chroot

```bash
echo arch-nvme > /etc/hostname
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo en_US.UTF-8 UTF-8 >> /etc/locale.gen
locale-gen
echo KEYMAP=us > /etc/vconsole.conf
echo FONT=lat9w-16 >> /etc/vconsole.conf
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
```

###### Edit the extlinux.conf

The cryptdevice=UUID= line(s) need to be set to the UUID of your encrypted NVME partition. You can use `blkid -s UUID` to get the UUID you need. In my case it was the output of `blkid -s UUID /dev/nvme0n1p2`

```config
DEFAULT arch
MENU TITLE Boot Menu
PROMPT 0
TIMEOUT 50

LABEL arch
MENU LABEL Arch Linux ARM on nvme
LINUX /Image
INITRD /initramfs-linux.img
FDT /dtbs/rockchip/rk3399-pinebook-pro.dtb
APPEND console=tty0 cryptdevice=UUID=${NVME-UUID}:cryptroot root=/dev/mapper/cryptroot rootwait rootflags=subvol=@ rw

LABEL arch-fallback
MENU LABEL Arch Linux ARM on nvme with fallback initramfs
LINUX /Image
INITRD /initramfs-linux-fallback.img
FDT /dtbs/rockchip/rk3399-pinebook-pro.dtb
APPEND console=tty0 cryptdevice=UUID=${NVME-UUID}:cryptroot root=/dev/mapper/cryptroot rootwait rootflags=subvol=@ rw
```

###### Edit the mkinitcpio.conf

Modify the `HOOKS=` and `COMPRESSION=` lines to support encyrption and uncompressed initramfs

```config
# mkinitcpio.conf for LUKS
# Adding encrypt before fileystems is the magic sauce. order matters!
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
# choosing uncompressed (cat) for initramfs as it's the most compatable, though outher formats may work
COMPRESSION="cat"
```

###### Update initramfs image

```bash
mkinitcpio -P
```

Once it completes, exit out of the chroot

```bash
exit
```

### References

#### Files

#### Repos

* <https://pacman.kiljan.org/archlinuxarm-pbp/os/aarch64/>
* <http://nj.us.mirror.archlinuxarm.org/aarch64/>

#### Other References

* <https://ryankozak.com/luks-encrypted-arch-linux-on-pinebook-pro-again/>
* <https://github.com/infinitechris/PineBookProArchSetup/tree/main>
* <https://wiki.pine64.org/wiki/Pinebook_Pro_Installing_Arch_Linux_ARM>
* <https://github.com/SvenKiljan/archlinuxarm-pbp/blob/main/INSTALL.md>
* <https://nerdstuff.org/posts/2020/2020-004_arch_linux_luks_btrfs_systemd-boot/>
