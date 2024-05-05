# Optimize installation _(optional)_

After the installation, it is possible to optimize the system by setting up a few features.

Table of contents:

- [Optimize installation _(optional)_](#optimize-installation-optional)
    - [Configuration](#configuration)

---

### Configuration

Add no timeout to `GRUB` settings on `/etc/default/grub` to speed up boot time:

```bash
GRUB_TIMEOUT=0
```

And refresh `GRUB` settings:

```bash
sed -i 's/#GRUB_TIMEOUT=5/GRUB_TIMEOUT=0/g' /etc/default/grub
sed -i 's/#GRUB_CMDLINE_LINUX_DEFAULT=""/GRUB_CMDLINE_LINUX_DEFAULT="quiet video=efifb:nobgrt"/g' /etc/default/grub
sed -i 's/#GRUB_DISABLE_SUBMENU=y/GRUB_DISABLE_SUBMENU=y/g' /etc/default/grub

grub-mkconfig -o /boot/grub/grub.cfg
```
