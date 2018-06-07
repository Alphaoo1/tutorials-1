[![By ULHPC](https://img.shields.io/badge/by-ULHPC-blue.svg)](https://hpc.uni.lu) [![Licence](https://img.shields.io/badge/license-GPL--3.0-blue.svg)](http://www.gnu.org/licenses/gpl-3.0.html) [![GitHub issues](https://img.shields.io/github/issues/ULHPC/tutorials.svg)](https://github.com/ULHPC/tutorials/issues/) [![](https://img.shields.io/badge/slides-PDF-red.svg)](https://github.com/ULHPC/tutorials/raw/devel/tools/easybuild/slides.pdf) [![Github](https://img.shields.io/badge/sources-github-green.svg)](https://github.com/ULHPC/tutorials/tree/devel/tools/easybuild/) [![Documentation Status](http://readthedocs.org/projects/ulhpc-tutorials/badge/?version=latest)](http://ulhpc-tutorials.readthedocs.io/en/latest/tools/easybuild/) [![GitHub forks](https://img.shields.io/github/stars/ULHPC/tutorials.svg?style=social&label=Star)](https://github.com/ULHPC/tutorials)

# Building [custom] software with EasyBuild on UL HPC platform

      Copyright (c) 2014-2018 S. Peter & UL HPC Team  <hpc-sysadmins@uni.lu>

[![](https://github.com/ULHPC/tutorials/raw/devel/tools/easybuild/cover_slides.png)](https://github.com/ULHPC/tutorials/raw/devel/tools/easybuild/slides.pdf)

The objective of this tutorial is to show how [EasyBuild](http://easybuild.readthedocs.io) can be used to ease, automate and script the build of software on the [UL HPC](https://hpc.uni.lu) platforms.

Indeed, as researchers involved in many cutting-edge and hot topics, you probably have access to many theoretical resources to understand the surrounding concepts. Yet it should _normally_ give you a wish to test the corresponding software.
Traditionally, this part is rather time-consuming and frustrating, especially when the developers did not rely on a "regular" building framework such as [CMake](https://cmake.org/) or the [autotools](https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html) (_i.e._ with build instructions as `configure --prefix <path> && make && make install`).

And when it comes to have a build adapted to an HPC system, you are somehow _forced_ to make a custom build performed on the target machine to ensure you will get the best possible performances.
[EasyBuild](https://github.com/easybuilders/easybuild) is one approach to facilitate this step.

Moreover, later on, you probably want to recover a system configuration matching the detailed installation paths through a set of environmental variable  (Ex: `JAVA_HOME`, `HADOOP_HOME` etc...). At least you would like to see the traditional `PATH`, `CPATH` or `LD_LIBRARY_PATH` updated.

**Question**: what is the purpose of the above mentioned environmental variable?

For this second aspect, the solution came long time ago (in 1991) with the [Environment Modules](http://modules.sourceforge.net/).
We will cover it in the first part of this tutorial.

Then, another advantage of [EasyBuild](http://easybuild.readthedocs.io) comes into account that justifies its wide-spread deployment across many HPC centers (incl. [UL HPC](http://hpc.uni.lu)): it has been designed to not only build any piece of software, but also to generate the corresponding module files to facilitate further interactions with it.
Thus we will cover [EasyBuild](http://easybuild.readthedocs.io) in the second part of this hands-on.
It allows for  automated and reproducable build of software. Once a build has been made, the build script (via the *EasyConfig file*) or the installed software (via the *module file*) can be shared with other users.

You might be interested to know that we rely on [EasyBuild](http://easybuild.readthedocs.io) to provide the [software environment](https://hpc.uni.lu/users/software/) to the users of the platform.

In this tutorial, we are going to first build software that are supported by EasyBuild. Then we will see through a simple example how to add support for a new software in EasyBuild, and eventually contribute back to the main repository

**Note**: The latest version of this tutorial is available on [Github](https://github.com/ULHPC/tutorials/tree/devel/tools/easybuild/).

--------------------
## Pre-requisites ##

Ensure you are able to [connect to the UL HPC clusters](https://hpc.uni.lu/users/docs/access.html)
**For all tests and compilation with Easybuild, you MUST work on a computing node**

In particular, the `module` command **is not** available on the access frontends.

```bash
# /!\ FOR ALL YOUR COMPILING BUSINESS, ENSURE YOU WORK ON A COMPUTING NODE
# Have an interactive job
############### iris cluster (slurm) ###############
(access-iris)$> si -n 2 -t 2:00:00        # 2h interactive reservation
# OR (long version)
(access-iris)$> srun -p interactive -n 2 -t 2:00:00 --pty bash

############### gaia/chaos clusters (OAR) ###############
(access-{gaia|chaos})$> oarsub -I -l core=2,walltime=2
```


------------------------------------------
## Part 1: Environment modules and LMod ##

[Environment Modules](http://modules.sourceforge.net/) are a standard and well-established technology across HPC sites, to permit developing and using complex software and libraries builds with dependencies, allowing multiple versions of software stacks and combinations thereof to co-exist.

The tool in itself is used to manage environment variables such as `PATH`, `LD_LIBRARY_PATH` and `MANPATH`, enabling the easy loading and unloading of application/library profiles and their dependencies.

| Command                        | Description                                                   |
|--------------------------------|---------------------------------------------------------------|
| `module avail`                 | Lists all the modules which are available to be loaded        |
| `module spider <pattern>`      | Search for <pattern> among available modules **(Lmod only)**  |
| `module load <mod1> [mod2...]` | Load a module                                                 |
| `module unload <module>`       | Unload a module                                               |
| `module list`                  | List loaded modules                                           |
| `module purge`                 | Unload all modules (purge)                                    |
| `module display <module>`      | Display what a module does                                    |
| `module use <path>`            | Prepend the directory to the MODULEPATH environment variable  |
| `module unuse <path>`          | Remove the directory from the MODULEPATH environment variable |


*Note:* for more information, see the reference man pages for [modules](http://modules.sourceforge.net/man/module.html) and [modulefile](http://modules.sourceforge.net/man/modulefile.html), or the [official FAQ](http://sourceforge.net/p/modules/wiki/FAQ/).

You can also see our [modules page](https://hpc.uni.lu/users/docs/modules.html) on the [UL HPC website](http://hpc.uni.lu/users/).

At the heart of environment modules interaction resides the following components:

* the `MODULEPATH` environment variable, which defined the list of searched directories for modulefiles
* `modulefile` (see [an example](http://www.nersc.gov/assets/modulefile-example.txt)) associated to each available software.

Then, [Lmod](https://www.tacc.utexas.edu/research-development/tacc-projects/lmod)  is a [Lua](http://www.lua.org/)-based module system that easily handles the `MODULEPATH` Hierarchical problem.

Lmod is a new implementation of Environment Modules that easily handles the MODULEPATH Hierarchical problem. It is drop-in replacement for TCL/C modules and reads TCL modulefiles directly.
In particular, Lmod add many interesting features on top of the traditional implementation focusing on an easier interaction (search, load etc.) for the users. Thus that's the tool I would advise to deploy.

* [User guide](https://www.tacc.utexas.edu/research-development/tacc-projects/lmod/user-guide)
* [Advanced user guide](https://www.tacc.utexas.edu/research-development/tacc-projects/lmod/advanced-user-guide)
* [Sysadmins Guide](https://www.tacc.utexas.edu/research-development/tacc-projects/lmod/system-administrators-guide)

**`/!\ IMPORTANT:` (reminder): the `module` command is ONLY available on the nodes, NOT on the access front-ends.**


```bash
$> module -h
$> echo $MODULEPATH
/opt/apps/resif/data/stable/default/modules/all
```

You have already access to a huge list of software:

```bash
$> module avail       # OR 'module av'
```

Now you can search for a given software using `module spider <pattern>`:

```
$> module spider python

-----------------------------------------------------------------------------------------
devel/protobuf-python:
-----------------------------------------------------------------------------------------
    Description:
      Python Protocol Buffers runtime library.

     Versions:
        devel/protobuf-python/3.3.0-intel-2017a-Python-2.7.13
        devel/protobuf-python/3.4.0-intel-2017a-Python-2.7.13

-----------------------------------------------------------------------------------------
  For detailed information about a specific "devel/protobuf-python" module (including how
  to load the modules) use the module's full name.
  For example:

     $ module spider devel/protobuf-python/3.4.0-intel-2017a-Python-2.7.13
-----------------------------------------------------------------------------------------

-----------------------------------------------------------------------------------------
  lang/Python:
-----------------------------------------------------------------------------------------
    Description:
      Python is a programming language that lets you work more quickly and integrate
      your systems more effectively.

     Versions:
        lang/Python/2.7.13-foss-2017a-bare
        lang/Python/2.7.13-foss-2017a
        lang/Python/2.7.13-intel-2017a
        lang/Python/3.5.3-intel-2017a
        lang/Python/3.6.0-foss-2017a-bare
        lang/Python/3.6.0-foss-2017a

-----------------------------------------------------------------------------------------
  For detailed information about a specific "lang/Python" module (including how to load
  the modules) use the module's full name.
  For example:

     $ module spider lang/Python/3.6.0-foss-2017a
-----------------------------------------------------------------------------------------
```

Let's see the effect of loading/unloading a module

```bash
$> module list
No modules loaded
$> which python
/usr/bin/python
$> python --version       # System level python
Python 2.7.5

$> module load lang/Python/3.6.0-foss-2017a      # use TAB to auto-complete
$> which python
/opt/apps/resif/data/production/v0.1-20170602/default/software/lang/Python/3.6.0-foss-2017a/bin/python
$> python --version
Python 3.6.0

$> module purge
```

Now let's assume that a given software your looking at does not exists, or not in the version you want.
That's where [EasyBuild](http://easybuild.readthedocs.io) comes into play.

-----------------------
## Part 2: Easybuild ##

[<img width='150px' src='http://easybuild.readthedocs.io/en/latest/_static/easybuild_logo_alpha.png'/>](https://easybuilders.github.io/easybuild/)

EasyBuild is a tool that allows to perform automated and reproducible compilation and installation of software. A large number of scientific software are supported (**[1504 supported software packages](http://easybuild.readthedocs.io/en/latest/version-specific/Supported_software.html)** in the last release 3.6.1) -- see also [What is EasyBuild?](http://easybuild.readthedocs.io/en/latest/Introduction.html)

All builds and installations are performed at user level, so you don't need the admin (i.e. `root`) rights.
The software are installed in your home directory (by default in `$HOME/.local/easybuild/software/`) and a module file is generated (by default in `$HOME/.local/easybuild/modules/`) to use the software.

EasyBuild relies on two main concepts: *Toolchains* and *EasyConfig file*.

A **toolchain** corresponds to a compiler and a set of libraries which are commonly used to build a software.
The two main toolchains frequently used on the UL HPC platform are the `foss` ("_Free and Open Source Software_") and the `intel` one.

1. `foss`  is based on the GCC compiler and on open-source libraries (OpenMPI, OpenBLAS, etc.).
2. `intel` is based on the Intel compiler and on Intel libraries (Intel MPI, Intel Math Kernel Library, etc.).

An **EasyConfig file** is a simple text file that describes the build process of a software. For most software that uses standard procedure (like `configure`, `make` and `make install`), this file is very simple.
Many [EasyConfig files](https://github.com/easybuilders/easybuild-easyconfigs/tree/master/easybuild/easyconfigs) are already provided with EasyBuild.
By default, EasyConfig files and generated modules are named using the following convention:
`<Software-Name>-<Software-Version>-<Toolchain-Name>-<Toolchain-Version>`.
However, we use a **hierarchical** approach where the software are classified under a category (or class) -- see  the `CategorizedModuleNamingScheme` option for the `EASYBUILD_MODULE_NAMING_SCHEME` environmental variable), meaning that the layout will respect the following hierarchy:
`<Software-Class>/<Software-Name>/<Software-Version>-<Toolchain-Name>-<Toolchain-Version>`

Additional details are available on EasyBuild website:

- [EasyBuild homepage](https://easybuilders.github.io/easybuild/)
- [EasyBuild documentation](http://easybuild.readthedocs.io/)
- [What is EasyBuild?](http://easybuild.readthedocs.io/en/latest/Introduction.html)
- [Toolchains](https://github.com/easybuilders/easybuild/wiki/Compiler-toolchains)
- [EasyConfig files](http://easybuild.readthedocs.io/en/latest/Writing_easyconfig_files.html)
- [List of supported software packages](http://easybuild.readthedocs.io/en/latest/version-specific/Supported_software.html)

### a. Installation.

* [the official instructions](http://easybuild.readthedocs.io/en/latest/Installation.html).

What is important for the installation of Easybuild are the following variables:

* `EASYBUILD_PREFIX`: where to install **local** modules and software, _i.e._ `$HOME/.local/easybuild`
* `EASYBUILD_MODULES_TOOL`, the type of [modules](http://modules.sourceforge.net/) tool you are using, _i.e._ `LMod` in this case
* `EASYBUILD_MODULE_NAMING_SCHEME`, the way the software and modules should be organized (flat view or hierarchical) -- we're advising on `CategorizedModuleNamingScheme`.

Add the following entries to your `~/.bashrc` (use your favorite CLI editor like `nano` or `vim`):

```bash
# Easybuild
export EASYBUILD_PREFIX=$HOME/.local/easybuild
export EASYBUILD_MODULES_TOOL=Lmod
export EASYBUILD_MODULE_NAMING_SCHEME=CategorizedModuleNamingScheme
# Use the below variable to run:
#    module use $LOCAL_MODULES
#    module load tools/EasyBuild
export LOCAL_MODULES=${EASYBUILD_PREFIX}/modules/all

alias ma="module avail"
alias ml="module list"
function mu(){
   module use $LOCAL_MODULES
   module load tools/EasyBuild
}
```

Then source this file to expose the environment variables:

```bash
$> source ~/.bashrc
$> echo $EASYBUILD_PREFIX
/home/users/<login>/.local/easybuild
```

Now let's install Easybuild following the [boostrapping procedure](http://easybuild.readthedocs.io/en/latest/Installation.html#bootstrapping-easybuild)

```bash
$> cd /tmp/
# download script
curl -o /tmp/bootstrap_eb.py  https://raw.githubusercontent.com/easybuilders/easybuild-framework/develop/easybuild/scripts/bootstrap_eb.py

# install Easybuild
$> python /tmp/bootstrap_eb.py $EASYBUILD_PREFIX
```

Now you can use your freshly built software:
The main EasyBuild command is `eb` (To get help on the EasyBuild options, use the `-h` or `-H` option flags:

    $> eb -h
    $> eb -H
:

```bash
$> eb --version             # expected ;)
-bash: eb: command not found

# Load the newly installed Easybuild
$> echo $MODULEPATH
/opt/apps/resif/data/stable/default/modules/all/

$> module use $LOCAL_MODULES
$> echo $MODULEPATH
/home/users/<login>/.local/easybuild/modules/all:/opt/apps/resif/data/stable/default/modules/all

$> module spider Easybuild
$> module load tools/EasyBuild       # TAB is your friend...
$> eb --version
This is EasyBuild 3.6.1 (framework: 3.6.1, easyblocks: 3.6.1) on host iris-001.
```

Since you are going to use quite often the above command to use locally built modules and load easybuild, an alias `mu` is provided and can be used from now on. Use it **now**

```
$> mu
$> module avail     # OR 'ma'
```

### b. Local vs. Global Usage

As you probably guessed, we are going to use two places for the installed software:

* local builds `~/.local/easybuild`          (see `$LOCAL_MODULES`)
* global builds (provided to you by the UL HPC team) in `/opt/apps/resif/data/stable/default/modules/all` (see default `$MODULEPATH`).

Default usage (with the `eb` command) would install your software and modules in `~/.local/easybuild`.

Before that, let's explore the basic usage of [EasyBuild](http://easybuild.readthedocs.io/) and the `eb` command.

```bash
# Search for an Easybuild recipY with 'eb -S <pattern>'
$> eb -S Spark
CFGS1=/opt/apps/resif/data/easyconfigs/ulhpc/default/easybuild/easyconfigs/s/Spark
CFGS2=/home/users/<login>/.local/easybuild/software/tools/EasyBuild/3.6.1/lib/python2.7/site-packages/easybuild_easyconfigs-3.6.1-py2.7.egg/easybuild/easyconfigs/s/Spark
 * $CFGS1/Spark-2.1.1.eb
 * $CFGS1/Spark-2.3.0-intel-2018a-Hadoop-2.7-Java-1.8.0_162-Python-3.6.4.eb
 * $CFGS2/Spark-1.3.0.eb
 * $CFGS2/Spark-1.4.1.eb
 * $CFGS2/Spark-1.5.0.eb
 * $CFGS2/Spark-1.6.0.eb
 * $CFGS2/Spark-1.6.1.eb
 * $CFGS2/Spark-2.0.0.eb
 * $CFGS2/Spark-2.0.2.eb
 * $CFGS2/Spark-2.2.0-Hadoop-2.6-Java-1.8.0_144.eb
 * $CFGS2/Spark-2.2.0-Hadoop-2.6-Java-1.8.0_152.eb
 * $CFGS2/Spark-2.2.0-intel-2017b-Hadoop-2.6-Java-1.8.0_152-Python-3.6.3.eb
```

### c. Build software using provided EasyConfig file

In this part, we propose to build [High Performance Linpack (HPL)](http://www.netlib.org/benchmark/hpl/) using EasyBuild.
HPL is supported by EasyBuild, this means that an EasyConfig file allowing to build HPL is already provided with EasyBuild.

First of all, let's check if that software is not available by default:

```
$> module spider HPL

Lmod has detected the following error: Unable to find: "HPL"
```

Then, search for available EasyConfig files with HPL in their name. The EasyConfig files are named with the `.eb` extension.

```bash
# Search for an Easybuild recipY with 'eb -S <pattern>'
$> eb -S HPL
```


If we try to build `HPL-2.0-goolf-1.4.10`, nothing will be done as it is already installed on the cluster.

    $> eb HPL-2.0-goolf-1.4.10.eb

    == temporary log file in case of crash /tmp/eb-JKadCH/easybuild-SoXdix.log
    == tools/HPL/2.0-goolf-1.4.10 is already installed (module found), skipping
    == No easyconfigs left to be built.
    == Build succeeded for 0 out of 0
    == temporary log file(s) /tmp/eb-JKadCH/easybuild-SoXdix.log* have been removed.
    == temporary directory /tmp/eb-JKadCH has been removed.


However the build can be forced using the `-f` option flag. Then this software will be re-built.
(Tip: prefix your command with `time` to know its duration)

    $> time eb HPL-2.0-goolf-1.4.10.eb -f

    == temporary log file in case of crash /tmp/eb-FAO8AO/easybuild-ea15Cq.log
    == processing EasyBuild easyconfig /opt/apps/resif/devel/v1.1-20150414/.installRef/easybuild-easyconfigs/easybuild/easyconfigs/h/HPL/HPL-2.0-goolf-1.4.10.eb
    == building and installing tools/HPL/2.0-goolf-1.4.10...
    == fetching files...
    == creating build dir, resetting environment...
    == unpacking...
    == patching...
    == preparing...
    == configuring...
    == building...
    == testing...
    == installing...
    == taking care of extensions...
    == packaging...
    == postprocessing...
    == sanity checking...
    == cleaning up...
    == creating module...
    == COMPLETED: Installation ended successfully
    == Results of the build can be found in the log file /home/users/mschmitt/.local/easybuild/software/tools/HPL/2.0-goolf-1.4.10/easybuild/easybuild-HPL-2.0-20150624.113223.log
    == Build succeeded for 1 out of 1
    == temporary log file(s) /tmp/eb-FAO8AO/easybuild-ea15Cq.log* have been removed.
    == temporary directory /tmp/eb-FAO8AO has been removed.

    real    1m10.619s
    user    0m49.387s
    sys     0m7.828s


Let's have a look at `HPL-2.0-ictce-5.3.0` which is not installed yet.
We can check if a software and its dependencies are installed using the `-Dr` option flag:

    $> eb HPL-2.0-ictce-5.3.0.eb -Dr

    == temporary log file in case of crash /tmp/eb-HlZDMR/easybuild-JbndYN.log
    Dry run: printing build status of easyconfigs and dependencies
    CFGS=/opt/apps/resif/devel/v1.1-20150414/.installRef/easybuild-easyconfigs/easybuild/easyconfigs
     * [x] $CFGS/i/icc/icc-2013.3.163.eb (module: compiler/icc/2013.3.163)
     * [x] $CFGS/i/ifort/ifort-2013.3.163.eb (module: compiler/ifort/2013.3.163)
     * [x] $CFGS/i/iccifort/iccifort-2013.3.163.eb (module: toolchain/iccifort/2013.3.163)
     * [x] $CFGS/i/impi/impi-4.1.0.030-iccifort-2013.3.163.eb (module: mpi/impi/4.1.0.030-iccifort-2013.3.163)
     * [x] $CFGS/i/iimpi/iimpi-5.3.0.eb (module: toolchain/iimpi/5.3.0)
     * [x] $CFGS/i/imkl/imkl-11.0.3.163-iimpi-5.3.0.eb (module: numlib/imkl/11.0.3.163-iimpi-5.3.0)
     * [x] $CFGS/i/ictce/ictce-5.3.0.eb (module: toolchain/ictce/5.3.0)
     * [ ] $CFGS/h/HPL/HPL-2.0-ictce-5.3.0.eb (module: tools/HPL/2.0-ictce-5.3.0)
    == temporary log file(s) /tmp/eb-HlZDMR/easybuild-JbndYN.log* have been removed.
    == temporary directory /tmp/eb-HlZDMR has been removed.


`HPL-2.0-ictce-5.3.0` is not available but all it dependencies are. Let's build it:

    $> time eb HPL-2.0-ictce-5.3.0.eb

    == temporary log file in case of crash /tmp/eb-UFlEv7/easybuild-uVbm24.log
    == processing EasyBuild easyconfig /opt/apps/resif/devel/v1.1-20150414/.installRef/easybuild-easyconfigs/easybuild/easyconfigs/h/HPL/HPL-2.0-ictce-5.3.0.eb
    == building and installing tools/HPL/2.0-ictce-5.3.0...
    == fetching files...
    == creating build dir, resetting environment...
    == unpacking...
    == patching...
    == preparing...
    == configuring...
    == building...
    == testing...
    == installing...
    == taking care of extensions...
    == packaging...
    == postprocessing...
    == sanity checking...
    == cleaning up...
    == creating module...
    == COMPLETED: Installation ended successfully
    == Results of the build can be found in the log file /home/users/mschmitt/.local/easybuild/software/tools/HPL/2.0-ictce-5.3.0/easybuild/easybuild-HPL-2.0-20150624.113547.log
    == Build succeeded for 1 out of 1
    == temporary log file(s) /tmp/eb-UFlEv7/easybuild-uVbm24.log* have been removed.
    == temporary directory /tmp/eb-UFlEv7 has been removed.

    real    1m25.849s
    user    0m49.039s
    sys     0m10.961s


To see the newly installed modules, you need to add the path where they were installed to the MODULEPATH. On the cluster you have to use the `module use` command:

    $> module use $HOME/.local/easybuild/modules/all/

Check which HPL modules are available now:

    $> module avail HPL

    ------------- /mnt/nfs/users/homedirs/mschmitt/.local/easybuild/modules/all -------------
        tools/HPL/2.0-goolf-1.4.10    tools/HPL/2.0-ictce-5.3.0 (D)

    ---------------- /opt/apps/resif/devel/v1.1-20150414/core/modules/tools ----------------
        tools/HPL/2.0-goolf-1.4.10

The two newly-built versions of HPL are now available for your user. You can use them with the usually `module load` command.

### Iris

Let's search for available EasyConfig files with HPL in their name. The EasyConfig files are named with the `.eb` extension.

    $> eb -S HPL

		CFGS1=/home/users/sdiehl/.local/easybuild/software/tools/EasyBuild/3.2.1/lib/python2.7/site-packages/easybuild_easyconfigs-3.2.1-py2.7.egg/easybuild/easyconfigs
		 * $CFGS1/h/HPL/HPL-2.0-foss-2014b.eb
		 * $CFGS1/h/HPL/HPL-2.0-goolf-1.4.10.eb
		 * $CFGS1/h/HPL/HPL-2.0-goolf-1.5.16.eb
		 * $CFGS1/h/HPL/HPL-2.0-ictce-5.3.0.eb
		 * $CFGS1/h/HPL/HPL-2.0-ictce-6.1.5.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayCCE-2015.06.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayCCE-2015.11.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayGNU-2015.06.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayGNU-2015.11.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayGNU-2016.03.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayGNU-2016.04.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayGNU-2016.06.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayIntel-2015.06.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayIntel-2015.11.eb
		 * $CFGS1/h/HPL/HPL-2.1-CrayIntel-2016.06.eb
		 * $CFGS1/h/HPL/HPL-2.1-foss-2015.05.eb
		 * $CFGS1/h/HPL/HPL-2.1-foss-2015a.eb
		 * $CFGS1/h/HPL/HPL-2.1-foss-2015b.eb
		 * $CFGS1/h/HPL/HPL-2.1-foss-2016.04.eb
		 * $CFGS1/h/HPL/HPL-2.1-foss-2016.06.eb
		 * $CFGS1/h/HPL/HPL-2.1-foss-2016a.eb
		 * $CFGS1/h/HPL/HPL-2.1-foss-2016b.eb
		 * $CFGS1/h/HPL/HPL-2.1-gimkl-2.11.5.eb
		 * $CFGS1/h/HPL/HPL-2.1-gmpolf-2016a.eb
		 * $CFGS1/h/HPL/HPL-2.1-gmvolf-1.7.20.eb
		 * $CFGS1/h/HPL/HPL-2.1-gmvolf-2016a.eb
		 * $CFGS1/h/HPL/HPL-2.1-goolf-1.7.20.eb
		 * $CFGS1/h/HPL/HPL-2.1-ictce-7.1.2.eb
		 * $CFGS1/h/HPL/HPL-2.1-ictce-7.3.5.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2014.06.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2014.10.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2014.11.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2014b.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2015.02.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2015.08.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2015a.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2015b.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016.00.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016.01.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016.02-GCC-4.9.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016.02-GCC-5.3.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016.03-GCC-4.9.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016.03-GCC-5.3.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016.03-GCC-5.4.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016a.eb
		 * $CFGS1/h/HPL/HPL-2.1-intel-2016b.eb
		 * $CFGS1/h/HPL/HPL-2.1-iomkl-2015.01.eb
		 * $CFGS1/h/HPL/HPL-2.1-iomkl-2015.02.eb
		 * $CFGS1/h/HPL/HPL-2.1-iomkl-2015.03.eb
		 * $CFGS1/h/HPL/HPL-2.1-iomkl-2016.07.eb
		 * $CFGS1/h/HPL/HPL-2.1-pomkl-2016.03.eb
		 * $CFGS1/h/HPL/HPL-2.1-pomkl-2016.04.eb
		 * $CFGS1/h/HPL/HPL-2.1-pomkl-2016.09.eb
		 * $CFGS1/h/HPL/HPL-2.1_LINKER-ld.patch
		 * $CFGS1/h/HPL/HPL-2.2-foss-2016.07.eb
		 * $CFGS1/h/HPL/HPL-2.2-foss-2016.09.eb
		 * $CFGS1/h/HPL/HPL-2.2-foss-2017a.eb
		 * $CFGS1/h/HPL/HPL-2.2-goolfc-2016.08.eb
		 * $CFGS1/h/HPL/HPL-2.2-goolfc-2016.10.eb
		 * $CFGS1/h/HPL/HPL-2.2-intel-2017.00.eb
		 * $CFGS1/h/HPL/HPL-2.2-intel-2017.01.eb
		 * $CFGS1/h/HPL/HPL-2.2-intel-2017.02.eb
		 * $CFGS1/h/HPL/HPL-2.2-intel-2017a.eb
		 * $CFGS1/h/HPL/HPL-2.2-intelcuda-2016.10.eb
		 * $CFGS1/h/HPL/HPL-2.2-iomkl-2016.09-GCC-4.9.3-2.25.eb
		 * $CFGS1/h/HPL/HPL-2.2-iomkl-2016.09-GCC-5.4.0-2.26.eb
		 * $CFGS1/h/HPL/HPL-2.2-iomkl-2017.01.eb
		 * $CFGS1/h/HPL/HPL-2.2-iomkl-2017a.eb
		 * $CFGS1/h/HPL/HPL-2.2-pomkl-2016.09.eb
		 * $CFGS1/h/HPL/HPL_parallel-make.patch

		Note: 15 matching archived easyconfig(s) found, use --consider-archived-easyconfigs to see them


Let's have a look at `HPL-2.2-intel-2017a` which is not installed yet.
We can check if a software and its dependencies are installed using the `-Dr` option flag:

    $> eb HPL-2.2-intel-2017a.eb -Dr

		== temporary log file in case of crash /tmp/eb-K1VnEh/easybuild-4C6ZpN.log
		Dry run: printing build status of easyconfigs and dependencies
		CFGS=/home/users/sdiehl/.local/easybuild/software/tools/EasyBuild/3.2.1/lib/python2.7/site-packages/easybuild_easyconfigs-3.2.1-py2.7.egg/easybuild/easyconfigs
		 * [x] $CFGS/m/M4/M4-1.4.17.eb (module: devel/M4/1.4.17)
		 * [x] $CFGS/b/Bison/Bison-3.0.4.eb (module: lang/Bison/3.0.4)
		 * [x] $CFGS/f/flex/flex-2.6.0.eb (module: lang/flex/2.6.0)
		 * [x] $CFGS/z/zlib/zlib-1.2.8.eb (module: lib/zlib/1.2.8)
		 * [x] $CFGS/b/binutils/binutils-2.27.eb (module: tools/binutils/2.27)
		 * [x] $CFGS/g/GCCcore/GCCcore-6.3.0.eb (module: compiler/GCCcore/6.3.0)
		 * [x] $CFGS/m/M4/M4-1.4.18-GCCcore-6.3.0.eb (module: devel/M4/1.4.18-GCCcore-6.3.0)
		 * [x] $CFGS/z/zlib/zlib-1.2.11-GCCcore-6.3.0.eb (module: lib/zlib/1.2.11-GCCcore-6.3.0)
		 * [x] $CFGS/h/help2man/help2man-1.47.4-GCCcore-6.3.0.eb (module: tools/help2man/1.47.4-GCCcore-6.3.0)
		 * [x] $CFGS/b/Bison/Bison-3.0.4-GCCcore-6.3.0.eb (module: lang/Bison/3.0.4-GCCcore-6.3.0)
		 * [x] $CFGS/f/flex/flex-2.6.3-GCCcore-6.3.0.eb (module: lang/flex/2.6.3-GCCcore-6.3.0)
		 * [x] $CFGS/b/binutils/binutils-2.27-GCCcore-6.3.0.eb (module: tools/binutils/2.27-GCCcore-6.3.0)
		 * [x] $CFGS/i/icc/icc-2017.1.132-GCC-6.3.0-2.27.eb (module: compiler/icc/2017.1.132-GCC-6.3.0-2.27)
		 * [x] $CFGS/i/ifort/ifort-2017.1.132-GCC-6.3.0-2.27.eb (module: compiler/ifort/2017.1.132-GCC-6.3.0-2.27)
		 * [x] $CFGS/i/iccifort/iccifort-2017.1.132-GCC-6.3.0-2.27.eb (module: toolchain/iccifort/2017.1.132-GCC-6.3.0-2.27)
		 * [x] $CFGS/i/impi/impi-2017.1.132-iccifort-2017.1.132-GCC-6.3.0-2.27.eb (module: mpi/impi/2017.1.132-iccifort-2017.1.132-GCC-6.3.0-2.27)
		 * [x] $CFGS/i/iimpi/iimpi-2017a.eb (module: toolchain/iimpi/2017a)
		 * [x] $CFGS/i/imkl/imkl-2017.1.132-iimpi-2017a.eb (module: numlib/imkl/2017.1.132-iimpi-2017a)
		 * [x] $CFGS/i/intel/intel-2017a.eb (module: toolchain/intel/2017a)
		 * [ ] $CFGS/h/HPL/HPL-2.2-intel-2017a.eb (module: tools/HPL/2.2-intel-2017a)
		== Temporary log file(s) /tmp/eb-K1VnEh/easybuild-4C6ZpN.log* have been removed.
		== Temporary directory /tmp/eb-K1VnEh has been removed.


`HPL-2.2-intel-2017a` is not available but all it dependencies are. Let's build it:

    $> time eb HPL-2.2-intel-2017a.eb

		== temporary log file in case of crash /tmp/eb-152mYB/easybuild-myA4bD.log
		== processing EasyBuild easyconfig /home/users/sdiehl/.local/easybuild/software/tools/EasyBuild/3.2.1/lib/python2.7/site-packages/easybuild_easyconfigs-3.2.1-py2.7.egg/easybuild/easyconfigs/h/HPL/HPL-2.2-intel-2017a.eb
		== building and installing tools/HPL/2.2-intel-2017a...
		== fetching files...
		== creating build dir, resetting environment...
		== unpacking...
		== patching...
		== preparing...
		== configuring...
		== building...
		== testing...
		== installing...
		== taking care of extensions...
		== postprocessing...
		== sanity checking...
		== cleaning up...
		== creating module...
		== permissions...
		== packaging...
		== COMPLETED: Installation ended successfully
		== Results of the build can be found in the log file(s) /home/users/sdiehl/.local/easybuild/software/tools/HPL/2.2-intel-2017a/easybuild/easybuild-HPL-2.2-20170609.155430.log
		== Build succeeded for 1 out of 1
		== Temporary log file(s) /tmp/eb-152mYB/easybuild-myA4bD.log* have been removed.
		== Temporary directory /tmp/eb-152mYB has been removed.

		real	0m54.624s
		user	0m14.651s
		sys	0m21.476s


To see the newly installed modules, you need to add the path where they were installed to the MODULEPATH. On the cluster you have to use the `module use` command:

    $> module use $HOME/.local/easybuild/modules/all

Check which HPL modules are available now:

    $> module avail HPL

		---------------------- /home/users/sdiehl/.local/easybuild/modules/all ----------------------
		   tools/HPL/2.2-intel-2017a


The newly-built version of HPL is now available for your user. You can use them with the usually `module load` command.

## Amending an existing EasyConfig file (Gaia only)

It is possible to amend existing EasyConfig file to build software with slightly different parameters.

As a example, we are going to build the lastest version of HPL (2.1) with ICTCE toolchain. We use the `--try-software-version` option flag to overide the HPL version.

    $> time eb HPL-2.0-ictce-5.3.0.eb --try-software-version=2.1

    == temporary log file in case of crash /tmp/eb-ocChbK/easybuild-liMmlk.log
    == processing EasyBuild easyconfig /tmp/eb-ocChbK/tweaked_easyconfigs/HPL-2.1-ictce-5.3.0.eb
    == building and installing tools/HPL/2.1-ictce-5.3.0...
    == fetching files...
    == creating build dir, resetting environment...
    == unpacking...
    == patching...
    == preparing...
    == configuring...
    == building...
    == testing...
    == installing...
    == taking care of extensions...
    == packaging...
    == postprocessing...
    == sanity checking...
    == cleaning up...
    == creating module...
    == COMPLETED: Installation ended successfully
    == Results of the build can be found in the log file /home/users/mschmitt/.local/easybuild/software/tools/HPL/2.1-ictce-5.3.0/easybuild/easybuild-HPL-2.1-20150624.114243.log
    == Build succeeded for 1 out of 1
    == temporary log file(s) /tmp/eb-ocChbK/easybuild-liMmlk.log* have been removed.
    == temporary directory /tmp/eb-ocChbK has been removed.

    real    1m24.933s
    user    0m53.167s
    sys     0m11.533s

    $> module avail HPL

    ------------- /mnt/nfs/users/homedirs/mschmitt/.local/easybuild/modules/all -------------
        tools/HPL/2.0-goolf-1.4.10    tools/HPL/2.0-ictce-5.3.0    tools/HPL/2.1-ictce-5.3.0 (D)

    ---------------- /opt/apps/resif/devel/v1.1-20150414/core/modules/tools ----------------
        tools/HPL/2.0-goolf-1.4.10

We obtained HPL 2.1 without writing any EasyConfig file.

**IMPORTANT**: LMod cache the modules available such that it may append that the `module avail HPL` command _does not_ report the newly created 2.1 version. In that case, you can use the following option:

    $> module --ignore-cache avail HPL

There are multiple ways to amend a EasyConfig file. Check the `--try-*` option flags for all the possibilities.


## Build software using your own EasyConfig file (Gaia only)

For this example, we create an EasyConfig file to build GZip 1.4 with the GOOLF toolchain.
Open your favorite editor and create a file named `gzip-1.4-goolf-1.4.10.eb` with the following content:

    easyblock = 'ConfigureMake'

    name = 'gzip'
    version = '1.4'

    homepage = 'http://www.gnu.org/software/gzip/'
    description = "gzip (GNU zip) is a popular data compression program as a replacement for compress"

    # use the GOOLF toolchain
    toolchain = {'name': 'goolf', 'version': '1.4.10'}

    # specify that GCC compiler should be used to build gzip
    preconfigopts = "CC='gcc'"

    # source tarball filename
    sources = ['%s-%s.tar.gz'%(name,version)]

    # download location for source files
    source_urls = ['http://ftpmirror.gnu.org/gzip']

    # make sure the gzip and gunzip binaries are available after installation
    sanity_check_paths = {
                          'files': ["bin/gunzip", "bin/gzip"],
                          'dirs': []
                         }

    # run 'gzip -h' and 'gzip --version' after installation
    sanity_check_commands = [True, ('gzip', '--version')]


This is a simple EasyConfig. Most of the fields are self-descriptive. No build method is explicitely defined, so it uses by default the standard *configure/make/make install* approach.


Let's build GZip with this EasyConfig file:

    $> time eb gzip-1.4-goolf-1.4.10.eb

    == temporary log file in case of crash /tmp/eb-hiyyN1/easybuild-ynLsHC.log
    == processing EasyBuild easyconfig /mnt/nfs/users/homedirs/mschmitt/gzip-1.4-goolf-1.4.10.eb
    == building and installing base/gzip/1.4-goolf-1.4.10...
    == fetching files...
    == creating build dir, resetting environment...
    == unpacking...
    == patching...
    == preparing...
    == configuring...
    == building...
    == testing...
    == installing...
    == taking care of extensions...
    == packaging...
    == postprocessing...
    == sanity checking...
    == cleaning up...
    == creating module...
    == COMPLETED: Installation ended successfully
    == Results of the build can be found in the log file /home/users/mschmitt/.local/easybuild/software/base/gzip/1.4-goolf-1.4.10/easybuild/easybuild-gzip-1.4-20150624.114745.log
    == Build succeeded for 1 out of 1
    == temporary log file(s) /tmp/eb-hiyyN1/easybuild-ynLsHC.log* have been removed.
    == temporary directory /tmp/eb-hiyyN1 has been removed.

    real    1m39.982s
    user    0m52.743s
    sys     0m11.297s


We can now check that our version of GZip is available via the modules:

    $> module avail gzip

    --------- /mnt/nfs/users/homedirs/mschmitt/.local/easybuild/modules/all ---------
        base/gzip/1.4-goolf-1.4.10



## To go further


- [EasyBuild homepage](http://easybuilders.github.io/easybuild)
- [EasyBuild documentation](http://easybuilders.github.io/easybuild/)
- [Getting started](https://github.com/easybuilders/easybuild/wiki/Getting-started)
- [Using EasyBuild](https://github.com/easybuilders/easybuild/wiki/Using-EasyBuild)
- [Step-by-step guide](https://github.com/easybuilders/easybuild/wiki/Step-by-step-guide)
