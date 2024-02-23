# Randomize disk content _(optional)_

Load random data to disk to obfuscate real encrypted data:

```bash
dd if=/dev/urandom of=/dev/nvme0n1p3
```

---

# Format backup disk _(optional)_

```bash
lsblk -o name,uuid
```

This section documents what I did to prepare a disk for backup storage:

```bash
fdisk /dev/sda
```

Use the following scheme:

```
Partition       Filesystem  Size	Description
/dev/sda1       LUKS        256GB   LUKS partition
# do not allocate anything else
```

Encrypt partition with `aes-xts-plain64` LUKS:

```bash
cryptsetup -c aes-xts-plain64 -s 512 -y luksFormat /dev/sda1
```

Open the LUKS partition so we can create the filesystem:

```bash
cryptsetup luksOpen /dev/sda1 rootbak
```

Create an ext4 filesystem on the LUKS partition:

```bash
mkfs.ext4 /dev/mapper/rootbak
```

---

# Create and add keyfile to LUKS _(optional)_

```bash
mkdir -p /etc/keys/
openssl genrsa -out /etc/keys/rootbak.key 4096

chmod -v 0400 /etc/keys/rootbak.key
chown root:root /etc/keys/rootbak.key

cryptsetup luksAddKey /dev/sda1 /etc/keys/rootbak.key
```

To open the LUKS partition with the keyfile:

```bash
cryptsetup luksClose rootbak
```

```bash
cryptsetup luksOpen /dev/sda1 rootbak --key-file /etc/keys/rootbak.key
```

```bash
lsblk -o name,uuid
```

- luksbak - 36a524c1-3312-4a65-b565-5db916f53ae5
- rootbak - 6be59466-94a1-45cb-a8dc-a038a1138a7c

---

# Mount encrypted backup disk _(optional)_

```bash
mkdir -p /mnt/backup

nano /etc/conf.d/dmcrypt
```

Add the following content to the file:

```bash
# /mnt/backup with regular keyfile (from /dev/mapper/rootbak)
target=rootbak
source=UUID="36a524c1-3312-4a65-b565-5db916f53ae5"
key=/etc/keys/rootbak.key

# An empty line is important at the end of the file
```

Mount the encrypted disk at boot with `fstab`:

```bash
cat << EOF >> /etc/fstab
# encrypted backup disk
UUID=6be59466-94a1-45cb-a8dc-a038a1138a7c   /mnt/backup                       ext4    defaults 0 0

EOF

rc-update add dmcrypt boot
```
