..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

This technote documents how sciplat-lab (the JupyterLab RSP container) is built.

.. Add content here.

Philosophy
==========

The dual-Python-environment design of the ``sciplat-lab`` container is described in `sqr-088<https://sqr-088.lsst.io>`.

Repository
==========

The assembly instructions for the Lab container reside at `the LSST SQuaRE GitHub sciplat-lab repository <https://github.com/lsst-sqre/sciplat-lab.git>`_.

Layout
------

There are several different categories of files in the repository
directory.

#. ``Dockerfile`` controls the build.

#. ``static`` holds files copied as-is into the container.

#. ``scripts`` holds scripts run inside the container at various stages of the build.

#. ``README.md`` describes the build process.

Branch Conventions
------------------

Standard Lab containers (that is, dailies, weeklies, release candidates,
and releases) are built from the ``main`` branch.  Experimental
containers may be built from any branch.  The build process enforces
this condition, and will force the tag to an experimental one when
building from any other branch than ``main``.

Build process
=============

The `build process<https://github.com/lsst-sqre/sciplat-lab/blob/main/.github/workflows/build.yaml>`_ is usually run from GitHub Actions.

It is, however, also possible to build the container locally.
The container builds on both ``amd64`` and ``arm64`` architectures for Linux.  Other architectures are not supported.


GitHub Actions input parameters for build
-----------------------------------------

#. ``tag`` -- mandatory: this is the tag from which to install the DM Stack, e.g. ``w_2026_14``.
   If it begins with a ``v`` (indicating a DM Stack release version) that ``v`` becomes an ``r`` in the output version.
#.  ``supplementary`` -- optional: this (with a prepended ``_``) is the text added to the end of the container tag (`exp_` will be automatically prepended).
   Usually this should describe the reason for the experimental container's existence, e.g. a value of ``rje`` used on a build from version tag ``d_2026_04_22`` would yield ``exp_d_2026_04_22_rje``.
#. ``image`` -- optional: this is the URI for the image you're building and pushing.
   It defaults to ``us-central1-docker.pkg.dev/rubin-shared-services-71ec/sciplat/sciplat-lab,ghcr.io/lsst-sqre/sciplat-lab,docker.io/lsstsqre/sciplat-lab``.
   As the default makes plain, it may be a comma-separated list of URIs, if you are pushing to multiple targets.
   If you are building an experimental image, you probably want to remove the ``ghcr.io`` and ``docker.io`` targets, assuming you're running at the IDF.
   If you need to run at USDF, only keep the ``ghcr.io`` target.
#. ``input`` -- optional: this is the name, and any tag prefix, of the input image you're starting with.
   It defaults to ``ghcr.io/lsst-sqre/nublado-jupyterlab-base:latest``.
   You almost certainly only want to change the tag.
   If you have built a modified base image (by opening a PR against `nublado <https://github.com/lsst-sqre/nublado`_ and changing things inside ``jupyterlab-base``), then the tag will be something like ``tickets-DM-53996``.
   The tag is case-sensitive, and dashes will replace slashes in the branch name.

Local build
-----------

Set ``ARGS`` for ``tag``, ``input``, and ``version``.
The ``tag`` and ``input`` are as described above.
To calculate ``version`` from the tag, set the environment variables ``tag``, ``image``, and ``supplementary`` (all as described above), source the `helper functions<https://github.com/lsst-sqre/sciplat-lab/blob/main/scripts/helper-functions.sh>`_, and use the output of ```calculate_tags | cut -d ',' -f 1``.

Retagging an image
==================

There is also a process for retagging an image.
We typically use this for moving the recommended tag, but it can be used to add an arbitrary tag to any image.

This too is a `Github Action<https://github.com/lsst-sqre/sciplat-lab/blob/main/.github/workflows/retag.yaml>`_.

Github Actions input Parameters For Retag
-----------------------------------------

``tag`` is the tag on the sciplat-lab input container, not the upstream DM Stack tag (for the case when the input tag represents a weekly build, they are identical, but for release versions, the DM Stack tag ``v<something>`` will have become the sciplat-lab tag ``r<something>``).

``supplementary`` is the new tag to be applied to the image.
No substitution is done.
It is mandatory in the ``retag`` case.

``image`` retains the same meaning and default: it is the target repository (or comma-separated list of repositories) to which the new tags should be pushed.

Lab build process
=================

When ``docker build`` runs, the following sequence of events takes place.

#. We copy ``/etc/passwd`` and ``/etc/group`` into place and generate their corresponding shadow files.
   This is necessary for the package installation in the next step to run, as ``dpkg`` assumes that accounts that either came with the base system or were installed by dependencies stayed installed; it uses them during installation.
#. We run ``scripts/install-system-packages``; this first updates installed system packages, as the ``jupyterlab-base`` image may be out of date, since it will date to the last release of ``Nublado``.
   Then we add system packages that we want in the RSP image, but not in the base container.
   Currently that's ``ssh``, ``quota``, and ``emacs-nox``.
#. We copy most of our static files into the container.
#. We install the DM Stack from the supplied ``tag`` using `lsstinstall<https://ls.st/lsstinstall>`_, then strip it down as much as possible.
#. We install remaining static files whose operation depends on the stack.
#. We run `install_rsp_user<https://github.com/lsst-sqre/sciplat-lab/blob/main/scripts/install-rsp-user>` to create the environment in which an RSP user works.
   This is the piece you are most likely to need to modify.
   There are several phases of installation for the RSP User environment.

   * First is ``rubin-env-rsp``.
     This is a conda metapackage, defined in the `feedstock recipe<https://github.com/conda-forge/rubinenv-feedstock/blob/main/recipe/recipe.yaml>`_.  Its version is pinned to the version of ``rubin-env`` installed with ``lsstinstall``, and we do not update dependencies.
     However, that doesn't mean that rebuilds of the same version separated by time will necessarily have all the same package versions.
     Although the DM Stack installation does lock its direct dependencies, it lets its transitive dependencies float.
     This has not been a huge operational problem thus far, but it is a lurking source of anxiety.
   * Next we add four Telescope and Site packages from their conda channel, which is *not* conda-forge.
     We also do not update dependencies for those.
     It would be very helpful if these were put onto conda-forge and then just added to ``rubin-env-rsp`` but neither SQuaRE nor T&S wants to accept responsibility for their maintenance as conda-forge packages, so we seem to be stuck with the status quo.
   * Next, we have a few tech debt steps.
     The desire to support old releases and have reasonable reproducibility of a particular release is at odds with CST's desire to add new features to old releases at will.
     We are currently (April 2026) in negotiations with Build Engineering and Pipelines to require a new point version of an old release if additional functionality is requested.
     That would eliminate these steps, as packages would just be added to a backport branch of the feedstock and a new point release then made.
   * Until we are done with the 29.x series of releases (post-DP2), we then must check to see whether ``lsdb`` has been installed already, and if not, install ``lsdb``, ``hats``, and ``cdshealpix``.
   * Then we do the same with ``reproject``.
   * If we had packages we had to ``pip`` install into the conda environment, we would do it here.
     Right now we do not and we hope to hold the line; pip-installing on top of conda leads really, really easily to dependency hell.
   * Here endeth the tech debt steps.
   * We install ``lsst-rsp`` separately.
     We have deliberately chosen to exclude it from ``rubin-env-rsp`` because we want to be able to iterate it more quickly than we update the ``recommended`` image, and it is a SQuaRE-maintained package.
   * We do conda cleanup.
   * Now we switch to the UI Python environment and perform needed modifications there.
     We deactivate the JupyterLab console (basically a fancy iPython REPL; apparently some people don't want to use a notebook for that).
     Finally we install the spellchecker, which is very strangely designed and implemented, and which is included at CST's request.
#. We install a small compatibility layer designed to ease the transition from the one-python model to the current model which separates the UI and payload Python environments.
   This can very likely go away soon.
#. We generate manifests for what's been installed.
   This turns out to be very useful for debugging regressions, because generally only a handful of packages change day-to-day, and the nature of the regression usually gives a good clue as to which package is the likely culprit.
#. Finally, we clean up files left behind by the build process.

Modifying Lab container contents
================================

This is probably why you're reading this document.
It's very likely that whatever changes you make will go into the ``install-rsp-user`` script.
The most common scenario is that you want to try out a new conda package for inclusion before you add it to ``rubin-env-rsp``.
In that case you'd add it between installation of ``rubin-env-rsp`` and ``lsst-rsp``.

Building from a different base container
========================================

It is quite possible that rather than changing anything in the payload environment, you want to modify the UI environment.

That is encapsulated in ``jupyterlab-base``, a part of `Nublado<https://github.com/lsst-sqre/nublado>`.

Installing a package from GitHub
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The most common change to the base container is that you want to install something that is not available on PyPi--either it has not been released as a Python package yet, or it resides on a branch and you're testing it before merge.

In this case, you must edit the `jupyterlab-base pyproject.toml<https://github.com/lsst-sqre/nublado/blob/main/jupyterlab-base/pyproject.toml>`_ and then update ``uv.lock``.

#. Edit pyproject.toml, and add a ``[tools.uv.sources]`` entry at the bottom.
As an example, if you wanted to install ``rsp-jupyter-extensions`` from branch ``tickets/DM-54736``, you'd add the following: ``rsp-jupyter-extensions = { git = "https://github.com/lsst-sqre/rsp-jupyter-extensions", branch = "tickets/DM-54736" }``
#. Then run ``uv lock --upgrade-package``, e.g. ``uv lock --upgrade-package rsp-jupyter-extensions``.
#. Commit the changes to your Nublado branch and push them.
#. Create a PR from the branch and mark it as draft; you're never going to merge it, but the PR means that the corresponding containers will be built.
#. As mentioned above, the container tag will replace slashes with dashes, but respect capitalization; thus in the above example, the container tag would be ``tickets-DM-54736``.

After you've done that, you should go to the `sciplat-lab actions page<https://github.com/lsst-sqre/sciplat-lab/actions>`_ and kick off a build.
Pick a supplementary tag, restrict the push targets as appropriate, and change the tag on the image field.
Test that image, which you will find down near the bottom of the dropdown list in the experimental builds section on the Hub spawner page.

Other changes
^^^^^^^^^^^^^

Other changes to the base container can be carried out as well.
The layout is very similar to sciplat-lab, except that the various static files to be copied into the container are not (yet) consolidated.

Scripts
=======

If you are working on the shell scripts (for either ``jupyterlab-base`` or ``sciplat-lab``), please use four-space indentations and convert tabs to spaces.

.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
