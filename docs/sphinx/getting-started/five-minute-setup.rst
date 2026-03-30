.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _five-minute-setup-label:

*******************
5-Minute Quick Start
*******************

Get a basic RADIUSS Shared CI pipeline running in 5 minutes.

.. important::
   Before starting, ensure you've completed the :doc:`prerequisites` checklist.

====================
Overview
====================

This guide will set up a minimal CI pipeline that:

- Checks if Dane machine is available
- Runs a single build/test job on Dane
- Reports status to GitHub

You can expand this later to add more machines and jobs.

====================
Step 1: Create Files
====================

In your project root, create the required directory structure:

.. code-block:: bash

   cd /path/to/your/project
   mkdir -p .gitlab/jobs

You'll create three files:

1. ``.gitlab-ci.yml`` - Main pipeline configuration
2. ``.gitlab/jobs/dane.yml`` - Dane-specific jobs
3. ``scripts/build-and-test.sh`` - Your build/test script (if you don't have one)

=============================
Step 2: Main Pipeline File
=============================

Create ``.gitlab-ci.yml`` in your project root:

.. code-block:: yaml

   ###############################################################################
   # RADIUSS Shared CI - Minimal Component-Based Setup
   ###############################################################################

   # Required: Define stages
   stages:
     - prerequisites
     - build-and-test

   # Configuration variables
   variables:
     # GitHub project info
     GITHUB_PROJECT_NAME: "myproject"      # ← CHANGE THIS
     GITHUB_PROJECT_ORG: "LLNL"            # ← CHANGE THIS if needed

     # Build command (use your actual script path)
     JOB_CMD:
       value: "./scripts/build-and-test.sh"
       expand: false

     # Optional: Service user
     LLNL_SERVICE_USER: ""  # Set if using service account

     # Optional: Git submodules
     GIT_SUBMODULE_STRATEGY: recursive  # or "none" if no submodules

   # Include RADIUSS Shared CI components
   include:
     # Base orchestration component
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
       inputs:
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG
         github_token: $GITHUB_STATUS_TOKEN

   # Machine availability check
   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"

   # Dane build and test pipeline
   dane-build-and-test:
     needs: [dane-up-check]
     extends: [.dane, .build-and-test]
     trigger:
       include:
         # Dane machine component
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD
             shared_alloc: "--nodes=1 --exclusive --time=30"
             job_alloc: "--nodes=1"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
             llnl_service_user: $LLNL_SERVICE_USER
         # Your Dane-specific jobs
         - local: '.gitlab/jobs/dane.yml'

**Required changes**:

1. Line 13: Change ``"myproject"`` to your GitHub project name
2. Line 14: Change ``"LLNL"`` if your org is different
3. Line 18: Change to match your build script path

=============================
Step 3: Dane Jobs File
=============================

Create ``.gitlab/jobs/dane.yml``:

.. code-block:: yaml

   ###############################################################################
   # Dane Machine Jobs
   ###############################################################################

   # Minimal job - inherits everything from .job_on_dane template
   basic-build:
     extends: .job_on_dane

**That's it!** This creates one job that runs your ``JOB_CMD`` on Dane.

To add more jobs with different configurations:

.. code-block:: yaml

   # Multiple compiler jobs
   gcc-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"
       COMPILER_VERSION: "11.0.0"

   clang-build:
     extends: .job_on_dane
     variables:
       COMPILER: "clang"
       COMPILER_VERSION: "14.0.0"

Your build script can read these ``COMPILER`` variables.

.. notes::
   RADIUSS projects that leverage radiuss-shared-ci typically use Spack in the
   build script and pass the spack spec as a variable.

=================================
Step 4: Build Script (Optional)
=================================

If you don't have a build/test script yet, create ``scripts/build-and-test.sh``:

.. code-block:: bash

   #!/bin/bash
   set -e  # Exit on first error

   echo "Building and testing project..."

   # Example: Load environment
   module load gcc/13.3.1
   module load cmake/3.25

   # Example: Configure
   cmake -B build \
     -DCMAKE_BUILD_TYPE=Release \
     -DCMAKE_CXX_COMPILER=g++

   # Example: Build
   cmake --build build -j8

   # Example: Test
   cd build
   ctest --output-on-failure

   echo "Build and test completed successfully!"

Make it executable:

.. code-block:: bash

   chmod +x scripts/build-and-test.sh

.. note::
   Adapt this script to your project's actual build system (CMake, Make,
   Spack, etc.).

====================================
Step 5: Configure GitHub Token
====================================

In LC GitLab, add your GitHub token as a CI/CD variable:

1. Navigate to your GitLab project settings:
   ``https://lc.llnl.gov/gitlab/<your-org>/<your-project>/-/settings/ci_cd``

2. Expand "Variables" section

3. Click "Add variable"

4. Configure:

   - **Key**: ``GITHUB_STATUS_TOKEN``
   - **Value**: (paste your GitHub personal access token)
   - **Type**: Variable
   - **Protected**: ☑ (recommended)
   - **Masked**: ☑ (recommended)
   - **Expand variable reference**: ☐ (unchecked)

5. Click "Add variable"

.. warning::
   Never commit your token to Git! Always set it as a CI/CD variable.

===========================
Step 6: Commit and Push
===========================

Add and commit your new files:

.. code-block:: bash

   git add .gitlab-ci.yml .gitlab/jobs/dane.yml scripts/build-and-test.sh
   git commit -m "Add RADIUSS Shared CI configuration

   - Configure component-based CI for Dane
   - Add basic build/test job
   - Include build script"

   git push origin main  # or your branch name

=========================
Step 7: Verify in GitLab
=========================

After pushing (and waiting for GitHub → GitLab mirroring):

1. **View Pipeline**:
   Go to your LC GitLab project → CI/CD → Pipelines

2. **Check Stages**:

   .. code-block:: text

      ┌─────────────────┐     ┌──────────────────┐
      │  prerequisites  │ ──→ │ build-and-test   │
      │                 │     │                  │
      │ dane-up-check ✓ │     │ dane-build-and-  │
      │                 │     │ test (child)   ⟳ │
      └─────────────────┘     └──────────────────┘

3. **View Child Pipeline**:
   Click on ``dane-build-and-test`` to see the actual build job(s)

4. **Check GitHub**:
   Go to your GitHub PR/commit and verify CI status appears

====================
What Just Happened?
====================

Let's break down what you created:

**Parent Pipeline** (``.gitlab-ci.yml``):

1. Defines two stages: ``prerequisites`` and ``build-and-test``
2. Includes ``base-pipeline`` component for orchestration templates
3. Creates ``dane-up-check`` job to verify Dane is available
4. Creates ``dane-build-and-test`` trigger that spawns a child pipeline

**Child Pipeline** (triggered for Dane):

1. Includes ``dane-pipeline`` component with templates for Dane
2. Includes your ``.gitlab/jobs/dane.yml`` with actual jobs
3. Jobs run within a shared allocation on Dane
4. Each job executes your ``JOB_CMD`` in the appropriate environment

**Reporting**:

- Parent pipeline reports to GitHub: "dane-up-check" and overall status
- Child pipeline reports to GitHub: "dane-build-and-test"
- You see both statuses on GitHub PR/commit

==============
Next Steps
==============

Now that you have a basic pipeline running, you can:

**Add More Machines**:
  Copy the ``dane-up-check`` and ``dane-build-and-test`` sections, changing
  ``dane`` to other machines (``matrix``, ``corona``, ``tioga``, etc.)

  See: :doc:`choosing-your-path`

**Add More Jobs**:
  Edit ``.gitlab/jobs/dane.yml`` to add more jobs with different
  configurations (compilers, build types, etc.)

**Customize Allocations**:
  Adjust ``shared_alloc`` and ``job_alloc`` parameters to your actual needs.

**Add Utilities**:
  Include ``utility-draft-pr-filter`` to skip CI on draft PRs

**Enable Performance Testing**:
  Add the ``performance-pipeline`` component and performance jobs

=======================
Troubleshooting
=======================

Pipeline Doesn't Start
======================

- **Check mirroring**: Verify your project exists on LC GitLab
- **Check syntax**: Validate YAML with ``yamllint .gitlab-ci.yml``
- **Check variables**: Ensure ``GITHUB_PROJECT_NAME`` matches your repo

Machine Check Fails
===================

- **Dane is down**: This is expected - the pipeline will skip gracefully
- **Lorenz file missing**: Contact LC support

Jobs Fail
=========

- **Script errors**: Check job logs for script failures
- **Path issues**: Ensure ``JOB_CMD`` path is correct
- **Module not found**: Check existence of the loaded modules on the machine
- **Permission denied**: Check service user permissions

GitHub Status Not Showing
==========================

- **Token missing**: Verify ``GITHUB_STATUS_TOKEN`` is set in GitLab variables
- **Token expired**: Regenerate token if needed
- **Wrong permissions**: Token needs ``repo:status`` scope

==============
Get Help
==============

- **Quick reference**: :doc:`../user_guide/quick-reference`
- **Troubleshooting guide**: :doc:`../dev_guide/troubleshooting`
- **Full setup documentation**: :doc:`../user_guide/setup_ci`
- **GitHub issues**: https://github.com/LLNL/radiuss-shared-ci/issues

.. seealso::

   - :doc:`choosing-your-path` - Decide which machines and features to enable
   - :doc:`../user_guide/concepts` - Understand the architecture
   - :doc:`../user_guide/components_migration` - Migrate from legacy setup
