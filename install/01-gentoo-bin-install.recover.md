# Gentoo base install - recovery

This guide is a continuation of the previous guide [Gentoo base install](./01-gentoo-bin-install.md) and is intended to be used as a reference to recover a Gentoo installation. The previous guide is a step-by-step guide to install Gentoo from scratch, and this guide is a step-by-step guide to recover a Gentoo installation.

Table of contents:

- [Gentoo base install - recovery](#gentoo-base-install---recovery)
    - [Recovering pre-cheks](#recovering-pre-cheks)
      - [Configure boot setup if needed](#configure-boot-setup-if-needed)
    - [Finalizing the recovery](#finalizing-the-recovery)

---

### Recovering pre-cheks

To recover the installation we need to mount all the drives and chroot into the system. The following steps are to be executed in the live environment.

Open the encrypted partition:

```bash
cryptsetup luksOpen /dev/nvme0n1p2 root
```

Get the partition details:

```bash
lsblk -o name,uuid
```

- boot - 393A-6EB3
- luks - b7b50886-85bb-47d8-af23-955f14ff954b
- root - 0a50c26f-d611-4a63-bf64-5b972bf85502

And setup the mount points:

```bash
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@ /dev/mapper/root /mnt/gentoo

mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run

mount -t tmpfs -o nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm

chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

mount /dev/nvme0n1p1 /boot

mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home /dev/mapper/root /home

mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial /dev/mapper/root /home/celestial
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial_cache /dev/mapper/root /home/celestial/.cache
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial_thumbs /dev/mapper/root /home/celestial/.thumbs
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial_space_tmp /dev/mapper/root /home/celestial/space/tmp
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial_space_workspace /dev/mapper/root /home/celestial/space/workspace

mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@media /dev/mapper/root /media
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@mnt /dev/mapper/root /mnt
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@root /dev/mapper/root /root
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@tmp /dev/mapper/root /tmp

mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@var_cache /dev/mapper/root /var/cache
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@var_crash /dev/mapper/root /var/crash
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@var_log /dev/mapper/root /var/log
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@var_tmp /dev/mapper/root /var/tmp
```

#### Configure boot setup if needed

```bash
cat << EOF > /etc/dracut.conf.d/luks.conf
hostonly="yes"
add_dracutmodules+=" crypt "
filesystems+=" btrfs "
force_drivers+=" amdgpu "
kernel_cmdline="rd.retry=10 rd.luks.allow-discards rd.luks.uuid=b7b50886-85bb-47d8-af23-955f14ff954b rd.luks.name=b7b50886-85bb-47d8-af23-955f14ff954b=root rootfstype=btrfs rootflags=subvol=@"
EOF

#nano /etc/dracut.conf.d/override.conf
dracut --force --kver 6.6.13-gentoo-dist
#nano /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
#nano /boot/grub/grub.cfg
```

---

### Finalizing the recovery

```bash
exit
cd

umount -l /mnt/gentoo/dev{/shm,/pts,}
umount /dev/nvme0n1p2
umount -R /mnt/gentoo

reboot
```
