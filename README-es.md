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
entornos de escritorio KDE / GNOME / XFCE.

## Uso rápido

```bash
sudo ./neonatox-bootstrap -L /mnt core    # sistema base (perfil glibc)
sudo ./neonatox-bootstrap -L /mnt kde     # KDE (host → chroot)
```

Ejemplos con las características nuevas (`--libc` y nhopkg al vuelo):

```bash
# Sistema base que apunta a los repos musl (n27/x86_64-musl)
sudo ./neonatox-bootstrap --libc musl -L /mnt core

# Escritorio GNOME sobre un target musl
sudo ./neonatox-bootstrap --libc musl -L /mnt gnome

# nhopkg se construye siempre (git+meson+ninja) como un único árbol
# con el prefix estándar (/usr /etc /var), instalado en el target vía
# DESTDIR; sirve de driver del host (libs resueltas por env hacia el
# interior de $LFS) y de nhopkg final del sistema. Si el host no tiene
# alguna tool, la receta habilita BusyBox estático.
sudo ./neonatox-bootstrap -L /mnt core
```

### Argumentos

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` o `/mnt` | Directorio raíz de destino |
| `--hostname <nombre>` | auto (DMI+random) | Nombre del equipo |
| `--device <disp>` | auto-detect | Dispositivo para fstab |
| `--uuid <uuid>` | auto-detect | UUID para fstab |
| `--fstype <tipo>` | auto-detect | Tipo de sistema de archivos |
| `-z, --timezone <zona>` | host o `America/Caracas` | Zona horaria |
| `-l, --locale <locale>` | `es_US.UTF-8` | Locale del sistema |
| `--libc {glibc\|musl}` | `glibc` | Perfil de libc del target (receta de nhopkg) |
| `--no-cleanup` | — | Salta limpieza post-instalación |
| `--pack-dir <dir>` | `bootstrap/packs/` | Listas de paquetes |

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
```

## Requisitos

- Linux con `chroot`, `mount --bind`
- `sudo` para `blkid` y `mount`
- Conexión a internet para descargar nhopkg y paquetes
- **nhopkg se construye siempre** — el del sistema **no se usa**: su
  config corresponde al flavor del host, no al del target. Se genera **un
  único árbol** (`git`, `meson` y `ninja` requeridos) con la receta del
  `--libc` elegido, instalado en `$LFS` vía `DESTDIR` con el prefix
  estándar (`--prefix=/usr --sysconfdir=/etc --localstatedir=/var`):
  1. Es el **driver del host** durante el bootstrap — las libs y la conf
     se resuelven por env (`NHOPKG_CONF`, `NHOPKG_LIB`, `NHOPKG_UDEP_LIB`,
     `NHOPKG_DOWNLOAD_LIB`, `NHOPKG_CRYPTO_LIB`) apuntando al interior de
     `$LFS`, así opera sobre `--root $LFS`.
  2. Es el **nhopkg final del sistema** — las rutas horneadas
     `/usr`/`/etc`/`/var` son las correctas en boot, sustituyendo al
     paquete `nhopkg`.
  Si falta alguna tool en el host, la receta activa BusyBox estático
  automáticamente.

## Flujo

Ver [`AGENTS.md`](AGENTS.md) para el detalle paso a paso de cada comando,
helpers de chroot (`chroot_prepare`, `chroot_run`, `chroot_cleanup`),
diagrama de flujo y notas de ejecución.

## Recursos

- [NeonatoX](https://neonatox.github.io)
- [nhopkg](https://github.com/NeonatoX/neonatox-nhopkg)
