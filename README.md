# Meta-abicompat

This is a work in progress!

This is a demo project that shows how libabigail can be integrated into a Yocto build to hook ABI compatibility and HashEquiv together. By using libabigail to get a textual representation of a shared libraries ABI, we can inject this into the hash equivalence mechanism instead of the file checksum. The result is the ability to avoid rebuilding applications when library ABIs are not changing.

# Background

## Yocto Tasks and Recipes

In Yocto, recipes are defined which contain tasks that Bitbake executes. These tasks are prefixed with "do_".

## Yocto Dependencies

Yocto has a clever isolation and dependencies mechanism. Basically recipes deploy files using populate_sysroot into a shared sysroot and recipes can only consume files from dependencies using that shared sysroot.

This dependency is between do_populate_sysroot of one recipe and do_prepare_recipe_sysroot of the other recipe.

```
libalpaca:do_fetch                         alpaca-farm:do_fetch
libalpaca:do_unpack                        alpaca-farm:do_unpack
libalpaca:do_patch                         alpaca-farm:do_patch
libalpaca:do_prepare_recipe_sysroot     -> alpaca-farm:do_prepare_recipe_sysroot
libalpaca:do_configure                  |  alpaca-farm:do_configure
libalpaca:do_compile                    |  alpaca-farm:do_compile
libalpaca:do_install                    |  alpaca-farm:do_install
libalpaca:do_populate_sysroot -----------  alpaca-farm:do_populate_sysroot
libalpaca:do_package                       alpaca-farm:do_package
```

## Sstate-cache

Sstate-caching is the main caching method used in Yocto projects. It captures all the relevant metadata in a recipe task and creates a checksum (or better known as the sstate hash). If this task is marked to be stored in sstate-cache, a tarball of the task output files is created with that hash in the filename. When Yocto is running, it can check for the existence of these tarballs and restore the files from it instead of re-generating the files.

This can save significant time, as essentially it can avoid recompiling applications for which it has already generated and stored into the sstate-cache.

The problem though is that this mechanism is very sensitive and these checksums are chained together for dependencies so changes in one recipe may trigger an avalance of rebuilds in other recipes that re-use those results - despite not really needing to rebuild.

Eg. libalpaca provides a shared library and alpaca-farm consumes this shared library. When the recipe for libalpaca recompiles, the recipe for alpaca-farm must recompile in case the shared library has a changed ABI and alpaca-farm needs to be re-linked.

To resolve this issue, some smart Yocto developers created a feature called Hashequiv.

## HashEquiv

Hashequiv is a powerful Yocto feature that works in addition to the sstate-cache. Essentially it allows adding a hook to the sstate-caching which calculates a full checksum of the sstate-cached files. This checksum is then tagged as an alias to the sstate-cache hash. This means if the sstate-cache hash changes, Yocto can identify equivalent hashes and sstate-cache tarballs and re-use these. It can then remap all the chained dependencies so that they do not rebuild - as the results of the changed task do not affect them since the files are the same.

Eg. libalpaca provides a shared library and alpaca-farm consumes this shared library. When the recipe for libalpaca recompiles but the library file stays the same, the recipe for alpaca-farm can avoid recompiling because its previous sstate-cached results are still valid.

The problem that meta-abicompat looks at though is what happens if the shared library changes, but in an ABI compatible manner? In this case the file checksums change so alpaca-farm recompiles. However if the ABI is stable for the shared library, alpaca-farm could avoid recompiling. This would be an optimisation to HashEquiv that could increase the speed of Yocto builds.

## Libabigail / abidw

Libabigail is a project that provides tools to compare and provide detailed information about the ELF ABI interface of shared libraries. The tool abidw can be used to create a textual representation of an ABI interface from a shared library.

See https://sourceware.org/libabigail/ for more information about libabigail.

See https://sourceware.org/libabigail/manual/abidw.html for more information about abidw.

## Meta-abicompat

The idea behind meta-abicompat is to demo hooking libabigail into the HashEquiv functionality so that instead of using the file checksum of a shared library in the checksum mechanism, to instead use the ABI of the shared library.

We can do this by using abidw to dump the ABI of shared libraries and use that instead of the file checksum when calculating the HashEquiv checksums.

The following puzzle pieces are put together to do so.

## Meta-abicompat: conf/layer.conf

The layer configuration file does some reconfiguration of Poky:
1) Injects a dependency on abidw-native for all target and nativesdk recipes.
2) Inject the abicompat bbclass into all recipes.
3) Switches the hashequiv algorithm to abicompat.sstatesig.ABICompatBasicHash which is provided by this layer.
4) Control which versions of the alpaca and bandicoot recipes to use.

## Meta-abicompat: classes/abicompat.bbclass

This bbclass is injected as a base inherit for all recipes. It adds our own SSTATE_PKGSPEC variable that allows avoiding recipe metadata from affecting the hashequiv checksum.

It also has a placeholder for some functions to restrict the abi checker to a select number of recipes instead of all recipes by default.

## Meta-abicompat: abicompat/sstatesig.py

This file brings in our HashEquiv hook function. It's based on the original hook from Poky but modified to do the following:

1) Shared libraries have their ABI dumped for the checksum calculation instead of the file checksum
2) Cataloguing of symlinks to shared libraries are modified to not include version specific filenames.

To dump the ABI, we essentially call:
```
abidw --no-corpus-path <so_file>
```

The use of --no-corpus-path is to avoid embedding the filename into the ABI dump. This filename typically includes the version numbers of the shared library which we do not want to include. We want the hash checksum to represent the ABI interface, not the version of the shared library as a change in version may have the same ABI interface.

## Meta-abicompat: abidw

Recipe to build and deploy into the Yocto environment the libabigail abidw tool.

## Meta-abicompat: alpaca

This is a toy project where a shared library named libalpaca is created that exposes a single function named count_alpacas. This function returns an integer. There are four versions of this project featuring ABI compatible updates and ABI incompatible updates.

v1.0.0: initial commit

v1.0.1: cosmetic change in build scripts (abi compatible)

v1.0.2: return 1 from count_alpacas (abi compatible)

v1.0.3: return 2 from count_alpacas (abi compatible)

v1.1.0: add magic_counter function (abi compatible)

v2.0.0: switch return type from int to int pointer (abi incompatible)

## Meta-abicompat: alpaca-farm

Project that consumes libalpaca and is compatible with v1.0.0 to v1.1.0.

## Meta-abicompat: bandicoot

This is a toy project where a shared library named libbandicoot is created that exposes a single function named count_bandicoots. This function returns an integer. There are two versions of this project featuring ABI compatible updates.

The main difference to libalpaca is however using the -fvisibility=hidden feature of gcc to restrict an internal function from being exposed to the library ABI. This demonstrates how the ABI of a shared library can be controlled in a way that improves the ABI checker usage.

In this case, the bump to v1.0.1 avoids a recompilation in bandicoot-farm while the corresponding change in libalpaca (v1.0.3 to v1.1.0) would cause a recompilation of alpaca-farm. 

v1.0.0: initial commit

v1.0.1: add magic_counter internal function (abi compatible)

See https://labjack.com/news/simple-cpp-symbol-visibility-demo for more information about controlling the exposed ABI of a shared library.

## Meta-abicompat: bandicoot-farm

Project that consumes libbandicoot and is compatible with v1.0.0 to v1.0.1.

## Poky Changes: Creation of do_packagesplit

Although the ABI of a library may stay the same, normally the package data of the recipe will change. The problem with the current configuration of Poky is that our dependency chain is not as simple as previously described in the section "Yocto Dependencies". Actually since the packaging of one recipe normally needs package data of the dependencies, there is an additional link. do_package of one recipe is dependent on the do_package of another recipe when they are chained via runtime dependencies.

```
libalpaca:do_fetch                         alpaca-farm:do_fetch
libalpaca:do_unpack                        alpaca-farm:do_unpack
libalpaca:do_patch                         alpaca-farm:do_patch
libalpaca:do_prepare_recipe_sysroot     -> alpaca-farm:do_prepare_recipe_sysroot
libalpaca:do_configure                  |  alpaca-farm:do_configure
libalpaca:do_compile                    |  alpaca-farm:do_compile
libalpaca:do_install                    |  alpaca-farm:do_install
libalpaca:do_populate_sysroot -----------  alpaca-farm:do_populate_sysroot
libalpaca:do_package --------------------> alpaca-farm:do_package
```

Therefore, if do_package needs to be re-run because the package data has changed, then do_install and so do_compile needs to be re-run (because they are not stored in the sstate-cache).

To allow avoiding recompilation of an application despite some packaging data changing in the dependency, this project includes the creation of a new task named do_packagesplit. The aim is to split do_package into two functions. One which just captures the packaged files, the other which handles the packaging data. The first is then stored into the sstate-cache.

By doing so, when a dependency recompiles, rather than needing to recompile the recipe, the sstate-cache of do_package can be re-used and the package data updated and the packages are re-created.

```
libalpaca:do_fetch                         alpaca-farm:do_fetch
libalpaca:do_unpack                        alpaca-farm:do_unpack
libalpaca:do_patch                         alpaca-farm:do_patch
libalpaca:do_prepare_recipe_sysroot     -> alpaca-farm:do_prepare_recipe_sysroot
libalpaca:do_configure                  |  alpaca-farm:do_configure
libalpaca:do_compile                    |  alpaca-farm:do_compile
libalpaca:do_install                    |  alpaca-farm:do_install
libalpaca:do_populate_sysroot -----------  alpaca-farm:do_populate_sysroot
libalpaca:do_package                       alpaca-farm:do_package
libalpaca:do_packagesplit ---------------> alpaca-farm:do_packagesplit
```

This problem is actually only visible when rm_work is applied. If rm_work is disabled, then the results of do_install can be re-used to avoid recompilation.

# Demo

Clone my fork of poky with the changes to include do_packagesplit (on branch abiequiv) and clone meta-abicompat into this poky setup.
```
git clone https://github.com/bmwcarit/meta-abicompat-poky.git poky
cd poky
git checkout abiequiv
git clone https://github.com/bmwcarit/meta-abicompat.git
```

Next initiate the poky environment
```
. oe-init-build-env
```

Add the meta-abicompat meta-layer to build/conf/bblayers.conf like in the following example.
```
BBLAYERS ?= " \
  /home/michael/Projects/poky/meta-abicompat \
  /home/michael/Projects/poky/meta \
  /home/michael/Projects/poky/meta-poky \
  /home/michael/Projects/poky/meta-yocto-bsp \
  "
```

Enable rm_work by adding the following statement to the build/conf/local.conf file so we ensure that no intermediate compile results in the workdir get reused.
```
INHERIT += "rm_work"
```

Now compile the alpaca-farm recipe.

```
bitbake alpaca-farm | tee
```

Note we see:

```
WARNING: alpaca-farm-1.0.0-r1 do_compile: **************************
WARNING: alpaca-farm-1.0.0-r1 do_compile: * Compiling alpaca-farm! *
WARNING: alpaca-farm-1.0.0-r1 do_compile: **************************
```

This shows us alpaca-farm was compiled. After this, we have alpaca v1.0.0 and alpaca-farm compiled against alpaca v1.0.0 in the sstate-cache.

Now edit the meta-abicompat configuration to pick the v1.0.1 alpaca recipe. This is an abi compatible update that should not trigger a rebuild of alpaca-farm.

```
vim meta-abicompat/conf/layer.conf

# Control version of alpaca recipe conveniently here
# PREFERRED_VERSION_alpaca = "1.0.0"
PREFERRED_VERSION_alpaca = "1.0.1"
```

Now compile the alpaca-farm recipe again.

```
bitbake alpaca-farm | tee
```

If you look through the output, note now we don't see do_compile! Instead we see:

```
WARNING: alpaca-1.0.1-r0 do_populate_sysroot: Task cc4c779ccf3bf619f05e25c11cf6268e22f5f66e293bb9e1cad18efbf58f94ec unihash changed cc4c779ccf3bf619f05e25c11cf6268e22f5f66e293bb9e1cad18efbf58f94ec -> 28e6681ab3904657418ba2c29b55859f0426cf3de9f75adfeff3ec8c2904aa6a by server unix:///home/michael/Projects/poky/build/hashserve.sock
...
WARNING: Task /home/michael/Projects/poky/meta-abicompat/recipes-examples/alpaca/alpaca_1.0.1.bb:do_populate_sysroot unihash changed to 28e6681ab3904657418ba2c29b55859f0426cf3de9f75adfeff3ec8c2904aa6a
WARNING: Found unihash 4a7c650138e6066ad20cc7b07c74315f4463014759c57c396d459ff5e3e70ab6 in place of 4a7c650138e6066ad20cc7b07c74315f4463014759c57c396d459ff5e3e70ab6 for /home/michael/Projects/poky/meta-abicompat/recipes-examples/alpaca-farm/alpaca-farm.bb:do_package from unix:///home/michael/Projects/poky/build/hashserve.sock
WARNING: Found unihash ce3309f290ecfa5035a817d588a46bd4a893f64c2b6c9fe041067179e32ade04 in place of ce3309f290ecfa5035a817d588a46bd4a893f64c2b6c9fe041067179e32ade04 for /home/michael/Projects/poky/meta-abicompat/recipes-examples/alpaca-farm/alpaca-farm.bb:do_populate_sysroot from unix:///home/michael/Projects/poky/build/hashserve.sock
```

This shows that Yocto was able to alias the changed alpaca:do_populate_sysroot results and in doing so, mark the previous sstate-cached results of alpaca-farm again.

This shows we managed to skip recompiling alpaca-farm despite libalpaca being updated!

Now we can trigger an abi incompatible update that should cause alpaca-farm to recompile and fail to build with an error.

```
vim meta-abicompat/conf/layer.conf

# Control version of alpaca recipe conveniently here
# PREFERRED_VERSION_alpaca = "1.0.0"
# PREFERRED_VERSION_alpaca = "1.0.1"
# PREFERRED_VERSION_alpaca = "1.0.2"
# PREFERRED_VERSION_alpaca = "1.0.3"
# PREFERRED_VERSION_alpaca = "1.1.0"
PREFERRED_VERSION_alpaca = "2.0.0"
```

```
bitbake alpaca-farm | tee
...
| /home/michael/Projects/poky/build/tmp/work/cortexa57-poky-linux/alpaca-farm/1.0.0-r1/git/main.cpp:6:32: error: invalid conversion from 'int*' to 'int' [-fpermissive]
|     6 |     int alpacas = count_alpacas();
|       |                   ~~~~~~~~~~~~~^~
|       |                                |
|       |                                int*
| ninja: build stopped: subcommand failed.
```

This error is expected because the library has changed in an incompatible way that requires adaptations in alpaca-farm. By making alpaca-farm recompile and fail, this ensures that cases where the ABI changes is correctly caught and validated in Yocto.

You can repeat the same process with bandicoot, and see despite a new internal function being added, that bandicoot-farm does not recompile. I hope you can easily figure out how that works ;)

And that's it! This demo shows how libabigail and Yocto Hashequiv can be integrated together to avoid expensive application recompilations when library changes are ABI compatible.

