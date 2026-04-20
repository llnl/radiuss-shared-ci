.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _setup-with-components:

**********************************
Setup CI with Components
**********************************

This guide walks through setting up RADIUSS Shared CI using GitLab CI
Components (requires GitLab 17.0+).

.. note::
   This guide assumes you have completed :doc:`../getting_started/five-minute-setup`
   and have a single-machine pipeline running on Dane. It covers extending that
   setup with more machines, custom job templates, variables, and performance testing.

   **Migrating from legacy setup?** See :doc:`components_migration` instead.

==============
Prerequisites
==============

Before starting, ensure you have:

- GitLab 17.0 or later
- LC GitLab account with machine access
- GitHub repository mirrored to LC GitLab
- Build/test script (single command)
- GitHub personal access token

See :doc:`../getting_started/prerequisites` for detailed requirements.

========
Overview
========

Component-based setup uses GitLab CI Components consumed via `component:`
directives with typed inputs. No template file copying required.

**File structure you'll create:**

.. code-block:: text

   my-project/
   ├── .gitlab-ci.yml           # Main pipeline file
   ├── .gitlab/
   │   ├── custom-variables.yml # Optional: variables
   │   ├── custom-jobs.yml      # Optional: job templates
   │   └── jobs/
   │       ├── dane.yml         # Machine-specific jobs
   │       ├── matrix.yml
   │       └── tioga.yml
   └── scripts/
       └── build-and-test.sh    # Your build script

====================
Add More Machines
====================

Your Dane setup from the quick-start is already working. To add another
machine, repeat the same pattern — one check job and one pipeline trigger —
for each additional machine.

Matrix Example (SLURM, NVIDIA GPU)
===================================

.. code-block:: yaml

   matrix-up-check:
     extends: [.matrix, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "matrix-build-and-test"

   matrix-build-and-test:
     needs: [matrix-up-check]
     extends: [.matrix, .build-and-test]
     trigger:
       include:
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/matrix-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD
             shared_alloc: "--partition=pci --exclusive --nodes=1 --time=30"
             job_alloc: "--partition=pci --overlap --nodes=1"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/matrix.yml'

Tioga Example (Flux, AMD GPU)
==============================

.. code-block:: yaml

   tioga-up-check:
     extends: [.tioga, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "tioga-build-and-test"

   tioga-build-and-test:
     needs: [tioga-up-check]
     extends: [.tioga, .build-and-test]
     trigger:
       include:
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/tioga-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD
             shared_alloc: "--queue=pci --exclusive --nodes=1 --time-limit=30m"
             job_alloc: "--nodes=1 --begin-time=+5s"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/tioga.yml'

See :doc:`../reference/components/machine-pipelines` for all machines and allocation options.

======================
Extend Job Files
======================

Once a machine pipeline is included, add jobs by extending its template in
``.gitlab/jobs/<machine>.yml``. Multiple jobs with different build
configurations look like this:

.. code-block:: yaml

   ###########################################################################
   # Dane Jobs
   ###########################################################################

   gcc-11-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"
       VERSION: "11"

   gcc-13-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"
       VERSION: "13"

   clang-build:
     extends: .job_on_dane
     variables:
       COMPILER: "clang"
       VERSION: "14"

Your build script can read variables set by the job instance (here ``$COMPILER``, ``$VERSION``).

==============
Customization
==============

Optional Variables File
=======================

Create `.gitlab/custom-variables.yml` for cleaner organization:

.. code-block:: yaml

   variables:
     # Allocation settings
     DANE_SHARED_ALLOC: "--reservation=ci --exclusive --nodes=1 --time=30"
     DANE_JOB_ALLOC: "--reservation=ci --overlap --nodes=1"

     MATRIX_SHARED_ALLOC: "--partition=pci --exclusive --nodes=1 --time=30"
     MATRIX_JOB_ALLOC: "--partition=pci --overlap --nodes=1"

Then include it in `.gitlab-ci.yml`:

.. code-block:: yaml

   include:
     - component: .../base-pipeline@v2026.02.2
       inputs: { ... }
     - local: '.gitlab/custom-variables.yml'  # Add this

   dane-build-and-test:
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2
           inputs:
             shared_alloc: $DANE_SHARED_ALLOC  # Use variable
             job_alloc: $DANE_JOB_ALLOC

Custom Job Templates
====================

Create `.gitlab/custom-jobs.yml` for job customization:

.. code-block:: yaml

   # Applies to all build/test jobs
   .custom_job:
     before_script:
       - echo "Project-specific setup"
       - module load cmake/3.25
     artifacts:
       reports:
         junit: build/junit.xml

Include in child pipeline triggers:

.. code-block:: yaml

   dane-build-and-test:
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2
         - local: '.gitlab/custom-jobs.yml'  # Add this
         - local: '.gitlab/jobs/dane.yml'

Adding Utility Components
==========================

Skip draft PRs:

.. code-block:: yaml

   include:
     - component: .../base-pipeline@v2026.02.2
     - component: .../utility-draft-pr-filter@v2026.02.2
       inputs:
         github_token: $GITHUB_STATUS_TOKEN
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG

Skip non-PR branches:

.. code-block:: yaml

     - component: .../utility-branch-skip@v2026.02.2
       inputs:
         github_token: $GITHUB_STATUS_TOKEN
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG

See :doc:`../reference/components/utility-components` for details.

Adding Performance Testing
===========================

Add performance stage and pipeline:

.. code-block:: yaml

   stages:
     - prerequisites
     - build-and-test
     - performance-measurements  # Add this

   performance-measurements:
     extends: [.performance-measurements]
     rules:
       - if: '$CI_COMMIT_BRANCH == "main"'
         when: on_success
       - when: manual
     trigger:
       include:
         - component: .../performance-pipeline@v2026.02.2
           inputs:
             job_cmd: "./scripts/run-benchmarks.sh"
             tioga_perf_alloc: "--queue=pci --exclusive --nodes=1 --time-limit=15m"
             perf_processing_cmd: "./scripts/process-results.py"
             github_token: $GITHUB_STATUS_TOKEN
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/performances.yml'

See :doc:`../reference/components/performance-pipeline` for full details.

======================
Allocation Strategies
======================

Shared Allocations (Recommended)
=================================

One allocation for all jobs on a machine:

.. code-block:: yaml

   inputs:
     shared_alloc: "--reservation=ci --exclusive --nodes=1 --time=30"  # Shared
     job_alloc: "--reservation=ci --nodes=1"                           # Per-job

Individual Allocations
======================

Each job gets own allocation:

.. code-block:: yaml

   inputs:
     shared_alloc: "OFF"                                               # Disable
     job_alloc: "--reservation=ci --exclusive --nodes=1 --time=20"     # Per-job

.. note::
   We include CI reservation/partition/queue parameters in our allocation
   examples because we strongly recommend using those dedicated resources for
   CI.

========
Examples
========

For complete, copy-ready pipeline files covering multiple machines, custom
jobs, and performance testing, see the ``examples/`` directory in the
repository:

- ``examples/example-gitlab-ci.yml`` — multi-machine parent pipeline
- ``examples/example-custom-jobs.yml`` — custom job templates
- ``examples/example-jobs-dane.yml`` — machine-specific jobs

========
See Also
========

- :doc:`../getting_started/five-minute-setup` - Quick start
- :doc:`../reference/components/index` - Component reference
- :doc:`how_to` - Common tasks
- :doc:`components_migration` - Migrate from legacy
