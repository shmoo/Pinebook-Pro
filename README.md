# Aarch64 Arch Linux Distro + LUKS + BTRFS + Yubikey

Adventures in getting everything to work together.

## What I wanted

I had recently ordered an NVME drive and the NVME adapter for my Pinebook Pro, and I wanted to move away from Manjaro for a number of reasons. I wanted to utlize btrfs on the nvme for possible performance and snapshots. I wanted to encrypt the entire working drive and I wanted to decrypt it with my Yubikey.

## What I had

A 2019 Pinebook Pro, A yubikey 5C NFC, A working Manjaro installation on the emmc, and a spare emmc (65GB), and emmc/SD adapter. 

##### Repos

* https://pacman.kiljan.org/archlinuxarm-pbp/os/aarch64/
* http://nj.us.mirror.archlinuxarm.org/aarch64/

###### References

* https://ryankozak.com/luks-encrypted-arch-linux-on-pinebook-pro-again/
* https://github.com/infinitechris/PineBookProArchSetup/tree/main
* https://wiki.pine64.org/wiki/Pinebook_Pro_Installing_Arch_Linux_ARM
* https://github.com/SvenKiljan/archlinuxarm-pbp/blob/main/INSTALL.md
* https://nerdstuff.org/posts/2020/2020-004_arch_linux_luks_btrfs_systemd-boot/
