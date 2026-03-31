.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _base-pipeline-reference:

*************
base-pipeline
*************

The foundational component that provides orchestration templates and machine
availability checks for RADIUSS Shared CI pipelines.

=======
Purpose
=======

The ``base-pipeline`` component is **required** for all RADIUSS Shared CI
setups. It provides:

- **Orchestration Templates**: Job templates for child pipeline triggers
- **Machine Check Templates**: Templates checking machine availability and reporting status
- **Customization Hooks**: Base templates for project-specific customization
- **Machine Rules**: Templates that define machine-specific behavior

This component does NOT define stages (you must define them yourself) to allow
customization.

===========
When to Use
===========

**Always.** Every RADIUSS Shared CI pipeline must include ``base-pipeline``.

Include it first, before any machine or utility components:

.. code-block:: yaml

   include:
     # Always include base-pipeline first
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
       inputs:
         github_project_name: "my-project"
         github_project_org: "LLNL"
         github_token: $GITHUB_STATUS_TOKEN

     # Then other components
     - component: .../dane-pipeline@v2026.02.2
       inputs: { ... }

======
Inputs
======

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 45

   * - Input
     - Type
     - Required
     - Description
   * - ``github_project_name``
     - string
     - Yes
     - GitHub repository name (e.g., "RAJA", "Umpire")
   * - ``github_project_org``
     - string
     - Yes
     - GitHub organization name (e.g., "LLNL")
   * - ``github_token``
     - string
     - No
     - GitHub personal access token for status updates. If not provided, status reporting will be skipped. Set as CI/CD variable in GitLab UI.

==================
Exported Templates
==================

The component exports several job templates you can extend:

.machine-check
==============

**Purpose**: Template for machine availability check jobs.

**Usage**: Create machine check jobs by extending this template with machine-specific rules.

.. code-block:: yaml

   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"

**What it does**:

1. Queries Lorenz status file for machine availability
2. Reports status to GitHub (if token provided)
3. Fails if machine is down (preventing child pipeline from starting)
4. Uses ``ASSOCIATED_CHILD_PIPELINE`` variable to set correct GitHub status context

**Key Variables**:

- ``CI_MACHINE``: Set by machine rule (e.g., ``.dane`` sets it to "dane")
- ``ASSOCIATED_CHILD_PIPELINE``: Name of the child pipeline job (e.g., "dane-build-and-test")

.. important::
   As of v2026.02.0, ``ASSOCIATED_CHILD_PIPELINE`` is **required**. This ensures
   machine check status uses the same GitHub context as the child pipeline,
   allowing failing checks to be overridden when the machine comes back up.

.build-and-test
===============

**Purpose**: Template for child pipeline trigger jobs.

**Usage**: Create pipeline trigger jobs that spawn machine-specific child pipelines.

.. code-block:: yaml

   dane-build-and-test:
     needs: [dane-up-check]
     extends: [.dane, .build-and-test]
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2
           inputs: { ... }
         - local: '.gitlab/jobs/dane.yml'

**What it does**:

1. Provides trigger configuration template
2. Uses ``strategy: depend`` to wait for child completion
3. Forwards pipeline variables to child
4. Sets appropriate stage (``build-and-test``)

.custom_job
===========

**Purpose**: Base customization template for all build/test jobs.

**Usage**: Override in your ``.gitlab/custom-jobs.yml`` to add project-specific setup.

.. code-block:: yaml

   # In .gitlab/custom-jobs.yml
   .custom_job:
     before_script:
       - echo "Project-specific setup"
       - module load my-dependencies
     artifacts:
       reports:
         junit: junit.xml

**What it does**:

- Provides an empty template that all job templates extend
- Allows to define project specific job configuration in one place
- Applies to all build/test jobs across all machines

.custom_perf
============

**Purpose**: Base customization template for performance jobs.

**Usage**: Override in your ``.gitlab/custom-jobs.yml`` for performance-specific setup.

.. code-block:: yaml

   # In .gitlab/custom-jobs.yml
   .custom_perf:
     before_script:
       - echo "Performance environment setup"
       - export PERF_FLAGS="..."

**What it does**:

- Similar to ``.custom_job`` but for performance pipeline
- Extends to all performance jobs
- Separate from ``.custom_job`` to allow different configurations

Machine-Specific Rules
=======================

The component also exports machine-specific rule templates (not typically used directly):

- ``.dane`` - Sets ``CI_MACHINE: "dane"`` and machine-specific rules
- ``.matrix`` - Sets ``CI_MACHINE: "matrix"``
- ``.corona`` - Sets ``CI_MACHINE: "corona"``
- ``.tioga`` - Sets ``CI_MACHINE: "tioga"``
- ``.tuolumne`` - Sets ``CI_MACHINE: "tuolumne"``

These are used with ``.machine-check`` and ``.build-and-test`` templates.

=============
Variables Set
=============

The component sets several internal variables:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Variable
     - Description
   * - ``RSCI_GITHUB_PROJECT_NAME``
     - Internal: GitHub project name from input
   * - ``RSCI_GITHUB_PROJECT_ORG``
     - Internal: GitHub organization from input
   * - ``RSCI_GITHUB_TOKEN``
     - Internal: GitHub token from input

.. note::
   ``RSCI_`` prefix was added in v2026.02.1 to avoid collisions with GitLab
   global variables. You don't need to use these directly - they're for
   internal component use.

The component also includes ID token configuration from ``lc-templates/id_tokens``.

========
Examples
========

Minimal Example
===============

Simplest possible setup with base-pipeline:

.. code-block:: yaml

   stages:
     - prerequisites
     - build-and-test

   variables:
     GITHUB_PROJECT_NAME: "my-project"
     GITHUB_PROJECT_ORG: "LLNL"

   include:
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
       inputs:
         github_project_name: $GITHUB_PROJECT_NAME
         github_project_org: $GITHUB_PROJECT_ORG
         github_token: $GITHUB_STATUS_TOKEN

Typical Example
===============

Standard setup with machine checks and child pipelines:

.. code-block:: yaml

   stages:
     - prerequisites
     - build-and-test

   variables:
     GITHUB_PROJECT_NAME: "my-project"
     GITHUB_PROJECT_ORG: "LLNL"
     JOB_CMD:
       value: "./scripts/build-and-test.sh"
       expand: false

   include:
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

   # Child pipeline trigger
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

Advanced Example with Custom Jobs
==================================

Using custom job templates:

.. code-block:: yaml

   # .gitlab-ci.yml
   include:
     - component: .../base-pipeline@v2026.02.2
       inputs: { ... }

   # In child pipeline trigger
   dane-build-and-test:
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2
         - local: '.gitlab/custom-jobs.yml'  # Your customizations
         - local: '.gitlab/jobs/dane.yml'

.. code-block:: yaml

   # .gitlab/custom-jobs.yml
   .custom_job:
     before_script:
       - echo "Setting up environment"
       - export MY_VAR="value"
     after_script:
       - echo "Cleaning up"
     artifacts:
       reports:
         junit: build/junit.xml
       paths:
         - build/logs/

==============
Common Issues
==============

Missing ASSOCIATED_CHILD_PIPELINE
==================================

**Error**: Machine check job fails with "ASSOCIATED_CHILD_PIPELINE variable not set"

**Solution**: Add the variable to your machine check job:

.. code-block:: yaml

   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"  # Add this

**Why**: This ensures the machine check and child pipeline share the same
GitHub status context.

GitHub Status Not Showing
==========================

**Symptom**: Pipeline runs but no status appears on GitHub PR/commit

**Possible causes**:

1. **Token not provided**: Add ``github_token`` input
2. **Token expired**: Regenerate token in GitHub settings (max 30 days expiration per LLNL policy)
3. **Wrong permissions**: Token needs ``repo:status`` scope
4. **Token not in GitLab**: Set ``GITHUB_STATUS_TOKEN`` variable in GitLab CI/CD settings
5. **Wrong project name**: Verify ``github_project_name`` matches GitHub repo name

Stages Not Defined
===================

**Error**: "Stage 'prerequisites' not defined" or similar

**Solution**: The component does NOT define stages. Add them to your ``.gitlab-ci.yml``:

.. code-block:: yaml

   stages:
     - prerequisites      # Required
     - build-and-test     # Required
     - performance-measurements  # If using performance pipeline

**Why**: Allowing you to define stages enables adding custom stages without
conflicts.

Template Not Found
==================

**Error**: "Job extends template that doesn't exist: .machine-check"

**Solution**: Ensure ``base-pipeline`` is included in your parent pipeline
(not child pipeline).

.. code-block:: yaml

   # In .gitlab-ci.yml (parent pipeline)
   include:
     - component: .../base-pipeline@v2026.02.2  # Include here

========
See Also
========

- :doc:`machine-pipelines` - Machine-specific pipeline components
- :doc:`../getting-started/five-minute-setup` - Quick start guide
- :doc:`../user_guide/quick-reference` - Quick reference card
- :doc:`../user_guide/concepts` - Architecture concepts

**Related Components**:

- All machine pipelines require ``base-pipeline``
- ``performance-pipeline`` uses ``.custom_perf`` template
- Utility components work alongside ``base-pipeline``
