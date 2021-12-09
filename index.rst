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

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is not yet published.**

   This technote documents how sciplat-lab (the JupyterLab RSP container) is built.

.. Add content here.

Repository
==========

Layout
------

The Lab container resides at `the LSST SQuaRE Github sciplat-lab
repository <https://github.com/lsst-sqre/sciplat-lab.git>`_.

Branch Conventions
------------------

Development of the Lab container contents typically happens on a ticket
branch, which is then PRed into ``main``.  Once the resulting containers
are built and tested and we feel they are ready to be promoted to
container builds, the changes are fed into ``prod_update`` either by
rebase or cherry-picking.  (At the moment, ``main`` and ``prod_update``
should generally be the same; if we find ourselves in another situation
like we did near the end of Jupyterlab 2.x, it is possible that ``main``
will have severly diverged and cherry-picking changes will be
necessary.)

Once ``prod_update`` is ready, changes from that branch are merged into
``prod``.  Our current (Jenkins) CI process builds containers from
``prod`` rather than ``main``.

Build Process
=============

GNU Make is used to drive the build process.  The Makefile accepts three
arguments and has three useful targets.

The arguments are as follows:

#. ``tag`` -- mandatory: this is the tag on the input DM Stack container,
   e.g. ``w_2021_50``.  If it starts with a ``v`` that ``v`` becomes an
   ``r`` in the output version.
#. ``image`` -- optional: this is the name of the image you're building
   and pushing.  It defaults to ``docker.io/lsstsqre/sciplat-lab``.
#. ``supplementary`` -- optional: if specified, this turns the build into an
   experimental build where the tag starts with ``exp_`` and ends with
   ``_<supplementary>``.

The targets are one of:

#. dockerfile -- just generate the Dockerfile from the template and the
   arguments.  Do not build or push.

#. image -- build the Lab container, but do not push it.

#. push -- build and push the container.

"push" is a synonym for "all" and is the default.  Note that we assume
that the building user already has appropriate push credentials for the
repository to which the image is pushed, and that no ``docker login`` is
needed.

Dockerfile template substitution
--------------------------------
`Dockerfile.template
<https://github.com/lsst-sqre/sciplat-lab/blob/main/Dockerfile.template>`_
looks like it's ready for Jinja 2: we're substituting ``{{TAG}}``,
``{{IMAGE}}``, and ``{{VERSION}}``.  It's nothing that sophisticated.
We just run ``sed`` in the ``dockerfile`` target of the `Makefile
<https://github.com/lsst-sqre/sciplat-lab/blob/main/Makefile>`_.


Examples
--------

Build and push the weekly 2021_50 container: ``make tag=w_2021_50``.

Build and push an experimental container with a ``newnumpy``
supplementary tag: ``make tag=w_2021_50 supplementary=newnumpy``.

Just create the Dockerfile for w_2021_49: ``make dockerfile
tag=w_2021_49``.

Build the ``newnumpy`` container, but don't push it: ``make image
tag=w_2021_50 supplementary=newnumpy``.

Build and push w_2021_50 to ghcr.io: ``make tag=w_2021_50
image=ghcr.io/lsst-sqre/sciplat-lab``.


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

Most of the action in the Dockerfile comes from five shell scripts
executed by ``docker build`` as ``RUN`` actions.

These are, in order:

#. rpm -- we will always be building on top of ``centos`` in the current
   regime.  This stage first reinstalls all the system packages but with
   man pages this time (the Stack container isn't really designed for
   interactive use, but ours is), and then adds some RPM packages we
   require, or at least find helpful, for our user environment.
#. os -- this installs os-level packages that are not packaged via RPM.
   Currently the biggest and hairiest of these is TeXLive--the conda TeX
   packaging story is not good, and if we don't install TeXLive a bunch
   of the export-as options in JupyterLab will not work.
#. py -- this is probably where you're going to be spending your time.
   Mamba is faster and reports errors better than conda, so we install
   and then use it.  Anything that is packaged as a Conda package should
   be installed from conda-forge.  However, that's not everything we
   need.  Thus, the first thing we do is add all the Conda packages we
   need.  Then we do a pip install of the rest, and a little bit of
   bookkeeping to create a kernel for the Stack Python.  It is likely
   that what you need to do will be done by inserting (or pinning
   versions of) python packages in the mamba or pip sections.
#. jup -- this is for installation of Jupyter packages--mostly Lab
   extensions, but there are also server and notebook extensions we rely
   upon.  Use pre-built Lab extensions if at all possible, which will
   mean they are packaged as conda-forge or pip-installable packages and
   handled in the previous Python stage.
#. ro -- this is Rubin Observatory-specific setup.  This, notably,
   creates quite a big layer because, among other things, it checks out
   the tutorial notebooks as they existed at build time, and people keep
   checking large figure outputs into these notebooks.

Other files
-----------
The rest of the files in this directory are either things copied to
various well-known locations (for example, all the ``local*.sh`` files
end up in ``/etc/profile.d``) or they control various aspects of the Lab
startup process.  For the most part they are moved into the container by
``COPY`` statements in the Dockerfile.

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
