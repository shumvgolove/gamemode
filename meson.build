project(
    'gamemode',
    'c',
    default_options : ['c_std=c11', 'warning_level=3'],
    version: '1.8.2',
    license: 'BSD',
    meson_version: '>= 1.3.1',
)

am_cflags = [
    '-fstack-protector',
    '-Wstrict-prototypes',
    '-Wundef',
    '-fno-common',
    '-Werror-implicit-function-declaration',
    '-Wformat-security',
    '-Werror=format-security',
    '-Wconversion',
    '-Wunreachable-code',
]

# Add our main flags
add_global_arguments(am_cflags, language: 'c')

cc = meson.get_compiler('c')

# additional compiler warnings, if supported
test_args = [
  '-Waggregate-return',
  '-Wunused',
  '-Warray-bounds',
  '-Wcast-align',
  '-Wclobbered',
  '-Wempty-body',
  '-Wformat=2',
  '-Wformat-nonliteral',
  '-Wformat-signedness',
  '-Wignored-qualifiers',
  '-Wimplicit-function-declaration',
  '-Winit-self',
  '-Wmissing-format-attribute',
  '-Wmissing-include-dirs',
  '-Wmissing-noreturn',
  '-Wmissing-parameter-type',
  '-Wnested-externs',
  '-Wno-discarded-qualifiers',
  '-Wno-missing-field-initializers',
  '-Wno-suggest-attribute=format',
  '-Wno-unused-parameter',
  '-Wold-style-definition',
  '-Woverride-init',
  '-Wpointer-arith',
  '-Wredundant-decls',
  '-Wreturn-type',
  '-Wshadow',
  '-Wsign-compare',
  '-Wstrict-aliasing=3',
  '-Wstrict-prototypes',
  '-Wstringop-overflow',
  '-Wstringop-truncation',
  '-Wtype-limits',
  '-Wundef',
  '-Wuninitialized',
  '-Wunused-but-set-variable',
  '-Wwrite-strings',
]

foreach arg: test_args
  if cc.has_argument(arg)
    add_global_arguments(arg, language : 'c')
  endif
endforeach


path_prefix = get_option('prefix')
path_bindir = join_paths(path_prefix, get_option('bindir'))
path_datadir = join_paths(path_prefix, get_option('datadir'))
path_includedir = join_paths(path_prefix, get_option('includedir'))
path_libdir = join_paths(path_prefix, get_option('libdir'))
path_libexecdir = join_paths(path_prefix, get_option('libexecdir'))
path_mandir = join_paths(path_prefix, get_option('mandir'))
path_metainfo = join_paths(path_datadir, 'metainfo')
path_sysconfdir = join_paths(path_datadir, 'gamemode')

# Find systemd / elogind via pkgconfig
sd_bus_provider = get_option('with-sd-bus-provider')

sd_bus_args = []
sd_bus_dep = []
if sd_bus_provider == 'systemd'
    sd_bus_dep = dependency('libsystemd')
elif sd_bus_provider == 'elogind'
    sd_bus_args += ['-DUSE_ELOGIND']
    sd_bus_dep = dependency('libelogind')
endif

# For the client, libdbus is used
dep_dbus = dependency('dbus-1')

# Allow meson to figure out how the compiler sets up threading
dep_threads = dependency('threads')

# On non glibc systems this might be a stub, i.e. for musl
libdl = cc.find_library('dl', required: false)

with_privileged_group = get_option('with-privileged-group')

# Determine the location for the systemd unit
if sd_bus_provider == 'systemd'
    with_systemd_unit = get_option('with-systemd-user-unit')
    if with_systemd_unit
        path_systemd_unit_dir = get_option('with-systemd-user-unit-dir')
        if path_systemd_unit_dir == ''
            message('Asking pkg-config for systemd\'s \'systemduserunitdir\' directory')
            pkgconfig_systemd = dependency('systemd')
            path_systemd_unit_dir = pkgconfig_systemd.get_variable(pkgconfig: 'systemduserunitdir')
        endif
    endif
    if with_privileged_group != ''
      with_systemd_group = get_option('with-systemd-group')
      if with_systemd_group
          path_systemd_group_dir = get_option('with-systemd-group-dir')
          if path_systemd_group_dir == ''
              message('Asking pkg-config for systemd\'s \'sysusersdir\' directory')
              pkgconfig_systemd = dependency('systemd')
              path_systemd_group_dir = pkgconfig_systemd.get_variable(pkgconfig: 'sysusersdir')
          endif
      endif
    else
      with_systemd_group = false
    endif
endif

if with_privileged_group != ''
  with_pam_renicing = get_option('with-pam-renicing')
  if with_pam_renicing
    path_pam_limits_dir = get_option('with-pam-limits-dir')
  endif
else
  with_pam_renicing = false
endif

# Set the dbus path as appropriate.
path_dbus_service_dir = get_option('with-dbus-service-dir')
if path_dbus_service_dir == ''
  path_dbus_service_dir = join_paths(path_datadir, 'dbus-1', 'services')
endif

path_polkit_dir = join_paths(path_datadir, 'polkit-1')
path_polkit_action_dir = join_paths(path_polkit_dir, 'actions')
path_polkit_rule_dir = join_paths(path_polkit_dir, 'rules.d')

with_examples = get_option('with-examples')
with_util = get_option('with-util')

# Provide a config.h
pidfd_open = cc.has_function('pidfd_open', args: '-D_GNU_SOURCE')

cdata = configuration_data()
cdata.set_quoted('LIBEXECDIR', path_libexecdir)
cdata.set_quoted('SYSCONFDIR', path_sysconfdir)
cdata.set_quoted('GAMEMODE_VERSION', meson.project_version())
cdata.set10('HAVE_FN_PIDFD_OPEN', pidfd_open)

config_h = configure_file(
    configuration: cdata,
    output: 'build-config.h',
)
config_h_dir = include_directories('.')

# common lib is always required
subdir('common')

# Library is always required
subdir('lib')

# Utilities are always required except when having both 64 and 32 bit versions
# of libgamemode installed
if with_util == true
    subdir('util')
endif

# The daemon can be disabled if necessary, allowing multilib builds of the
# main library
if sd_bus_provider != 'no-daemon'
    # inih currently only needed by the daemon
    inih_dependency = dependency(
        'inih',
        fallback : ['inih', 'inih_dep']
    )

    subdir('daemon')

    # All installed data is currently daemon specific
    subdir('data')
endif

# Optionally allow building of examples
if with_examples == true
    subdir('example')
endif


report = [
    '    Build configuration:',
    '    ====================',
    '',
    '    prefix:                                 @0@'.format(path_prefix),
    '    bindir:                                 @0@'.format(path_bindir),
    '    datadir:                                @0@'.format(path_datadir),
    '    libdir:                                 @0@'.format(path_libdir),
    '    libexecdir:                             @0@'.format(path_libexecdir),
    '    includedir:                             @0@'.format(path_includedir),
]

if with_pam_renicing
report += [
    '    PAM limits.d directory:                 @0@'.format(path_pam_limits_dir),
]
endif
if sd_bus_provider == 'systemd'
if with_systemd_unit
report += [
    '    systemd user unit directory:            @0@'.format(path_systemd_unit_dir),
]
endif
if with_systemd_group
report += [
    '    systemd group directory:                @0@'.format(path_systemd_group_dir),
]
endif
endif
report += [
    '    D-BUS service directory:                @0@'.format(path_dbus_service_dir),
]

report += [

    '    PolKit Action Directory:                @0@'.format(path_polkit_action_dir),
    '',
    '    Options:',
    '    ========',
    '',
    '    sd-bus provier:                         @0@'.format(sd_bus_provider),
    '    examples:                               @0@'.format(with_examples),
    '    util:                                   @0@'.format(with_util),
]

# Output some stuff to validate the build config
message('\n\n\n' + '\n'.join(report) + '\n\n')
