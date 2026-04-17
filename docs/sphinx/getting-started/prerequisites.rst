.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _prerequisites-label:

*************
Prerequisites
*************

Before setting up RADIUSS Shared CI, ensure you have the following requirements
in place.

==========================
System Requirements
==========================

GitLab Version
==============

**For component-based setup (recommended):**
  GitLab 17.0 or later is required. Check your LC GitLab version at
  https://lc.llnl.gov/gitlab/help

**For legacy include-based setup:**
  Any GitLab version, but components are strongly recommended for new projects.

.. note::
   To check your GitLab version, visit the LC GitLab help page or contact your
   GitLab administrator.

Access Requirements
===================

You will need:

**LC GitLab Account**
  At least one team member must have an account on the LC GitLab instance
  (https://lc.llnl.gov/gitlab). This account will be used to:

  - Configure the CI pipeline
  - Set up mirroring from GitHub
  - Configure CI/CD variables
  - Monitor pipeline execution

**GitHub Repository**
  Your project must be hosted in a GitHub organization (typically LLNL) and
  accessible for mirroring to LC GitLab.

**Machine Access**
  Access to at least one LC machine where you want to run CI. Your GitLab
  account (or service user) needs:

  - Login access to the machine
  - Ability to submit jobs to the scheduler
  - Adequate allocation or queue access

.. _service-user-setup:

=================================
Service User Account (Recommended)
=================================

We **strongly recommend** using an LLNL service user account for CI rather than
individual user accounts.

Why Use a Service User?
========================

**Consistent Permissions**
  All CI runs use the same account, avoiding permission issues when different
  team members trigger pipelines.

**Quota Management**
  Service accounts typically have separate disk quotas, preventing CI from
  filling individual user home directories.

**Team Access**
  Multiple team members can be added to the service user group, allowing them
  to:

  - Restart failed pipelines
  - Debug CI issues
  - Access CI build artifacts

**Continuity**
  CI continues to work when individual team members leave the project.

How to Request a Service User
==============================

1. **Submit Request**:
   Contact LC support to request a service user account for your project.
   Provide:

   - Project name
   - Justification (CI/CD automation)
   - Initial list of team members who need access

2. **Configure Access**:
   Once created, team members can be added to the service user group by LC
   support.

3. **Set in CI**:
   Add the service user to your ``.gitlab-ci.yml``:

   .. code-block:: yaml

      variables:
        LLNL_SERVICE_USER: "myproject"  # Your service user name

.. note::
   While not strictly required, projects without a service user may encounter
   disk quota issues and permission problems. Plan to set this up early.

================================
Build and Test Script
================================

RADIUSS Shared CI requires a **single command** that builds and tests your
project. This is the ``JOB_CMD`` variable you'll configure.

.. note::
   This requirement is intentional to keep the build and test process separated
   from CI configuration. It allows you to maintain a single build/test script
   that works both locally and in CI. It also keeps the CI configuration simple
   and focused on orchestration.

Requirements
============

Your build/test script must:

1. **Be executable** - Either a shell script or a command that can be invoked
   directly

2. **Be self-contained** - Handle all setup, build, test, and cleanup steps

3. **Exit with appropriate codes**:

   - ``0`` for success
   - Non-zero for failure

4. **Work on target machines** - Script should handle machine-specific
   environment setup

Example Scripts
===============

**Simple bash script** (``scripts/build-and-test.sh``):

.. code-block:: bash

   #!/bin/bash
   set -e  # Exit on error

   # Load modules
   module load gcc/13.3.1
   module load cmake/3.25

   # Configure
   cmake -B build -DCMAKE_BUILD_TYPE=Release

   # Build
   cmake --build build -j8

   # Test
   cd build
   ctest --output-on-failure

**With Spack** (using RADIUSS Spack Configs):

.. code-block:: bash

   #!/bin/bash
   set -e

   # Build and test
   python scripts/uberenv/uberenv.py \
     --spec="${SPEC}" ${pwd}

   # Run tests
   cd build
   make test

.. tip::
   Keep CI scripts in a ``scripts/`` or ``.gitlab/scripts/`` directory for
   organization.

Testing Your Script Locally
============================

Before configuring CI, verify your script works on target machines:

.. code-block:: bash

   # SSH to target machine (e.g., dane)
   ssh dane

   # Clone your project
   git clone https://github.com/LLNL/myproject.git
   cd myproject

   # Run your build script
   ./scripts/build-and-test.sh

If your script works locally, it should work in CI (assuming the same
environment).

==========================
GitHub Integration
==========================

GitHub Personal Access Token
=============================

For CI status reporting to GitHub, you'll need a personal access token.

**Create Token**:

1. Go to GitHub: Settings → Developer settings → Personal access tokens →
   Tokens (classic)

2. Click "Generate new token (classic)"

3. Configure token:

   - **Note**: "RADIUSS Shared CI - MyProject"
   - **Expiration**: 30 days (maximum allowed by LLNL policy)
   - **Scopes**: Select ``admin:repo_hook`` and ``repo``

4. Click "Generate token" and **copy it immediately** (you won't see it again)

**Add to GitLab**:

You'll add this token as a CI/CD variable in GitLab (covered in setup guide).

.. warning::
   Keep your token secure! Anyone with this token can access your repository
   according to the permissions granted.

Mirroring Setup
===============

Your GitHub repository needs to be mirrored to LC GitLab so CI can run.

**For RADIUSS Projects**:

  If your project is in the LLNL organization and part of RADIUSS, mirroring
  may already be set up. Check if your project exists at:
  https://lc.llnl.gov/gitlab/radiuss/<your-project>

**For Other Projects**:

  Follow GitLab's official documentation:
  https://docs.gitlab.com/ee/ci/ci_cd_for_external_repos/github_integration.html

  Key steps:

  1. In LC GitLab, create a new project
  2. Choose "CI/CD for external repo"
  3. Connect to GitHub with a personal access token
  4. Select your repository
  5. GitLab will automatically sync changes

.. note::
   Mirroring typically has a delay (a few minutes). Changes pushed to GitHub
   won't trigger CI instantly.

====================
Quick Checklist
====================

Before proceeding to setup, confirm you have:

.. list-table::
   :widths: 10 90
   :header-rows: 0

   * - ☐
     - GitLab 17.0+ (for components) or any version (for legacy)
   * - ☐
     - LC GitLab account with access to target machines
   * - ☐
     - Service user account (recommended) or plan to use personal account
   * - ☐
     - Build/test script that works on target machines
   * - ☐
     - GitHub personal access token with ``admin:repo_hook`` and ``repo`` permissions
   * - ☐
     - GitHub repository mirrored to LC GitLab
   * - ☐
     - Machine access and allocation/queue permissions

==============
Next Steps
==============

Once you've confirmed all prerequisites:

→ **Continue to** :doc:`five-minute-setup`

Need More Context?
==================

- **Understand the architecture**: :doc:`../user_guide/concepts`
- **Quick reference**: :doc:`../user_guide/quick-reference`
- **Full setup guide**: :doc:`../user_guide/setup-with-components`

.. seealso::

   - `LC GitLab Documentation <https://lc.llnl.gov/gitlab/help>`_
   - `GitHub Token Documentation <https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token>`_
   - `GitLab Mirroring Guide <https://docs.gitlab.com/ee/ci/ci_cd_for_external_repos/github_integration.html>`_
