.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _concepts-label:

*************
Core Concepts
*************

This section explains the fundamental concepts behind RADIUSS Shared CI,
helping you understand how the system works before diving into setup and
configuration.

=============================
What is RADIUSS Shared CI?
=============================

RADIUSS Shared CI is a **templated CI infrastructure for GitLab** designed
specifically for LLNL open-source projects hosted on GitHub that need to run
tests on Livermore Computing (LC) systems.

Rather than each project maintaining its own complex CI configuration, RADIUSS
Shared CI provides a **shared, reusable framework** that projects can consume
and customize. This approach:

- **Reduces maintenance burden** - Bug fixes and improvements benefit all users
- **Ensures consistency** - Common patterns across RADIUSS projects
- **Simplifies onboarding** - New projects can adopt proven CI practices quickly
- **Enables specialization** - Projects focus on their build/test scripts, not CI internals

The infrastructure is **project-agnostic**: you provide a command to build and
test your project, and RADIUSS Shared CI helps you with orchestration, machine
scheduling, GitHub integration, and reporting.

============================
Component Architecture Model
============================

Starting with v2025.12.0, RADIUSS Shared CI uses **GitLab CI Components** as
its primary architecture (requires GitLab 17.0+).

What are Components?
====================

GitLab CI Components are reusable, versioned, parameterized CI templates that
can be consumed from a catalog. Think of them as "functions" for CI
configuration:

- **Inputs**: Type-checked parameters you provide
- **Outputs**: Job templates and variables your pipeline can use
- **Versioning**: Pin to specific versions (e.g., ``@v2026.02.2``)
- **Discovery**: Browse available components in GitLab's CI/CD Catalog

Components vs Include-Based Approach
=====================================

The legacy approach used GitLab's ``include: project:`` with file paths.
Components provide several advantages:

.. list-table::
   :header-rows: 1
   :widths: 20 40 40

   * - Feature
     - Legacy (Include-Based)
     - Components (Modern)
   * - **Versioning**
     - ``ref: 'v2026.02.2'`` in each include
     - ``@v2026.02.2`` in component path
   * - **Type Safety**
     - No validation, runtime errors
     - Input validation at parse time
   * - **Documentation**
     - Scattered in files/docs
     - Self-documenting in catalog
   * - **Reusability**
     - Some template files had to be copied
     - Direct component consumption
   * - **Discoverability**
     - Search GitHub/docs
     - Browse GitLab catalog UI

Parent vs Child Pipelines
==========================

RADIUSS Shared CI uses GitLab's **parent-child pipeline** pattern to organize
work efficiently:

.. code-block:: text

   Parent Pipeline (.gitlab-ci.yml)
   ├── Stage: prerequisites
   │   ├── dane-up-check        (Machine availability check)
   │   ├── matrix-up-check
   │   └── tioga-up-check
   │
   └── Stage: build-and-test
       ├── Child: dane-build-and-test
       │   ├── Job: gcc-build
       │   ├── Job: clang-build
       │   └── Job: intel-build
       │
       ├── Child: matrix-build-and-test
       │   └── Job: clang-cuda-build
       │
       └── Child: tioga-build-and-test
           └── Job: rocm-build

**Parent pipeline** (defined in ``.gitlab-ci.yml``):
  - Orchestrates the overall workflow
  - Checks machine availability
  - Triggers child pipelines
  - Reports to GitHub

**Child pipelines** (triggered by parent):
  - Run machine-specific jobs
  - Execute within shared allocations (SLURM/Flux)
  - Each reports independently to GitHub
  - Inherit variables from parent

This separation provides:

- **Clarity**: Each machine's jobs grouped together
- **Scalability**: Add machines without cluttering main config
- **Parallelism**: Child pipelines run concurrently
- **GitHub integration**: One status per machine

Machine Abstraction Layer
==========================

RADIUSS Shared CI provides components for different LC machines, abstracting
their scheduler differences to some extent:

**LSF Spectrum (Lassen)**:
  Uses ``bsub`` and ``jsrun`` for job submission.

**SLURM (Dane, Matrix)**:
  Uses ``salloc`` for shared allocations and ``srun`` for jobs.

**Flux (Corona, Tioga, Tuolumne)**:
  Uses ``flux alloc`` for shared allocations and ``flux run`` for jobs.

Each machine component handles:

- Scheduler-specific allocation commands
- Job execution within allocations
- Resource management
- Reproducer generation for local testing

As a user, you primarily interact through **allocation parameters** (e.g.,
``shared_alloc``, ``job_alloc``) which may still be scheduler specific.
The component handles scheduler details, e.g. how to share allocations.

Template Extension Pattern
===========================

RADIUSS Shared CI components export **job templates** that you extend to
define your specific jobs:

.. code-block:: yaml

   # Component exports .job_on_dane template
   # You extend it with your custom variables

   gcc-build:
     extends: .job_on_dane
     variables:
       SPEC: "gcc@11.0.0"

   clang-build:
     extends: .job_on_dane
     variables:
       SPEC: "clang@14.0.0"

Each template includes:

- **Scheduler commands**: Allocation and job execution
- **Reproducer logic**: Generates commands for local testing
- **Machine tags**: Ensures jobs run on correct runners
- **Customization hooks**: ``.custom_job`` for project-specific setup

This pattern enables:

- **DRY principle**: Define once, reuse many times
- **Consistency**: All jobs follow same structure
- **Flexibility**: Override only what you need

=============
Key Workflows
=============

GitHub to GitLab Integration
=============================

RADIUSS Shared CI bridges GitHub (where code lives) and GitLab (where CI runs):

1. **Mirroring**: GitHub repository mirrored to LC GitLab (a GitLab feature)
2. **Trigger**: Push/PR triggers GitLab pipeline (after mirroring delay)
3. **Execution**: Pipeline runs on LC machines
4. **Reporting**: Status posted back to GitHub PR/commit (RADIUSS Shared CI
   adds custom statuses)

Machine Availability Checks
============================

Before running expensive builds, RADIUSS Shared CI checks if machines are
available using the ``lorenz`` status system:

.. code-block:: yaml

   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"

If a machine is down:

- Check job reports "unavailable" to GitHub (Users see clear status on GitHub).
- Child pipeline is skipped (No wasted time on failed allocations).

Shared Allocations
==================

For machines using SLURM/Flux, RADIUSS Shared CI can create a **shared
allocation** for all jobs on that machine:

.. code-block:: text

   Shared Allocation (30 minutes, 1 node)
   ├── Job 1: gcc-build (5 minutes)
   ├── Job 2: clang-build (5 minutes)
   ├── Job 3: intel-build (5 minutes)
   └── Job 4: gcc-debug (5 minutes)

Benefits:

- **Faster**: No allocation wait time between jobs
- **Efficient**: Better resource utilization
- **Simpler**: One allocation to manage

Set ``shared_alloc: "OFF"`` to disable for machines where this doesn't make sense.

================
Key Terminology
================

.. glossary::

   Component
      A reusable, versioned GitLab CI template with typed inputs and documented
      outputs. Consumed via ``component: path@version`` syntax.

   Parent Pipeline
      The main pipeline defined in ``.gitlab-ci.yml`` that orchestrates machine
      checks and triggers child pipelines.

   Child Pipeline
      A dynamically-triggered pipeline for a specific machine that contains the
      actual build and test jobs.

   Machine Component
      A component providing CI templates for a specific LC machine (e.g.,
      ``matrix-pipeline``, ``dane-pipeline``).

   Base Pipeline
      The ``base-pipeline`` component that provides orchestration templates and
      utilities used by all pipelines.

   Job Template
      A YAML template (e.g., ``.job_on_dane``) that jobs extend to inherit
      common configuration.

   Allocation
      A reservation of compute resources on an HPC machine, managed by a
      scheduler (LSF, SLURM, or Flux).

   Shared Allocation
      A single allocation within which multiple jobs run sequentially or in parallel,
      avoiding repeated allocation wait times.

   RSCI_ Variables
      Internal environment variables prefixed with ``RSCI_`` set by components
      to avoid collisions with GitLab globals.

   Service User
      An LLNL service account used to run CI, providing consistent permissions
      and avoiding individual user quota issues.

   Reproducer
      Shell commands printed in job logs that allow you to reproduce the exact
      CI environment and command locally for debugging.

   Machine Check
      A preliminary job that queries machine availability before triggering
      expensive child pipelines.

   GitHub Token
      A personal access token with adequate premissions, e.g. ``repo:status``
      to post CI status updates to GitHub.

   Job Command (JOB_CMD)
      The single command that builds and tests your project, provided by you
      and executed by RADIUSS Shared CI in the appropriate environment.

====================
Architecture Diagram
====================

.. code-block:: text

   ┌──────────────────────────────────────────────────────────────┐
   │ GitHub Repository (LLNL/my-project)                          │
   │  - Source code                                               │
   │  - Pull requests                                             │
   │  - Commit statuses ←────────────────────┐                    │
   └────────────┬────────────────────────────┼────────────────────┘
                │ (mirrored)                 │ (status reports)
                ↓                            │
   ┌───────────────────────────────────────┐ │
   │ LC GitLab (lc.llnl.gov/gitlab)        │ │
   │                                       │ │
   │  .gitlab-ci.yml (Parent Pipeline)     │ │
   │  ├─ Include: base-pipeline component  │ │
   │  ├─ Include: utility components       │ │
   │  └─ Machine pipeline triggers         │ │
   │                                       │ │
   │  ┌─────────────────────────────────┐  │ │
   │  │ dane-up-check ──→ Available?    │──┼─┤
   │  └─────────────────────────────────┘  │ │
   │           ↓ (if available)            │ │
   │  ┌─────────────────────────────────┐  │ │
   │  │ Child: dane-build-and-test      │  │ │
   │  │  ├─ Component: dane-pipeline    │  │ │
   │  │  ├─ Local: custom-jobs.yml      │  │ │
   │  │  └─ Local: jobs/dane.yml        │  │ │
   │  │                                 │  │ │
   │  │     Jobs run on Dane:           │  │ │
   │  │     ├─ gcc-build ──────────────────┼─┤
   │  │     ├─ clang-build ────────────────┼─┤
   │  │     └─ intel-build ────────────────┼─┤
   │  └─────────────────────────────────┘  │ │
   │                                       │ │
   │  (Similar for lassen, corona, etc.)   │ │
   └───────────────────────────────────────┘ │
                                             │
   ┌─────────────────────────────────────────┘
   │
   │  Status Update: "dane-build-and-test: ✓ Success"
   └──→ GitHub PR shows check results

===================
Next Steps
===================

Now that you understand the core concepts, you can:

- **New users**: Proceed to :doc:`../getting-started/five-minute-setup` (once created)
- **Migrating users**: See :doc:`components_migration`
- **Learn more**: Explore :doc:`setup_ci` for detailed setup instructions

.. seealso::

   - :doc:`components_migration` - Migrate from legacy include-based setup
   - :doc:`how_to` - Common tasks and recipes
   - :ref:`radiuss_ci_explained-label` - Developer perspective on architecture
