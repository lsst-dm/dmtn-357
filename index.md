# Using LSST Science Pipelines without EUPS

```{abstract}
Historically the LSST Science Pipelines have required that each package be installed using EUPS and that EUPS is used to configure the environment. With tens of packages installed the search path environment variables can be very long and this can slow down imports. This tech note describes the changes needed to allow the pipelines to run without EUPS being used.
```

## Introduction

The LSST Science Pipelines {cite:p}`PSTN-019` has been built with support for EUPS {cite:p}`EUPS` from the very beginning, inheriting the software from SDSS.
EUPS tracks every installed package and knows which package each depends on.
EUPS provides a very powerful way to mix and match different versions of software packages by allowing you to "set up" a specific collection and then do local overrides of development packages.
These facilities provide a powerful mechanism for pipeline developers to experiment and debug.

The way EUPS works is by using environment variables to override default search paths.
It commonly uses `$PYTHONPATH`, `$PATH` and `$[DY]LD_LIBRARY_PATH` to dynamically affect how Python code, binaries, and shared libraries are discovered.
This works very well but comes at a cost when the number of installed packages approaches the hundreds.
Using large paths like these significantly adds to the overhead of importing Python code and resolving link dependencies.
This is especially true for slow networked file systems where every single Python import is coupled with a search of the `PYTHONPATH` to look for the correct file to load, and this is made worse by the pipeline software sharing directory trees such that `python/lsst/daf` and `python/lsst/pipe`, for example, corresponds to multiple distinct Python packages.

The pipeline code has been designed to assume that EUPS is creating these environment variables and, importantly, has a corresponding `${PRODUCT}_DIR` environment variable set for every product, pointing at the root of the product tree.
Historically a pipelines package separates out config files from Python code and shared library files and Python code is allowed to refer to any file in the tree via the product environment variable (or the `lsst.utils.getPackageDir` API that knows about that environment variable).

## Going EUPS-Less

For the vast majority of pipelines users (the non-developers) the existence of EUPS is at best irrelevant and at worst confusing.
On the Rubin Science Platform {cite:p}`2024SPIE13101E..2BO` people are instructed to type `setup lsst_distrib` in their terminal sessions without really know why they need to do that, especially given that each RSP session uses a single Docker container corresponding to a single pipeline release version.
During data release processing using the CernVM-FS distribution {cite:p}`2024EPJWC.29501042H` of the science pipelines each job has to search the 100 paths trying to find which code to load, leading to much larger overhead on the file system than is desirable when thousands of jobs are active simultaneously.

Being able to make a CernVM-FS distribution and Docker container distributions, both of which are inherently a single coherent release of the pipeline software, with the software integrated into the Conda tree would result in significant performance gains for file access and remove end user complexity.
It is for this reason that we decided to change how we organize each pipelines package to make this possible.

## Default Package Layout

LSST Science Pipelines packages are built mostly using SCons with a standard layout (not all packages have all directories):

```
|- bin
|- config
|- doc
|- policy
|- pipelines
|- python
|- src
|- ups
```

Configs would refer to other configs using `getPackageDir` and pipelines would refer to pipeline fragments using `$PRODUCT_DIR`.
These approaches all assume that EUPS is creating the relevant `_DIR` environment variable.
We considered implementing a system where a user would run a script that would try to create all the missing `_DIR` environment variables but if we adopt a standard Python installation inside a Conda tree there is no suitable environment variable that can be set for each project.

The solution was to realize that Python packages know where they are installed (via the `importlib.metadata` package) which meant that if the config, pipelines, and policy files were somehow part of the Python tree rather than at their own top levels, then other code could find them without needing to set the environment variable.

## The Solution: Phase 1

We developed a phased approach to migrating the code away from assuming that there is a product environment variable.

* We developed a new `eups://` URI scheme in the `lsst-resources` package.
* When we see a file URI starting with `$<X>_DIR` we automatically convert it to an EUPS URI.
* We modified the `pex_config` package to understand `lsst-resources` URIs.
* We migrated each package such that the `config`, `pipelines`, and `policy` content would also be visible in the Python tree.
  This latter uses soft links to ensure that the old `getPackageDir` approach and the newer Python package resources approach would work.

To avoid confusing namespace clashing, when we move directories into the Python tree we adopted the convention of adding a `resources` path.
For example, for `obs_lsst` the `config` directory is soft linked to `python/lsst/obs/lsst/resources/config/`.
The `eups://` URI scheme is designed such that for `eups://product/` the URI handler first looks for a `PRODUCT_DIR` environment variable, and if that it not found it looks for `resource://lsst.product/resources` (and it searches with and without the `lsst` prefix).

Once every package has been modified to use the `eups` or `resource` URI schemes, the pipelines suite is then ready to be used without EUPS.

## The Solution: Installing into Conda

Having a built LSST Science Pipelines tree inside EUPS the critical phase for going EUPS-less is to install all the code into the host Conda environment.
We have written a script, distributed as part of `sconsUtils`.
The script scans the EUPS package list and copies the `python/` tree into the installed Python tree and also copyies `bin` files into the path and shared libraries and include files into the appropriate locations.
The shared libraries have to be patched to align the library load path with the actual installed location so that there is no need to set `LD_LIBRARY_PATH`.
Once this is done the pipeline software should be available in the Conda environment.

We have not considered whether it is worth trying to package all the files up as a single conda-forge package.

## What About Developers?

Nothing changes with the default installation using `lsstsw` or `eups distrib install` and the shared stack at SLAC will continue to be based on EUPS.
If a developer wishes to test some code by doing a local setup the intent is for this still to be possible, with the downside of there being some potential issues with shadowing (it seems that entry point discovery will return the installed entry points and the override entry points).
A `setup -j -r .` can still be used for a local setup and Python-only packages should override without any expected issues.
For packages depending on C++ packages the installation script will install the relevant `.cfg` files into the installed `sconsUtils` distribution to allow SCons to understand how to link them.

## Conclusions

This document explains the motivation for going EUPS-less and explains how packages have been modified to support this.
Having a version of the LSST Science Pipelines software distributed in CernVM-FA and Docker containers that do not require EUPS can result in significant speed ups of import times.
In some cases we have seen the import times half.

## References

```{bibliography}
```
