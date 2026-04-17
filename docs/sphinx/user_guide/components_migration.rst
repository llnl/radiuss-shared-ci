.. _components-migration:

===================================
Migrating to GitLab CI Components
===================================

This guide helps you migrate from the legacy include-based approach to
GitLab CI Components.

.. note::
   **New to RADIUSS Shared CI?** You don't need this guide. See
   :doc:`../getting-started/five-minute-setup` instead.

Quick Migration Checklist
--------------------------

.. list-table::
   :widths: 10 90
   :header-rows: 0

   * - ☐
     - Verify GitLab 17.0+ (check at https://lc.llnl.gov/gitlab/help)
   * - ☐
     - Add stages to ``.gitlab-ci.yml`` (components don't define them)
   * - ☐
     - Replace ``include: project:`` with ``component:`` directives
   * - ☐
     - Split ``custom-jobs-and-variables.yml`` into two files
   * - ☐
     - Update machine pipeline triggers to use components
   * - ☐
     - Test on a branch before merging
   * - ☐
     - Update documentation/README if you have migration notes

Prerequisites
-------------

* GitLab 17.0 or later (required for components)
* Familiarity with your current CI setup
* Access to your project's ``.gitlab-ci.yml`` and ``.gitlab/`` directory

Benefits of Components
----------------------

The component-based approach provides several advantages over the legacy include method:

**Type-Safe Inputs**
  Components validate input parameters, catching configuration errors early.

**Better Versioning**
  Components appear in the GitLab CI/CD Catalog with clear versioning.

**Cleaner Syntax**
  Use ``component:`` with named inputs instead of ``include: project:`` with file paths.

**No Copy-Pasting**
  Templates that previously required copying (like ``customization/gitlab-ci.yml``)
  are now reusable components.

**Improved Discoverability**
  Browse available components in the GitLab CI/CD Catalog UI.

Migration Steps
---------------

Step 1: Define Required Stages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**IMPORTANT:** The base-pipeline component does NOT define stages to allow you to
add your own custom stages. You must define the required stages in your ``.gitlab-ci.yml``:

.. code-block:: yaml

   # .gitlab-ci.yml
   stages:
     - prerequisites           # Required for machine checks
     - build-and-test          # Required for build/test jobs
     - performance-measurements # Required if using perf pipeline
     # Add your custom stages here:
     # - my-custom-stage

Step 2: Update Main Pipeline File
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Before (Legacy approach):**

.. code-block:: yaml

   # .gitlab-ci.yml
   include:
     - project: 'lc-templates/id_tokens'
       file: 'id_tokens.yml'
     - project: 'radiuss/radiuss-shared-ci'
       ref: 'v2026.02.2'
       file: 'utilities/preliminary-ignore-draft-pr.yml'
     - local: '.gitlab/subscribed-pipelines.yml'

   variables:
     GITHUB_PROJECT_NAME: "my-project"
     GITHUB_PROJECT_ORG: "LLNL"
     JOB_CMD:
       value: "./scripts/build-and-test.sh"
       expand: false

**After (Component-based):**

.. code-block:: yaml

   # .gitlab-ci.yml
   include:
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
       inputs:
         github_project_name: "my-project"
         github_project_org: "LLNL"
         github_token: $GITHUB_TOKEN

     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/utility-draft-pr-filter@v2026.02.2
       inputs:
         github_token: $GITHUB_TOKEN
         github_project_name: "my-project"
         github_project_org: "LLNL"

     - local: '.gitlab/custom-variables.yml'

   variables:
     JOB_CMD:
       value: "./scripts/build-and-test.sh"
       expand: false

.. note::
   **File Split:** The legacy ``custom-jobs-and-variables.yml`` has been split into two files:

   * ``.gitlab/custom-variables.yml`` - Variables only (included in parent pipeline)
   * ``.gitlab/custom-jobs.yml`` - Job templates only (included in child pipelines)

   This prevents duplication and makes it clear where each piece is used.

Step 3: Update Machine Pipeline Triggers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Before (in subscribed-pipelines.yml):**

.. code-block:: yaml

   dane-build-and-test:
     variables:
       CI_MACHINE: "dane"
     needs: [dane-up-check]
     extends: [.build-and-test]
     rules:
       # Runs except if we explicitly deactivate dane by variable.
       - if: '$ON_DANE == "OFF"'
         when: never
       - when: on_success

**After (in .gitlab-ci.yml):**

.. code-block:: yaml

   dane-build-and-test:
     variables:
       CI_MACHINE: "dane"
     needs: [dane-up-check]
     extends: [.build-and-test]
     rules:
       - if: '$ON_DANE == "OFF"'
         when: never
       - when: on_success
     trigger:
       include:
         - local: '.gitlab/custom-jobs.yml'
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@v2026.02.2
           inputs:
             job_cmd: $JOB_CMD
             shared_alloc: "--nodes=1 --exclusive --reservation=ci --time=30"
             job_alloc: "--nodes=1 --reservation=ci"
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/dane.yml'

Step 4: Update Custom Jobs (No Changes Required)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Your existing job definitions in ``.gitlab/jobs/<machine>.yml`` continue to work
without modification. They still extend the same templates:

.. code-block:: yaml

   # .gitlab/jobs/dane.yml (no changes needed)
   gcc-build:
     extends: .job_on_dane
     variables:
       COMPILER: "gcc"

Step 5: Split Custom Files
^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you were using the legacy ``custom-jobs-and-variables.yml`` it should be
split into two files, which are both optional:

**Create** ``.gitlab/custom-jobs.yml`` (job templates only):

.. code-block:: yaml

   # .gitlab/custom-jobs.yml
   .custom_job:
     before_script:
       - echo "Setting up environment..."

   .custom_perf:
     before_script:
       - echo "Setting up performance environment..."

**Create** ``.gitlab/custom-variables.yml`` (variables only):

.. code-block:: yaml

   # .gitlab/custom-variables.yml
   variables:
     DANE_SHARED_ALLOC: "--reservation=ci --exclusive --nodes=1 --time=30"
     DANE_JOB_ALLOC: "--reservation=ci --overlap --nodes=1"
     # ... etc

.. note::
    These files have different purposes and are used in different parts of the
    pipeline:
    * ``custom-variables.yml`` is included in the parent pipeline to define
      variables. This is simply a convenience to gather all allocations
      information in one place.
    * ``custom-jobs.yml`` is included in each child pipeline to populate the
      customization templates if needed. Gathering templates in a file avoids
      duplication across child pipelines, but the same result could be achieved
      by defining them directly in each .gitlab/jobs/<machine>.yml file.

Complete Example
----------------

See the ``examples/`` directory in the repository for complete migration examples:

* ``examples/example-gitlab-ci.yml`` - Complete main CI file using components
* ``examples/example-custom-jobs.yml`` - Optional job templates (child pipelines)
* ``examples/example-jobs-dane.yml`` - Machine-specific jobs

.. seealso::

   For full input, output, and exported template details for each component,
   see :doc:`../reference/components/index`.

.. seealso::

   Hit an error during migration? See :doc:`troubleshooting` for solutions to
   common component, input validation, and template errors.

Legacy Compatibility
--------------------

The legacy include-based approach (using ``pipelines/*.yml`` and ``utilities/*.yml``)
remains available and supported. Projects can continue using this approach if:

* Using GitLab version < 17.0
* Not ready to migrate yet
* Require features not yet available in components

Both approaches will only be maintained in parallel for this release.

==================
Testing Migration
==================

Before merging, test your migration:

1. **Create a test branch:**

   .. code-block:: bash

      git checkout -b test-component-migration

2. **Make changes** following the steps above

3. **Push and verify:**

   .. code-block:: bash

      git push origin test-component-migration

4. **Check GitLab pipeline:**

   - Verify all stages appear correctly
   - Check machine check jobs run
   - Verify child pipelines trigger
   - Check GitHub status updates appear

Getting Help
------------

* **Quick reference**: :doc:`quick-reference`
* **Component details**: :doc:`../reference/components/index`
* **Setup guide**: :doc:`setup-with-components`
* **Examples**: Check the ``examples/`` directory in the repository
* **Issues**: https://github.com/LLNL/radiuss-shared-ci/issues
