.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
..
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _faq:

***
FAQ
***

Frequently asked questions about RADIUSS Shared CI.

=======
General
=======

What is RADIUSS Shared CI?
===========================

A shared GitLab CI/CD framework for LLNL projects to run tests on LC machines.
Provides templates and components for multi-machine testing with minimal setup.

Who can use this?
=================

Projects that:

- Are hosted on GitHub
- Want to test on LC machines
- Can build/test with a single command
- Have access to LC GitLab instance

Both RADIUSS and non-RADIUSS projects can use this framework.

Do I need to be part of RADIUSS?
=================================

No. While developed for RADIUSS projects, any LLNL project can use it.

Non-RADIUSS projects need to:

1. Create GitHub token with ``repo:status`` scope
2. Set ``GITHUB_STATUS_TOKEN`` in GitLab CI/CD variables
3. Follow standard setup process

See :doc:`setup-with-components` for details.

==================
Components vs Legacy
==================

Should I use components or legacy approach?
===========================================

See :ref:`components-vs-legacy` in the Choosing Your Path guide for the
decision tree.

Can I migrate from legacy to components?
=========================================

Yes. See :doc:`components_migration` for a step-by-step guide.

Will legacy approach be maintained?
====================================

Both approaches are maintained in current release. Component-based approach
is recommended for new setups on GitLab 17.0+.

================
Machine Selection
================

Do I need to test on all machines?
===================================

No. Choose machines that match your hardware requirements:

- **Dane:** CPU testing (Intel Xeon)
- **Matrix:** NVIDIA GPU testing (H100)
- **Tioga:** AMD GPU testing (MI250X), development/testing
- **Tuolumne:** AMD GPU testing (MI300A), production-like environment
- **Corona:** AMD GPU testing (MI50), older hardware

Most projects test on 1-3 machines. See :doc:`../getting_started/choosing-your-path`.

Can I add more machines later?
===============================

Yes. Add machine check job and pipeline trigger to ``.gitlab-ci.yml``, create
job file in ``.gitlab/jobs/``. No changes needed to existing machines.

Can I disable a machine temporarily?
=====================================

Yes. Set the corresponding ``ON_<MACHINE>`` variable to ``"OFF"``. See
:ref:`machine-control-variables` for the full variable list.

===========
Allocations
===========

What's the difference between shared and individual allocations?
=================================================================

**Shared allocation:**

- One allocation for all jobs on a machine
- Jobs run within that allocation
- Faster job startup (no queue wait)
- More efficient resource usage
- Recommended for most projects

**Individual allocations:**

- Each job requests its own allocation
- Jobs queue independently
- More flexible for different resource needs
- Set ``shared_alloc: "OFF"``

See :doc:`setup-with-components` for examples.

Do I need CI reservations/partitions?
======================================

No, but recommended. CI-specific resources provide:

- Shorter queue times
- Dedicated capacity

Without CI resources, use regular partitions/queues but expect longer wait times.

Why do jobs wait indefinitely?
===============================

Common causes:

1. Shared allocation failed but not reported
2. Time limit too short for job sequence
3. No access to requested resources
4. Resources unavailable

Check allocation job logs, increase time limit, or disable shared allocation.

See :doc:`troubleshooting` for details.

=============
Configuration
=============

What's the difference between custom-jobs.yml and custom-variables.yml?
========================================================================

**custom-variables.yml:**

- Included in parent pipeline
- Contains only variables
- Used to organize allocation parameters and global settings
- Optional convenience file

**custom-jobs.yml:**

- Included in each child pipeline (machine-specific)
- Contains job templates (like ``.custom_job``)
- Used for project-specific job customization
- Optional customization file

See :doc:`setup-with-components` for examples.

How do variables work between parent and child pipelines?
==========================================================

Variables defined in parent pipeline are available in child pipelines when
defined in the trigger job or passed as inputs to the component trigged. 

.. code-block:: yaml

   # Parent (.gitlab-ci.yml)
   variables:
     JOB_CMD: "./script.sh" # Defined in parent only

   dane-build-and-test:
     variables:
       CI_MACHINE: "dane"  # Passed to child
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD  # Uses parent variable

.. note::
   In the legacy approach, variables defined in the parent pipeline were
   automatically inherited by child pipelines using ``trigger:forward``. With
   the components implementation, we follow GitLab's recommendation to pass
   variables as inputs to components instead. 

Can I add custom stages?
=========================

Yes. The parent pipeline remains fully expandable: define stages in
``.gitlab-ci.yml``:

.. code-block:: yaml

   stages:
     - prerequisites
     - build-and-test
     - my-custom-stage      # Custom stage
     - performance-measurements

In the build-and-test child pipelines, jobs default to ``jobs-stage-1``, but
``jobs-stage-2`` and ``jobs-stage-3`` are also available. You can override the
stage for any job to respect a specific sequence.

.. code-block:: yaml

   primary-job: # runs in jobs-stage-1 by default
     extends: .job_on_dane

   secondary-job:
     extends: .job_on_dane
     stage: jobs-stage-2  # Override default (jobs-stage-1)

This will prevent secondary-job from running until primary-job successfully
completes.

What's ASSOCIATED_CHILD_PIPELINE?
==================================

It's a variable added in v2026.02.0 linking machine check to its child
pipeline.

It allows machine check job reports status using the same context as the
associated downstream pipeline:

.. code-block:: yaml

   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"  # Links to child

Without this, the machine check status would never be overridden by build/test
results on GitHub side, causing an intial failure status to never update once
the machine is back up.

=============
Service Users
=============

Do I need a service user?
==========================

Not required, but recommended. Service users provide:

- Easier permission management
- Shared allocations across team
- Consistent environment
- No dependency on individual accounts

How do I set up a service user?
================================

1. Request service account from LC
2. Set ``LLNL_SERVICE_USER`` variable in CI configuration
3. Verify user has access to required machines

See :doc:`../getting_started/prerequisites` for details.

==================
GitHub Integration
==================

Why isn't GitHub status updating?
==================================

Check:

1. ``GITHUB_STATUS_TOKEN`` set in GitLab CI/CD variables
2. Token has ``repo:status`` scope
3. ``github_token`` input provided to components
4. ``GITHUB_PROJECT_NAME`` and ``GITHUB_PROJECT_ORG`` correct
5. Network access from GitLab runners to GitHub API

See :doc:`troubleshooting` for detailed debugging.

What permissions does the GitHub token need?
=============================================

See :ref:`github-token-scopes` in the Quick Reference for the full scope
breakdown by use case.

Do I need separate status per machine?
=======================================

Child pipelines report independently to GitHub. Each machine shows as separate
status check, allowing you to see per-machine results.

=============
Job Execution
=============

Can I run multiple commands in JOB_CMD?
========================================

``JOB_CMD`` should be single command. For multiple operations, create a script:

.. code-block:: bash

   #!/bin/bash
   # scripts/build-and-test.sh
   module load cmake
   mkdir build && cd build
   cmake ..
   make
   make test

Then:

.. code-block:: yaml

   variables:
     JOB_CMD:
       value: "./scripts/build-and-test.sh"
       expand: false

How do I pass different arguments to jobs?
===========================================

Use job variables:

.. code-block:: yaml

   # .gitlab/jobs/dane.yml
   gcc-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"
       VERSION: "11"

   clang-build:
     extends: .job_on_dane
     variables:
       COMPILER: "clang"
       VERSION: "14"

Script reads variables: ``$COMPILER``, ``$VERSION``

Can I run tests outside CI?
============================

Yes. Check job logs for reproducer instructions:

.. code-block:: text

   ===== REPRODUCER =====
   ssh dane
   # ... allocation and setup commands ...

How do I debug job failures?
=============================

1. Check GitLab job logs for error messages
2. Look for reproducer in job output
3. Test script locally on the machine
4. Verify module loads and dependencies
5. Check allocation permissions

See :doc:`troubleshooting` for common issues.

===========
Performance
===========

How do I add performance testing?
==================================

Add performance stage and pipeline:

.. code-block:: yaml

   stages:
     - prerequisites
     - build-and-test
     - performance-measurements

   performance-measurements:
     extends: [.performance-measurements]
     trigger:
       include:
         - component: .../performance-pipeline@v2026.02.2
           inputs:
             job_cmd: "./scripts/run-benchmarks.sh"
             tioga_perf_alloc: "--nodes=1 --time-limit=15m"
             # ... other inputs

See :doc:`../reference/components/performance-pipeline` for full details.

Can I run performance tests on subset of machines?
===================================================

Yes. Only specify ``<machine>_perf_alloc`` for desired machines:

.. code-block:: yaml

   inputs:
     tioga_perf_alloc: "--nodes=1 --time-limit=15m"    # Only Tioga
     # Other machines not specified = not tested

How do I report results to GitHub?
===================================

Performance pipeline includes reporting templates. Provide processing command:

.. code-block:: yaml

   inputs:
     perf_processing_cmd: "./scripts/process-results.py"
     github_token: $GITHUB_STATUS_TOKEN

Processing script should output GitHub benchmark JSON format.

See :doc:`../reference/components/performance-pipeline`.

========
Updating
========

How do I update RADIUSS Shared CI version?
===========================================

**Component-based:**

Update version tag in all component references:

.. code-block:: yaml

   - component: .../base-pipeline@v2026.02.2  # Update version

**Legacy:**

Update ``ref:`` value:

.. code-block:: yaml

   variables:
     SHARED_CI_REF: v2026.02.2  # Update once

See :doc:`how_to` for details.

Do I need to update all components to same version?
====================================================

Yes. Use consistent version across all components in your configuration to
avoid compatibility issues.

=================
Advanced Usage
=================

Can I extend the pipeline with custom stages?
==============================================

Yes. Add stages to ``.gitlab-ci.yml`` and create jobs using those stages.
Jobs can run before, after, or between standard stages.

Can I import jobs from another repository?
===========================================

Yes. Use ``include:`` with ``artifact:`` from a prerequisite job that merges
job files.

See :doc:`how_to` for shared jobs example.

Can I override shared templates?
=================================

Yes. GitLab YAML allows overriding any previously defined section. Define your
version after including shared configuration.

.. warning::
   Requires understanding of shared CI implementation. Consider opening issue
   to discuss customization needs.

How do I contribute changes?
=============================

See :doc:`../dev_guide/how_to` for developer documentation.

Process:

1. Fork repository
2. Create feature branch
3. Make changes with tests
4. Submit pull request
5. Address review feedback

========
See Also
========

- :doc:`troubleshooting` - Common issues and solutions
- :doc:`../getting_started/five-minute-setup` - Quick start
- :doc:`setup-with-components` - Setup guide
- :doc:`../reference/components/index` - Component reference
