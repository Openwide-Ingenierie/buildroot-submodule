# Buildroot Submodule

This project is an infrastructure to help you manage a firmware project based on [buildroot](www.buildroot.org) while respecting the best practices of configuration management with git.

* Buildroot modifications are made in the buildroot subdirectory
* project specific modifications are cleanly separated
* everything is saved as a git project
* your project's git repository doesn't need to contain all of buildroot's history
* handle variants of the same project properly

## Starting a simple project

If your project has no variants (you target only a single type of hardware), starting a new project is pretty simple:

* download the latest version of this project as a zip file (do not git-clone it, see the section about contributing)
* uncompress it in the subdirectory you want to work in
* run `git init` to initialize your new git repository
* run `git add common.mk Config.in external.mk Makefile LICENSE`
* run `git submodule add git://git.buildroot.net/buildroot buildroot` to clone buildroot in the proper subdirectory.
* If you want to use a specific version of buildroot run the following commands:
```
cd buildroot
git checkout <version tag>
cd ..
git add buildroot
```
* run `git commit`

You now have an infrastructure that allows you to work on a project based on buildroot but that allows you to keep your changes cleanly separated from changes to buildroot itself.

* You can call any buildroot make target directly (menuconfig, xxx_defconfig etc...)
* You can add your own configuration options in the _Config.in_ file at the root of the project
* You can add your own _make_ comand to the toplevel _external.mk_ 
* Your configuration will be saved in a toplevel _defconfig_ file that you can manage with git
* Any change that is meant to be upstreamed can be done in the _buildroot_ submodule
* You can add files to overlay in an _overlay_ subdirectory
* You can add patches (following the buildroot naming convention) in the _patch_ subdirectory
* You can override the source for a specific package by setting an override in _local.mk_


All these mechanism are standard buildroot features that have been properly configured for you. Please refer to [the buildroot documentation](http://buildroot.org/docs.html) to learn more about them.

  
## Project with variants

It is possible to have multiple projects built in the same directory using buildroot-submodule.
* copy the file _Makefile_ to a new name (like _Makefile.Project1_) 
* create a directory for your subproject (will contain _defconfig_ and _output_)
* update the PROJECT_NAME variable to that directory (in `<new makefile>`)
* run `make -f <new makefile> <target>` to run make on that target

By default, variants will share
* The _dl_ directory (downloaded sources)
* The _buildroot_ directory (infrastructure and versions to use)
* The _Config.in_ file (extra configuration options for menuconfig)
* The _external.mk_ file (extra make targets and instructions)
* The _local.mk_ file (defining OVERRIDE_SRCDIR variables)
* The _patch_ directory (patches to be applied by buildroot before build)
* The _overlay_ directory (files to be added to the target filesystem after everything is built)

But they will not share
* The _defconfig_ file (main buildroot configuration)
* The _output_ directory (build intermediate steps and products)
 
Most of these choices are set via normal buildroot configuration options that you can override via _menuconfig_.

## Contributing to buildroot

It is common, when developing a buildroot-based project, that you need to update some package provided by buildroot. buildroot-submodule tries to ease that process by separating buildroot changes from your code. The buildroot subdirectory is a normal buildroot clone on which you can work on upstreamable changes following best practices described in the buildroot documentation.

## Contributing to buildroot-submodule
If the infrastructure provided by buildroot-submodule does not satify your use-case, you can change the infrastructure and we will gladly look at your changes and upstream them if they are good.

Note that the normal method of deploying buildroot-submodule does **not** clone buildroot-submodule. Cloning buildroot-submodule is a bad idea for normal developement because the git history of buildroot-submodule would pollute your own project's history.

## Licence

buildroot-submodule is provided under the GPLv3 or later. The licence is provided in the _LICENCE_ file. Note that this licence only covers the files provided by buildroot-submodule. It does not cover buildroot (which is GPLv2 or later) nor any software installed by buildroot (they have their own licences) nor your own code (which you are free to licence as you want).

## Using buildroot-submodule to build a toolchain separately
One of the best ways to speed-up buildroot builds is to build the toolchain separately. buildroot-submodule can be used to simplify that process while still allowing you to build your toolchain using buildroot and using git to make sure all users of your project have the same toolchain configuration.

The idea is to make two variants of the same project, one to build the toolchain and one for the target filesystem, the second variant using the first one as an external toolchain

* Create a new _Makefile_ for the variant `cp Makefile Makefile.toolchain`
* Edit the new file to rename the project to _toolchain_
* Configure the new subproject properly with `make -f Makefile.toolchain menuconfig`
  * Configure the toolchain itself according to your need
  * deactivate the linux kernel option
  * set init system to _None_ and deactivate busybox
* make the toolchain with `make -f Makefile.toolchain toolchain`
* configure your main project to use the built toolchain
  * change the toolchain type to _external toolchain_
  * set the toolchain location to _$(BR2_EXTERNAL_BUILDROOT_SUBMODULE_PATH)/toolchain/output/host/usr_
  * set all toolchain options to reflect what you have set in your toolchain project
* check that the configuration is correct : `make toolchain`

you can now build your normal project and _make clean_ won't erase the compiler entirely

## Adding gdb to the toolchain built separately
The configuration decribed above does not include GDB in the toolchain, if you need it, follow the extra steps bellow (it can be done direcly).
* Update the configuration of the toolchain subproject with `make -f Makefile.toolchain menuconfig`
  * disable all rootfs specific tasks:
    * disable root login with password
    * set /bin/sh to none
    * disable Run a getty (login prompt) after boot
    * disable remount root filesystem read-write during boot
    * disable all Filesystem images options (tar, ext, cpio...)
  * select build cross gdb for the host and the options you need
  * add the gdb + gdb-server package (from Target packages > Debugging, profiling and benchmark as for buildroot this is part of your rootfs, but here it's a very minimal one)
* make the toolchain with `make -f Makefile.toolchain` (note the full build: you need to build the gdb-server target package)
* update the configuration of your main project
  * select Copy gdb server to the Target
* check that the configuration is correct : `make toolchain`

## Some notes on configuration management
One of the design goals of buildroot-submodule is to keep buildroot's _absolutely rebuildable_ philosophy. 
* Users should clone it, build it and no other steps should be needed to create a filesystem
* All build should be identical. They should depends only on a minimal set of external tools

To properly use buildroot-submodule it is important to keep all relevant files under git. This includes
* The defconfig file itself
* all files in the _overlay_ and _patch_ subdirectories
* any post-build scripts
* all custom packages
* All sources of sub-projects that are not kept under a configuration system elsewhere
* Checking out subproject as submodules when they need to be modified locally but are kept under git elsewhere

It is a highly recommanded practice to regularly rebuild your project from scratch
* from a different directory (to make sure there are no hardcoded path)
* as much as possible on a freshly installed machine (to make sure it doesn't depend on external packages)


## Customising your project
The following section is a simplified summary of the different ways to customize a buildroot build. Please refer to the buildroot documentation for more details

### Adding files/directories to the target
Buildroot provides a way to easily add or replace a file on the target filesystem. Adding the file in the _overlay_ subdirectory will add the file to the target filesystem.
For instance, adding a file  _overaly/etc/inittab_ will replace the inittab provided by buildroot with your custom version.

This is the recommended way to add custom configuration files to the target
### Patching a package provided by buildroot
If one of the packages provided by buildroot has a bug and you find a patch fixing the bug that hasn't been commited upstream (or that is only available in unreleased builds), you will want to have buildroot apply the patch automatically:

* Find the exact name of the package. That is the name of the directory in _buildroot/package/_
* Create a directory _patch/< package >/_
* put your patch in it with the following naming convention : _< package >-< number >-< description >.patch_ where the number will be used to determine the order in whitch patches are applied

This is the recommanded way to integrate external patches. You should always consider the upstream status of patches and if upstreaming is possible.


### Adding a custom package to buildroot
If you need to add a package that is not part of buildroot you can do that externally to buildroot. There are a few cases where you might want to do that
* Your project depends on a specific version of buildroot that does not have that package yet
* Your project depends on software that is internal to your company and is not opensourced.

If you are not in one of those cases, you probably want to upstream your changes to buildroot and work directly in the buildroot subdirectory.

To add a custom package
* create a subdirectory named after your package
* add a _.mk_ file in that directory (see the buildroot documentation for its content)
* include the new makefile in _external.mk_
* add a _Config.in_ file in that directory (again, see the buildroot documentation for its content)
* include the new config file in the toplevel _Config.in_

The new package and its options will appear in menuconfig.

Note that buildroot will usually fetch sources from version control servers, but is also able to fetch sources from a local directory or tarball. See the buildroot documentation for details.


### Compiling your own sources for a package provided by buildroot

If you need to modify the source code of a package provided by buildroot it is handy to have buildroot use a local, uncompressed copy of the sources instead of fetching from the internet.

This only applies for packages that are managed by buildroot proper. If the package is a custom package, you probably want to set the fetch method to _local_ instead.

* if the package is using git, clone the source you want to modify as a submodule (to guarantee source consistancy) 
  * Clone the package itself
  * Checkout the exact version used by buildroot
  * Create a branch for your local changes
  * Apply any patches that Buildroot will usually apply using _git am_
* if not copy/checkout the source locally and use another tool to follow apply buildroot patches (_quilt_ can probably do that)
* add the following line to the file _local.mk_

```
 <packagename>_OVERRIDE_SRCDIR=$(BR2_EXTERNAL_BUILDROOT_SUBMODULE_PATH)/path/to/source
```
   The variable _$(BR2_EXTERNAL_BUILDROOT_SUBMODULE_PATH)_ will expand to the toplevel directory of your project. Note that _packagename_ should be all cap here.

Buildroot will use these source instead of the version recommande by its own configuration but will still use its internal knowledge of the package to compile it.

Once your changes are ready you can regenerate the patch serie, including the patches Buildroot already integrates using _git format patch_

### Customizing the final filesystem
Usually, you want to customize your target filesystem by ovelaying files as described above in this document. But in some case it is not possible to use static files and running a shell script on the generated filesystem can be very handy.

* set the script to run in the menuconfig option _system configuration => script to run before generating images_ (use $(BR2_EXTERNAL_BUILDROOT_SUBMODULE_PATH) to have a path relative to your toplevel project directory)
* add the script at the proper place

### Adding your own documentation to a buildroot-submodule project

Buildroot has a very complete infrastructure to generate documentation based on _asciidoc_ This infrastructure can easily be reused to create your own documents. As per usual, please refer to the buildroot documentation for all the details. Here is a quick run-through

* create a directory named after the document you want to create
* add a _.mk_ file in that directory with the same name as the directory.
* include the new makefile in _external.mk_
* add the source of your document ( _asciidoc_ text files) to the directory
* add the following in the _.mk_ file (with _foo_ replaced with the name of your document)

```
################################################################################
#
# foo-document
#
################################################################################

FOO_SOURCES = $(sort $(wildcard $(BR2_EXTERNAL_BUILDROOT_SUBMODULE_PATH)/foo/*))
$(eval $(call asciidoc-document))
```

* run _make foo_
