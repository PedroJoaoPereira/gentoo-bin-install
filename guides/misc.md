# List sorted disk usage _(hint)_

To be able to understand what is taking up space on the disk, we can use the following command:

```bash
du -sch * .[!.]* | sort -rh
```

# Regular cleanups _(hint)_

Removing _distfiles_ regularly can save a lot of space:

```bash
eclean-dist --deep
```

# To update system:

```bash
emerge --sync
emerge -uUDNav @world
emerge --depclean
```
