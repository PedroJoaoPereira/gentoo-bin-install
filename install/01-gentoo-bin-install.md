# Gentoo base install

This is a guide on how to install Gentoo Linux with UEFI, with LUKS encrypted BTRFS. The installation will be done with the `amd64` architecture and `OpenRC` as the init system.

This guide creates a system based on the distribution kernel `gentoo-kernel-bin`, with `dracut` to generate a bootable image and `grub` as the bootloader.

This distribution kernel was used as the first step to get the system up and running. The next step is to compile a custom kernel with the `gentoo-sources` and `gentoo-kernel` overrife configuration feature so we get the best of both worlds - better support for updates as well as kernel customizations.

The `gentoo-kernel-bin` is a precompiled kernel that we can use to get the list of loaded modules and create a custom kernel configuration more easily.

Table of contents:

- [Gentoo base install](#gentoo-base-install)
    - [Installation pre-cheks](#installation-pre-cheks)
      - [Partition disks](#partition-disks)
        - [Partition scheme](#partition-scheme)
      - [Encrypt root](#encrypt-root)
      - [Create filesystem](#create-filesystem)
      - [Get partition details](#get-partition-details)
    - [Install stage 3](#install-stage-3)
      - [Configure `make.conf`](#configure-makeconf)
    - [Chroot into system](#chroot-into-system)
      - [Update Gentoo](#update-gentoo)
      - [Configure _fstab_](#configure-fstab)
    - [Installing dependencies](#installing-dependencies)
      - [Dracut with numlock _(optional)_](#dracut-with-numlock-optional)
      - [Boot configuration check](#boot-configuration-check)
      - [Configuring the system](#configuring-the-system)
    - [Finalizing the installation](#finalizing-the-installation)
    - [Sources:](#sources)

---

### Installation pre-cheks

Using an admin LiveCD from gentoo.org, boot into the system and check the internet connection:

```bash
ping -c 3 gentoo.org
```

#### Partition disks

Format disk to your liking, but this is an example using a NVME SSD:

```bash
fdisk /dev/nvme0n1
```

- d -- delete all paritions
- p -- print config
- w -- write config
- g -- create gpt label
- n -- create partition

##### Partition scheme

```
Partition       Filesystem  Size	Description
/dev/nvme0n1p1	fat32       256MB	Boot partition
/dev/nvme0n1p2	LUKS        rest	LUKS partition
```

Label them with:

```bash
parted /dev/nvme0n1
```

```bash
name 1 boot
name 2 luks

set 1 boot on

print
quit
```

#### Encrypt root

Check which algorithm is best according to your hardware:

```bash
cryptsetup benchmark
```

In my case the algorithm `aes-xts-plain64` has the most IO performance, so it will be used to encrypt a LUKS partition:

```bash
cryptsetup -c aes-xts-plain64 -s 512 -y luksFormat /dev/nvme0n1p2
```

Open the LUKS partition so we can create the filesystem:

```bash
cryptsetup luksOpen /dev/nvme0n1p2 root
```

#### Create filesystem

```bash
mkfs.vfat -F32 /dev/nvme0n1p1
mkfs.btrfs -L Gentoo /dev/mapper/root

mount /dev/mapper/root /mnt/gentoo

btrfs subvolume create /mnt/gentoo/@

btrfs subvolume create /mnt/gentoo/@home
btrfs subvolume create /mnt/gentoo/@home_celestial
btrfs subvolume create /mnt/gentoo/@home_celestial_cache
btrfs subvolume create /mnt/gentoo/@home_celestial_thumbs
btrfs subvolume create /mnt/gentoo/@home_celestial_space_tmp
btrfs subvolume create /mnt/gentoo/@home_celestial_space_workspace

btrfs subvolume create /mnt/gentoo/@media
btrfs subvolume create /mnt/gentoo/@mnt
btrfs subvolume create /mnt/gentoo/@root
btrfs subvolume create /mnt/gentoo/@tmp

btrfs subvolume create /mnt/gentoo/@var_cache
btrfs subvolume create /mnt/gentoo/@var_crash
btrfs subvolume create /mnt/gentoo/@var_log
btrfs subvolume create /mnt/gentoo/@var_tmp

umount /mnt/gentoo

mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@ /dev/mapper/root /mnt/gentoo
```

#### Get partition details

Get UUID for the boot partition on the dev `nvme0n1p1` and for the root file system mapped on dev `root` - encrypted volume:

```bash
lsblk -o name,uuid
```

- boot - 393A-6EB3
- luks - b7b50886-85bb-47d8-af23-955f14ff954b
- root - 0a50c26f-d611-4a63-bf64-5b972bf85502

---

### Install stage 3

Synchronize date before installation and download stage 3 tarball:

```bash
chronyd -q
date

cd /mnt/gentoo
links gentoo.org
```

Choose the `amd64` architecture and download the latest stage 3 tarball with `OpenRC`, quit with `q` and untar the tarball:

```bash
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

#### Configure `make.conf`

Alter `portage` configuration to your liking. The following is an example that works for me and that I prefer:

```bash
nano /mnt/gentoo/etc/portage/make.conf
```

Content of the file with some options to be addded later:

```conf
# Global use flags, recommended is granular flag setting with
# Change /etc/portage/package.use/<dependency name>
USE="cryptsetup dist-kernel -systemd"

# Portage defaults
# MAKEOPTS="-j16 -l16"
# EMERGE_DEFAULT_OPTS="--ask --verbose --quiet-build"
FEATURES="parallel-fetch parallel-install clean-logs"

# Compiler settings
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

# GPU settings
VIDEO_CARDS="amdgpu radeonsi"

# Fastest Gentoo mirrors
GENTOO_MIRRORS="https://mirrors.ptisp.pt/gentoo/ \
    http://mirrors.ptisp.pt/gentoo/ \
    https://ftp.rnl.tecnico.ulisboa.pt/pub/gentoo/gentoo-distfiles/ \
    http://ftp.rnl.tecnico.ulisboa.pt/pub/gentoo/gentoo-distfiles/ \
    ftp://ftp.rnl.tecnico.ulisboa.pt/pub/gentoo/gentoo-distfiles/ \
    rsync://ftp.rnl.tecnico.ulisboa.pt/pub/gentoo/gentoo-distfiles/ \
    http://ftp.dei.uc.pt/pub/linux/gentoo/ \
    https://repo.ifca.es/gentoo-distfiles \
    rsync://repo.ifca.es/gentoo-distfiles \
    ftp://repo.ifca.es/gentoo-distfiles"

# GRUB efit settings - mandatory
GRUB_PLATFORMS="efi-64"

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C.utf8

```

Update CPU architecture flags:

```bash
echo "*/* $(cpuid2cpuflags)" > /mnt/gentoo/etc/portage/package.use/00-cpuid2cpuflags
```

Mirrors can be changed with `mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf`

---

### Chroot into system

```bash
mkdir -p /mnt/gentoo/etc/portage/repos.conf
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf

mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run

test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount -t tmpfs -o nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm

chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

mount /dev/nvme0n1p1 /boot

mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home /dev/mapper/root /home

mkdir -p /home/celestial
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial /dev/mapper/root /home/celestial
mkdir -p /home/celestial/.cache
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial_cache /dev/mapper/root /home/celestial/.cache
mkdir -p /home/celestial/.thumbs
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial_thumbs /dev/mapper/root /home/celestial/.thumbs
mkdir -p /home/celestial/space/tmp
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial_space_tmp /dev/mapper/root /home/celestial/space/tmp
mkdir -p /home/celestial/space/workspace
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@home_celestial_space_workspace /dev/mapper/root /home/celestial/space/workspace

mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@media /dev/mapper/root /media
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@mnt /dev/mapper/root /mnt
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@root /dev/mapper/root /root
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@tmp /dev/mapper/root /tmp

mkdir -p /var/cache
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@var_cache /dev/mapper/root /var/cache
mkdir -p /var/crash
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@var_crash /dev/mapper/root /var/crash
mkdir -p /var/log
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@var_log /dev/mapper/root /var/log
mkdir -p /var/tmp
mount -t btrfs -o compress-force=zstd:1,noatime,subvol=@var_tmp /dev/mapper/root /var/tmp
```

#### Update Gentoo

Update the portage tree and system, as well as setting up the timezone and locale:

```bash
emerge --sync
emerge --update --deep --newuse @world
emerge --depclean

echo "Europe/Lisbon" > /etc/timezone
emerge --config sys-libs/timezone-data

sed -i "s/#en_US ISO-8859-1/en_US ISO-8859-1/g" /etc/locale.gen
sed -i "s/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g" /etc/locale.gen

locale-gen
eselect locale set 6
eselect locale list

env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```

#### Configure _fstab_

Edit _fstab_ with appropriate settings for the `BTRFS` filesystem:

```bash
cat << EOF > /etc/fstab
# /etc/fstab: static file system information.
#
# See the manpage fstab(5) for more information.
#
# NOTE: The root filesystem should have a pass number of either 0 or 1.
#       All other filesystems should have a pass number of 0 or greater than 1.
#
# NOTE: Even though we list ext4 as the type here, it will work with ext2/ext3
#       filesystems.  This just tells the kernel to use the ext4 driver.
#
# NOTE: You can use full paths to devices like /dev/sda3, but it is often
#       more reliable to use filesystem labels or UUIDs. See your filesystem
#       documentation for details on setting a label. To obtain the UUID, use
#       the blkid(8) command.

# <fs>                                      <mountpoint>            <type>  <opts> <dump> <pass>

# boot partition
UUID=393A-6EB3                              /boot                             vfat    noatime 0 1

# static file system
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /                                 btrfs   compress-force=zstd:1,noatime,subvol=@ 0 0

UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /home                             btrfs   compress-force=zstd:1,noatime,subvol=@home 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /home/celestial                   btrfs   compress-force=zstd:1,noatime,subvol=@home_celestial 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /home/celestial/.cache            btrfs   compress-force=zstd:1,noatime,subvol=@home_celestial_cache 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /home/celestial/.thumbs           btrfs   compress-force=zstd:1,noatime,subvol=@home_celestial_thumbs 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /home/celestial/space/tmp         btrfs   compress-force=zstd:1,noatime,subvol=@home_celestial_space_tmp 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /home/celestial/space/workspace   btrfs   compress-force=zstd:1,noatime,subvol=@home_celestial_space_workspace 0 0

UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /media                            btrfs   compress-force=zstd:1,noatime,subvol=@media 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /mnt                              btrfs   compress-force=zstd:1,noatime,subvol=@mnt 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /root                             btrfs   compress-force=zstd:1,noatime,subvol=@root 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /tmp                              btrfs   compress-force=zstd:1,noatime,subvol=@tmp 0 0

UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /var/cache                        btrfs   compress-force=zstd:1,noatime,subvol=@var_cache 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /var/crash                        btrfs   compress-force=zstd:1,noatime,subvol=@var_crash 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /var/log                          btrfs   compress-force=zstd:1,noatime,subvol=@var_log 0 0
UUID=0a50c26f-d611-4a63-bf64-5b972bf85502   /var/tmp                          btrfs   compress-force=zstd:1,noatime,subvol=@var_tmp 0 0

# tmps
tmpfs                                       /run                              tmpfs   size=128M 0 0
shm                                         /dev/shm                          tmpfs   nodev,nosuid,noexec 0 0

EOF
```

---

### Installing dependencies

Set appropriate flags for packages to be pulled and install them - decide what works best for you. In this guide I am installing `dracut` and `grub` to generate a bootable image automatically with `installkernel` which will run the former dependencies any time the kernel is updated. Additionally, the `gentoo-kernel-bin` because of reasons previously mentioned. `linux-firmware` is also installed to get the latest firmware for the kernel:

```bash
echo "sys-kernel/installkernel dracut grub" > /etc/portage/package.use/installkernel
emerge sys-kernel/installkernel

mkdir -p /etc/dracut.conf.d
cat << EOF > /etc/dracut.conf.d/override.conf
hostonly="yes"
add_dracutmodules+=" crypt "
filesystems+=" btrfs "
force_drivers+=" amdgpu "
kernel_cmdline="rd.retry=10 rd.luks.allow-discards rd.luks.uuid=b7b50886-85bb-47d8-af23-955f14ff954b rd.luks.name=b7b50886-85bb-47d8-af23-955f14ff954b=root rootfstype=btrfs rootflags=subvol=@"
EOF

#dracut --force --kver 6.6.13-gentoo-dist
#grub-mkconfig -o /boot/grub/grub.cfg

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id="Gentoo Grub Bootloader"

echo "sys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE" > /etc/portage/package.license
emerge sys-kernel/linux-firmware

# system dependencies
emerge sys-block/io-scheduler-udev-rules sys-fs/dosfstools sys-fs/e2fsprogs sys-fs/btrfs-progs sys-fs/cryptsetup sys-kernel/gentoo-kernel-bin net-misc/dhcpcd net-misc/netifrc app-admin/sysklogd sys-process/cronie net-misc/chrony app-misc/neofetch
```

- `io-scheduler-udev-rules` - NVME SSD scheduler
- `dosfstools` - FAT32 filesystem tools
- `e2fsprogs` - EXT2/3/4 filesystem tools
- `btrfs-progs` - BTRFS filesystem tools
- `cryptsetup` - LUKS encryption tools
- `gentoo-kernel-bin` - precompiled kernel
- `dhcpcd` - DHCP client
- `netifrc` - network interface configuration
- `sysklogd` - system logger
- `cronie` - cron daemon
- `chrony` - NTP client
- `neofetch` - system information - no system installation is finished without it

#### Dracut with numlock _(optional)_

If you want to have the numlock enabled at boot, you can add the `numlock` service to the `dracut` configuration:

```bash
mkdir -p /usr/lib/dracut/modules.d/51numlock/

cat << EOF > /usr/lib/dracut/modules.d/51numlock/enable-numlock.sh
#!/bin/bash
if [ -x /usr/bin/setleds ] ; then
   INITTY=/dev/tty[1-8]
   for tty in $INITTY ; do
       setleds -D +num < $tty
   done
fi
EOF

cat << EOF > /usr/lib/dracut/modules.d/51numlock/module-setup.sh
#!/bin/bash
# This file is part of dracut.
# SPDX-License-Identifier: GPL-2.0-or-later

# Prerequisite check(s) for module.
check() {

    # If the binary(s) requirements are not fulfilled the module can't be installed.
    require_binaries setleds || return 1

    # Return 0 to always include the module.
    return 0

}

# Module dependency requirements.
depends() {

    # Return 0 to include the dependent module(s) in the initramfs.
    return 0

}

# Install the required file(s) and directories for the module in the initramfs.
install() {
    inst /usr/bin/setleds
    inst_hook cmdline 99 "$moddir/enable-numlock.sh"
}
EOF
```

#### Boot configuration check

After the above section you should double check the boot details so it can properly prompt the decryption of the disk and loading of the system:

```bash
#nano /etc/dracut.conf.d/override.conf
dracut --force --kver 6.6.13-gentoo-dist
#nano /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
#nano /boot/grub/grub.cfg
```

#### Configuring the system

Set multiple services and details - also note that some personal customizations were added such as the `hostname`, `keymaps` and `hosts`:

```bash
sed -i 's/hostname="localhost"/hostname="terra"/g' /etc/conf.d/hostname

cat << EOF > /etc/hosts
# /etc/hosts: Local Host Database
#
# This file describes a number of aliases-to-address mappings for the for
# local hosts that share this file.
#
# The format of lines in this file is:
#
# IP_ADDRESS    canonical_hostname      [aliases...]
#
#The fields can be separated by any number of spaces or tabs.
#
# In the presence of the domain name service or NIS, this file may not be
# consulted at all; see /etc/host.conf for the resolution order.
#

# IPv4 and IPv6 localhost aliases
127.0.0.1       localhost terra
::1             localhost

#
# Imaginary network.
#10.0.0.2               myname
#10.0.0.3               myfriend
#
# According to RFC 1918, you can use the following IP networks for private
# nets which will never be connected to the Internet:
#
#       10.0.0.0        -   10.255.255.255
#       172.16.0.0      -   172.31.255.255
#       192.168.0.0     -   192.168.255.255
#
# In case you want to be able to connect directly to the Internet (i.e. not
# behind a NAT, ADSL router, etc...), you need real official assigned
# numbers.  Do not try to invent your own network numbers but instead get one
# from your network provider (if any) or from your regional registry (ARIN,
# APNIC, LACNIC, RIPE NCC, or AfriNIC.)

EOF

sed -i 's/keymap="us"/keymap="pt-latin9"/g' /etc/conf.d/keymaps
```

Change root password:

```bash
passwd
```

Take note of the ethernet interface - in my case the ethernet interface is named `enp31s0` - and configure it with `dhcpcd` and `netifrc`:

```bash
ifconfig
```

Configure network interfaces and update OpenRC services:

```bash
echo 'config_enp31s0="dhcp"' > /etc/conf.d/net
cd /etc/init.d
ln -s net.lo net.enp31s0

rc-update add dhcpcd default
rc-service dhcpcd start
rc-update add net.enp31s0 default
rc-update add sysklogd default
rc-update add cronie default
rc-update add sshd default
rc-update add chronyd default
rc-update add numlock default
```

### Finalizing the installation

Remove stage3 tarball previously downloaded. exit chroot and unmount drives:

```bash
rm /stage3-*.tar.*

exit
cd

umount -l /mnt/gentoo/dev{/shm,/pts,}
umount /dev/nvme0n1p1
umount -R /mnt/gentoo

reboot
```
