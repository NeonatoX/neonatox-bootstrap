# NeonatoX Bootstrap

![Bash](https://img.shields.io/badge/bash-4EAA25?logo=gnubash&logoColor=fff)
![systemd](https://img.shields.io/badge/systemd-052A46?logo=systemd&logoColor=fff)
![btrfs](https://img.shields.io/badge/btrfs-2D8659?logo=btrfs&logoColor=fff)
![Linux](https://img.shields.io/badge/linux-FCC624?logo=linux&logoColor=000)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/cargabsj175/neonatox-bootstrap)

Bootstrap scripts para [NeonatoX](https://neonatox.vegnux.com), una
distribución Linux desde cero. Crea la jerarquía del sistema de archivos,
instala `nhopkg` (gestor de paquetes propio), genera la configuración
base, e instala los paquetes para los entornos de escritorio
KDE / GNOME / XFCE.

## Uso rápido

```bash
sudo ./neonatox-bootstrap -L /mnt core    # sistema base
sudo ./neonatox-bootstrap -L /mnt kde     # KDE (host → chroot)
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
FLUJO.md                    # diagrama de flujo y detalle paso a paso
```

## Requisitos

- Linux con `chroot`, `mount --bind`, `git`
- `sudo` para `blkid` y `mount`
- Conexión a internet para descargar nhopkg y paquetes

## Flujo

Ver [`FLUJO.md`](FLUJO.md) para el detalle paso a paso de cada comando,
helpers de chroot (`chroot_prepare`, `chroot_run`, `chroot_cleanup`),
diagrama de flujo y notas de ejecución.

## Recursos

- [NeonatoX](https://neonatox.vegnux.com)
- [nhopkg](https://github.com/cargabsj175/neonatox-nhopkg)
