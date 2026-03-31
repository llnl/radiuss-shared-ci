.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _machine-pipelines-reference:

*****************
Machine Pipelines
*****************

Machine pipeline components provide CI templates for running build and test
jobs on specific LC supercomputers. Each component provides scheduler-specific
job templates and allocation patterns.

========
Overview
========

RADIUSS Shared CI provides 5 machine pipeline components:

.. list-table::
   :header-rows: 1
   :widths: 15 15 20 50

   * - Component
     - Scheduler
     - Architecture
     - Best For
   * - **dane-pipeline**
     - SLURM
     - Intel Sapphire Rapids
     - **Recommended: CPU-only projects**
   * - **matrix-pipeline**
     - SLURM
     - Intel Sapphire Rapids + NVIDIA H100
     - **Recommended: CUDA/NVIDIA GPU projects**
   * - **tioga-pipeline**
     - Flux
     - AMD Trento + AMD MI250X
     - **Recommended: ROCm/AMD GPU projects**
   * - **tuolumne-pipeline**
     - Flux
     - AMD EPYC + AMD MI300A
     - **Production-like AMD environments**
   * - **corona-pipeline**
     - Flux
     - AMD Rome + AMD MI50
     - Older AMD GPU hardware

Choosing Machines
=================

**For most projects, we recommend:**

- **CPU testing**: Start with **dane-pipeline**
- **NVIDIA GPU testing**: Use **matrix-pipeline**
- **AMD GPU testing**: Use **tioga-pipeline**
- **Production AMD**: Add **tuolumne-pipeline**

**Corona** is available but less commonly used for new projects (older hardware).

=================
Common Structure
=================

All machine components share a similar structure with scheduler-specific
differences.

Common Inputs
=============

All machine pipelines require:

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Input
     - Required
     - Description
   * - ``job_cmd``
     - Yes
     - Command to build and test your project
   * - ``github_project_name``
     - Yes
     - GitHub repository name
   * - ``github_project_org``
     - Yes
     - GitHub organization name
   * - ``llnl_service_user``
     - No
     - LLNL service account username (recommended)

Scheduler-Specific Inputs
==========================

**SLURM machines (Dane, Matrix)**:

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Input
     - Required
     - Description
   * - ``shared_alloc``
     - Yes
     - Shared allocation args (``salloc``) or "OFF"
   * - ``job_alloc``
     - Yes
     - Per-job allocation args (``srun``)
   * - ``alloc_name``
     - No
     - Shared allocation name (default: "ci_<machine>")

**Flux machines (Corona, Tioga, Tuolumne)**:

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Input
     - Required
     - Description
   * - ``shared_alloc``
     - Yes
     - Shared allocation args (``flux alloc``) or "OFF"
   * - ``job_alloc``
     - Yes
     - Per-job allocation args (``flux run``)
   * - ``alloc_name``
     - No
     - Shared allocation name (default: "ci_<machine>")

Common Exported Templates
==========================

Each machine component exports:

- ``.job_on_<machine>`` - Main job template (extend this for your jobs)
- ``.on_<machine>`` - Machine-specific rules
- ``.<machine>_reproducer_init`` - Reproducer initialization
- ``.<machine>_reproducer_vars`` - Reproducer variables (override if needed)
- ``.<machine>_reproducer_job`` - Reproducer job command
- ``.custom_jobs`` - Customization template hook

==================
dane-pipeline
==================

**Recommended for: CPU-only testing**

SLURM-based machine with Intel Sapphire Rapids CPUs.

Usage
=====

.. code-block:: yaml

   # In .gitlab-ci.yml (parent pipeline)
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

   # In .gitlab/jobs/dane.yml (child pipeline)
   gcc-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"

Allocation Examples
===================

.. code-block:: yaml

   # Conservative: 30-minute shared allocation
   shared_alloc: "--nodes=1 --exclusive --time=30"
   job_alloc: "--nodes=1"

   # With reservation (recommended for CI)
   shared_alloc: "--nodes=1 --exclusive --reservation=ci --time=30"
   job_alloc: "--nodes=1 --reservation=ci"

   # Disable shared allocation (each job gets its own allocation)
   shared_alloc: "OFF"
   job_alloc: "--nodes=1 --exclusive --time=20"

===================
matrix-pipeline
===================

**Recommended for: CUDA/NVIDIA GPU testing**

SLURM-based machine with Intel Sapphire Rapids CPUs and NVIDIA H100 GPUs.

Usage
=====

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
             shared_alloc: "--nodes=1 --exclusive --time=30"
             job_alloc: "--nodes=1"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/matrix.yml'

   # In .gitlab/jobs/matrix.yml
   cuda-build:
     extends: .job_on_matrix
     variables:
       COMPILER: "gcc"
       CUDA_VERSION: "12.0"

Allocation Examples
===================

.. code-block:: yaml

   # GPU allocation
   shared_alloc: "--nodes=1 --exclusive --time=30 --gres=gpu:1"
   job_alloc: "--nodes=1 --gres=gpu:1"

==================
tioga-pipeline
==================

**Recommended for: ROCm/AMD GPU testing**

Flux-based machine with AMD Trento CPUs and AMD MI250X GPUs.

Usage
=====

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
             shared_alloc: "--nodes=1 --exclusive --time-limit=30m"
             job_alloc: "--nodes=1 --begin-time=+5s"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/tioga.yml'

   # In .gitlab/jobs/tioga.yml
   rocm-build:
     extends: .job_on_tioga
     variables:
       COMPILER: "llvm-amdgpu"
       ROCM_VERSION: "6.4.3"

Allocation Examples
===================

.. code-block:: yaml

   # Standard Flux allocation
   shared_alloc: "--nodes=1 --exclusive --time-limit=30m"
   job_alloc: "--nodes=1 --begin-time=+5s"

   # With GPU specification
   shared_alloc: "--nodes=1 --exclusive --time-limit=30m -g 1"
   job_alloc: "--nodes=1 --begin-time=+5s -g 1"

=====================
tuolumne-pipeline
=====================

**Recommended for: Production-like AMD environments**

Flux-based machine with AMD EPYC CPUs and AMD MI300A APUs.

Usage
=====

.. code-block:: yaml

   tuolumne-up-check:
     extends: [.tuolumne, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "tuolumne-build-and-test"

   tuolumne-build-and-test:
     needs: [tuolumne-up-check]
     extends: [.tuolumne, .build-and-test]
     trigger:
       include:
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/tuolumne-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD
             shared_alloc: "--nodes=1 --exclusive --time-limit=30m"
             job_alloc: "--nodes=1 --begin-time=+5s"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/tuolumne.yml'

Allocation Examples
===================

.. code-block:: yaml

   # APU allocation
   shared_alloc: "--nodes=1 --exclusive --time-limit=30m"
   job_alloc: "--nodes=1 --begin-time=+5s"

==================
corona-pipeline
==================

**Available for: Older AMD GPU hardware (less common)**

Flux-based machine with AMD Rome CPUs and AMD MI50 GPUs.

.. note::
   Corona uses older AMD GPU hardware. For most AMD GPU projects, **tioga-pipeline**
   or **tuolumne-pipeline** are recommended.

Usage
=====

.. code-block:: yaml

   corona-up-check:
     extends: [.corona, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "corona-build-and-test"

   corona-build-and-test:
     needs: [corona-up-check]
     extends: [.corona, .build-and-test]
     trigger:
       include:
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/corona-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD
             shared_alloc: "--nodes=1 --exclusive --time-limit=30m"
             job_alloc: "--nodes=1 --begin-time=+5s"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/corona.yml'

================
Job Customization
================

Extending Job Templates
========================

All machine components export ``.job_on_<machine>`` templates:

.. code-block:: yaml

   # .gitlab/jobs/dane.yml
   gcc-11-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"
       VERSION: "11.0.0"

   gcc-13-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"
       VERSION: "13.0.0"

   clang-build:
     extends: .job_on_dane
     variables:
       COMPILER: "clang"
       VERSION: "14.0.0"

Custom Job Template
===================

Override ``.custom_jobs`` in ``.gitlab/custom-jobs.yml``:

.. code-block:: yaml

   .custom_jobs:
     before_script:
       - echo "Machine: ${CI_MACHINE}"
       - module load cmake/3.25
     artifacts:
       paths:
         - build/logs/

This applies to all jobs on all machines.

====================
Reproducer Variables
====================

Each component provides reproducer commands in job logs. Customize what
variables appear:

.. code-block:: yaml

   # .gitlab/custom-jobs.yml
   .dane_reproducer_vars:
     script:
       - |
         echo "export COMPILER=\"${COMPILER}\""
         echo "export VERSION=\"${VERSION}\""

This adds your custom variables to the reproducer output for easier local
reproduction.

==============
Common Patterns
==============

Multi-Machine Setup
===================

Typical setup with CPU + GPU testing:

.. code-block:: yaml

   # .gitlab-ci.yml
   include:
     - component: .../base-pipeline@v2026.02.2

   # CPU testing on Dane
   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"

   dane-build-and-test:
     needs: [dane-up-check]
     extends: [.dane, .build-and-test]
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2
         - local: '.gitlab/jobs/dane.yml'

   # GPU testing on Matrix
   matrix-up-check:
     extends: [.matrix, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "matrix-build-and-test"

   matrix-build-and-test:
     needs: [matrix-up-check]
     extends: [.matrix, .build-and-test]
     trigger:
       include:
         - component: .../matrix-pipeline@v2026.02.2
         - local: '.gitlab/jobs/matrix.yml'

Conditional Machine Activation
===============================

Disable machines without removing configuration:

.. code-block:: yaml

   variables:
     ON_CORONA: "OFF"  # Temporarily disable Corona

Useful during outages or when debugging specific machines.

==============
Common Issues
==============

Allocation Timeout
==================

**Symptom**: Jobs fail with "allocation timeout" or "resource unavailable"

**Solutions**:

1. Increase allocation time if too short
2. Use appropriate queue/reservation (e.g., ``--reservation=ci``)
3. Check if machine has available nodes

.. note::

   Remember that using a shared allocation is recommended but not required. If
   you want to disable the shared allocation and have each job get its own
   allocation, set ``shared_alloc: "OFF"``.

Job Template Not Found
======================

**Error**: "extends template .job_on_dane that doesn't exist"

**Solution**: Ensure machine component is included in child pipeline trigger:

.. code-block:: yaml

   dane-build-and-test:
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2  # Must be here!
         - local: '.gitlab/jobs/dane.yml'

Permission Denied
=================

**Symptom**: "permission denied" when allocating or running jobs

**Solutions**:

1. Check service user has access to target machine
2. Verify allocation parameters (queue, reservation, etc.)
3. Check disk quotas (use service user to avoid personal quota issues)

========
See Also
========

- :doc:`base-pipeline` - Required orchestration component
- :doc:`../getting-started/choosing-your-path` - Choosing machines guide
- :doc:`../user_guide/quick-reference` - Quick allocation examples
- :doc:`../user_guide/concepts` - Machine abstraction explanation

**Related Topics**:

- Shared allocations vs individual allocations
- Service user setup
- Reproducer usage for local debugging
