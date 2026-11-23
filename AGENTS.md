# AGENTS.md — NeonatoX Bootstrap

Guía de referencia única para colaboradores y asistentes de IA.
Fusiona la antigua `FLUJO.md`: descripción del proyecto, uso,
arquitectura, flujo paso a paso y notas de mantenimiento.

## Índice

1. [Qué es este repositorio](#qué-es-este-repositorio)
2. [Uso](#uso)
3. [Arquitectura](#arquitectura)
4. [Flujo detallado](#flujo-detallado)
5. [nhopkg — gestor de paquetes](#nhopkg--gestor-de-paquetes)
6. [Workflow btrfs + systemd-nspawn](#workflow-btrfs--systemd-nspawn)
7. [Estructura de archivos](#estructura-de-archivos)
8. [Problemas conocidos](#problemas-conocidos)
9. [Contribuir y planificar](#contribuir-y-planificar)

---

## Qué es este repositorio

`neonatox-bootstrap` instala [NeonatoX](https://neonatox.vegnux.com),
una distribución Linux desde cero, en el sistema de archivos destino. Se
ejecuta desde cualquier distro hospedera —incluida la propia NeonatoX
(live-boot, netinstall)—: crea la jerarquía del sistema, instala el gestor
de paquetes `nhopkg`, genera la configuración base y despliega los
escritorios KDE / GNOME / XFCE.

`neonatox-bootstrap` es un **único ejecutable en bash**. Los archivos de
configuración estáticos viven en `bootstrap/data/` (reflejan el `/`
destino); las listas de paquetes en `bootstrap/packs/` (un paquete por
línea). Los scripts originales `01-base-neonatox.sh` y `02-corespacks`
quedan intactos como referencia.

Se puede ejecutar desde el source tree o instalar con meson. La instalación
usa `/usr` + `/etc` (nunca modifica el host: `LFS=/` sigue prohibido):

```bash
meson setup build --prefix=/usr --sysconfdir=/etc
ninja -C build
sudo ninja -C build install                       # directo en /usr + /etc
DESTDIR=/tmp/nb-root ninja -C build install       # o staging para verificar
```

Instala el script en `/usr/bin/neonatox-bootstrap`, el config de ejemplo en
`/etc/neonatox-bootstrap.conf` y los datos/packs en
`/usr/share/neonatox-bootstrap/{data,packs}`. El mismo
`neonatox-bootstrap.conf` se lee desde la raíz del source tree.

---

## Uso

```
neonatox-bootstrap [-c <config>] [--lfs <dir>] [opciones] <comando>
```

Precedencia de configuración: **CLI > archivo de config > entorno > defaults**.
El config se busca en `-c/--config` → `NB_CONFIG` → `/etc/neonatox-bootstrap.conf`
(instalado) o `./neonatox-bootstrap.conf` (source tree).

### Comandos

| Comando | Contexto | Descripción |
|---------|----------|-------------|
| `core` | host (`nhopkg --root`) | Sistema base: directorios, nhopkg, config dinámica, paquetes core |
| `kde` | host → chroot (`chroot_run`) | Escritorio KDE + desktop-common |
| `gnome` | host → chroot (`chroot_run`) | Escritorio GNOME + desktop-common |
| `xfce` | host → chroot (`chroot_run`) | Escritorio XFCE + desktop-common |
| `user` | host → chroot | Crear usuario (`user --help` para opciones) |
| `grub` | host → chroot | Instalar GRUB (`grub --help` para opciones) |
| `chroot` | host → chroot | Shell interactivo dentro de `$LFS` |

### Opciones globales

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` o `/mnt` | Directorio raíz de destino (nunca `/`) |
| `-c, --config <archivo>` | `/etc/neonatox-bootstrap.conf` | Archivo de configuración (env `NB_CONFIG`) |
| `--create-sysroot {glibc\|musl}` | — | Modo propio: crea solo la jerarquía y sale (fija `LIBC`; sin fstab/paquetes/sentinels) |
| `--hostname <nombre>` | auto (DMI+random) | Nombre del equipo |
| `--device <disp>` | `findmnt`/`df` | Dispositivo del root para fstab (debe ser block device) |
| `--uuid <uuid>` | `blkid` | UUID del root para fstab |
| `--fstype <tipo>` | `blkid` | Tipo de FS del root, solo se escribe en fstab (nunca llama a `mkfs`) |
| `--subvol <nombre>` | detectado de `findmnt` | Subvolumen btrfs del root |
| `--root-opts <opts>` | `compress=zstd:3,noatime[,subvol=…]` / `defaults` | Reemplaza las opciones de montaje del root literalmente |
| `--root-passno <0\|1\|2>` | `0` (btrfs), `1` (otros) | Campo `pass` de la entrada root (dump queda `0`) |
| `--swap auto\|none\|<UUID>` | `auto` | Entrada swap en fstab |
| `--efi-uuid <uuid>` | `blkid` | UUID de la partición EFI para fstab |
| `--efi-mount <dir>` | detectado o `/boot` | Punto de montaje EFI en fstab |
| `--no-efi-entry` | — | Omite la entrada EFI autodetectada |
| `--no-fstab` | — | No genera `/etc/fstab` (ni detecta UUID/FS/swap/EFI) |
| `-z, --timezone <zona>` | host o `America/Caracas` | Zona horaria |
| `-l, --locale <locale>` | `en_US.UTF-8` | Locale del sistema |
| `--libc {glibc\|musl}` | `glibc` | Perfil de libc del target (receta de nhopkg temporal) |
| `--no-busybox` | — | Construye nhopkg sin BusyBox estático (PATH privado) |
| `--host-nhopkg` | — | Reutiliza el nhopkg del host como driver (salta la build; env `NB_HOST_NHOPKG=1`) |
| `--skip-sentinel` | — | No exige el sentinel de `core` para `chroot`/`user`/`grub` (env `NB_SKIP_SENTINEL=1`) |
| `--no-cleanup` | — | Salta `rm -rf /usr/src/* /tmp/*` al final |
| `--pack-dir <dir>` | `bootstrap/packs/` | Directorio de listas de paquetes |
| `-h, --help` | — | Muestra la ayuda |
| `-v, --version` | — | Muestra la versión |

**Modo `--create-sysroot`**: no admite combinarse con `core`/DE/`user`/`grub`/
`chroot`, no exige `nhopkg` en el host y tampoco crea `.core-installed`.
El particionado, formateo y montaje previos siguen siendo responsabilidad del
usuario. La jerarquía la genera `create_sysroot()`, la misma función que usa
`core` como primer paso.

### Ejemplos

```bash
sudo ./neonatox-bootstrap -L /mnt core                 # sistema base (glibc)
sudo ./neonatox-bootstrap -L /mnt kde                  # KDE (host → chroot)
sudo ./neonatox-bootstrap --libc musl -L /mnt core     # target musl (n27/x86_64-musl)
sudo ./neonatox-bootstrap --libc musl -L /mnt gnome    # GNOME sobre target musl
sudo ./neonatox-bootstrap -L /mnt --create-sysroot musl # solo jerarquía (layout musl)
sudo ./neonatox-bootstrap -L /mnt --no-fstab core      # no tocar /etc/fstab
sudo ./neonatox-bootstrap user --username neonatox -p <pass>
sudo ./neonatox-bootstrap grub --efi -L /mnt

# Instalado con meson (lee /etc/neonatox-bootstrap.conf):
sudo neonatox-bootstrap -L /mnt core
sudo neonatox-bootstrap -c /etc/neonatox-bootstrap.conf -L /mnt core
```

---

## Arquitectura

- **`core`** crea la jerarquía de directorios, copia `bootstrap/data/etc/*`,
  genera la config dinámica (`fstab` configurable o ausente con `--no-fstab`,
  `hostname`, `machine-id`, timezone, `locale.conf`), puebla la BD de nhopkg,
  construye o reutiliza nhopkg y despliega los packs base/base-extra (más `grub`).
- **`--create-sysroot {glibc|musl}`** expone solo la jerarquía como modo
  independiente: no genera fstab, no instala nada y no crea sentinels.
- **DE (`kde`/`gnome`/`xfce`)** instalan `desktop-common` + el pack del DE,
  ejecutan `systemctl enable lightdm` en chroot y limpian.
  - GNOME además elimina qt5/qt6; XFCE elimina qt6.
- **Driver de nhopkg**: por defecto el config de ejemplo activa el
  **perfil netinstall** → `HOST_NHOPKG=1`, `NHOPKG_GETTEXT=no`,
  `NHOPKG_USE_BUSYBOX=no`. El `nhopkg`, `zstd` y BusyBox los provee
  `neonatox-live-boot`; bootstrap no compila ni descarga estáticos.
  - **Modo host** (`HOST_NHOPKG=1`, `--host-nhopkg` o `NB_HOST_NHOPKG=1`):
    `nhopkg_build()` no hace nada (sin `git`/`meson`/`ninja` ni `DESTDIR`).
    `nhopkg_preflight_host()` exige `nhopkg`, `zstd`, `tar`, `sha256sum` y
    `wget` o `curl`. `nhopkg_host_conf_prepare()` respeta `NB_NHOPKG_CONF`
    y añade `NHOPKG_CONF_OVERRIDES` en una copia temporal. No se antepone
    el PATH privado de BusyBox. El nhopkg final se instala como paquete en
    `core` (`-RS nhopkg --no-check-deps`).
  - **Modo build** (`HOST_NHOPKG=0`): `nhopkg_build()` genera un único
    árbol con el **prefix estándar** (`--prefix=/usr --sysconfdir=/etc
    --localstatedir=/var`) y lo instala en `$LFS` vía `DESTDIR`. Ese árbol
    es el driver del host (por ruta absoluta, con las libs/conf exportadas
    hacia `$LFS`) y el nhopkg final del sistema. Se reconstruye cuando
    cambia el hash de la receta (marker `.recipe-<hash>` en
    `$LFS/etc/nhopkg/`), que cubre `--libc`, BusyBox, `repo-version` y
    `NHOPKG_MESON_ARGS`. Ver [nhopkg: modos de driver](#nhopkg-modos-de-driver).
- **Sentinels** (`/var/nhopkg/.core-installed`,
  `/var/nhopkg/.desktop-common-installed`) evitan reinstalaciones.

**Enlaces de interés:** [NeonatoX](https://neonatox.vegnux.com) ·
[nhopkg](https://github.com/cargabsj175/neonatox-nhopkg)

---

## Flujo detallado

### Helpers de chroot

| Helper | Descripción |
|--------|-------------|
| `chroot_prepare` | Monta `/proc`, `/dev`, `/sys`, `/run` en `$LFS` |
| `chroot_run <cmd>` | Ejecuta `chroot "$LFS" /bin/bash -c "<cmd>"` |
| `chroot_cleanup` | Desmonta la VFS en orden inverso |

### Flujo: core (host)

1. **Jerarquía de directorios** — `create_sysroot()`: `mkdir -p` y symlinks
   (`/bin → usr/bin`, `/lib64 → usr/lib`, `lib32` solo con `glibc`). Es la
   misma función que usa `--create-sysroot` cuando se invoca como modo propio.
2. **Archivos estáticos** — `cp -a bootstrap/data/etc/. → $LFS/etc/`, `skel/ → $LFS/root/`
3. **Symlinks del sistema** — `awk → gawk`, `sh → bash`, `mtab`
4. **Metadata de nhopkg** — `mkdir -p $LFS/var/nhopkg/{cache,files,logs,packages,repo}`
5. **Archivos dinámicos** — `generate_fstab()` (device/UUID/FS/swap/EFI
   configurables; `--no-fstab` lo omite), `hosts`/`hostname` (DMI+random),
   timezone (host), `locale.conf` (default `en_US.UTF-8`), `machine-id`
6. **Paquetes core** — `nhopkg --root $LFS -U` (poblar BD), `-RS busybox` (+ symlink `sh → busybox` para acceso temprano al shell), `-RS base base-extra`, `-RS grub --no-check-deps`
7. **nhopkg final en el target** — en modo build el tree único ya quedó en `$LFS` con prefix estándar `/usr`/`/etc`/`/var`, sustituyendo al paquete `nhopkg`; en modo host (netinstall) ese árbol no existe, así que va `-RS nhopkg --no-check-deps`
8. **Sentinel** — se crea `$LFS/var/nhopkg/.core-installed`

Todo lo anterior se ejecuta directamente sobre `$LFS` con `nhopkg --root`,
usando como driver el nhopkg del live (netinstall) o el instalado en `$LFS`.

### nhopkg: modos de driver

El config de ejemplo trae activo el **perfil netinstall**: `HOST_NHOPKG=1`,
`export NHOPKG_GETTEXT=no`, `export NHOPKG_USE_BUSYBOX=no`. El `nhopkg`,
`zstd` y BusyBox los provee `neonatox-live-boot`; bootstrap no descarga ni
compila estáticos.

1. **Modo host (netinstall)** — `HOST_NHOPKG=1` (o `--host-nhopkg` /
   `NB_HOST_NHOPKG=1`):
   - `nhopkg_build()` retorna sin hacer nada.
   - `nhopkg_preflight_host()` valida `nhopkg`, `zstd`, `tar`, `sha256sum` y
     `wget` o `curl` (nhopkg llama `zstd` desnudo; **no es applet de BusyBox**).
   - `nhopkg_host_conf_prepare()` usa `NB_NHOPKG_CONF` (o `NHOPKG_CONF`) y,
     si hay `NHOPKG_CONF_OVERRIDES`, los agrega a una copia temporal de la
     conf del live para no modificarla. `NHOPKG_GETTEXT` se exporta según el
     config.
   - No se antepone PATH privado de BusyBox (`NHOPKG_USE_BUSYBOX=no`).
   - El nhopkg final del target se instala como paquete
     (`-RS nhopkg --no-check-deps`) tras los packs base.
2. **Modo build** (`HOST_NHOPKG=0`) — solo si se desactiva el perfil netinstall:
   - `nhopkg_build()` corre meson con `--prefix=/usr --sysconfdir=/etc
     --localstatedir=/var` + `DESTDIR=$LFS`, creando un único árbol dentro
     de `$LFS`. Requiere `git`, `meson` y `ninja` del host.
   - El marker de rebuild es `.recipe-<hash>` en `$LFS/etc/nhopkg/`, calculado
     sobre **toda** la receta (libc, BusyBox, `repo-version`,
     `NHOPKG_MESON_ARGS`). Cambios de `n2026 → n27` o de args fuerzan rebuild.
   - Recetas por libc (ambas usan `n27`):
     - `glibc`: `repo-version=n27`, `repo-arch=x86_64`, `git-branch=n27`
     - `musl`: `binlocate=plocate`, `repo-version=n27`,
       `repo-arch=x86_64-musl`, `libc=musl`, `git-branch=musl`
   - `nhopkg_prepare()` reusa ese árbol: invoca `$LFS/usr/bin/nhopkg` por
     ruta absoluta (nunca anteponiendo el `/usr/bin` del target al `PATH`
     del host) y exporta `NHOPKG_CONF`, `NHOPKG_LIB`, `NHOPKG_UDEP_LIB`,
     `NHOPKG_DOWNLOAD_LIB`, `NHOPKG_CRYPTO_LIB` hacia `$LFS`.
   - `NHOPKG_CONF_OVERRIDES` se agrega al final de la conf del target en un
     bloque reemplazable (`# BEGIN/END neonatox-bootstrap overrides`).
   - BusyBox estático sigue disponible en esta rama (`use-busybox=yes` +
     `static-busybox=auto` + `static-zstd=auto`; `--no-busybox` lo apaga),
     para `nhouser` y tools que falten en el host. El PATH privado solo se
     inyecta en los hooks del chroot, nunca en el host.

El cleanup final borra el clon en `/tmp/nhopkg` y la conf temporal del host,
pero **conserva** el nhopkg ya instalado en `$LFS/usr`.

### Flujo: DE (kde / gnome / xfce)

Los tres comandos DE se ejecutan desde el **host** y usan `chroot_run`
para operar dentro de `$LFS`. Cada comando asegura `core` y
`desktop-common` antes de proceder.

**Desktop common (compartido):** `nhopkg --root $LFS -RS desktop-common`;
`chroot_prepare` → `chroot_run "systemctl enable lightdm"` →
`chroot_cleanup`; se crea `/var/nhopkg/.desktop-common-installed`.

La contraseña de root se gestiona con:

```
neonatox-bootstrap user --username root -p <contraseña>
```

**Paquetes del DE** (desde el host, `nhopkg --root $LFS`):

| DE | Paquetes | Remociones |
|----|----------|------------|
| KDE | `-RS kde kde-extra` | — |
| GNOME | `-RS gnome gnome-extra` | `-Rr qt5 qt6` |
| XFCE | `-RS xfce xfce-extra` | `-Rr qt6` |

### Dependencias entre comandos

- `core` debe ejecutarse **primero** (desde el host).
- `kde`/`gnome`/`xfce` asumen que `core` ya pobló `$LFS`; si el sentinel
  falta, ejecutan `core` y `desktop-common` automáticamente.
- `user` y `grub` requieren `core` instalado (verifican el sentinel).
- `chroot` requiere `core` instalado (verifica el sentinel).
- Los tres gates anteriores usan `requiere_core()`, que se salta con
  `--skip-sentinel` / `NB_SKIP_SENTINEL=1` (sistemas o instalaciones sin
  sentinel); exige sí o sí que `$LFS` exista. No afecta a la
  idempotencia de `core` (sigue saltándose si el sentinel existe) ni a
  los auto-triggers de los DEs (`core_instalado || install_core`).

### Diagrama

```
neonatox-bootstrap
    │
    ├── core (host)      → mkdir → cp data/etc → symlinks →
    │                      metadata nhopkg → archivos dinámicos
    │                      (fstab, hostname, timezone, machine-id)
    │                      → nhopkg_build/prepare (tree único) → -U →
    │                      -RS packs → sentinel
    │
    ├── user/grub (host→chroot)
    │                      verificar sentinel core → chroot_prepare →
    │                      useradd/chpasswd | grub-install/grub-mkconfig
    │                      → chroot_cleanup
    │
    ├── DEs (host→chroot) → asegurar core + desktop-common →
    │                      systemctl enable lightdm →
    │                      nhopkg --root -RS <de> <de>-extra →
    │                      remociones (QT según DE) → cleanup
    │
    └── chroot interactivo → chroot_prepare → shell → chroot_cleanup
```

---

## nhopkg — gestor de paquetes

- `nhopkg --root $LFS -U` — poblar BD en el target
- `nhopkg --root $LFS -S <pkg>` / `-r <pkg>`
- `nhopkg -S <pkg> --no-check-deps` — saltar chequeo de dependencias
- `nhopkg --install-group ttf-fonts` — instalar un grupo
- `NHOPKG_CONF` — ruta de configuración (default
  `@sysconfdir@/nhopkg/nhopkg.conf`); recuerda
  `nhopkg -S pam` tras crear usuarios (permisos de `libpwquality`).

---

## Workflow btrfs + systemd-nspawn

(notas en `testing_neoanatox_btrfs`)

- Formatear: `mkfs.btrfs -L neonatox -f /dev/sda3`
- Crear subvolumen `@core`, montar con `-o subvol=@core`
- Snapshot → `@kde`, `@gnome`, `@xfce`
- Montar en `/var/lib/machines/<nombre>` para `systemd-nspawn`

---

## Estructura de archivos

```
neonatox-bootstrap              # ejecutable principal (bash)
neonatox-bootstrap.conf         # config de ejemplo (perfil netinstall activo)
meson.build                     # instalación en /usr + /etc
bootstrap/
  data/
    etc/                        # configuración estática (refleja /)
      {passwd,group,profile,bashrc,os-release,inputrc,shells,...}
      polkit-1/rules.d/00-mount-internal.rules
      profile.d/{extrapaths,readline,umask,i18n}.sh
      default/grub
      skel/{.bash_profile,.bashrc,.profile}
  packs/                        # listas de paquetes (uno por línea)
    base / base-extra           # sistema base
    desktop-common              # común a todos los DE
    kde / kde-extra             # KF6+Plasma y apps KDE
    gnome / gnome-extra         # GNOME core y apps
    xfce / xfce-extra           # XFCE4 y apps
01-base-neonatox.sh             # referencia original (intacto)
02-corespacks                   # referencia original (intacto)
PROPOSALS/                      # propuestas de features futuras
AGENTS.md                       # este archivo
README.md                       # documentación pública (inglés)
README-es.md                    # documentación pública (español)
NEWS                            # changelog por release
release-0.2.0.md                # notas de la release 0.2.0
release-0.3.0.md                # notas de la release 0.3.0
testing_neoanatox_btrfs         # notas workflow btrfs + systemd-nspawn
```

---

## Problemas conocidos

- `drkonqi` requiere `gdb` (dep no declarada)
- `transmission-qt` requiere `qt6-core` (dep no declarada)
- `vim` tiene dependencia errónea con `gpm`
- `gnome-control-center` necesita `udisks2` (corregir dep)
- Reinstalar `nhopkg -S pam` tras crear usuarios para permisos correctos
  de `libpwquality`

---

## Contribuir y planificar

- Solo hacer **commit** cuando el usuario lo indique explícitamente.
- Las features futuras se registran en `PROPOSALS/PROPOSAL-*.md`,
  interactuando con el usuario **antes** de escribir código.