.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _quick-reference-label:

***************
Quick Reference
***************

This page provides a quick lookup for common RADIUSS Shared CI components,
variables, and configurations.

==================
Component Catalog
==================

All components are consumed with the pattern:

.. code-block:: yaml

   component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/<component-name>@<version>

Core Components
===============

.. list-table::
   :header-rows: 1
   :widths: 20 50 30

   * - Component
     - Purpose
     - Required Inputs
   * - **base-pipeline**
     - Orchestration templates, machine checks, GitHub integration
     - ``github_project_name``, ``github_project_org``
   * - **dane-pipeline**
     - Build/test jobs on Dane (SLURM scheduler)
     - ``job_cmd``, ``shared_alloc``, ``job_alloc``, ``github_project_name``, ``github_project_org``
   * - **matrix-pipeline**
     - Build/test jobs on Matrix (SLURM scheduler)
     - ``job_cmd``, ``shared_alloc``, ``job_alloc``, ``github_project_name``, ``github_project_org``
   * - **corona-pipeline**
     - Build/test jobs on Corona (Flux scheduler)
     - ``job_cmd``, ``shared_alloc``, ``job_alloc``, ``github_project_name``, ``github_project_org``
   * - **tioga-pipeline**
     - Build/test jobs on Tioga (Flux scheduler)
     - ``job_cmd``, ``shared_alloc``, ``job_alloc``, ``github_project_name``, ``github_project_org``
   * - **tuolumne-pipeline**
     - Build/test jobs on Tuolumne (Flux scheduler)
     - ``job_cmd``, ``shared_alloc``, ``job_alloc``, ``github_project_name``, ``github_project_org``
   * - **performance-pipeline**
     - Performance testing and GitHub benchmark reporting
     - ``job_cmd``, machine-specific ``*_perf_alloc``
   * - **utility-draft-pr-filter**
     - Skip CI on draft pull requests
     - ``github_token``, ``github_project_name``, ``github_project_org``
   * - **utility-branch-skip**
     - Skip CI on branches not associated with PRs
     - ``github_token``, ``github_project_name``, ``github_project_org``

==================
Required Stages
==================

When using components, some stages **must** be defined in your ``.gitlab-ci.yml``:

.. code-block:: yaml

   stages:
     - prerequisites            # Required for machine checks
     - build-and-test           # Required for machine child pipelines
     - performance-measurements # Required if using performance child pipeline

==================
Common Variables
==================

Parent Pipeline Variables
=========================

Set these in ``.gitlab-ci.yml`` or ``.gitlab/custom-variables.yml``:

.. list-table::
   :header-rows: 1
   :widths: 30 50 20

   * - Variable
     - Description
     - Example
   * - ``GITHUB_PROJECT_NAME``
     - Your GitHub project name
     - ``"my-project"``
   * - ``GITHUB_PROJECT_ORG``
     - Your GitHub organization
     - ``"LLNL"``
   * - ``GITHUB_TOKEN``
     - GitHub token (set in GitLab CI/CD settings)
     - ``$GITHUB_TOKEN``
   * - ``LLNL_SERVICE_USER``
     - Service account username (optional)
     - ``"myproject"``
   * - ``JOB_CMD``
     - Build and test command (non-expandable)
     - ``"./scripts/build.sh"``
   * - ``GIT_SUBMODULE_STRATEGY``
     - How to handle submodules
     - ``recursive`` or ``none``

.. _machine-control-variables:

Machine Control Variables
==========================

Choose the machines you want to run CI on by including the relevant machine
components. Then, you can use variables to disable machines without changing
includes. This can be convenient for temporary overrides, like skipping a
machine with known issues, or for debugging.

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Variable
     - Effect
   * - ``ON_DANE="OFF"``
     - Disable CI on Dane
   * - ``ON_CORONA="OFF"``
     - Disable CI on Corona
   * - ``ON_TIOGA="OFF"``
     - Disable CI on Tioga
   * - ``ON_TUOLUMNE="OFF"``
     - Disable CI on Tuolumne
   * - ``ON_MATRIX="OFF"``
     - Disable CI on Matrix

Allocation Variables
====================

SLURM (Dane, Matrix)
--------------------

.. code-block:: yaml

   inputs:
     shared_alloc: "--nodes=1 --exclusive --time=30"
     job_alloc: "--nodes=1"

Flux (Corona, Tioga, Tuolumne)
-------------------------------

.. code-block:: yaml

   inputs:
     shared_alloc: "--nodes=1 --exclusive --time-limit=30m"
     job_alloc: "--nodes=1 --begin-time=+5s"

Disable shared allocation:

.. code-block:: yaml

   inputs:
     shared_alloc: "OFF"

.. seealso::

   For complete input tables and exported template lists, see
   :doc:`../reference/components/index`.

.. _github-token-scopes:

===========================
GitHub Token Setup
===========================

Required scopes vary by feature. Create a dedicated token per use case:

.. list-table::
   :header-rows: 1
   :widths: 35 20 45

   * - Purpose
     - Required Scopes
     - Notes
   * - **Status Reporting** (required)
     - ``repo:status``
     - Set as ``GITHUB_STATUS_TOKEN``; used every pipeline run
   * - **Performance Reporting** (optional)
     - ``repo``, ``workflow``
     - For posting benchmark results to GitHub
   * - **GitHub Integration Setup** (one-time)
     - ``admin:repo_hook``, ``repo``
     - Used when first connecting LC GitLab to GitHub

.. warning::
   Tokens must have a lifetime of no more than 30 days per LLNL policy.

1. **Generate Token**:

   - Go to GitHub Settings → Developer settings → Personal access tokens
   - Create token with the scopes for your use case

2. **Add to GitLab**:

   - Navigate to your GitLab project → Settings → CI/CD → Variables
   - Add variable: ``GITHUB_STATUS_TOKEN`` = your token
   - Mark as "Masked" and "Protected" (recommended)

3. **Use in Pipeline**:

   .. code-block:: yaml

      include:
        - component: .../base-pipeline@v2026.02.2
          inputs:
            github_token: $GITHUB_STATUS_TOKEN

=============================
Common Job Customization
=============================

Override Custom Job Template
=============================

In ``.gitlab/custom-jobs.yml``:

.. code-block:: yaml

   .custom_job:
     before_script:
       - echo "Project-specific setup"
       - module load my-dependencies
     artifacts:
       reports:
         junit: junit.xml

Extend Machine Job Template
============================

In ``.gitlab/jobs/dane.yml``:

.. code-block:: yaml

   gcc-11-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"
       COMPILER_VERSION: "11.0.0"

   clang-14-build:
     extends: .job_on_dane
     variables:
       COMPILER: "clang"
       COMPILER_VERSION: "14.0.0"

=====================
File Structure
=====================

Typical project layout with components:

.. code-block:: text

   my-project/
   ├── .gitlab-ci.yml           # Main pipeline (includes components)
   ├── .gitlab/
   │   ├── custom-variables.yml # Variables (optional)
   │   ├── custom-jobs.yml      # Job templates (optional)
   │   └── jobs/
   │       ├── dane.yml         # Dane-specific jobs
   │       ├── matrix.yml       # Matrix-specific jobs
   │       ├── corona.yml       # Corona-specific jobs
   │       └── performances.yml # Performance jobs (optional)
   └── scripts/
       ├── build-and-test.sh    # Your build/test script
       └── run-benchmarks.sh    # Your performance script (optional)

===================
Minimal Example
===================

Simplest possible setup (single machine, one job):

``.gitlab-ci.yml``:

.. code-block:: yaml

   stages:
     - prerequisites
     - build-and-test

   variables:
     GITHUB_PROJECT_NAME: "my-project"
     GITHUB_PROJECT_ORG: "LLNL"
     JOB_CMD:
       value: "./build-and-test.sh"
       expand: false

   include:
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
       inputs:
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG
         github_token: $GITHUB_TOKEN

   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"

   dane-build-and-test:
     needs: [dane-up-check]
     extends: [.dane, .build-and-test]
     trigger:
       include:
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD
             shared_alloc: "--nodes=1 --exclusive --time=30"
             job_alloc: "--nodes=1"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/dane.yml'

``.gitlab/jobs/dane.yml``:

.. code-block:: yaml

   my-build-job:
     extends: .job_on_dane

=============
Quick Links
=============

- **Getting Started**: :doc:`../getting-started/five-minute-setup` (once created)
- **Core Concepts**: :doc:`concepts`
- **Migration Guide**: :doc:`components_migration`
- **Detailed Setup**: :doc:`setup-with-components`
- **How-To Guides**: :doc:`how_to`
- **Troubleshooting**: :doc:`../dev_guide/troubleshooting`

.. seealso::

   For complete examples, see the ``examples/`` directory in the repository.
