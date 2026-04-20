.. ## Copyright (c) 2019-2025, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

#################
RADIUSS Shared CI
#################

*In a hurry? Quickstart instructions can be found in* :doc:`our five-minute
setup guide <getting_started/five-minute-setup>`.

The RADIUSS Shared CI project provides a *templated CI infrastructure for LC
GitLab* that we intend to be sufficiently flexible and general to be adopted
and shared by any LLNL open-source projects hosted on GitHub.

The infrastructure is project-agnostic. Each machine we support has a templated
pipeline ready for use. Users only need to add jobs for machines of interest
and set some variables according to their needs.

Projects using the RADIUSS Shared CI infrastructure keep control over their
build and test process. The pre-requisite being that this process can be
invoked through a single command line.

A project is then built and tested as usual and *most of the CI infrastructure
is shared* to reduce duplicated effort and ease maintenance for individual
projects.

.. note:: LLNL's RADIUSS project (Rapid Application Development via an
   Institutional Universal Software Stack) aims to broaden usage across LLNL
   and the open source community of a set of libraries and tools used for HPC
   scientific application development.

=========================
Background and Motivation
=========================

LLNL open-source projects are prevalently hosted on GitHub. However, for those
meant to run on Livermore Computing (LC) systems, testing automatically on LC
GitLab instance is paramount.

Initially, the RADIUSS Shared CI infrastructure was designed for projects also
using Spack to describe the toolchain and build the dependencies. This "build
infrastructure" is defined in `RADIUSS Spack Configs`_ documentation.

===========================================
Cool features provided by RADIUSS Shared CI
===========================================

* A shared configuration, for reduced maintenance.
* Ability to define your jobs command and add as many jobs as desired.
* Leverage GitLab child pipelines and components to keep the pipeline readable.
* Each child pipeline reports independently to GitHub: one status per machine.
* Projects can extend their pipeline at will with additional stages.
* Print an exact reproducer in the logs.
* Provide a template for a pipeline dedicated to performance measurements.
* Optimize machine usage with shared allocation (optional).
* Filter out pipelines coming from a mirrored draft Pull Request (optional).

.. toctree::
   :maxdepth: 2
   :caption: Getting Started

   getting_started/index
   getting_started/prerequisites
   getting_started/five-minute-setup
   getting_started/choosing-your-path

.. toctree::
   :maxdepth: 3
   :caption: User Documentation

   user_guide/concepts
   user_guide/quick-reference
   user_guide/setup-with-components
   user_guide/setup-legacy
   user_guide/how_to
   user_guide/troubleshooting
   user_guide/faq

.. toctree::
   :maxdepth: 3
   :caption: Developer Documentation

   dev_guide/radiuss_ci_explained
   dev_guide/how_to
   dev_guide/contributing
   dev_guide/troubleshooting

.. toctree::
   :maxdepth: 2
   :caption: Reference

   reference/index

.. _RADIUSS Spack Configs: https://radiuss-spack-configs.readthedocs.io/en/latest/index.html
