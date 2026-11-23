project(
  'neonatox-bootstrap',
  version: '0.2.0',
  license: 'GPL-3.0-or-later',
  meson_version: '>= 0.55.0',
)

prefix = get_option('prefix')
bindir = join_paths(prefix, get_option('bindir'))
datadir = join_paths(prefix, get_option('datadir'))

sysconf_opt = get_option('sysconfdir')
if sysconf_opt.startswith('/')
  sysconfdir = sysconf_opt
else
  sysconfdir = join_paths(prefix, sysconf_opt)
endif

nb_datadir = join_paths(datadir, 'neonatox-bootstrap')

nb_conf = configuration_data()
nb_conf.set('NB_VERSION', meson.project_version())
nb_conf.set('NB_DATADIR', nb_datadir)
nb_conf.set('NB_SYSCONFDIR', sysconfdir)

# El ejecutable y el config de ejemplo son los mismos archivos del repo: meson
# solo sustituye los tokens @NB_*@. Si no se instala, los tokens quedan como
# literales y el script cae al directorio de origen.
configure_file(
  input: 'neonatox-bootstrap',
  output: 'neonatox-bootstrap',
  configuration: nb_conf,
  install: true,
  install_dir: bindir,
  install_mode: 'rwxr-xr-x',
)

configure_file(
  input: 'neonatox-bootstrap.conf',
  output: 'neonatox-bootstrap.conf',
  configuration: nb_conf,
  install: true,
  install_dir: sysconfdir,
)

install_subdir('bootstrap/data', install_dir: nb_datadir)
install_subdir('bootstrap/packs', install_dir: nb_datadir)
