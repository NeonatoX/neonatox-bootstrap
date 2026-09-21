# NeonatoX Bootstrap

![Bash](https://img.shields.io/badge/bash-4EAA25?logo=gnubash&logoColor=fff)
![systemd](https://img.shields.io/badge/systemd-052A46?logo=systemd&logoColor=fff)
![btrfs](https://img.shields.io/badge/btrfs-2D8659?logo=btrfs&logoColor=fff)
![Linux](https://img.shields.io/badge/linux-FCC624?logo=linux&logoColor=000)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/cargabsj175/neonatox-bootstrap)

`neonatox-bootstrap` installs [NeonatoX](https://neonatox.github.io), a
Linux distribution built from scratch, onto the target filesystem. It runs
from any host distro —including NeonatoX itself (live-boot, netinstall)—:
it creates the system hierarchy, installs the `nhopkg` package manager,
generates the base configuration and deploys the KDE / GNOME / XFCE desktop
environments.

> Documentación en español: [`README-es.md`](README-es.md)

## Quick start

```bash
sudo ./neonatox-bootstrap -L /mnt core    # base system (glibc profile)
sudo ./neonatox-bootstrap -L /mnt kde     # KDE (host → chroot)
```

Examples with the new features (`--libc` and on-the-fly nhopkg):

```bash
# Base system pointing at the musl repositories (n27/x86_64-musl)
sudo ./neonatox-bootstrap --libc musl -L /mnt core

# GNOME desktop on a musl target
sudo ./neonatox-bootstrap --libc musl -L /mnt gnome

# nhopkg is always built (git+meson+ninja) as a single tree with
# the standard prefix (/usr /etc /var), installed into the target via
# DESTDIR; it acts both as the host-side driver (libs resolved by env
# inside $LFS) and as the final system nhopkg. If the host lacks some
# tool, the recipe enables static BusyBox.
sudo ./neonatox-bootstrap -L /mnt core
```

### Arguments

| Option | Default | Description |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` or `/mnt` | Target root directory |
| `--hostname <name>` | auto (DMI+random) | Machine name |
| `--device <dev>` | auto-detect | Device for fstab |
| `--uuid <uuid>` | auto-detect | UUID for fstab |
| `--fstype <type>` | auto-detect | Filesystem type |
| `-z, --timezone <zone>` | host or `America/Caracas` | Timezone |
| `-l, --locale <locale>` | `es_US.UTF-8` | System locale |
| `--libc {glibc\|musl}` | `glibc` | Target libc profile (temporary nhopkg recipe) |
| `--no-cleanup` | — | Skip post-install cleanup |
| `--pack-dir <dir>` | `bootstrap/packs/` | Package list files |

### Commands

| Command | Context | Description |
|---------|----------|-------------|
| `core` | host (`nhopkg --root`) | Base system: directories, nhopkg, dynamic config, base packages |
| `kde` | host → chroot (`chroot_run`) | KDE desktop + desktop-common |
| `gnome` | host → chroot (`chroot_run`) | GNOME desktop + desktop-common |
| `xfce` | host → chroot (`chroot_run`) | XFCE desktop + desktop-common |
| `user` | host → chroot | Create a system user |
| `grub` | host → chroot | Install GRUB |
| `chroot` | host → chroot | Interactive shell inside `$LFS` |

## Structure

```
neonatox-bootstrap          # main executable (bash)
bootstrap/
  data/etc/                 # static configuration files
    {passwd,group,profile,bashrc,os-release,inputrc,shells,...}
    polkit-1/rules.d/
    profile.d/
    skel/
  packs/                    # package lists (one per line)
    base                    # base system packages
    base-extra              # extra system packages
    desktop-common          # packages common to all DEs
    kde / kde-extra         # KF6+Plasma and KDE apps
    gnome / gnome-extra     # GNOME core and apps
    xfce / xfce-extra       # XFCE4 and apps
AGENTS.md                   # guide: usage, architecture and step-by-step flow
```

## Requirements

- Linux with `chroot`, `mount --bind`
- `sudo` for `blkid` and `mount`
- Internet connection to download nhopkg and packages
- **nhopkg is always built** — the system one is **never used**: its
  config belongs to the host flavor, not the target. A **single tree**
  (`git`, `meson` and `ninja` required) is built with the recipe for the
  selected `--libc` and installed into `$LFS` with the standard prefix
  (`--prefix=/usr --sysconfdir=/etc --localstatedir=/var`) via `DESTDIR`:
  1. It is the **host-side driver** during bootstrap — the libs/conf are
     resolved by env (`NHOPKG_CONF`, `NHOPKG_LIB`, `NHOPKG_UDEP_LIB`,
     `NHOPKG_DOWNLOAD_LIB`, `NHOPKG_CRYPTO_LIB`) pointing inside `$LFS`,
     so it operates on `--root $LFS`.
  2. It is the **final system nhopkg** — the baked `/usr`/`/etc`/`/var`
     paths are correct at boot, replacing the `nhopkg` package.
  If the host lacks some tool, the recipe enables static BusyBox
  automatically.

## Flow

See [`AGENTS.md`](AGENTS.md) for the step-by-step detail of each command,
chroot helpers (`chroot_prepare`, `chroot_run`, `chroot_cleanup`), flow
diagram and execution notes.

## Resources

- [NeonatoX](https://neonatox.github.io)
- [nhopkg](https://github.com/NeonatoX/neonatox-nhopkg)
