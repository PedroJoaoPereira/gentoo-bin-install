# Journaling with BTRFS and backup system _(optional)_

Guide on how to setup journaling for easily rolling back versioned snapshots of the system and how to setup a backup system.

Table of contents:

- [Journaling with BTRFS and backup system _(optional)_](#journaling-with-btrfs-and-backup-system-optional)
  - [Configuration](#configuration)

---

### Configuration

In this guide we use `snapper` to create snapshots and `borgbackup` to create backups. The following steps are to be executed in the environment as a super user.

```bash
emerge app-backup/snapper app-backup/borgbackup

rc-service dbus start
rc-update add dbus default

snapper --config root create-config /
snapper --config home create-config /home/celestial/space

snapper -c root set-config ALLOW_USERS=${LOGNAME} SYNC_ACL=yes
snapper -c home set-config ALLOW_USERS=${LOGNAME} SYNC_ACL=yes

chown -R :${LOGNAME} /.snapshots
chown -R :${LOGNAME} /home/celestial/space/.snapshots

mkdir -p /etc/borg
cat << EOF > /etc/borg/exclude.conf
# https://borgbackup.readthedocs.io/en/stable/quickstart.html#automating-backups

sh:/dev/*
sh:/proc/*
sh:/run/*
sh:/sys/*

sh:/home/*

sh:/.snapshots/*
sh:/media/*
sh:/mnt/*
sh:/root/*
sh:/tmp/*
sh:/usr/tmp/*

sh:/var/cache/*
sh:/var/crash/*
sh:/var/log/*
sh:/var/tmp/*

EOF
```

Change configuration files for generated timeline snapshots:

```bash
nano /etc/snapper/configs/root
nano /etc/snapper/configs/home
```

Create backup repository:

```bash
borg init --encryption=repokey --make-parent-dirs /mnt/backup/terra/local/manual
```

Create snapshot and backup:

```bash
borg create --stats --list --exclude-from /etc/borg/exclude.conf --compression zstd,3 /mnt/backup/terra/local/manual::"base_system_install" /

# delete all snapshots starting on index 1
snapper -c root delete $(snapper -c root list --columns number | tail +4)
snapper -c home delete $(snapper -c home list --columns number | tail +4)

snapper -c root create -d "base_system_install"
snapper -c home create -d "base_system_install"

snapper -c root list
snapper -c home list
```
