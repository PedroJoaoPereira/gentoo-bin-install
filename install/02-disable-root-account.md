# Disable root account _(optional)_

To reduce the attack surface of the system, the root account can be disabled. This is a security measure that can be taken to prevent unauthorized access to the system. The following steps are to be executed in the environment as a super user.

Table of contents:

- [Disable root account _(optional)_](#disable-root-account-optional)
    - [Configuration](#configuration)

---

### Configuration

In this guide we use `doas` as a replacement for the usual `sudo`. The following steps are to be executed in the environment as a super user.

```bash
echo "app-admin/doas persist" > /etc/portage/package.use/doas
emerge app-admin/doas

cat << EOF > /etc/doas.conf
# https://wiki.gentoo.org/wiki/Doas
permit  persist :wheel
permit  nopass  :wheel as root  cmd shutdown
permit  nopass  :wheel as root  cmd reboot
EOF

chown -c root:root /etc/doas.conf
```

Setup new user and groups, as well as its password:

```bash
useradd -m -G users,wheel,audio,video -s /bin/bash celestial
chown -R -c celestial:celestial /home/celestial

passwd celestial
```

Make the root account not accessible:

```bash
su celestial
doas passwd -l root
```

```bash
exit
```
