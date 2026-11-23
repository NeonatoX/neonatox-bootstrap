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
environments. The target is NeonatoX `n27` ("Rolling Gatiste"), for both
glibc and musl.

> Documentación en español: [`README-es.md`](README-es.md)

## Install

It can be run from the source tree or installed under `/usr` + `/etc` with
meson (installing never touches the host: `LFS=/` is still refused):

```bash
meson setup build --prefix=/usr --sysconfdir=/etc
ninja -C build
sudo ninja -C build install
```

This installs:

| Path | Content |
|------|---------|
| `/usr/bin/neonatox-bootstrap` | executable |
| `/etc/neonatox-bootstrap.conf` | example config (netinstall profile active) |
| `/usr/share/neonatox-bootstrap/data` | static target configuration |
| `/usr/share/neonatox-bootstrap/packs` | package lists |

The very same `neonatox-bootstrap.conf` is read from the source-tree root
when running `./neonatox-bootstrap`. Configuration precedence is
**CLI > config file > environment > script defaults**. The config is looked
up in `-c/--config` → `NB_CONFIG` → `/etc/neonatox-bootstrap.conf` (installed)
or `./neonatox-bootstrap.conf` (source tree).

## Quick start

```bash
sudo neonatox-bootstrap -L /mnt core      # base system (glibc profile)
sudo neonatox-bootstrap -L /mnt kde       # KDE (host → chroot)

# Only the sysroot skeleton (musl layout), no packages or generated config:
sudo neonatox-bootstrap -L /mnt --create-sysroot musl
```

Examples with the new features (`--libc` and nhopkg driver selection):

```bash
# Base system pointing at the musl repositories (n27/x86_64-musl)
sudo neonatox-bootstrap --libc musl -L /mnt core

# GNOME desktop on a musl target
sudo neonatox-bootstrap --libc musl -L /mnt gnome

# Netinstall profile (active in the shipped config): reuse the nhopkg from
# neonatox-live-boot. No clone, no git/meson/ninja, no static BusyBox/zstd.
# zstd and BusyBox are provided by the live image.
sudo neonatox-bootstrap -L /mnt core

# Same, selected explicitly / via env:
sudo neonatox-bootstrap --host-nhopkg -L /mnt core
sudo NB_HOST_NHOPKG=1 neonatox-bootstrap -L /mnt core

# Build mode: if HOST_NHOPKG=0 is set in the config, a single nhopkg tree is
# built with the standard prefix (/usr /etc /var) and installed into the
# target via DESTDIR; it is both the host-side driver and the final nhopkg.
# Static BusyBox (private PATH) is available there; --no-busybox disables it.
sudo HOST_NHOPKG=0 neonatox-bootstrap -L /mnt core

# Alternate config file:
sudo neonatox-bootstrap -c /etc/neonatox-bootstrap.conf -L /mnt core

# Fully explicit root entry in fstab (no swap, no EFI entry):
sudo neonatox-bootstrap -L /mnt --device /dev/sda2 --fstype ext4 \
    --uuid 1234-abcd --root-opts noatime --root-passno 1 --swap none \
    --no-efi-entry core

# Keep an existing /etc/fstab untouched:
sudo neonatox-bootstrap -L /mnt --no-fstab core
```

### Arguments

| Option | Default | Description |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` or `/mnt` | Target root directory (never `/`) |
| `-c, --config <file>` | `/etc/neonatox-bootstrap.conf` | Config file (env `NB_CONFIG`) |
| `--create-sysroot {glibc\|musl}` | — | Create **only** the target directory tree and exit (sets `LIBC`; no fstab, packages or sentinels) |
| `--hostname <name>` | auto (DMI+random) | Machine name |
| `--device <dev>` | `findmnt`/`df` | Root device for fstab (must be a block device) |
| `--uuid <uuid>` | `blkid` | Root UUID for fstab |
| `--fstype <type>` | `blkid` | Root filesystem type **written in fstab** (bootstrap never runs `mkfs`) |
| `--subvol <name>` | detected from `findmnt` | btrfs subvolume for the root entry |
| `--root-opts <opts>` | `compress=zstd:3,noatime[,subvol=…]` / `defaults` | Replace root mount options verbatim |
| `--root-passno <0\|1\|2>` | `0` (btrfs), `1` (others) | `pass` field of the root entry (dump stays `0`) |
| `--swap auto\|none\|<UUID>` | `auto` | Swap entry in fstab |
| `--efi-uuid <uuid>` | `blkid` | EFI partition UUID for fstab |
| `--efi-mount <dir>` | detected or `/boot` | EFI mount point in fstab |
| `--no-efi-entry` | — | Omit the auto-detected EFI entry |
| `--no-fstab` | — | Do not generate `/etc/fstab` (no UUID/FS/swap/EFI detection either) |
| `-z, --timezone <zone>` | host or `America/Caracas` | Timezone |
| `-l, --locale <locale>` | `en_US.UTF-8` | System locale |
| `--libc {glibc\|musl}` | `glibc` | Target libc profile (`n27` repositories for both) |
| `--no-busybox` | — | Build nhopkg without static BusyBox (build mode only) |
| `--host-nhopkg` | — | Reuse the host's `nhopkg` as driver (already the default in the shipped config; env `NB_HOST_NHOPKG=1`) |
| `--skip-sentinel` | — | Don't require the `core` sentinel for `chroot`/`user`/`grub` (env `NB_SKIP_SENTINEL=1`) |
| `--no-cleanup` | — | Skip post-install cleanup |
| `--pack-dir <dir>` | `bootstrap/packs/` | Package list files |

`--create-sysroot` is a standalone mode: it cannot be combined with another
command and does not require `nhopkg` in the host `PATH`. Partitioning,
formatting and mounting the target are still the caller's responsibility.

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
neonatox-bootstrap.conf     # example config (netinstall profile active)
meson.build                 # install under /usr + /etc
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
README-es.md                # Spanish documentation
```

## Requirements

- Linux with `chroot`, `mount --bind`
- `sudo` for `blkid` and `mount`
- Internet connection to download the packages
- **Netinstall profile (default in the shipped config)**: `HOST_NHOPKG=1`,
  `NHOPKG_GETTEXT=no` and `NHOPKG_USE_BUSYBOX=no`. `neonatox-live-boot`
  provides `nhopkg`, `zstd` and BusyBox; bootstrap does not build or
  download static tools. A preflight checks `nhopkg`, `zstd`, `tar`,
  `sha256sum` and `wget`/`curl` early (`zstd` is called directly by
  nhopkg and is **not** a BusyBox applet). `NB_NHOPKG_CONF` selects the
  live config and `NHOPKG_CONF_OVERRIDES` appends settings to a temporary
  copy of it. The final system nhopkg is installed as a package
  (`nhopkg -RS nhopkg --no-check-deps`) during `core`.
- **Build mode** (`HOST_NHOPKG=0` in the config): a **single tree** (`git`,
  `meson` and `ninja` required) is built with the recipe for the selected
  `--libc` (`n27` for both) and installed into `$LFS` with the standard
  prefix (`--prefix=/usr --sysconfdir=/etc --localstatedir=/var`) via
  `DESTDIR`. It is both the host-side driver (invoked by absolute path,
  with libs/conf resolved by env inside `$LFS`) and the final system
  nhopkg. The rebuild marker is a hash of the whole recipe
  (`.recipe-<hash>`), so changes to libc, BusyBox, `repo-version` or
  `NHOPKG_MESON_ARGS` force a rebuild. Static BusyBox is available in this
  mode (private PATH applied only inside the chroot); `--no-busybox`
  disables it.

## Flow

See [`AGENTS.md`](AGENTS.md) for the step-by-step detail of each command,
chroot helpers (`chroot_prepare`, `chroot_run`, `chroot_cleanup`), flow
diagram and execution notes.

## Resources

- [NeonatoX](https://neonatox.github.io)
- [nhopkg](https://github.com/NeonatoX/neonatox-nhopkg)
