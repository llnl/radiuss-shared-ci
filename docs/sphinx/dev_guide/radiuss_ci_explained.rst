.. ##
.. ## Copyright (c) 2022-2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _radiuss_ci_explained-label:

***************************
RADIUSS Shared CI Explained
***************************

This guide explains how RADIUSS Shared CI works internally - the architecture,
design patterns, and implementation details.

===================
Repository Structure
===================

The repository contains both component-based (current) and legacy (compatibility)
implementations:

.. code-block:: text

   radiuss-shared-ci/
   ├── templates/              # Component implementations (GitLab 17.0+)
   │   ├── base-pipeline/
   │   ├── dane-pipeline/
   │   ├── matrix-pipeline/
   │   ├── tioga-pipeline/
   │   ├── tuolumne-pipeline/
   │   ├── corona-pipeline/
   │   ├── performance-pipeline/
   │   ├── utility-draft-pr-filter/
   │   └── utility-branch-skip/
   ├── pipelines/              # Legacy include-based files
   ├── utilities/              # Legacy utility files
   ├── customization/          # Template files for users
   ├── examples/               # Example configurations
   └── docs/                   # Documentation source

Component-Based Implementation
===============================

Location: ``templates/*/template.yml``

Components follow GitLab CI Components specification:

.. code-block:: yaml

   # templates/dane-pipeline/template.yml
   spec:
     inputs:
       job_cmd:
         type: string
       shared_alloc:
         type: string
         default: "OFF"
   ---
   # Component implementation
   .job_on_dane:
     tags: [shell, dane]
     script:
       - ${{ inputs.job_cmd }}

Users consume components via:

.. code-block:: yaml

   include:
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@v2026.02.2
       inputs:
         job_cmd: "./script.sh"
         shared_alloc: "--nodes=1"

Legacy Implementation
=====================

Location: ``pipelines/*.yml``, ``utilities/*.yml``

Legacy files use traditional ``include: project:`` syntax:

.. code-block:: yaml

   include:
     - project: 'radiuss/radiuss-shared-ci'
       ref: 'v2026.02.2'
       file: 'pipelines/dane.yml'

Both implementations provide the same functionality. Components add input
validation and catalog integration.

===================
Pipeline Architecture
===================

Parent-Child Pattern
====================

RADIUSS Shared CI uses GitLab's parent-child pipeline pattern:

1. **Parent pipeline** (``.gitlab-ci.yml``):
   - Can be extended by users as usual
   - Checks machine availability
   - Triggers child pipelines for each machine

2. **Child pipelines** (machine-specific):
   - Execute build/test jobs
   - Report results independently to GitHub

This pattern allows:

- Independent machine status reporting to GitHub
- Parallel execution across machines
- Clean separation of concerns

Machine Abstraction
===================

Each machine has:

- Scheduler type (SLURM or Flux)
- Allocation syntax
- Runner tags
- Job templates

Machine components abstract these differences:

.. code-block:: yaml

   # Dane (SLURM)
   .job_on_dane:
     tags: [shell, dane]
     # SLURM-specific allocation logic

   # Tioga (Flux)
   .job_on_tioga:
     tags: [shell, tioga]
     # Flux-specific allocation logic

Users extend machine templates without worrying about scheduler details.

Shared Allocations
==================

Shared allocation pattern:

1. ``allocate-resources`` job requests allocation and names it.
2. Jobs find allocation ID using name and run within it.
3. ``release-resources`` job cancels allocation.

Jobs can use ``jobs-stage-1``, ``jobs-stage-2``, ``jobs-stage-3`` stages
between allocation and release.

Individual allocation alternative: each job requests its own resources.

=======================
Component Implementation
=======================

Input Specification
===================

Components define typed inputs in spec section:

.. code-block:: yaml

   spec:
     inputs:
       job_cmd:
         type: string
         description: "Command to execute for build and test"
       shared_alloc:
         type: string
         default: "OFF"
         description: "Shared allocation parameters or OFF"

GitLab validates inputs before pipeline execution. Inputs are the preferred way
to pass parameters to components instead of variables.

Template Export
===============

Our components typically export templates for user jobs:

.. code-block:: yaml

   # Component exports this
   .job_on_dane:
     stage: jobs-stage-1
     tags: [shell, slurm, dane]
     script:
       - ${{ inputs.job_cmd }}

   # User extends it
   my-job:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"

This allows users to define multiple jobs with different configurations.

Variable Passing
================

Variables flow from parent to child via component inputs:

.. code-block:: yaml

   # Parent (.gitlab-ci.yml)
   variables:
     JOB_CMD: "./script.sh"

   dane-pipeline-trigger:
     trigger:
       include:
         - component: .../dane-pipeline@version
           inputs:
             job_cmd: $JOB_CMD  # Pass parent variable

Child pipeline jobs receive variables with ``RSCI_`` prefix (v2026.02.1+).

.. warning::
   Avoid setting variables with ``RSCI_`` prefix by yourself (be it in UI or in
   .gitlab-ci.yml) as they are reserved for components intenal use and may
   cause conflicts.

=====================
GitHub Integration
=====================

Status Reporting
================

Each child pipeline reports status to GitHub using:

1. ``GITHUB_PROJECT_NAME`` and ``GITHUB_PROJECT_ORG`` identify repository
2. ``GITHUB_STATUS_TOKEN`` provides API access
3. Status context: ``gitlab-ci/<machine>`` or custom via ``CI_STATUS_CONTEXT``

Machine checks must set ``ASSOCIATED_CHILD_PIPELINE`` to use the same context
as build-and-test status reports.

This creates separate status checks per machine on GitHub pull requests.

Reproducers
===========

Jobs print reproducer instructions to help debug failures locally:

.. code-block:: bash

   ### CI job ${CI_JOB_ID} reproducer on dane (${SYS_TYPE})
   working_dir=...
   mkdir -p $working_dir
   cd $working_dir
   git clone ...
   [...]
   # Commands to recreate environment and run test

Reproducer includes clone, environement setup, allocation parameters and job
command.

===================
File Organization
===================

Repository README
=================

Location: ``README.md`` (repository root)

The README serves dual purposes:

1. **GitLab CI/CD Catalog entry:**
   - Summary of component capabilities
   - Table of contents (for multiple components)
   - ``## Components`` section with subsections per component
   - Each component: description, usage example, link to published component
   - ``## Contribute`` section
   - **Important:** Do not duplicate input documentation (inputs appear automatically on component pages)

2. **GitHub repository landing page:**
   - Overview for potential users
   - Installation instructions
   - Links to resources

GitLab uses README content when displaying the project in CI/CD Catalog.
Follow GitLab's guidelines for component project README structure.

Customization Templates
=======================

Location: ``customization/``

Template files users copy and customize:

- ``gitlab-ci.yml`` - Main CI file template
- ``subscribed-pipelines.yml`` - Machine triggers (legacy)
- ``custom-jobs-and-variables.yml`` - Customization template (legacy)
- ``jobs/*.yml`` - Per-machine job templates

These files show recommended patterns but users can modify as needed.

Examples
========

Location: ``examples/``

Complete working examples demonstrating:

- Component-based setup
- Job customization
- Machine-specific configurations
- Performance testing

Examples use current best practices and latest version.

Documentation
=============

Location: ``docs/sphinx/``

Sphinx documentation organized as:

- ``getting-started/`` - New user guides
- ``user_guide/`` - Setup and usage
- ``reference/`` - Component API reference
- ``dev_guide/`` - Developer documentation

Built with Sphinx and published to ReadTheDocs.

======================
Design Decisions
======================

Why Components?
===============

Components provide several advantages over legacy includes:

**Type Safety**
  Input validation catches configuration errors before pipeline runs

**Catalog Integration**
  Components appear in GitLab CI/CD Catalog with documentation

**Cleaner Syntax**
  Named inputs clearer than variable-based configuration

**Versioning**
  ``@version`` syntax explicit in component reference

Why Parent-Child Pipelines?
============================

Parent-child pattern chosen for:

**Independent Reporting**
  Each machine reports separately to GitHub

**Parallel Execution**
  Machines run concurrently without blocking

**Configuration and UI Scalability**
  Easy to add new machines, new jobs, and clear display of status

Why Shared Allocations?
========================

Shared allocations offer:

**Faster Job Startup**
  Jobs run immediately within existing allocation

**Resource Efficiency**
  One allocation shared across jobs, with controlled concurrency

**Ease of configuration**
  Only resource and duration of the shared allocation, not each individual job

Trade-off: All jobs must complete within allocation time limit.

Individual allocations alternative when jobs have different resource needs.

======================
Implementation Details
======================

Machine Check Pattern
=====================

Machine checks verify availability before triggering child pipeline:

.. code-block:: yaml

   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"

Check uses ``ASSOCIATED_CHILD_PIPELINE`` to link with child pipeline status
on GitHub.

If machine down, check fails and child pipeline doesn't trigger, saving resources.

Allocation Naming
=================

Shared allocations use unique names to allow multiple pipelines on same machine:

.. code-block:: bash

   ALLOC_NAME="${ALLOC_NAME:-${CI_PROJECT_NAME}_ci_${CI_PIPELINE_ID}}"

This prevents conflicts when multiple pipelines run concurrently.

Jobs find allocation by name rather than storing ID in artifacts.

Stage Organization
==================

Child pipelines have three job stages between allocation and release:

- ``jobs-stage-1`` - Default, most jobs run here
- ``jobs-stage-2`` - Jobs that depend on stage-1
- ``jobs-stage-3`` - Jobs that depend on stage-2

This allows users to define job dependencies without modifying stage list.

RSCI Variable Prefix
====================

Components set variables with ``RSCI_`` prefix (v2026.02.1+) to avoid conflicts
with user variables.

Users should not set ``RSCI_`` variables themselves.

==================
See Also
==================

- :doc:`how_to` - Step-by-step procedures for common tasks
- :doc:`contributing` - Contribution guidelines
- :doc:`troubleshooting` - Common development issues
- `GitLab CI Components <https://docs.gitlab.com/ee/ci/components/>`_
- `GitLab Child Pipelines <https://docs.gitlab.com/ee/ci/pipelines/downstream_pipelines.html>`_
