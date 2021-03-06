conda-build-prepare
===================

This repository contains a tool for unified conda package building.

The ``conda-build-prepare`` tool creates almost completely fixed package recipe, conda environment and git repositories by rendering metadata and embedding crucial data inside of them.
It makes the package building much more consistent regardless of the building time and environment.

The data is also extracted from the repository containing the recipe to populate the ``recipe_append.yaml`` file, which is automatically added to the metadata (as of ``conda-build`` v3.0).

Installation
------------

``conda-build-prepare`` can be installed using ``pip``, repository address and revision or branch name (optionally; ``@2a3dd9f67c4810fbcae82a76da6adbf35a6aa264`` is an example commit usage):

.. code-block:: bash
   :name: install_cbp

   python3 -m pip install git+https://github.com/antmicro/conda-build-prepare@2a3dd9f67c4810fbcae82a76da6adbf35a6aa264

Usage
-----

After installing ``conda-build-prepare`` or cloning the repository and using repository root as working directory, run:

.. code-block:: bash
   :name: prepare_package

   python3 -m conda_build_prepare --dir $DIRECTORY $PACKAGE

to prepare the *PACKAGE* recipe and store output files in a *DIRECTORY*.

After running the tool, the package can be built using the prepared environment and recipe using these commands:

.. code-block:: bash
   :name: build_package

   conda activate $DIRECTORY/conda-env
   conda build $DIRECTORY/recipe
   conda deactivate  # restores previous conda environment

*DIRECTORY* can be virtually any name but it has to match the ``--dir`` argument passed to prepare the package.

Building the example package
----------------------------

``wishbone-tool`` recipe is included for testing in this repository under the ``test`` directory.
The package can be successfully built with the aforementioned commands after specifying recipe path with ``PACKAGE`` variable and using any ``DIRECTORY``, e.g.:

.. code-block:: bash
   :name: set_envs

   PACKAGE=test/wishbone-tool
   DIRECTORY=X

Specifying additional channels
------------------------------

Additional channels can be specified by using ``--channels CHANNEL [CHANNEL ...]`` option.
Each ``CHANNEL`` will be added to the ``$DIRECTORY/conda-env/.condarc``.
The last one used will be the most important channel for ``conda-build`` during preparing and building.

Restoring original conda configurations
---------------------------------------

``conda-build-prepare`` "neutralizes" conda configuration files by commenting them out to make environment as reproducible as it's possible.
The original configuration can be restored later using:

.. code-block:: bash
   :name: restore_condarcs

   python3 -m conda_build_prepare restore

How conda-build-prepare works
-----------------------------

Preparation process consists of:

#. preparing the working directory,
#. extracting the build environment information,
#. preparing the build environment,
#. rendering the recipe,
#. cloning git source repositories,
#. preparing git tags for better version description,
#. embedding script_env variables.

Preparing the working directory
+++++++++++++++++++++++++++++++

The argument passed to ``--dir`` (``$DIRECTORY``) is used as a target directory name, in which ``conda-env``, ``git-repos`` and ``recipe`` directories will be created.
The directory specified as a ``$PACKAGE`` will be copied as the ``$DIRECTORY/recipe`` directory.

While all packages are expected to have a ``meta.yaml``, a *prescript* file (``prescript.${TOOLCHAIN_ARCH}.sh``) can be used to download or generate it.
*Prescript* file is executed right after copying the ``$PACKAGE``.

Extracting the build environment information
++++++++++++++++++++++++++++++++++++++++++++

Additional metadata is added based on the build environment:

#. repository containing the recipe: it's address, branch, commit and result of ``git describe``
#. type of the environment: local build (default), Travis CI or Github Actions,
#. additional information from Travis or Github Actions like event that started the build, job/run id etc.,
#. ``TOOLCHAIN_ARCH`` it's being prepared for (if such variable is set),
#. package's additional ``condarc`` contents (if any ``condarc`` is used).

Preparing the build environment
+++++++++++++++++++++++++++++++

Conda environment created in ``$DIRECTORY/conda-env`` will contain basic packages necessary for building and rendering metadata. 
Precisely, those basic packages are: ``anaconda-client``, ``conda-build``, ``conda-verify``, ``jinja2``, ``pexpect``, ``python`` and ``ripgrep`` (``ripgrep`` only on Linux and macOS).

In the next step, ``conda-build-prepare`` will look for all ``condarc`` files influencing freshly created environment.
All such files found by the tool will be "neutralized" by commenting them out.
Paths are added to the ``conda-build-prepare_srcs.txt`` file inside the system's temp dir (``tempfile.gettempdir()``) for the eventual future restoration, which can be triggered by the user.

Then, package's condarc (``condarc``, ``condarc_linux``, ``condarc_macos`` or ``condarc_windows`` from ``$PACKAGE``) will be set as the most important one (``conda-env/condarc``) and other basic settings will be applied to that environment.

Rendering the recipe
++++++++++++++++++++

The goal is to set each package used for building with specific version.
This will allow to always use the same packages during building even if any of the required ``build`` or ``host`` package gets updated in used channels.

Conda environment created in ``$DIRECTORY/conda-env`` is used for rendering the recipe to ensure the same settings with future building (channels, channel priority etc.).

Rendered version of the recipe will replace the original ``meta.yaml`` copied from the ``$PACKAGE``.
Contents of the original ``meta.yaml`` will be left at the end of the new file as a comment.

``recipe_append.yaml`` is incorporated in the rendered recipe automatically by the ``conda render``.

Cloning git source repositories
+++++++++++++++++++++++++++++++

Due to problems with Conda's own repository management [TODO: what exactly?], ``conda-build-prepare`` clones the repository and changes the recipe for ``conda-build`` to operate on the cloned one.

Each ``git_url`` source repository is cloned to the ``$DIRECTORY/git-repos`` directory.
The relative submodules of those repositories (where submodule's url starts with ``../``) will also get cloned because ``git`` will search them in the same parent directory during building.

The resulting recipe will have each ``git_url`` replaced with the local path to a repository cloned from the original repository URL.

Preparing git tags for better version description
+++++++++++++++++++++++++++++++++++++++++++++++++

The previous version of conda-related tooling used to rewrite git tags in order to let conda automatically detect the version via ``git-describe``.

The ``conda-build-prepare`` tool makes version format unified among various packages.
This is achieved by checking recipe source's git tags for any version-like part and modifying it by leaving only this version-like part prefixed with a ``v`` after rewriting.

The following python code describes supported version formats best::

    version_spec = r"""[0-9]+[_.\-][0-9]+  # required major and minor
                       ([_.\-][0-9]+)?     # optional micro
                       ([_.\-][0-9]+)?     # optional extra number
                       ([._\-]*rc[0-9]+)?  # optional release candidate"""

Therefore version specifier consists of two to four numbers separated from each other and an optional release candidate number after ``rc`` which can be separated (e.g. ``2.1-rc2``).
Each separator can be an underscore, a punctuation or a dash.

If no valid tags are found, a ``v0.0`` tag is created on the oldest commit in the repository.

Finally, the package version will be set with the ``git describe`` result on such repository.
Any dashes in version will be replaced by underscores because of the conda's restrictions in setting ``package/version`` key.

Embedding script_env variables
++++++++++++++++++++++++++++++

The recipe can allow some environment variables to influence building through ``build/script_env`` key.
To make building process reproducible, ``conda-build-prepare`` embeds all such variables inside the ``conda-env`` with the values found in the current shell environment during preparation.
Such embedded variables will be later set in the shell while activating conda environment.

If during this stage there are some ``script_env`` variables not set to any value in shell, they will be removed from the ``script_env`` to never influence building this package.

