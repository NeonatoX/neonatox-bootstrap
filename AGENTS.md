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

---

## Uso

```
neonatox-bootstrap [--lfs <dir>] [opciones] <comando>
```

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
| `-L, --lfs <dir>` | `$LFS` o `/mnt` | Directorio raíz de destino |
| `--hostname <nombre>` | auto (DMI+random) | Nombre del equipo |
| `--device <disp>` | auto-detect | Dispositivo para fstab |
| `--uuid <uuid>` | auto-detect | UUID para fstab |
| `--fstype <tipo>` | auto-detect | Tipo de sistema de archivos |
| `-z, --timezone <zona>` | host o `America/Caracas` | Zona horaria |
| `-l, --locale <locale>` | `es_US.UTF-8` | Locale del sistema |
| `--libc {glibc\|musl}` | `glibc` | Perfil de libc del target (receta de nhopkg temporal) |
| `--no-cleanup` | — | Salta `rm -rf /usr/src/* /tmp/*` al final |
| `--pack-dir <dir>` | `bootstrap/packs/` | Directorio de listas de paquetes |
| `-h, --help` | — | Muestra la ayuda |
| `-v, --version` | — | Muestra la versión |

### Ejemplos

```bash
sudo ./neonatox-bootstrap -L /mnt core                 # sistema base (glibc)
sudo ./neonatox-bootstrap -L /mnt kde                  # KDE (host → chroot)
sudo ./neonatox-bootstrap --libc musl -L /mnt core     # target musl (n27/x86_64-musl)
sudo ./neonatox-bootstrap --libc musl -L /mnt gnome    # GNOME sobre target musl
sudo ./neonatox-bootstrap user --username neonatox -p <pass>
sudo ./neonatox-bootstrap grub --efi -L /mnt
```

---

## Arquitectura

- **`core`** crea la jerarquía de directorios, copia `bootstrap/data/etc/*`,
  genera la config dinámica (`fstab`, `hostname`, `machine-id`, timezone),
  puebla la BD de nhopkg e instala los packs base/base-extra (más `nhopkg`
  y `grub`).
- **DE (`kde`/`gnome`/`xfce`)** instalan `desktop-common` + el pack del DE,
  ejecutan `systemctl enable lightdm` en chroot y limpian.
  - GNOME además elimina qt5/qt6; XFCE elimina qt6.
- **nhopkg se construye siempre como árbol temporal en el host.** El
  nhopkg del sistema **no se usa**: su config corresponde al flavor del
  host, no al del target, así que no garantiza los repos del `--libc`
  elegido. `nhopkg_prepare()` construye (o recupera) `/tmp/nhopkg-tools`
  con la receta meson del `--libc` (ver
  [nhopkg al vuelo](#nhopkg-al-vuelo-driver-del-host));
  si el árbol ya existe, se reutiliza.
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

1. **Jerarquía de directorios** — `mkdir -pv` y symlinks (`/bin → usr/bin`, etc.)
2. **Archivos estáticos** — `cp -a bootstrap/data/etc/. → $LFS/etc/`, `skel/ → $LFS/root/`
3. **Symlinks del sistema** — `awk → gawk`, `sh → bash`, `mtab`
4. **Metadata de nhopkg** — `mkdir -p $LFS/var/nhopkg/{cache,files,logs,packages,repo}`
5. **Archivos dinámicos** — `fstab` (blkid), `hosts`/`hostname` (DMI+random), timezone (host), `machine-id`
6. **Paquetes core** — `nhopkg --root $LFS -U` (poblar BD), `-RS nhopkg --no-check-deps`, `-RS base base-extra`, `-RS grub --no-check-deps`
7. **Sentinel** — se crea `$LFS/var/nhopkg/.core-installed`

Todo se ejecuta directamente sobre `$LFS` con `nhopkg --root`.

### nhopkg al vuelo (driver del host)

`neonatox-bootstrap` construye **siempre** su propio nhopkg: el del
sistema no se usa porque su config corresponde al flavor del host, no al
del target. Antes del primer uso se llama `nhopkg_prepare()`, que:

1. **Build temporal** — receta meson del `--libc` elegido en
   `/tmp/nhopkg-tools` (se antepone su `bin` al `PATH` y se exporta
   `NHOPKG_CONF` al conf generado del propio árbol). Si el árbol ya
   existe, se reutiliza (no se recompila):
   - `glibc`: `repo-version=n2026`, `repo-arch=x86_64`, `git-branch=n2026`
   - `musl`: `binlocate=plocate`, `repo-version=n27`,
     `repo-arch=x86_64-musl`, `libc=musl`, `git-branch=musl`
   - Sin BusyBox por defecto (`use-busybox=no`); si al host le falta
     `wget`/`curl`, `grep`, `sed`, `awk`, `sort`, `tar`, `zstd` o
     `sha256sum`, la receta añade `use-busybox=yes` +
     `static-busybox=auto` + `static-zstd=auto` (para ambos libc).
   - Requiere `git`, `meson` y `ninja` del host (no se instalan en el
     target).
   - El conf generado queda **horneado** en
     `$NH_TMP/etc/nhopkg/nhopkg.conf` (vía `--sysconfdir` del build), así
     que nhopkg lo resuelve por defecto; `NHOPKG_CONF` se exporta por
     robustez.
2. Si el build falla → error claro con las vías (instalar `git`/`meson`/
   `ninja` o usar un descargable prebuilt por flavor — futuro).

El cleanup final borra `/tmp/nhopkg-tools`.

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

### Diagrama

```
neonatox-bootstrap
    │
    ├── core (host)      → mkdir → cp data/etc → symlinks →
    │                      metadata nhopkg → archivos dinámicos
    │                      (fstab, hostname, timezone, machine-id)
    │                      → nhopkg_prepare → -U → -RS packs → sentinel
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
README.md                       # documentación pública
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