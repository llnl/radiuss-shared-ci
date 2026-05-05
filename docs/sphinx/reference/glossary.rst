.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _glossary:

********
Glossary
********

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
      A personal access token with adequate permissions, e.g. ``repo:status``
      to post CI status updates to GitHub.

   Job Command (JOB_CMD)
      The single command that builds and tests your project, provided by you
      and executed by RADIUSS Shared CI in the appropriate environment.
