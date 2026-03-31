.. ##
.. ## Copyright (c) 2022-2025, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _setup-legacy:

***************************
Setup CI (Legacy Approach)
***************************

.. note::
   This guide covers the legacy include-based approach for GitLab < 17.0
   or projects not yet ready to migrate to components.

   **For GitLab 17.0+**, we recommend using the component-based approach:
   :doc:`setup-with-components`

.. figure:: images/Shared-CI-Infrastructure.png
   :scale: 40 %
   :align: center

   The Shared CI Infrastructure is project agnostic.

The pre-requisite for using the RADIUSS Shared CI Infrastructure is that your
build and test process can be invoked through a single command line.

Strict separation between the build/test process and CI infrastructure helps
with maintenance: it's easier to debug a standalone script that can be run
outside CI.

.. note::
   RADIUSS CI setups typically don't split build and test phases to avoid
   complexity and artifacts. However, complex workflows are supported.
   See :ref:`complex-workflows` for details.

RADIUSS projects typically leverage Spack to manage dependencies and configure
the project. This is documented in `RADIUSS Spack Configs`_.

.. warning::
   radiuss-shared-ci is meant to live on the LC GitLab instance. The main repo
   is hosted on GitHub for accessibility and visibility and is mirrored on LC
   GitLab. To include files from radiuss-shared-ci, point to the mirror repo
   on GitLab.

========
Overview
========

By sharing the CI definition, projects share the maintenance burden.

With centralized CI configuration, we create an interface between local and
shared configurations, keeping it minimal while allowing customization.

Projects set variables and add job instances (inheriting from job templates).
Files in the ``customization`` directory allow fine tuning.

.. note::
   GitLab allows projects to include external files to configure their CI.
   Legacy RADIUSS Shared CI relies on this mechanism.

==============
File structure
==============

.. figure:: images/Shared-CI-File-Structure.png
   :scale: 30 %
   :align: center

   RADIUSS Shared CI contains files included remotely and template files
   to copy and complete.

.. _instructions:

=================
The short version
=================

Summary of integrating RADIUSS Shared CI (legacy approach):

.. code-block:: bash

   ### Prerequisites
   # write CI script

   ### CI Setup
   cd my_project/..
   git clone https://github.com/LLNL/radiuss-shared-ci.git
   cd my_project
   cp ../radiuss-shared-ci/customization/gitlab-ci.yml .gitlab-ci.yml
   mkdir -p .gitlab/jobs
   cp ../radiuss-shared-ci/customization/subscribed-pipelines.yml .gitlab
   cp ../radiuss-shared-ci/customization/custom-jobs-and-variables.yml .gitlab
   cp ../radiuss-shared-ci/customization/jobs/<machine>.yml .gitlab/jobs/<machine>.yml
   vim .gitlab/subscribed-pipelines.yml
   # comment out machines you don't want
   vim .gitlab/custom-jobs-and-variables.yml
   # set variables
   vim .gitlab/jobs/<machine>.yml
   # add jobs

   ### Mirroring Setup
   # https://docs.gitlab.com/ee/ci/ci_cd_for_external_repos/github_integration.html

   ### Non-RADIUSS projects
   # Set GITHUB_STATUS_TOKEN in GitLab CI/CD variables

Jump to :ref:`customize-ci` and :ref:`add-jobs` for details.

====================
The detailed version
====================

CI implementation parts:

* Local build-and-test script
* Shared files
* Customization files
* Jobs

Setting up CI follows these steps.

Write CI Script
===============

Prepare a CI script called via ``JOB_CMD`` in the CI.

Core CI implementation
======================

Clone RADIUSS Shared CI locally:

.. code-block:: bash

   cd my_project/..
   git clone https://github.com/LLNL/radiuss-shared-ci.git
   cd my_project

Copy the main CI file:

.. code-block:: bash

   cp ../radiuss-shared-ci/customization/gitlab-ci.yml .gitlab-ci.yml

In ``.gitlab-ci.yml``, adapt these variables:

========================================== ==========================================================================================================================
 Parameter                                  Description
========================================== ==========================================================================================================================
 ``LLNL_SERVICE_USER``                      Service User Account (optional but recommended)
 ``CUSTOM_CI_BUILD_DIR``                    CI working directories location (if not using service user)
 ``GIT_SUBMODULES_STRATEGY``                Clone strategy (``recursive`` if you have submodules)
 ``GITHUB_PROJECT_NAME``                    GitHub project name
 ``GITHUB_PROJECT_ORG``                     GitHub organization
 ``JOB_CMD``                                Build and test command
 ``ALWAYS_RUN_PATTERN``                     Branches skipping draft PR filter
========================================== ==========================================================================================================================

.. note::
   Blank variables don't require values. Variables with "..." require one.

.. warning::
   We strongly recommend using a service user account for easier permissions
   and allocation management.

Your CI now includes remote files from GitLab mirror of radiuss-shared-ci.

.. _customize-ci:

Customize the CI
================

Copy customization templates:

.. code-block:: bash

   mkdir -p .gitlab
   cp ../radiuss-shared-ci/customization/subscribed-pipelines.yml .gitlab
   cp ../radiuss-shared-ci/customization/custom-jobs-and-variables.yml .gitlab

The ``.gitlab/subscribed-pipelines.yml`` file
---------------------------------------------

Select machines by commenting out unwanted ones.

.. note::
   To add a new machine, see :ref:`add-a-new-machine`.

The ``.gitlab/custom-jobs-and-variables.yml`` file
--------------------------------------------------

Variables in this file:

========================================== ==========================================================================================================================
 Parameter                                  Description
========================================== ==========================================================================================================================
 ``ALLOC_NAME``                             Shared allocation name
 ``<MACHINE>_SHARED_ALLOC``                 Shared allocation parameters (optional, may be set to "OFF")
 ``<MACHINE>_JOB_ALLOC``                    Job allocation parameters
 ``PROJECT_<MACHINE>_VARIANTS``             Global variants for shared specs
 ``PROJECT_<MACHINE>_DEPS``                 Global dependencies for shared specs
========================================== ==========================================================================================================================

You may modify ``.custom_job`` for project-specific setup (e.g., `export jUnit test reports`_).

.. _add-jobs:

Add jobs
========

Create job files using templates:

.. code-block:: bash

   cp ../radiuss-shared-ci/customization/jobs/<machine>.yml .gitlab/jobs/<machine>.yml

Edit to add jobs extending the machine template.

.. warning::
   Use unique job names to avoid overriding shared jobs.

.. note::
   Jobs can be imported from other repositories. See :ref:`import-shared-jobs`.

Mirroring Setup
===============

GitLab provides GitHub integration for mirroring. See `GitLab documentation`_.

Non-RADIUSS Projects
====================

For projects outside RADIUSS group, create a GitHub token with ``repo:status``
permissions and register as ``GITHUB_STATUS_TOKEN`` CI/CD variable in GitLab.

Visit ``https://lc.llnl.gov/gitlab/<group>/<project>/-/settings/ci_cd``

.. _Radiuss Shared CI: https://radiuss-shared-ci.readthedocs.io/en/latest/index.html
.. _export jUnit test reports: https://github.com/LLNL/Umpire/blob/develop/.gitlab/custom-jobs-and-variables.yml
.. _sharing spack configuration files: https://github.com/LLNL/radiuss-spack-configs
.. _RADIUSS Spack Configs: https://radiuss-spack-configs.readthedocs.io/en/latest/index.html
.. _GitLab documentation: https://docs.gitlab.com/ee/ci/ci_cd_for_external_repos/github_integration.html#connect-with-personal-access-token
