# -*- mode: python; coding: utf-8; -*-

#  Codezero -- a microkernel for embedded systems.
#
#  Copyright © 2009  B Labs Ltd
#
#  This program is free software: you can redistribute it and/or modify it under the terms of the GNU
#  General Public License as published by the Free Software Foundation, either version 3 of the License, or
#  (at your option) any later version.
#
#  This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even
#  the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public
#  License for more details.
#
#  You should have received a copy of the GNU General Public License along with this program.  If not, see
#  <http://www.gnu.org/licenses/>.
#
#  Author: Russel Winder

#  To support Python 2.5 we need the following, which seems to do no harm in Python 2.6.  Only if Python 2.6
#  is the floor version supported can be dispensed with.

from __future__ import with_statement

import os

posixServicesDirectory = "containers/posix/"
includeDirectory = 'include'
toolsDirectory = 'tools'
cml2ToolsDirectory = toolsDirectory + '/cml2-tools'
buildDirectory = 'build'

cml2CompileRulesFile = buildDirectory + '/cml2Rules.out'
cml2ConfigPropertiesFile = buildDirectory + '/cml2Config.out'
cml2ConfigHeaderFile = buildDirectory + '/cml2Config.h'

#  The choice of which parts of the kernel to compile and include in the build depends on the configuration
#  which is managed using CML2.  CML2 uses a base configuration file (currently #configs/arm.cml) to drive
#  an interaction with the user which results in a trio of files that specify the user choice.
#
#  cml2RulesFile is the pickled version of the source configuration driver.
#
#  cml2Config.out is a properties type representation of the user selected configuration data.
#
#  cml2Config.h is a C include file representation of the user selected configuration data derived from
#  cml2Config.out, it is essential for the compilation of the C code of the kernel and the tasks.
#
#  Since the DAG for building the kernel relies on the information from the configuration split the build
#  into two distinct phases as Autotools and Waf do, configure then build.  Achieve this by partitioning the
#  SCons DAG building in two depending on the command line.

if 'configure' in COMMAND_LINE_TARGETS  :

    def performCML2Configuration(target, source, env):
        if not os.path.isdir(buildDirectory) : os.mkdir(buildDirectory)
        os.system(cml2ToolsDirectory + '/cmlcompile.py -o ' + cml2CompileRulesFile + ' ' + source[0].path)
        os.system(cml2ToolsDirectory +  '/cmlconfigure.py -c -o ' + cml2ConfigPropertiesFile + ' ' + cml2CompileRulesFile)
        os.system(toolsDirectory +  '/cml2header.py -o ' + cml2ConfigHeaderFile + ' -i ' + cml2ConfigPropertiesFile)

    if len ( COMMAND_LINE_TARGETS ) != 1:
        print '#### Warning####: configure is part of the command line, all the other targets are being ignored as this is a configure step.'
    Command('configure',  ['#configs/arm.cml'], performCML2Configuration)
    Clean('configure', buildDirectory)

else :

    if not os.path.exists(cml2ConfigPropertiesFile):
        print "####\n#### Configuration has not been undertaken, please run 'scons configure'.\n####"
        Exit()

##########  Create the base environment and process the configuration ########################

    def processCML2Config():
        configItems = {}
        with file(cml2ConfigPropertiesFile) as configFile:
            for line in configFile:
                item = line.split('=')
                if len(item) == 2:
                    configItems[item[0].strip()] = (item[1].strip() == 'y')
        return configItems

    baseEnvironment = Environment(tools = ['gnulink', 'gcc', 'gas', 'ar'],
                                    ENV = {'PATH': os.environ['PATH']},
                                    configFiles = ('#' + cml2CompileRulesFile,  '#' + cml2ConfigPropertiesFile,  '#' + cml2ConfigHeaderFile))

    #  It is assumed that the C code is assuming that the configuration file will be found at l4/config.h so create it there.
    #
    #  Kernel code include config.h in a different way to all the other bits of code.
    #
    #  TODO:  Decide if this is an issue or not.

    configHPath = buildDirectory + '/l4/config.h'
    configuration = Configure(baseEnvironment, config_h =  configHPath)
    configData = processCML2Config()
    arch = None
    platform = None
    subarch = None
    for key, value in configData.items():
        if value:
            items = key.split('_')
            if items[0] == 'ARCH':
                arch = items[1].lower()
    for key, value in configData.items():
        if value:
            items = key.split('_')
            if items[0] == arch.upper():
                if items[1] == 'PLATFORM':
                    platform = items[2].lower()
                if items[1] == 'SUBARCH':
                    subarch = items[2].lower()
            if items[0] == 'DRIVER':
                configuration.env.Append(driverList = [('irq' if items[1] == 'IRQCTRL' else items[1].lower()) + '/' + items[2].lower()])
    configuration.Define('__ARCH__', arch)
    configuration.Define('__PLATFORM__', platform)
    configuration.Define('__SUBARCH__', subarch)
    configuration.env['ARCH'] = arch
    configuration.env['PLATFORM'] = platform
    configuration.env['SUBARCH'] = subarch
    baseEnvironment = configuration.Finish()
    baseEnvironment.Append(configFiles = ('#' + configHPath,))

##########  Build the libraries ########################

    libraryEnvironment = baseEnvironment.Clone(
        CC = 'arm-none-linux-gnueabi-gcc',
        CCFLAGS = ['-g', '-nostdinc', '-nostdlib', '-ffreestanding', '-std=gnu99', '-Wall', '-Werror'],
        LINKFLAGS = ['-nostdlib'],
        LIBS = ['gcc'],
        ARCH = arch,
        PLATFORM = platform)

    libs = {}
    crts = {}
    for variant in ['baremetal']:
        (libs[variant], crts[variant]) = SConscript('libs/c/SConscript', variant_dir = buildDirectory + '/lib/c/' + variant, duplicate = 0, exports = {'environment': libraryEnvironment, 'variant': variant})
        Depends((libs[variant], crts[variant]), libraryEnvironment['configFiles'])

    baseEnvironment['baremetal_libc'] = libs['baremetal']
    baseEnvironment['baremetal_crt0'] = crts['baremetal']

    libelf = SConscript('libs/elf/SConscript', variant_dir = buildDirectory + '/lib/elf', duplicate = 0, exports = {'environment': libraryEnvironment})
    Depends(libelf, libraryEnvironment['configFiles'])

    Alias('libs', crts.values() + libs.values() + [libelf])

##########  Build the kernel ########################

    kernelEnvironment = baseEnvironment.Clone(
        CC = 'arm-none-eabi-gcc',
        # We don't use -nostdinc because sometimes we need standard headers, such as stdarg.h e.g. for variable
        # args, as in printk().
        CCFLAGS = ['-mcpu=arm926ej-s', '-g', '-nostdlib', '-ffreestanding', '-std=gnu99', '-Wall', '-Werror'],
        LINKFLAGS = ['-nostdlib', '-T' +  includeDirectory + '/l4/arch/' + arch + '/linker.lds'],
        ASFLAGS = ['-D__ASSEMBLY__'],
        PROGSUFFIX = '.axf',
        LIBS = ['gcc'],
        CPPPATH = ['#' + buildDirectory, '#' + buildDirectory + '/l4', '#' + includeDirectory, '#' + includeDirectory + '/l4'],

        ####
        ####  TODO:  Why are these files forcibly included, why not just leave it up to the C code to include things?
        ####

        CPPFLAGS = ['-include', 'config.h', '-include', 'cml2Config.h', '-include', 'macros.h', '-include', 'types.h', '-D__KERNEL__'])

    startAxf = SConscript('src/SConscript' , variant_dir = buildDirectory + '/kernel' , duplicate = 0, exports = {'environment': kernelEnvironment})
    Depends(startAxf, kernelEnvironment['configFiles'])

    Alias('kernel', startAxf)

##########  Build the task libraries ########################

    taskSupportLibraryEnvironment = baseEnvironment.Clone(
        CC = 'arm-none-linux-gnueabi-gcc',
        CCFLAGS = ['-g', '-nostdlib', '-ffreestanding', '-std=gnu99', '-Wall', '-Werror'],
        LINKFLAGS = ['-nostdlib'],
        ASFLAGS = ['-D__ASSEMBLY__'],
        LIBS = ['gcc'],
        CPPPATH = ['#' + buildDirectory, '#' + buildDirectory + '/l4', '#' + includeDirectory])

    taskLibraryNames = [f.name for f in Glob(posixServicesDirectory + 'lib*')]

    taskLibraries = []
    for library in taskLibraryNames:
        taskLibraries.append(SConscript(posixServicesDirectory + library + '/SConscript', variant_dir = buildDirectory + '/' + posixServicesDirectory + library, duplicate = 0, exports = {'environment': taskSupportLibraryEnvironment, 'posixServicesDirectory': posixServicesDirectory}))

    Depends(taskLibraries, taskSupportLibraryEnvironment['configFiles'])

    Alias ('tasklibs', taskLibraries)

##########  Build the tasks ########################

    def buildTask(programName, sources, environment, previousImage, extraCppPath=None):
        e = environment.Clone()
        e.Append(LINKFLAGS=['-T' + posixServicesDirectory + programName + '/include/linker.lds'])
        e.Append(LIBPATH=['#build/' + posixServicesDirectory + programName])
        if extraCppPath: e.Append(CPPPATH=extraCppPath)
        objects = e.StaticObject(sources)
        Depends(objects, e['configFiles'])
        program = e.Program(programName, objects)
        environment['physicalBaseLinkerScript'] = Command('include/physical_base.lds', previousImage, 'tools/pyelf/readelf.py --first-free-page ' + previousImage[0].path + ' >> $TARGET')
        Depends(program, [environment['physicalBaseLinkerScript']])
        return program

    tasksEnvironment = baseEnvironment.Clone(
        CC = 'arm-none-linux-gnueabi-gcc',
        CCFLAGS = ['-g', '-nostdlib', '-ffreestanding', '-std=gnu99', '-Wall', '-Werror'],
        LINKFLAGS = ['-nostdlib'],
        ASFLAGS = ['-D__ASSEMBLY__'],
        LIBS =  taskLibraries + ['gcc'] + taskLibraries,
        PROGSUFFIX = '.axf',
        CPPDEFINES = ['__USERSPACE__'],
        CPPPATH = ['#' + buildDirectory, '#' + buildDirectory + '/l4', '#' + includeDirectory, 'include', \
	'#' + posixServicesDirectory + 'libl4/include', '#' + posixServicesDirectory + 'libc/include', \
	'#' + posixServicesDirectory + 'libmem', '#' + posixServicesDirectory + 'libposix/include'],
        buildTask = buildTask)

####
####  TODO: Why does the linker require crt0.o to be in the current directory and named as such.  Is it
####  because of the text in the linker script?
####
    execfile(posixServicesDirectory + 'taskOrder.py')
    imageOrderData = [(taskName, []) for taskName in taskOrder]
    imageOrderData[0][1].append(startAxf)
    tasks = []
    for i in range(len(imageOrderData)):
        taskName = imageOrderData[i][0]
        dependency = imageOrderData[i][1]
        program = SConscript(posixServicesDirectory + taskName + '/SConscript', variant_dir = buildDirectory + '/' + posixServicesDirectory + taskName, duplicate = 0, exports = {'environment': tasksEnvironment, 'previousImage': dependency[0], 'posixServicesDirectory':posixServicesDirectory})
        tasks.append(program)
        if i < len(imageOrderData) - 1:
            imageOrderData[i+1][1].append(program)
    Depends(tasks, tasksEnvironment['configFiles'])

    Alias ('tasks', tasks)

##########  Create the boot description ########################

    taskName = 'bootdesc'

    bootdescEnvironment = baseEnvironment.Clone(
        CC = 'arm-none-linux-gnueabi-gcc',
        CCFLAGS = ['-g', '-nostdlib', '-ffreestanding', '-std=gnu99', '-Wall', '-Werror'],
        LINKFLAGS = ['-nostdlib', '-T' + posixServicesDirectory + taskName + '/linker.lds'],
        ASFLAGS = ['-D__ASSEMBLY__'],
        PROGSUFFIX = '.axf',
        LIBS = ['gcc'],
        CPPPATH = ['#' + includeDirectory])

    bootdesc = SConscript(posixServicesDirectory + taskName + '/SConscript', variant_dir = buildDirectory + '/' + posixServicesDirectory + taskName, duplicate = 0, exports = {'environment': bootdescEnvironment, 'images': [startAxf] + tasks})

    Alias('bootdesc', bootdesc)

##########  Do the packing / create loadable ########################

    loaderEnvironment = baseEnvironment.Clone(
        CC = 'arm-none-linux-gnueabi-gcc',
        CCFLAGS = ['-g', '-nostdlib', '-ffreestanding', '-std=gnu99', '-Wall', '-Werror'],
        LINKFLAGS = ['-nostdlib', '-T' + buildDirectory + '/loader/linker.lds'],
        PROGSUFFIX = '.axf',
        LIBS = [libelf, libs['baremetal'], 'gcc', libs['baremetal']],
        CPPPATH = ['#libs/elf/include', '#' + buildDirectory + '/loader'])

    ####  TODO: Fix the tasks data structure so as to avoid all the assumptions.

    loader = SConscript('loader/SConscript', variant_dir = buildDirectory + '/loader', duplicate = 0, exports = {'environment': loaderEnvironment, 'images':[startAxf, bootdesc] + tasks, 'posixServicesDirectory': posixServicesDirectory})

    Alias('final', loader)

    #  The test does not terminate and Ctrl-C and Ctrl-Z have no effect.  Run the job in the background so
    #  the initiating terminal retains control and allows the process to be killed from this terminal.  Add
    #  the sleep to force SCons to wait until the test has run before it decides all targets are built and
    #  return to the prompt.  Remind the user they have a running process in the background.

    Command('runTest', loader, "qemu-system-arm -kernel $SOURCE -nographic -m 128 -M versatilepb & sleep 10 ; echo '####\\n#### You will need to kill the qemu-system-arm process\\n#### that is running in the background.\\n####\\n'")

##########  Other rules. ########################

    Default(crts.values() + libs.values() + [libelf, startAxf] + tasks + bootdesc + loader)

    Clean('.', [buildDirectory])

##########  Be helpful ########################

Help('''
Explicit targets are:

  configure -- configure the build area ready for a build.

  libs -- build the support library.
  kernel -- build the kernel.
  tasklibs -- build all the support libraries for the tasks.
  tasks -- build all the tasks.
  bootdesc -- build the tasks and the boot descriptor.
  final -- build the loadable.

The default target is to compile everything and to do a final link.

Compilation can only be undertaken after a configuration.
''')
