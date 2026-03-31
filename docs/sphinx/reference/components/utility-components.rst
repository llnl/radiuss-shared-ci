.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _utility-components-reference:

*******************
Utility Components
*******************

Utility components provide optional workflow enhancements for RADIUSS Shared CI
pipelines. They aim at adding features that are not provided by GitLab.

========
Overview
========

RADIUSS Shared CI provides 2 utility components:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Component
     - Purpose
   * - **utility-draft-pr-filter**
     - Skip CI on draft pull requests to save resources
   * - **utility-branch-skip**
     - Skip CI on branches not associated with a PR

Both components are **optional** and can be used independently or together.

=========================
utility-draft-pr-filter
=========================

Skip CI execution when a GitHub pull request is in draft state, saving
machine time and resources until the PR is ready for review.

Purpose
=======

- **Resource Conservation**: Avoid running CI on work-in-progress PRs
- **Clear Communication**: Explicitly marks draft PRs as "skipped" on GitHub
- **Flexible Override**: Allow specific branches to always run (e.g., main, develop)

When to Use
===========

Enable this utility when:

- Your team uses GitHub draft PRs for work-in-progress
- You want to reduce CI load on incomplete work
- You have limited CI resources/allocations

**Skip if**:

- Your team doesn't use draft PRs
- You want CI to run on all commits regardless of state

Inputs
======

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Input
     - Required
     - Description
   * - ``github_token``
     - Yes
     - GitHub token with ``repo:status`` permission
   * - ``github_project_name``
     - Yes
     - GitHub repository name
   * - ``github_project_org``
     - Yes
     - GitHub organization name
   * - ``always_run_pattern``
     - No
     - Regex pattern for branches that always run (default: ``"^(main|master|develop)$"``)

Behavior
========

The component:

1. Checks if current commit is from a GitHub PR
2. If yes, queries GitHub API to check if PR is draft
3. If draft:
   - Reports "Draft PR - CI skipped" to GitHub
   - Exits pipeline gracefully
4. If not draft or branch matches ``always_run_pattern``:
   - Pipeline continues normally

Usage
=====

.. code-block:: yaml

   include:
     # Add after base-pipeline
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
       inputs: { ... }

     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/utility-draft-pr-filter@v2026.02.2
       inputs:
         github_token: $GITHUB_STATUS_TOKEN
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG

Default behavior skips draft PRs except on ``main``, ``master``, or ``develop``.

Custom Override Pattern
=======================

Customize which branches always run:

.. code-block:: yaml

   - component: .../utility-draft-pr-filter@v2026.02.2
     inputs:
       github_token: $GITHUB_STATUS_TOKEN
       github_project_name: $GITHUB_PROJECT_NAME
       github_project_org: $GITHUB_PROJECT_ORG
       always_run_pattern: "^(main|develop|release/.*)$"

This allows:

- ``main`` - Always runs
- ``develop`` - Always runs
- ``release/v1.0`` - Always runs
- ``feature/draft-pr`` - Skips if draft

Examples
========

Basic Setup
-----------

.. code-block:: yaml

   # .gitlab-ci.yml
   include:
     - component: .../base-pipeline@v2026.02.2
       inputs:
         github_project_name: "my-project"
         github_project_org: "LLNL"
         github_token: $GITHUB_STATUS_TOKEN

     # Add draft PR filter
     - component: .../utility-draft-pr-filter@v2026.02.2
       inputs:
         github_token: $GITHUB_STATUS_TOKEN
         github_project_name: "my-project"
         github_project_org: "LLNL"

No Override (Run on All Branches)
----------------------------------

To always run CI regardless of draft status:

.. code-block:: yaml

   - component: .../utility-draft-pr-filter@v2026.02.2
     inputs:
       github_token: $GITHUB_STATUS_TOKEN
       github_project_name: "my-project"
       github_project_org: "LLNL"
       always_run_pattern: ".*"  # Match all branches

This effectively disables the filter while keeping it in configuration.

====================
utility-branch-skip
====================

Skip CI execution on branches that are not associated with an open GitHub pull
request, reducing noise from experimental or personal branches.

Purpose
=======

- **Reduce CI Noise**: Avoid running CI on experimental branches
- **Save Resources**: Only run CI on branches being reviewed
- **PR-Focused Workflow**: Enforce "no CI without PR" policy

When to Use
===========

Enable this utility when:

- You want CI only on pull requests
- You have many feature/experimental branches
- You want to reduce unnecessary CI runs

**Skip if**:

- You want CI on all branch pushes
- Your main/develop branches need CI without PRs
- Your workflow doesn't center around PRs

.. warning::
   This utility will skip CI on your main/develop branches unless you:

   1. Use ``always_run_pattern`` to override (recommended), OR
   2. Always have PRs open for those branches

Inputs
======

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Input
     - Required
     - Description
   * - ``github_token``
     - Yes
     - GitHub token with ``repo:status`` permission
   * - ``github_project_name``
     - Yes
     - GitHub repository name
   * - ``github_project_org``
     - Yes
     - GitHub organization name
   * - ``always_run_pattern``
     - No
     - Regex pattern for branches that always run (default: ``"^(main|master|develop)$"``)

Behavior
========

The component:

1. Checks if current branch has an open PR on GitHub
2. If no PR found:
   - Checks if branch matches ``always_run_pattern``
   - If no match: Reports "Not a PR - CI skipped" and exits
   - If match: Pipeline continues
3. If PR found:
   - Pipeline continues normally

Usage
=====

.. code-block:: yaml

   include:
     - component: .../base-pipeline@v2026.02.2
       inputs: { ... }

     - component: .../utility-branch-skip@v2026.02.2
       inputs:
         github_token: $GITHUB_STATUS_TOKEN
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG

Default behavior runs CI on:

- Any branch with an open PR
- ``main``, ``master``, or ``develop`` (even without PR)

Custom Override Pattern
=======================

Customize which branches always run:

.. code-block:: yaml

   - component: .../utility-branch-skip@v2026.02.2
     inputs:
       github_token: $GITHUB_STATUS_TOKEN
       github_project_name: $GITHUB_PROJECT_NAME
       github_project_org: $GITHUB_PROJECT_ORG
       always_run_pattern: "^(main|develop|release/.*|hotfix/.*)$"

Examples
========

Basic Setup
-----------

.. code-block:: yaml

   include:
     - component: .../base-pipeline@v2026.02.2
       inputs: { ... }

     # Skip branches without PRs
     - component: .../utility-branch-skip@v2026.02.2
       inputs:
         github_token: $GITHUB_STATUS_TOKEN
         github_project_name: "my-project"
         github_project_org: "LLNL"

Strict PR-Only Mode
--------------------

Require PR for all branches (even main/develop):

.. code-block:: yaml

   - component: .../utility-branch-skip@v2026.02.2
     inputs:
       github_token: $GITHUB_STATUS_TOKEN
       github_project_name: "my-project"
       github_project_org: "LLNL"
       always_run_pattern: "^$"  # Empty pattern = no exceptions

.. warning::
   This will skip CI on main/develop if they don't have open PRs!

===================
Combining Utilities
===================

Both utilities can be used together for fine-grained control:

.. code-block:: yaml

   include:
     - component: .../base-pipeline@v2026.02.2
       inputs: { ... }

     # Skip draft PRs
     - component: .../utility-draft-pr-filter@v2026.02.2
       inputs:
         github_token: $GITHUB_STATUS_TOKEN
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG

     # Skip non-PR branches
     - component: .../utility-branch-skip@v2026.02.2
       inputs:
         github_token: $GITHUB_STATUS_TOKEN
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG

This configuration:

- Skips CI on draft PRs (but runs when converted to ready)
- Skips CI on branches without PRs
- Always runs CI on main/master/develop (even without PR)
- Runs CI on non-draft PRs for any branch

==============
Common Issues
==============

Token Missing or Invalid
=========================

**Error**: "GitHub API request failed" or "remote: Permission to
LLNL/my-project.git denied"

**Solutions**:

1. Verify ``GITHUB_STATUS_TOKEN`` is set in GitLab UI (CI/CD variables)
2. Check token hasn't expired (regenerate if needed)
3. Ensure token has ``repo:status`` permission minimum

.. note::
   Token must have a maximum of 30 days validity to comply with LLNL's security
   policies.

CI Skipped on Main Branch
==========================

**Symptom**: Main branch shows "Not a PR - CI skipped"

**Solution**: Verify ``always_run_pattern`` includes your main branch:

.. code-block:: yaml

   inputs:
     always_run_pattern: "^(main|master|develop)$"

Check your branch name matches (e.g., "main" vs "master").

CI Runs on Draft PRs
=====================

**Symptom**: Draft PRs still trigger full CI

**Possible causes**:

1. **Component not included**: Verify draft-pr-filter is in ``.gitlab-ci.yml``
2. **Token missing**: Check ``github_token`` input provided
3. **Branch override**: Branch matches ``always_run_pattern``

Regex Pattern Not Matching
===========================

**Symptom**: Branch you expected to run gets skipped

**Solution**: Test your regex pattern. Common mistakes:

.. code-block:: yaml

   # Wrong: Matches branches containing these words anywhere
   always_run_pattern: "main|develop"

   # Right: Matches branches that are exactly these names
   always_run_pattern: "^(main|develop)$"

   # Right: Matches branches starting with "release/"
   always_run_pattern: "^(main|develop|release/.*)"

========
See Also
========

- :doc:`base-pipeline` - Core orchestration component
- :doc:`../getting-started/choosing-your-path` - Choosing utilities guide
- :doc:`../user_guide/quick-reference` - Quick utility setup reference

**Related Topics**:

- GitHub token permissions and scopes
- GitLab CI rules and workflow control
- Draft PR workflows
