# Hyprland

This guide allows us to install Hyprland using the official portage tree and without display managers.

Update the `USE` flags in `/etc/portage/make.conf` to contain `elogind wayland -X` and update the system:

```bash
doas emerge -uUDNav @world
doas emerge --depclean
```

Install `hyprland` and `kitty`:

```bash
echo 'x11-libs/libdrm video_cards_radeon' | doas tee -a /etc/portage/package.use/libdrm > /dev/null

doas emerge --ask=n gui-wm/hyprland x11-terms/kitty

doas rc-update add elogind default

# running Hyprland so the auto .config files get generated
# if it is executed, quit it
Hyprland
```

You should reboot the system to make sure everything is working properly - specifically the `elogind` service.

```bash
doas reboot
```

After rebooting, we add the default configuration for `hyprland` and make it start on login:

```bash
sed -i 's/autogenerated = 1/#autogenerated = 1/g' ~/.config/hypr/hyprland.conf
sed -i 's/kb_layout = us/kb_layout = pt/g' ~/.config/hypr/hyprland.conf
sed -i 's/force_default_wallpaper = -1/force_default_wallpaper = 0/g' ~/.config/hypr/hyprland.conf

cat << EOF > ~/.bash_profile
# run Hyprland on login from tty
# do not run within an SSH-session
if [ -z "\$SSH_CLIENT" ] && [ -z "\$SSH_TTY" ]; then
    dbus-run-session Hyprland > /dev/null 2>&1
fi

EOF
```
