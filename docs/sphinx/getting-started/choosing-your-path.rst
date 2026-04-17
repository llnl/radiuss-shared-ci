.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _choosing-your-path-label:

*******************
Choosing Your Path
*******************

After completing the :doc:`five-minute-setup`, this guide helps you decide
which machines, components, and features to enable for your project.

========================
Quick Decision Tree
========================

.. code-block:: text

   ┌─ Using GitLab < 17.0?
   │  └→ Use legacy include-based approach
   │     See: user_guide/setup-legacy
   │
   ├─ Using GitLab 17.0+?
   │  ├─ New project?
   │  │  └→ Use components (you're on the right path!)
   │  │
   │  └─ Existing project with legacy setup?
   │     └→ Migrate to components
   │        See: user_guide/components_migration
   │
   └─ What do you need?
      ├─ Build and test only → Machine components
      ├─ Performance tracking → + performance-pipeline
      ├─ Skip draft PRs → + utility-draft-pr-filter
      └─ Skip non-PR branches → + utility-branch-skip

=========================
Choosing Machines
=========================

RADIUSS Shared CI supports multiple LC machines. Choose based on your needs:

Available Machines
==================

.. list-table::
   :header-rows: 1
   :widths: 15 15 20 50

   * - Machine
     - Scheduler
     - Architecture
     - Best For
   * - **Dane**
     - SLURM
     - AMD EPYC + NVIDIA A100 GPUs
     - CUDA development, modern GPU testing
   * - **Matrix**
     - SLURM
     - AMD EPYC + AMD MI300A APUs
     - ROCm development, APU testing
   * - **Corona**
     - Flux
     - AMD EPYC + AMD MI250X GPUs
     - ROCm development, AMD GPU testing
   * - **Tioga**
     - Flux
     - AMD EPYC + AMD MI250X GPUs
     - ROCm development, production-like environment
   * - **Tuolumne**
     - Flux
     - AMD EPYC + AMD MI300A APUs
     - ROCm development, latest AMD hardware

Decision Criteria
=================

**Q: Which machine should I start with?**

Start with **one machine** where your team already has:

- Working build process
- Allocation or queue access
- Familiarity with the environment

Common starting points:

- **CUDA projects**: Start with Matrix
- **ROCm projects**: Start with Tioga or Tuolumne
- **CPU-only**: Any machine works, Dane is a good default

**Q: How many machines should I enable?**

Start small, expand gradually:

1. **One machine** - Get CI working reliably
2. **Two machines** - Add a second architecture (e.g., CUDA + ROCm)
3. **Multiple machines** - Full coverage once CI is stable

**Q: Should I enable all machines?**

No! Only enable machines where:

- You have active development
- You have allocation/access
- The architecture matters for your project

More machines = more CI time and resources.

Adding Machines
===============

To add a machine, add its check and trigger jobs to ``.gitlab-ci.yml``:

.. code-block:: yaml

   # Add to existing stages
   stages:
     - prerequisites
     - build-and-test

   # Add machine check
   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"

   # Add machine pipeline
   dane-build-and-test:
     needs: [dane-up-check]
     extends: [.dane, .build-and-test]
     trigger:
       include:
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD
             shared_alloc: "--reservation=ci --exclusive --nodes=1 --time=30"
             job_alloc: "--reservation=ci --overlap --nodes=1"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/dane.yml'

Then create ``.gitlab/jobs/dane.yml`` with your jobs.

See: :doc:`../user_guide/quick-reference` for all machine configurations.

Temporarily Disabling Machines
===============================

To temporarily disable a machine without removing its configuration:

.. code-block:: yaml

   variables:
     ON_DANE: "OFF"  # Disables Dane CI

This is useful during:

- Machine outages
- Debugging issues on specific machines
- Testing changes on subset of machines

=============================
Choosing Utility Components
=============================

Utility components add optional behavior to your pipeline.

utility-draft-pr-filter
=======================

**Purpose**: Skip CI on draft pull requests to save resources.

**When to use**:

- Your team uses GitHub draft PRs
- You want to avoid running CI until PR is ready for review
- You want to save CI time and machine resources

**How it works**:

1. Checks if current commit is from a draft PR
2. If yes, reports "Draft PR - CI skipped" to GitHub and exits
3. If no, continues with normal CI

**Adding it**:

.. code-block:: yaml

   include:
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/utility-draft-pr-filter@v2026.02.2
       inputs:
         github_token: $GITHUB_TOKEN
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG
         always_run_pattern: "^(main|develop)$"  # Optional: branches that always run

**Configuration**:

- ``always_run_pattern``: Regex for branches that skip the filter (e.g.,
  protected branches should always run even if from a draft PR)

utility-branch-skip
===================

**Purpose**: Skip CI on branches that aren't associated with a pull request.

**When to use**:

- You only want CI on PRs, not on random branch pushes
- You want to reduce noise from experimental branches
- You have many feature branches

**How it works**:

1. Checks if current branch has an open PR
2. If no PR, reports "Not a PR - CI skipped" and exits
3. If PR exists, continues with normal CI

**Adding it**:

.. code-block:: yaml

   include:
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/utility-branch-skip@v2026.02.2
       inputs:
         github_token: $GITHUB_TOKEN
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG

.. warning::
   This will skip CI on your main/develop branches unless they have PRs.
   Consider your workflow before enabling.

.. note::
   Both draft PRs filter and branch skip utilities can be used together depending on your needs.

============================
Performance Pipeline
============================

**Purpose**: Run performance benchmarks and report results to GitHub.

**When to use**:

- You have performance-critical code
- You want to track performance over time
- You want to detect performance regressions in PRs

**Requirements**:

- Performance test script that produces results
- (Optional) Processing script to format results
- GitHub token with ``repo`` and ``workflow`` permissions for reporting

Decision Questions
==================

**Q: Should I enable performance testing?**

Consider enabling if:

- Performance is critical to your project
- You have dedicated performance benchmarks
- You can afford the additional CI time

Skip if:

- You're just getting started with CI
- You don't have performance tests yet
- CI time is already too long

.. note::
   To reduce the burden of running performance tests, see "How often should
   performance tests run?" below for scheduling options.

**Q: Which machines should run performance tests?**

Typically, run performance tests on:

- **Fewer machines** than regular CI (1-2 representative machines)
- **Consistent hardware** (same machine for trend tracking)
- **Production-like environments** (e.g. Tuolumne for ROCm)

**Q: How often should performance tests run?**

Common patterns:

- **On main/develop only**: Use rules to restrict to protected branches
- **Scheduled**: Use GitLab schedules for nightly performance runs
- **Manual on PRs**: Set ``when: manual`` for PR performance testing

Adding Performance Pipeline
============================

Basic setup:

.. code-block:: yaml

   stages:
     - prerequisites
     - build-and-test
     - performance-measurements  # Add this stage

   performance-measurements:
     extends: [.performance-measurements]
     rules:
       # Only on main/develop, or manual on PRs
       - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"'
         when: on_success
       - when: manual
     trigger:
       include:
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/performance-pipeline@v2026.02.2
           inputs:
             job_cmd: "./scripts/run-benchmarks.sh"
             dane_perf_alloc: "--reservation=ci --nodes=1 --exclusive --time=30"
             perf_processing_cmd: "./scripts/process-results.py"
             github_token: $GITHUB_STATUS_TOKEN
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/performances.yml'

Then create ``.gitlab/jobs/performances.yml`` with performance jobs.

See: :doc:`../user_guide/quick-reference` for full performance pipeline reference.

=====================================
Service User vs Personal Account
=====================================

Already covered in :doc:`prerequisites`, but worth revisiting:

**Use Service User If**:

- ✓ Multiple team members need to manage CI
- ✓ You want consistent permissions across runs
- ✓ Disk quota is a concern
- ✓ Project is long-term / production

**Use Personal Account If**:

- ✓ Solo developer / small project
- ✓ Quick prototyping / temporary project
- ✓ Can't get service account approved
- ✓ Okay with quota limitations

.. note::
   You can always migrate from personal to service account later by changing
   the ``LLNL_SERVICE_USER`` variable.

=====================
Allocation Strategy
=====================

Understanding Shared Allocations
=================================

For SLURM/Flux machines, you can use:

**Shared Allocation** (recommended):

- One allocation for all jobs on that machine
- Jobs run sequentially or in parallel within allocation
- Faster (no wait between jobs)
- More efficient (better utilization)

**Individual Allocations** (``shared_alloc: "OFF"``):

- Each job gets its own allocation
- Slower (wait time between jobs)
- Simpler (no shared state)

.. note::
   We recommend shared allocations for regular CI jobs, and inidividual
   allocations for performance jobs to avoid interference.

Choosing Allocation Size
=========================

Consider:

**Time**:

- Start conservative (30 minutes)
- Monitor actual job duration
- Add buffer for variability
- Max out at queue limits

**Resources**:

- Match your typical build (1 node usually sufficient)
- Don't over-allocate (wastes resources)
- Can override per-job if needed

Example configurations:

.. note::
   For flexibility, we advise not to set a time limit for jobs running under a
   shared allocation: the top level allocaton suffices.

==============
Next Steps
==============

After deciding your configuration:

**Implement Your Choices**:

- Add chosen machines to ``.gitlab-ci.yml``
- Create job files for each machine
- Add utility components if desired
- Configure performance pipeline if needed

**Learn More**:

- **Detailed setup**: :doc:`../user_guide/setup-with-components`
- **How-to guides**: :doc:`../user_guide/how_to`
- **Quick reference**: :doc:`../user_guide/quick-reference`
- **Advanced patterns**: :doc:`../cookbook/index` (once created)

**Get Help**:

- **Troubleshooting**: :doc:`../dev_guide/troubleshooting`
- **GitHub issues**: https://github.com/LLNL/radiuss-shared-ci/issues

.. seealso::

   - :doc:`../user_guide/concepts` - Architecture and design
   - :doc:`../user_guide/components_migration` - Migrate from legacy
   - ``examples/example-gitlab-ci.yml`` - Complete working example
