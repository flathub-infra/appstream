project('AppStream', 'c',
  meson_version: '>=0.62',
  default_options: ['c_std=gnu17', 'cpp_std=c++20'],

  license: 'LGPL-2.1-or-later',
  version: '1.0.6',
)

cc = meson.get_compiler('c')
source_root = meson.project_source_root()

as_version = meson.project_version()
as_api_level = '5'

varr = as_version.split('.')
as_major_version = varr[0]
as_minor_version = varr[1]
as_micro_version = varr[2]
as_data_installdir = get_option('prefix') / get_option('datadir') / 'appstream'

#
# Configure files
#
conf = configuration_data()
conf.set('AS_MAJOR_VERSION_CONF', as_major_version)
conf.set('AS_MINOR_VERSION_CONF', as_minor_version)
conf.set('AS_MICRO_VERSION_CONF', as_micro_version)
conf.set_quoted('PACKAGE_VERSION', as_version)
conf.set_quoted('GETTEXT_PACKAGE', 'appstream')
conf.set_quoted('LOCALEDIR',
                get_option('prefix') / get_option('localedir'))
conf.set_quoted('LOCALSTATEDIR',
                get_option('prefix') / get_option('localstatedir'))
conf.set_quoted('LIBEXECDIR',
                get_option('prefix') / get_option('libexecdir'))
conf.set_quoted('LIBDIR',
                get_option('prefix') / get_option('libdir'))
conf.set_quoted('AS_DATADIR',
                as_data_installdir)
conf.set_quoted('BINDIR',
                get_option('prefix') / get_option('bindir'))
conf.set_quoted('SYSCONFDIR',
                get_option('prefix') / get_option('sysconfdir'))
conf.set('_DEFAULT_SOURCE', true)
conf.set('HAVE_APT_SUPPORT', get_option('apt-support'))
conf.set('HAVE_STEMMING', get_option('stemming'))
conf.set('HAVE_SYSTEMD', get_option('systemd'))
conf.set('HAVE_SVG_SUPPORT', get_option('svg-support'))
conf.set('HAVE_ZSTD', get_option('zstd-support'))

configure_file(output: 'config.h', configuration: conf)
root_inc_dir = include_directories ('.')

#
# Custom C flags
#

if cc.get_id() == 'msvc'
  error('MSVC is not supported as it does not support modern C11 features and __attribute__((cleanup))')
endif

sanitizer_libs = []
if get_option('maintainer')
    maintainer_c_args = ['-Werror',
                         '-Wall',
                         '-Wextra',
                         '-Wcast-align',
                         '-Wno-uninitialized',
                         '-Wempty-body',
                         '-Winit-self',
                         '-Wnull-dereference',
                         '-Wfloat-equal',
                         '-Winline',
                         '-Wno-error=comment',
                         '-Werror=format-security',

                         # used in rsvg's headers, so we can not make this fatal yet
                         '-Wno-error=expansion-to-defined',
    ]
    add_project_arguments(maintainer_c_args, language: 'c')
    add_project_arguments(maintainer_c_args, language: 'cpp')
endif

if get_option('static-analysis') and host_machine.system() != 'windows'
  if cc.get_id() != 'gcc'
    error('You need to compile with GCC to run the static analyzer!')
  endif

  # enable static analyzer
  add_project_arguments(['-fanalyzer'], language : 'c')

  # make false-positive non-fatal with GCC 12 for now
  if cc.version().version_compare('>11<13')
    add_project_arguments(['-Wno-error=analyzer-use-of-uninitialized-value'], language : 'c')
  endif

  # g_error has a deliberate infinite loop, so we have to suppress the new warning for now
  if cc.version().version_compare('>13<15')
    add_project_arguments(['-Wno-error=analyzer-infinite-loop'], language : 'c')
  endif
endif

# a few compiler warning flags we always want enabled
add_project_arguments(
  cc.get_supported_arguments([
    '-Werror=shadow',
    '-Werror=empty-body',
    '-Werror=strict-prototypes',
    '-Werror=missing-prototypes',
    '-Werror=implicit-function-declaration',
    '-Werror=pointer-arith',
    '-Werror=missing-declarations',
    '-Werror=return-type',
    '-Werror=int-conversion',
    '-Werror=incompatible-pointer-types',
    '-Werror=misleading-indentation',
    '-Werror=missing-include-dirs',
    '-Werror=declaration-after-statement',
    '-Werror=format-security',

    '-Wno-missing-field-initializers',
    '-Wno-error=missing-field-initializers',
    '-Wno-unused-parameter',
    '-Wno-error=unused-parameter',
  ]),
  language: 'c'
)
add_project_arguments(
  '-Wno-unused-parameter',
  '-Werror=empty-body',
  '-Werror=pointer-arith',
  '-Werror=init-self',
  '-Werror=missing-declarations',
  '-Werror=return-type',
  '-Werror=misleading-indentation',
  '-Werror=format-security',

  language: 'cpp'
)

# Vendor extensions in system headers
if host_machine.system() != 'netbsd'
    # on NetBSD, defining POSIX_C_SOURCE *removes* non-POSIX symbols from namespace,
    # while on all other systems it adds them. Hence we only define this when not on NetBSD.
    add_project_arguments('-D_POSIX_C_SOURCE=201710L', language: 'c')
endif

if host_machine.system() == 'darwin'
    # See https://github.com/ximion/appstream/issues/551
    add_project_arguments('-D_DARWIN_C_SOURCE', language: 'c')
endif

if cc.get_id() == 'clang'
    # Clang doesn't understand autofree helpers on GMutexLocker and thinks
    # these variables are irrelevant, so this warning when used with Clang
    # gives many false-positives
    add_project_arguments(
      '-Wno-unused-variable',
      language: 'c'
    )
endif

add_project_arguments('-DAS_COMPILATION', language: 'c')

#
# Dependencies
#
glib_dep = dependency('glib-2.0', version: '>= 2.62')
gobject_dep = dependency('gobject-2.0', version: '>= 2.62')
gio_dep = dependency('gio-2.0', version: '>= 2.62')
curl_dep = dependency('libcurl', version: '>= 7.62')
xml2_dep = dependency('libxml-2.0')
yaml_dep = dependency('yaml-0.1')
xmlb_dep = dependency('xmlb', version: '>= 0.3.14',
                      fallback: ['libxmlb', 'libxmlb_dep'],
                      default_options: ['gtkdoc=false', 'introspection=false'])
if get_option('zstd-support')
    zstd_dep = dependency('libzstd', required: get_option('zstd-support'))
else
    zstd_dep = dependency('', required: false)
endif
if get_option('systemd')
    libsystemd_dep = dependency('libsystemd', required: get_option('systemd'))
else
    libsystemd_dep = dependency('', required: false)
endif

if get_option ('gir')
    # ensure we have a version of GIR that isn't broken with Meson
    # (prior versions failed when any non-GObject library was linked)
    dependency('gobject-introspection-1.0', version: '>=1.56')
endif

stemmer_inc_dirs = include_directories()
if get_option('stemming')
    stemmer_lib = cc.find_library('stemmer', required: true)
    stemmer_inc_dirs = include_directories(['/usr/include'])
    if not cc.has_header('libstemmer.h')
        if cc.has_header('libstemmer/libstemmer.h')
            stemmer_inc_dirs = include_directories('/usr/include/libstemmer')
        else
            error('Unable to find Snowball header "libstemmer.h". Please ensure libstemmer/Snowball is installed properly in order to continue.')
        endif
    endif
endif

# use gperf for faster string -> enum matching
gperf = find_program('gperf')

#
# Modules
#
fs   = import('fs')
glib = import('gnome')
i18n = import('i18n')
pkgc = import('pkgconfig')

#
# Directories
#
subdir('src/')
if get_option('compose')
    subdir('compose/')
endif
subdir('tools/')
subdir('po/')
subdir('data/')
subdir('contrib/')
subdir('docs/')
subdir('tests/')
if get_option('qt')
    subdir('qt/')
endif
