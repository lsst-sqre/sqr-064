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

Repository
==========

The Lab container resides at `the LSST SQuaRE GitHub sciplat-lab
repository <https://github.com/lsst-sqre/sciplat-lab.git>`_.

Layout
------

There are several different categories of files in the repository
directory.

#. ``Makefile`` and ``Dockerfile.template`` directly control the build
   process; GNU Make is used to generate a ``Dockerfile`` from the
   template and arguments, and then ``docker build`` generates the
   ``sciplat-lab`` container.  ``bld`` provides compatibility with our
   old build system and is a wrapper for ``make``; it will be removed
   soon.

#. The ``stage`` shell files are executed during the ``docker build``
   and each control a fairly large section of the container build.
   ``texlive.profile`` is used to control the build of ``TeX`` in the
   container.

#. The other executable files, except for ``lsstlaunch.bash``, are used
   during JupyterLab startup.  The most important, and most likely to
   need modification, is ``runlab.sh``, which sets up the JupyterLab
   environment prior to launching the Lab.

#. Everything else is copied into the container during build and
   controls various runtime behaviors of the Lab.

Branch Conventions
------------------

Standard Lab containers (that is, dailies, weeklies, release candidates,
and releases) are built from the ``prod`` branch.  Experimental
containers may be built from any branch.  The build process enforces
this condition, and will force the tag to an experimental one when
building from a non-prod branch.

Note that from the GitHub perspective, ``prod`` rather than ``main`` is
the default branch.

Updating the Default Branch
---------------------------

#. Do your work in a ticket branch, as with any other repository.
#. PR that ticket branch into ``main``.  Note that the default branch to
   PR into is going to be ``prod`` and you will have to change the
   selection to ``main``.
#. Rebase (if possible) or cherry-pick the changes from ``main`` into
   ``prod_update``.  At the time of writing, there's no difference
   between ``main`` and ``prod_update``, but as we migrate between major
   versions of JupyterLab, it is possible for the two branches to
   diverge significantly (as they did in the JL2-JL3 transition).
#. Merge ``prod_update`` into ``prod``.

It is worth noting that the only place we use a PR in this process is
getting changes into ``main``.  Typically you would build an
experimental container from your branch, test that, and once satisfied,
proceed with the PR.

Once your changes are on ``main``, in the usual case where ``main`` and
``prod_update`` do not differ, the following incantation will suffice::

    git checkout main && \
    git pull && \
    git checkout prod_update && \
    git rebase main && \
    git push && \
    git checkout prod && \
    git merge prod_update && \
    git push

Build Process
=============

GNU Make is used to drive the build and tagging process.  The Makefile
has four useful targets and accepts four arguments (one or two
mandatory, depending on the target).

The targets are one of:

#. ``clean`` -- remove the generated ``Dockerfile``.  Not terribly
   useful on its own, but a good first step before running the next
   target (because the template rarely changes, ``make`` cannot tell on
   its own that the ``Dockerfile`` needs rebuilding when the arguments
   change).
#. ``dockerfile`` -- just generate the Dockerfile from the template and
   the arguments.  Do not build or push.
#. ``image`` -- build the Lab container, but do not push it.
#. ``push`` -- build and push the container.
#. ``retag`` -- attach a new tag to an already-built image.  The meaning
   of the input parameters differs slightly here, and will be treated
   separately.

``push`` is the default, and ``all`` is a synonym for it.  ``build`` is a
synonym for ``image``.  Note that we assume that the building user
already has appropriate push credentials for the repository to which the
image is pushed, and that any necessary ``docker login`` has already
been performed.

If the image is built from a branch that is not ``prod``, and the
``supplementary`` tag is not specified, the supplementary tag will be
set to a value derived from the branch name.  This prevents building
standard containers from branches other than ``prod``.

Input Parameters for Build Targets
----------------------------------

#. ``tag`` -- mandatory: this is the tag on the input DM Stack container,
   e.g. ``w_2021_50``.  If it starts with a ``v`` that ``v`` becomes an
   ``r`` in the output version.
#. ``image`` -- optional: this is the URI for the image you're building
   and pushing.  It defaults to
   ``docker.io/lsstsqre/sciplat-lab,us-central1-docker.pkg.dev/rubin-shared-services-71ec/sciplat/sciplat-lab``.
   As the default makes plain, it may be a comma-separated list of URIs,
   if you are pushing to multiple targets.
#. ``input`` -- optional: this is the name, and any tag prefix, of the
   input image you're starting with.  It defaults to
   ``docker.io/lsstsqre/centos:7-stack-lsst_distrib-``.

   Note that if there is no tag prefix, the image name should end with a
   colon, and also that if you do specify the input image, you're on
   your own: SQuaRE expects its containers to be built on top of the DM
   stack image.

   If you're just adding things to the stack image for your input
   container, you are likely to be fine, but it's entirely possible to
   introduce version incompatibilities while so doing.  It is certainly
   not going to work if you start with something that isn't based on the
   stack image.
#. ``supplementary`` -- optional: if specified, this turns the build into an
   experimental build where the tag starts with ``exp_`` and ends with
   ``_<supplementary>``.

Make Targets
------------

The targets are one of:

#. ``clean`` -- remove the generated ``Dockerfile``.  Not terribly
   useful on its own, but a good first step before running the next
   target (because the template rarely changes, ``make`` cannot tell on
   its own that the ``Dockerfile`` needs rebuilding when the arguments
   change).
#. ``dockerfile`` -- just generate the Dockerfile from the template and
   the arguments.  Do not build or push.
#. ``image`` -- build the Lab container, but do not push it.
#. ``push`` -- build and push the container.
#. ``retag`` -- tag an already-created container with a new tag and push
   that tag to the repositories specified in ``image``.  This is mainly
   intended for moving the ``recommended`` tag when consensus is
   achieved that an updated version is recommendable.
   See :ref:`make-retag`.

``push`` is the default, and ``all`` is a synonym for it.  ``build`` is a
synonym for ``image``.  Note that we assume that the building user
already has appropriate push credentials for the repository to which the
image is pushed, and that any necessary ``docker login`` has already
been performed.

If the image is built from a branch that is not ``prod``, and the
``supplementary`` tag is not specified, the supplementary tag will be
set to a value derived from the branch name.  This prevents building
standard containers from branches other than ``prod``.

.. _make-retag:

Input Parameters For "Retag" Target
-----------------------------------

The meaning and defaults for ``input``, ``tag``, and ``supplementary`` differ
slightly for the ``retag`` target.

For ``retag`` a sciplat-lab container should be ``input``, and the name
should not end in a colon.  The default is
``docker.io/lsstsqre/sciplat-lab``.  This is subject to change if and
when we move away from Docker Hub as our primary repository.

``tag`` is the tag on the sciplat-lab input container, not the upstream
DM stack tag (for the common case when the input tag is a weekly, they
are identical).

``supplementary`` is the new tag to be applied to the image.  No
substitution is done.  It is mandatory in the ``retag`` case.

``image`` retains the same meaning and default: it is the target
repository to which the new tags should be pushed.


Dockerfile template substitution
--------------------------------
`Dockerfile.template
<https://github.com/lsst-sqre/sciplat-lab/blob/main/Dockerfile.template>`_
substitutes ``{{TAG}}``, ``{{IMAGE}}``, ``{{INPUT}}`` and
``{{VERSION}}``.  Despite the fact that we use double-curly-brackets,
the substitution is nothing as sophisticated as Jinja 2: instead, we
just run ``sed`` in the ``dockerfile`` target of the
`Makefile <https://github.com/lsst-sqre/sciplat-lab/blob/main/Makefile>`_.


Examples
--------

Build and push the weekly 2021_50 container:

.. code-block:: sh

    make tag=w_2021_50

Build and push an experimental container with a ``newnumpy``
supplementary tag:

.. code-block:: sh

   make tag=w_2021_50 supplementary=newnumpy

Just create the ``Dockerfile`` for ``w_2021_49``:

.. code-block:: sh

   make dockerfile tag=w_2021_49

Build the ``newnumpy`` container, but don't push it:

.. code-block:: sh

   make image tag=w_2021_50 supplementary=newnumpy

Build and push ``w_2021_50`` to ``ghcr.io``:

.. code-block:: sh

   make tag=w_2021_50 image=ghcr.io/lsst-sqre/sciplat-lab

Build and push ``w_2021_50`` to both ``docker.io`` and ``ghcr.io``:

.. code-block:: sh

   make tag=w_2021_50 image=docker.io/lsstsqre/sciplat-lab,ghcr.io/lsst-sqre/sciplat-lab

Build and push a Telescope and Site image based on their ``sal-sciplat`` image
(note differing tag format):

.. code-block:: sh

   make tag=w_2021_49_c0023.008 input=ts-dockerhub.lsst.org/sal-sciplat: \
   image=ts-dockerhub.lsst.org/sal-sciplat-lab

Retag ``w_2022_12`` (from ghcr.io) as ``recommended`` and push to Docker
Hub and GHCR:

.. code-block:: sh

   make tag=w_2022_12 input=ghcr.io/lsst-sqre/sciplat-lab \
   image=docker.io/lsstsqre/sciplat-lab,ghcr.io/lsst-sqre/sciplat-lab \
   supplementary=recommended


Modifying Lab container Contents
================================

This is probably why you're reading this document.

You will need to understand the structure of `Dockerfile.template
<https://github.com/lsst-sqre/sciplat-lab/blob/main/Dockerfile.template>`_
a little.  It is very likely that the piece you need to modify is in one
of the ``stage*.sh`` scripts, although it is plausible that what you
want is actually one of the container setup-at-runtime pieces.

stage*.sh scripts
-----------------

Most of the action in the ``Dockerfile`` comes from five shell scripts
executed by ``docker build`` as ``RUN`` actions.

These are, in order:

#. ``stage1-rpm.sh`` -- we will always be building on top of ``centos``
   in the current regime.  This stage first reinstalls all the system
   packages but with man pages this time (the Stack container isn't
   really designed for interactive use, but ours is), and then adds some
   RPM packages we require, or at least find helpful, for our user
   environment.
#. ``stage2-os.sh`` -- this installs os-level packages that are not
   packaged via RPM.  Currently the biggest and hairiest of these is
   TeXLive--the conda TeX packaging story is not good, and if we don't
   install TeXLive a bunch of the export-as options in JupyterLab will
   not work.
#. ``stage3-py.sh`` -- this is probably where you're going to be
   spending your time.  Mamba is faster and reports errors better than
   conda, so we install and then use it.  Anything that is packaged as a
   Conda package should be installed from conda-forge.  However, that's
   not everything we need.  Thus, the first thing we do is add all the
   Conda packages we need.  Then we do a pip install of the rest, and a
   little bit of bookkeeping to create a kernel for the Stack Python.
   It is likely that what you need to do will be done by inserting (or
   pinning versions of) python packages in the mamba or pip sections.
#. ``stage4-jup.sh`` -- this is for installation of Jupyter
   packages--mostly Lab extensions, but there are also server and
   notebook extensions we rely upon.  Use pre-built Lab extensions if at
   all possible, which will mean they are packaged as conda-forge or
   pip-installable packages and handled in the previous Python stage.
#. ``stage5-ro.sh`` -- this is Rubin Observatory-specific setup.  This,
   notably, creates quite a big layer because, among other things, it
   checks out the tutorial notebooks as they existed at build time, and
   people keep checking large figure outputs into these notebooks.

Other files
-----------
The rest of the files in this directory are either things copied to
various well-known locations (for example, all the ``local*.sh`` files
end up in ``/etc/profile.d``) or they control various aspects of the Lab
startup process.  For the most part they are moved into the container by
``COPY`` statements in the ``Dockerfile``.  They do not often need
modification.

`runlab.sh
<https://github.com/lsst-sqre/sciplat-lab/blob/main/runlab.sh>`_ is the
other file you are likely to need to modify.  This is executed, as the
target user, and the last thing it does is start ``jupyterlab`` (well,
almost: it also knows if it's a dask worker or a noninteractive
container, and does something different in those cases).

Indentation conventions
-----------------------

There's a lot of shell scripting in here.  Please use four-space
indentations, and convert tabs to spaces, if you're working on the
scripts.

.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
