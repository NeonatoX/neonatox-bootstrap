# NeonatoX Bootstrap

![Bash](https://img.shields.io/badge/bash-4EAA25?logo=gnubash&logoColor=fff)
![systemd](https://img.shields.io/badge/systemd-052A46?logo=systemd&logoColor=fff)
![btrfs](https://img.shields.io/badge/btrfs-2D8659?logo=btrfs&logoColor=fff)
![Linux](https://img.shields.io/badge/linux-FCC624?logo=linux&logoColor=000)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/cargabsj175/neonatox-bootstrap)

`neonatox-bootstrap` instala [NeonatoX](https://neonatox.github.io),
una distribución Linux desde cero, en el sistema de archivos destino.
Se ejecuta desde cualquier distro hospedera —incluida la propia NeonatoX
(live-boot, netinstall)—: crea la jerarquía del sistema, instala el
gestor de paquetes `nhopkg`, genera la configuración base y despliega los
entornos de escritorio KDE / GNOME / XFCE. El target es NeonatoX `n27`
("Rolling Gatiste"), tanto para glibc como para musl.

## Instalación

Se puede ejecutar desde el source tree o instalar en `/usr` + `/etc` con
meson (la instalación nunca toca el host: `LFS=/` sigue prohibido):

```bash
meson setup build --prefix=/usr --sysconfdir=/etc
ninja -C build
sudo ninja -C build install
```

Instala:

| Ruta | Contenido |
|------|-----------|
| `/usr/bin/neonatox-bootstrap` | ejecutable |
| `/etc/neonatox-bootstrap.conf` | config de ejemplo (perfil netinstall activo) |
| `/usr/share/neonatox-bootstrap/data` | configuración estática del target |
| `/usr/share/neonatox-bootstrap/packs` | listas de paquetes |

El mismo `neonatox-bootstrap.conf` se lee desde la raíz del source tree.
Precedencia: **CLI > archivo de config > entorno > defaults**. El config se
busca en `-c/--config` → `NB_CONFIG` → `/etc/neonatox-bootstrap.conf`
(instalado) o `./neonatox-bootstrap.conf` (source tree).

## Uso rápido

```bash
sudo neonatox-bootstrap -L /mnt core      # sistema base (perfil glibc)
sudo neonatox-bootstrap -L /mnt kde       # KDE (host → chroot)

# Solo el esqueleto del sysroot (layout musl), sin paquetes ni config generada:
sudo neonatox-bootstrap -L /mnt --create-sysroot musl
```

Ejemplos con las características nuevas (`--libc` y selección de driver de nhopkg):

```bash
# Sistema base que apunta a los repos musl (n27/x86_64-musl)
sudo neonatox-bootstrap --libc musl -L /mnt core

# Escritorio GNOME sobre un target musl
sudo neonatox-bootstrap --libc musl -L /mnt gnome

# Perfil netinstall (activo en el config incluido): reutiliza el nhopkg del
# neonatox-live-boot. Sin clone, sin git/meson/ninja y sin estáticos.
# zstd y BusyBox los provee la imagen live.
sudo neonatox-bootstrap -L /mnt core

# Lo mismo, de forma explícita / vía entorno:
sudo neonatox-bootstrap --host-nhopkg -L /mnt core
sudo NB_HOST_NHOPKG=1 neonatox-bootstrap -L /mnt core

# Modo build: si el config fija HOST_NHOPKG=0, se genera un único árbol
# con el prefix estándar (/usr /etc /var) instalado en el target vía
# DESTDIR; sirve de driver del host y de nhopkg final. BusyBox estático
# está disponible en esta rama (PATH privado solo dentro del chroot).
sudo HOST_NHOPKG=0 neonatox-bootstrap -L /mnt core

# Archivo de configuración alternativo:
sudo neonatox-bootstrap -c /etc/neonatox-bootstrap.conf -L /mnt core

# Entrada root explícita en fstab (sin swap ni entrada EFI):
sudo neonatox-bootstrap -L /mnt --device /dev/sda2 --fstype ext4 \
    --uuid 1234-abcd --root-opts noatime --root-passno 1 --swap none \
    --no-efi-entry core

# Dejar intacto un /etc/fstab existente:
sudo neonatox-bootstrap -L /mnt --no-fstab core
```

### Argumentos

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` o `/mnt` | Directorio raíz de destino (nunca `/`) |
| `-c, --config <archivo>` | `/etc/neonatox-bootstrap.conf` | Archivo de configuración (env `NB_CONFIG`) |
| `--create-sysroot {glibc\|musl}` | — | Crea **solo** la jerarquía de directorios y sale (fija `LIBC`; sin fstab, paquetes ni sentinels) |
| `--hostname <nombre>` | auto (DMI+random) | Nombre del equipo |
| `--device <disp>` | `findmnt`/`df` | Dispositivo del root para fstab (debe ser block device) |
| `--uuid <uuid>` | `blkid` | UUID del root para fstab |
| `--fstype <tipo>` | `blkid` | Tipo de FS del root **escrito en fstab** (bootstrap nunca ejecuta `mkfs`) |
| `--subvol <nombre>` | detectado de `findmnt` | Subvolumen btrfs del root |
| `--root-opts <opts>` | `compress=zstd:3,noatime[,subvol=…]` / `defaults` | Reemplaza las opciones de montaje del root literalmente |
| `--root-passno <0\|1\|2>` | `0` (btrfs), `1` (otros) | Campo `pass` de la entrada root (dump sigue `0`) |
| `--swap auto\|none\|<UUID>` | `auto` | Entrada swap en fstab |
| `--efi-uuid <uuid>` | `blkid` | UUID de la partición EFI para fstab |
| `--efi-mount <dir>` | detectado o `/boot` | Punto de montaje EFI en fstab |
| `--no-efi-entry` | — | Omite la entrada EFI autodetectada |
| `--no-fstab` | — | No genera `/etc/fstab` (ni detecta UUID/FS/swap/EFI) |
| `-z, --timezone <zona>` | host o `America/Caracas` | Zona horaria |
| `-l, --locale <locale>` | `en_US.UTF-8` | Locale del sistema |
| `--libc {glibc\|musl}` | `glibc` | Perfil de libc del target (repos `n27` para ambos) |
| `--no-busybox` | — | Construye nhopkg sin BusyBox estático (solo modo build) |
| `--host-nhopkg` | — | Reutiliza el nhopkg del host como driver (ya es el default del config incluido; env `NB_HOST_NHOPKG=1`) |
| `--skip-sentinel` | — | No exige el sentinel de `core` para `chroot`/`user`/`grub` (env `NB_SKIP_SENTINEL=1`) |
| `--no-cleanup` | — | Salta limpieza post-instalación |
| `--pack-dir <dir>` | `bootstrap/packs/` | Listas de paquetes |

`--create-sysroot` es un modo propio: no se combina con otros comandos y no
exige `nhopkg` en el `PATH` del host. El particionado, formateo y montaje del
destino siguen siendo responsabilidad de quien llama al script.

### Comandos

| Comando | Contexto | Descripción |
|---------|----------|-------------|
| `core` | host (`nhopkg --root`) | Sistema base: directorios, nhopkg, config dinámica, paquetes base |
| `kde` | host → chroot (`chroot_run`) | Escritorio KDE + desktop-common |
| `gnome` | host → chroot (`chroot_run`) | Escritorio GNOME + desktop-common |
| `xfce` | host → chroot (`chroot_run`) | Escritorio XFCE + desktop-common |
| `user` | host → chroot | Crear usuario del sistema |
| `grub` | host → chroot | Instalar GRUB |
| `chroot` | host → chroot | Shell interactivo dentro de `$LFS` |

## Estructura

```
neonatox-bootstrap          # ejecutable principal (bash)
neonatox-bootstrap.conf     # config de ejemplo (perfil netinstall activo)
meson.build                 # instalación en /usr + /etc
bootstrap/
  data/etc/                 # archivos de configuración estáticos
    {passwd,group,profile,bashrc,os-release,inputrc,shells,...}
    polkit-1/rules.d/
    profile.d/
    skel/
  packs/                    # listas de paquetes (uno por línea)
    base                    # paquetes base del sistema
    base-extra              # paquetes extra del sistema
    desktop-common          # paquetes comunes a todos los DE
    kde / kde-extra         # KF6+Plasma y apps KDE
    gnome / gnome-extra     # GNOME core y apps
    xfce / xfce-extra       # XFCE4 y apps
AGENTS.md                   # guía: uso, arquitectura y flujo paso a paso
README.md                   # documentación pública (inglés)
```

## Requisitos

- Linux con `chroot`, `mount --bind`
- `sudo` para `blkid` y `mount`
- Conexión a internet para descargar los paquetes
- **Perfil netinstall (default en el config incluido)**: `HOST_NHOPKG=1`,
  `NHOPKG_GETTEXT=no` y `NHOPKG_USE_BUSYBOX=no`. `neonatox-live-boot`
  provee `nhopkg`, `zstd` y BusyBox; bootstrap no compila ni descarga
  estáticos. Un preflight valida `nhopkg`, `zstd`, `tar`, `sha256sum` y
  `wget`/`curl` al arrancar (`zstd` es llamado directamente por nhopkg y
  **no es applet de BusyBox**). `NB_NHOPKG_CONF` selecciona la conf del
  live y `NHOPKG_CONF_OVERRIDES` agrega ajustes a una copia temporal.
  El nhopkg final del target se instala como paquete
  (`nhopkg -RS nhopkg --no-check-deps`) durante `core`.
- **Modo build** (`HOST_NHOPKG=0` en el config): se genera un **único
  árbol** (`git`, `meson` y `ninja` requeridos) con la receta del `--libc`
  (`n27` para ambos), instalado en `$LFS` vía `DESTDIR` con el prefix
  estándar (`--prefix=/usr --sysconfdir=/etc --localstatedir=/var`). Sirve
  de driver del host (por ruta absoluta, con libs/conf hacia `$LFS`) y de
  nhopkg final del sistema. El marker de rebuild es un hash de toda la
  receta (`.recipe-<hash>`), así que cambios de libc, BusyBox,
  `repo-version` o `NHOPKG_MESON_ARGS` fuerzan reconstrucción. BusyBox
  estático está disponible en este modo (PATH privado solo en el chroot);
  `--no-busybox` lo desactiva.

## Flujo

Ver [`AGENTS.md`](AGENTS.md) para el detalle paso a paso de cada comando,
helpers de chroot (`chroot_prepare`, `chroot_run`, `chroot_cleanup`),
diagrama de flujo y notas de ejecución.

## Recursos

- [NeonatoX](https://neonatox.github.io)
- [nhopkg](https://github.com/NeonatoX/neonatox-nhopkg)
