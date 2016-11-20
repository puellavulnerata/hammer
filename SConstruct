# -*- python -*-
import os
import os.path
import platform
import sys
from distutils.version import LooseVersion
import re
import subprocess


vars = Variables(None, ARGUMENTS)
vars.Add(PathVariable('DESTDIR', "Root directory to install in (useful for packaging scripts)", None, PathVariable.PathIsDirCreate))
vars.Add(PathVariable('prefix', "Where to install in the FHS", "/usr/local", PathVariable.PathAccept))
vars.Add(PathVariable('libdir', "Where to install libraries", None, PathVariable.PathAccept))
vars.Add(PathVariable('includedir', "Where to install headers", None, PathVariable.PathAccept))
vars.Add(ListVariable('bindings', 'Language bindings to build', 'none', ['cpp', 'dotnet', 'perl', 'php', 'python', 'ruby']))

tools = ['default', 'scanreplace']
if 'dotnet' in ARGUMENTS.get('bindings', []):
	tools.append('csharp/mono')

envvars = {'PATH' : os.environ['PATH']}
if 'PKG_CONFIG_PATH' in os.environ:
    envvars['PKG_CONFIG_PATH'] = os.environ['PKG_CONFIG_PATH']

env = Environment(ENV = envvars,
                  variables = vars,
                  tools=tools,
                  toolpath=['tools'])

if not 'bindings' in env:
    env['bindings'] = []

def calcInstallPath(*elements):
    path = os.path.abspath(os.path.join(*map(env.subst, elements)))
    if 'DESTDIR' in env:
        path = os.path.join(env['DESTDIR'], os.path.relpath(path, start="/"))
    return path

rel_prefix = not os.path.isabs(env['prefix'])
env['prefix'] = os.path.abspath(env['prefix'])
if 'DESTDIR' in env:
    env['DESTDIR'] = os.path.abspath(env['DESTDIR'])
    if rel_prefix:
        print >>sys.stderr, "--!!-- You used a relative prefix with a DESTDIR. This is probably not what you"
        print >>sys.stderr, "--!!-- you want; files will be installed in"
        print >>sys.stderr, "--!!--    %s" % (calcInstallPath("$prefix"),)

if 'includedir' in env:
    env['incpath'] = calcInstallPath("$includedir", "hammer")
else:
    env['includedir'] = os.path.abspath(os.path.join(*map(env.subst, ["$prefix", "include"])))
    env['incpath'] = calcInstallPath("$prefix", "include", "hammer")
if 'libdir' in env:
    env['libpath'] = calcInstallPath("$libdir")
    env['pkgconfigpath'] = calcInstallPath("$libdir", "pkgconfig")
else:
    env['libpath'] = calcInstallPath("$prefix", "lib")
    env['pkgconfigpath'] = calcInstallPath("$prefix", "lib", "pkgconfig")
    env['libdir'] = os.path.abspath(os.path.join(*map(env.subst, ["$prefix", "lib"])))
env['parsersincpath'] = calcInstallPath("$includedir", "hammer", "parsers")
env['backendsincpath'] = calcInstallPath("$includedir", "hammer", "backends")

env.MergeFlags("-std=gnu11 -Wno-unused-parameter -Wno-attributes -Wno-unused-variable -Wall -Wextra -Werror")

if env['PLATFORM'] == 'darwin':
    env.Append(SHLINKFLAGS = '-install_name ' + env["libpath"] + '/${TARGET.file}')
elif os.uname()[0] == "OpenBSD":
    pass
else:
    env.MergeFlags("-lrt")

AddOption("--variant",
          dest="variant",
          nargs=1, type="choice",
          choices=["debug", "opt"],
          default="opt",
          action="store",
          help="Build variant (debug or opt)")

AddOption("--coverage",
          dest="coverage",
          default=False,
          action="store_true",
          help="Build with coverage instrumentation")

AddOption("--in-place",
          dest="in_place",
          default=False,
          action="store_true",
          help="Build in-place, rather than in the build/<variant> tree")

AddOption("--disable-llvm-backend",
          dest="use_llvm",
          default=False,
          action="store_false",
          help="Disable the LLVM backend (and don't require LLVM library dependencies)")
AddOption("--enable-llvm-backend",
          dest="use_llvm",
          default=False,
          action="store_true",
          help="Enable the LLVM backend (and require LLVM library dependencies)")

dbg = env.Clone(VARIANT='debug')
dbg.MergeFlags("-g -O0")

opt = env.Clone(VARIANT='opt')
opt.Append(CCFLAGS=["-O3"])

if GetOption("variant") == 'debug':
    env = dbg
else:
    env = opt

env["CC"] = os.getenv("CC") or env["CC"]
env["CXX"] = os.getenv("CXX") or env["CXX"]

if GetOption("use_llvm"):
    # Overridable default path to llvm-config
    env['LLVM_CONFIG'] = "llvm-config"
    env["LLVM_CONFIG"] = os.getenv("LLVM_CONFIG") or env["LLVM_CONFIG"]

if GetOption("coverage"):
    env.Append(CFLAGS=["--coverage"],
               CXXFLAGS=["--coverage"],
               LDFLAGS=["--coverage"])
    if env["CC"] == "gcc":
        env.Append(LIBS=['gcov'])
    else:
        env.ParseConfig('%s --cflags --ldflags --libs core executionengine mcjit analysis x86codegen x86info' % \
                        env["LLVM_CONFIG"])

if os.getenv("CC") == "clang" or env['PLATFORM'] == 'darwin':
    env.Replace(CC="clang",
                CXX="clang++")

env["ENV"].update(x for x in os.environ.items() if x[0].startswith("CCC_"))

#rootpath = env['ROOTPATH'] = os.path.abspath('.')
#env.Append(CPPPATH=os.path.join('#', "hammer"))

if GetOption("use_llvm"):
# Set up LLVM config stuff to export

# some llvm versions are old and will not work; some require --system-libs
# with llvm-config, and some will break if given it
    llvm_config_version = subprocess.Popen('%s --version' % env["LLVM_CONFIG"], \
                                           shell=True, \
                                           stdin=subprocess.PIPE, stdout=subprocess.PIPE).communicate()
    if LooseVersion(llvm_config_version[0]) < LooseVersion("3.6"):
        print "This LLVM version %s is too old" % llvm_config_version[0].strip()
        Exit(1)

    if LooseVersion(llvm_config_version[0]) < LooseVersion("3.9") and \
        LooseVersion(llvm_config_version[0]) >= LooseVersion("3.5"):
        llvm_system_libs_flag = "--system-libs"
    else:
        llvm_system_libs_flag = ""

    # Only keep one copy of this
    llvm_required_components = "core executionengine mcjit analysis x86codegen x86info"
    # Stubbing this out so we can implement static-only mode if needed later
    llvm_use_shared = True
    # Can we ask for shared/static from llvm-config?
    if LooseVersion(llvm_config_version[0]) < LooseVersion("3.9"):
        # Nope
        llvm_linkage_type_flag = ""
        llvm_use_computed_shared_lib_name = True
    else:
        # Woo, they finally fixed the dumb
        llvm_use_computed_shared_lib_name = False
        if llvm_use_shared:
            llvm_linkage_type_flag = "--link-shared"
        else:
            llvm_linkage_type_flag = "--link-static"

    if llvm_use_computed_shared_lib_name:
        # Okay, pull out the major and minor version numbers (barf barf)
        p = re.compile("^(\d+)\.(\d+).*$")
        m = p.match(llvm_config_version[0])
        if m:
            llvm_computed_shared_lib_name = "LLVM-%d.%d" % ((int)(m.group(1)), (int)(m.group(2)))
        else:
            print "Couldn't compute shared library name from LLVM version '%s', but needed to" % \
                llvm_config_version[0]
            Exit(1)
    else:
        # We won't be needing it
        llvm_computed_shared_lib_name = None

    # llvm-config 'helpfully' supplies -g and -O flags; educate it with this
    # custom ParseConfig function arg; make it a class with a method so we can
    # pass it around with scons export/import

    class LLVMConfigSanitizer:
        def sanitize(self, env, cmd, unique=1):
            # cmd is output from llvm-config
            flags = cmd.split()
            # match -g or -O flags
            p = re.compile("^-[gO].*$")
            filtered_flags = [flag for flag in flags if not p.match(flag)]
            filtered_cmd = ' '.join(filtered_flags)
            # print "llvm_config_sanitize: \"%s\" => \"%s\"" % (cmd, filtered_cmd)
            env.MergeFlags(filtered_cmd, unique)
    llvm_config_sanitizer = LLVMConfigSanitizer()

    # LLVM defines, which the python bindings need
    try:
        llvm_config_cflags = subprocess.Popen('%s --cflags' % env["LLVM_CONFIG"], \
                                              shell=True, \
                                              stdin=subprocess.PIPE, stdout=subprocess.PIPE).communicate()
        flags = llvm_config_cflags[0].split()
        # get just the -D ones
        p = re.compile("^-D(.*)$")
        llvm_defines = [p.match(flag).group(1) for flag in flags if p.match(flag)]
    except:
        print "%s failed. Make sure you have LLVM and clang installed." % env["LLVM_CONFIG"]
        Exit(1)

    # Get the llvm includedir, which the python bindings need
    try:
        llvm_config_includes = subprocess.Popen('%s --includedir' % env["LLVM_CONFIG"], \
                                                shell=True, \
                                                stdin=subprocess.PIPE, stdout=subprocess.PIPE).communicate()
        llvm_includes = llvm_config_includes[0].splitlines()
    except:
        print "%s failed. Make sure you have LLVM and clang installed." % env["LLVM_CONFIG"]
        Exit(1)

    # This goes here so we already know all the LLVM crap
    # Make a fresh environment to parse the config into, to read out just LLVM stuff
    llvm_dummy_env = Environment()
    # Get LLVM stuff into LIBS/LDFLAGS
    llvm_dummy_env.ParseConfig('%s --ldflags %s %s %s' % \
                               (env["LLVM_CONFIG"], llvm_system_libs_flag, llvm_linkage_type_flag, \
                                llvm_required_components), \
                               function=llvm_config_sanitizer.sanitize)
    # Get the right -l lines in
    if llvm_use_shared:
        if llvm_use_computed_shared_lib_name:
            llvm_dummy_env.Append(LIBS=[llvm_computed_shared_lib_name, ])
        else:
            llvm_dummy_env.ParseConfig('%s %s --libs %s' % \
                                       (env["LLVM_CONFIG"], llvm_linkage_type_flag, llvm_required_components), \
                                       function=llvm_config_sanitizer.sanitize)
    llvm_dummy_env.Append(LIBS=['stdc++', ], )
#endif GetOption("use_llvm")

# The .pc.in file has substs for llvm_lib_flags and llvm_libdir_flags, so if
# we aren't using LLVM, set them to the empty string
if GetOption("use_llvm"):
    env['llvm_libdir_flags'] = llvm_dummy_env.subst('$_LIBDIRFLAGS')
    env['llvm_lib_flags'] = llvm_dummy_env.subst('$_LIBFLAGS')
else:
    env['llvm_libdir_flags'] = ""
    env['llvm_lib_flags'] = ""

pkgconfig = env.ScanReplace('libhammer.pc.in')
Default(pkgconfig)
env.Install("$pkgconfigpath", pkgconfig)

testruns = []

targets = ["$libpath",
           "$incpath",
           "$parsersincpath",
           "$backendsincpath",
           "$pkgconfigpath"]

Export('env')
Export('testruns')
Export('targets')
# LLVM-related flags
if GetOption("use_llvm"):
    Export('llvm_computed_shared_lib_name')
    Export('llvm_config_sanitizer')
    Export('llvm_config_version')
    Export('llvm_defines')
    Export('llvm_includes')
    Export('llvm_linkage_type_flag')
    Export('llvm_required_components')
    Export('llvm_system_libs_flag')
    Export('llvm_use_computed_shared_lib_name')
    Export('llvm_use_shared')

if not GetOption("in_place"):
    env['BUILD_BASE'] = 'build/$VARIANT'
    lib = env.SConscript(["src/SConscript"], variant_dir='$BUILD_BASE/src')
    env.Alias("examples", env.SConscript(["examples/SConscript"], variant_dir='$BUILD_BASE/examples'))
else:
    env['BUILD_BASE'] = '.'
    lib = env.SConscript(["src/SConscript"])
    env.Alias(env.SConscript(["examples/SConscript"]))

for testrun in testruns:
    env.Alias("test", testrun)

env.Alias("install", targets)
