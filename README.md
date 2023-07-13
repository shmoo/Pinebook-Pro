# Aarch64 Arch Linux Distro + LUKS + BTRFS + Yubikey

Adventures in getting everything to work together.

## What I wanted

I had recently ordered an NVME drive and the [NVME adapter](https://pine64.com/product/pinebook-pro-m-2-ngff-nvme-ssd-interface-adapter/) for my Pinebook Pro, and I wanted to move away from [Manjaro](https://wiki.manjaro.org/index.php/Manjaro-ARM) for a number of reasons. I'm already a huge fan and user of [ZFS](https://en.wikipedia.org/wiki/ZFS), so I wanted to try out [btrfs](https://wiki.archlinux.org/title/Btrfs) on the nvme for possible performance and snapshots, and to get a better sense of it as a filesystem. I wanted to encrypt the entire working drive and I wanted to decrypt it with my Yubikey.

## What I had

A 2020 [Pinebook Pro](https://www.pine64.org/pinebook-pro/), A [Yubikey 5C NFC](https://www.yubico.com/product/yubikey-5c-nfc/), A working Manjaro installation on the internal emmc (128GB), a spare emmc (64GB), and an [emmc/SD adapter](https://pine64.com/product/usb-adapter-for-emmc-module/). And some free time.

## What this is

A record of the final, working, and (hopefully) repeatable process. I took quite a few notes during the process and had many dead ends and resets, so this will likely still have a series of assumptions about my target audience as well as omissions and mistakes.

## NVME installation of aarch64 Arch w/ LUKS container, using btrfs

This was done from a bootable Aarch64 Arch Linux image on my spare 64GB emmc drive mounted into the SD card adapter and in the SD card slot of they system. The boot/working drive all the commands were run from was mounted as /dev/mmcblk1p1 (boot) and mmcblk1p2 (root).
All commands were run as `root`/UID0 from a `sudo -s` shell.

I started with a working Manjaro installation on the internal 128GB emmc of my Pinebook Pro. My early attempts to install Arch Linux directly to the nvme failed as I was trying to do ALL THE THINGS at once. On subsequent attempts I broke it down into parts; Install stock aarch64 Arch Linux onto an SD card, and then us that SD card install to install to the nvme with all the bells & whistles.

## Phase 0: Downloads

I needed the following:

* The latest Aarch64 Arch Linux from <https://archlinuxarm.org/platforms/armv8/generic>. I used the 01-Mar-2023 release.
* The latest [Tow-Boot](https://tow-boot.org) for Pinebook Pro from <https://github.com/Tow-Boot/Tow-Boot/releases>. I used the 2022.07-006 release.

## Phase 0.5: Tow-Boot on SPI

TBD, for now see <https://forum.pine64.org/showthread.php?tid=17529>

## Phase 1: Creating a bootable Arch based SD card

The following workflow was all done from a terminal from a Manjaro boot on my intenral emmc. All commands were run as root from a `sudo -s` shell. I did so many installations here trying to get something to work from Manjaro that eventually I just wrote a bash script. You shoould be able to copy & paste it into a root shell and have it work, up until the `arch-chroot` anyway. The chroot has an interactive portion where the UUID of your root partition on the SD card needs to be entered into the `Append` line of the `extlinux.conf` file. If you use vi like me, you can easily cheat by just using `:read !blkid -s UUID -o value /dev/mmcblk1p2` to get the UUID inside of the vi session.

```bash
# zero out SD card for use
dd if=/dev/zero of=/dev/mmcblk1 bs=1M count=32
# Create a GPT partition table on SD card (mmcblk1) of two parittions, the boot partition of 550MB (mmcblk1p1) and the rest into the root (mmcblk1p2)
parted -s /dev/mmcblk1 mklabel gpt mkpart primary 1 550MiB name 1 boot mkpart primary 550Mib 100% name 2 root
# Create filesystems on both new partitions on the SD card
mkfs --type=ext4 -F /dev/mmcblk1p1
mkfs --type=ext4 -F /dev/mmcblk1p2
# Mount the new file systems. 
## noatime because who needs extra writes on an SD card?
mount -o noatime /dev/mmcblk1p2 /mnt
### create a /mnt/boot dir for the mount of the boot partition
mkdir -p /mnt/boot 
mount -o noatime /dev/mmcblk1p1 /mnt/boot
### Create a directory for the extlinux file that will be copied or created later
mkdir -p /mnt/boot/extlinux

# extract the latest Arch Linux for ARM bundle, downloaded in Phase 0, to the root partition of the SD card
bsdtar -xvpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
# add some additional packages now that will be important once we boot from the SD card later
## ap6256-firmware is drivers for the wifi chipset in the Pinebook Pro. Very important if you want to have WiFi access from the card boot
## dialog and wpa_supplicant so that `wifi-menu` will work post boot
## parted, btrfs-progs, and arch-install-scripts for doing the installs to nvme later
pacstrap -K /mnt ap6256-firmware wget nvme-cli dialog parted wpa_supplicant btrfs-progs arch-install-scripts
# create a preliminary /etc/fstab from the mounts already used. saves us from having to do it in the chroot later.
genfstab -U /mnt >> /mnt/etc/fstab
# Grab the reference pacman config from my repo, or create your own.
curl -o /mnt/etc/pacman.conf https://raw.githubusercontent.com/shmoo/Pinebook-Pro/main/References/Files/etc/pacman.conf
# Grab the reference extlinux config from my repo, or create your own from scratch - 
curl -o /mnt/boot/extlinux/extlinux.conf https://raw.githubusercontent.com/shmoo/Pinebook-Pro/main/References/Files/boot/extlinux/extlinux.conf.sd
arch-chroot /mnt
    # The following actions all take place INSIDE the chroot instantiated from the above command…
    echo arch-sd > /etc/hostname
    echo LANG=en_US.UTF-8 > /etc/locale.conf
    echo en_US.UTF-8 UTF-8 >> /etc/locale.gen
    locale-gen
    echo KEYMAP=us > /etc/vconsole.conf
    echo FONT=lat9w-16 >> /etc/vconsole.conf
    ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
    vi /boot/extlinux/extlinux.conf
        # Insert the correct UUID from your SD card root partition, probably /dev/mmcblk1p2, into the APPEND line and save
    exit # Leave the Chroot
umount -a
poweroff
```

The reason we're using poweroff here and not reboot is I've experienced issues with the Pinebook Pro not recognizing Wifi on reboots. A power-off seems to reset everything however.

After the Pinebook Pro has powered off, power it back and and hit [Esc] when prompted by Tow-Boot to enter the boot selection Menu. Boot from SD. Choose the default boot option, and you should be greeted by a wall of text as we boot into the non-GUI stock vanilla aarch64 Arch Linux.

### Phase 1b: Post SD install update to Arch OS

TBD

## Phase 2: Installing to NVMe from Arch SD Card

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
# Extract the aarch64 arch linux distro to the mounted root
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

The `cryptdevice=UUID=` line(s) need to be set to the UUID of your encrypted NVME partition. You can use `blkid -s UUID -o value` to get the UUID you need. In my case it was the output of `blkid -s UUID -o value /dev/nvme0n1p2`

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

Umount everything and reboot

```bash
umount -a
reboot
```

## Phase 3: First boot into arch on nvme and setting up the Yubikey

TBD
I followed guide from <https://github.com/agherzan/yubikey-full-disk-encryption>.

## References

### Files

A copy of the important files I referred to repeatedly from my notes for my system are stored here in this repo, in case they're of any use to anyone else.

* [/etc/pacman.conf](./References/Files/etc/pacman.conf) # A list of the repos I used
* [/etc/mkinitcpio.conf](./References/Files/etc/mkinitcpio.conf) # My working mkinitcpio.conf file for building a Yubikey + LUKS + btrfs initramfs boot image
* [/boot/extlinux/extlinux.conf](./References/Files/boot/extlinux/extlinux.conf) # My boot options

### Repos

* <https://pacman.kiljan.org/archlinuxarm-pbp/os/aarch64/>
* <http://nj.us.mirror.archlinuxarm.org/aarch64/>

### Other References

* <https://ryankozak.com/luks-encrypted-arch-linux-on-pinebook-pro-again/>
* <https://github.com/infinitechris/PineBookProArchSetup/tree/main>
* <https://wiki.pine64.org/wiki/Pinebook_Pro_Installing_Arch_Linux_ARM>
* <https://github.com/SvenKiljan/archlinuxarm-pbp/blob/main/INSTALL.md>
* <https://nerdstuff.org/posts/2020/2020-004_arch_linux_luks_btrfs_systemd-boot/>
* <https://github.com/agherzan/yubikey-full-disk-encryption>
