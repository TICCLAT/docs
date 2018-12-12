# TICCL Tools installation

The default way to install [TICCL Tools](https://github.com/LanguageMachines/ticcltools) as an end-user is through the [LaMachine suite](https://proycon.github.io/LaMachine/).
However, this is quite a heavy distribution with long installation and update procedures.
It also installs its own Python and other dependencies that may already be on your system.
In addition, [it doesn't play nice with Conda](https://github.com/proycon/LaMachine/issues/61#issuecomment-391157792) or Macports on macOS.
For these reasons, we did not find LaMachine suitable for our development purposes.

Here we document a few alternative ways to install TICCL Tools: from source, from Conda using released versions (including how to build the conda-forge packages) and using a Conda recipe to build your own master branch Conda package. **The latter option to be added soon.**

Note that we only tested building on macOS and Linux.

## Dependencies

For building:
- ``autotools``
- ``autoconf-archive``
- ``pkg-config``

For including headers and linking to (i.e. library/run-time dependencies):
- icu (``icu-dev``)
- xml2 (``libxml2-dev``)
- [TiCC Utils](https://github.com/LanguageMachines/ticcutils), which in turn depends on (in addition to the above build dependencies and also icu and xml2):
    - Boost and Boost Regex (``libboost-dev`` and ``libboost-regex-dev``)
    - libtar (``libtar-dev``)
    - bz2 bindings (``libbz2-dev``)
    - zlib bindings (``zlib1g-dev``)

## Manual build from source

After installing the dependencies, TICCL can be configured using GNU Autotools and built with `make`.

```sh
sh bootstrap.sh
./configure --prefix=$YOUR_CHOICE
make
make install
```

The `bootstrap.sh` script does some quick environment checks and then runs `autoreconf`, which generates the `configure` script.

Optionally, after installing, one can call `make check` to run some tests.

## Install from conda-forge
Easy as:

```sh
conda install ticcltools -c conda-forge
```

This requires Conda to be installed, [Conda installation instructions can be found here](https://conda.io/docs/user-guide/install/index.html).

## Build a Conda package
Of course, to installing from conda-forge, we had to build the Conda package and some of its dependencies.

The recipes we wrote can be found in the conda-forge GitHub organization:
- [autoconf-archive feedstock](https://github.com/conda-forge/autoconf-archive-feedstock)
- [libtar feedstock](https://github.com/conda-forge/libtar-feedstock)
- [ticcutils feedstock](https://github.com/conda-forge/ticcutils-feedstock)
- [ticcltools feedstock](https://github.com/conda-forge/ticcltools-feedstock)

### Writing a C(++) Conda recipe
The first thing to do is to fork the [conda-forge/staged-recipes](https://github.com/conda-forge/staged-recipes) repository on GitHub.
Clone your forked repo and create a branch off master for your recipe, for instance for a `ticcltools` branch:
```sh
git clone https://github.com/[YOUR USER/ORG]/staged-recipes
cd staged-recipes
git checkout -b ticcltools
```

Then create the first file of your recipe by copying the `example` folder:
```sh
cp recipes/example recipes/ticcltools
```

#### `meta.yaml`
This will give you a `recipes/ticcltools/meta.yaml` file that is mostly self-explanatory to edit.
For C(++) projects, you'll probably want to remove the `build/script` and `test/imports` keys.
Also, it is important to properly separate the three classes of requirements/dependencies: build, host and run.

- Build dependencies are only used at build time, things like the compiler, autoconf/make, cmake and pkg-config.
- Host dependencies are those that you will link against during build time.
  The difference with build dependencies is that host ones are the ones you will use on the target platform, i.e. the one you're building the package for.
  This allows for cross-compilation
- Run dependencies are the things that are needed only at run time.
  These will be installed together with your package.

See the Conda documentation for [more on the different requirements sections](https://conda.io/docs/user-guide/tasks/build-packages/define-metadata.html#requirements-section).

One more thing to consider when building libraries is [run exports (see documentation)](https://conda.io/docs/user-guide/tasks/build-packages/define-metadata.html#export-runtime-requirements).
This allows downstream dependent recipes to pin versions of your package.
In our libtar and TiCC Utils recipes, we had to add this.

#### `build.sh`
For C(++) packages, we still need to add a `build.sh` script containing the commands to compile the code.
For instance, this could contain the commands given above for [building from source](#Manual-build-from-source).
And indeed, this is what we do in the [TICCL Tools recipe `build.sh` script](https://github.com/TICCLAT/staged-recipes/blob/ticcltools/recipes/ticcltools/build.sh).

### Building C(++) Conda recipes locally
While writing and testing a Conda recipe, it is very convenient to first get it to build on your local machine (and, in fact, [the conda-forge guidelines strongly encourage this](https://conda-forge.org/docs/recipe.html#checklist) before making a PR).

This can be done by installing conda-build (of course, using Conda ;)):
```sh
conda install conda-build
```

You can run it on your recipe by simply pointing conda-build to it.
For instance, if you're in the `staged-recipes` base folder, run it like:
```sh
conda-build recipes/ticcltools
```

Conda build by default cleans up after itself.
However, during debugging, it can be useful to keep the temporary files that conda build created to see what went wrong in your build.
Pass the `--dirty` flag to `conda-build` in that case.

After a successful build, the package will be stored in Conda's local cache.
This is, again, really useful during development, because this means that you can now install the package from the "local channel" if you need it immediately (merging packages into conda-forge can take some time, as it is run by volunteers):
```sh
conda install ticcltools -c local
```

Alternatively, you could now upload the package to your own channel.
The `conda-build` output tells you how to do this (create an account on Anaconda Cloud and follow instructions).
That way others can also benefit from your build, if they are using the same operating system.

#### macOS: XCode SDK version
On macOS, when the code you're building links to system calls, you need to have the right version of the XCode SDK.
The current conda-forge toolchain is based on SDK 10.9.
The Conda documentation explains [how to modify your recipe to use SDK 10.9](https://conda.io/docs/user-guide/tasks/build-packages/compiler-tools.html?highlight=sdk#macos-sdk).

1. Download the proper SDK, e.g. from phracker.
2. Put it somewhere on your disk.
3. Create a file in your home directory called `conda_build_config.yaml` ([you can put it somewhere else if you like](https://conda.io/docs/user-guide/tasks/build-packages/variants.html#conda-build-variant-config-files), but don't put it in the recipe, conda-forge doesn't allow that).
4. Add the `CONDA_BUILD_SYSROOT` configuration (see linked documentation).

### Submitting to conda-forge
When your recipe builds on your local machine, make a pull request for your branch back to the conda-forge repo.
When everything builds on the CI services as well (and the linter is happy), ping reviewers with a simple PR comment like *"@conda-forge/staged-recipes ready for review."*