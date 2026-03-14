Migrating to GitLab CI Components
===================================

Starting with v2025.12.0, RADIUSS Shared CI provides GitLab CI Components as the
recommended way to consume shared CI configuration. This guide will help you migrate
from the legacy include-based approach to the new component-based architecture.

Prerequisites
-------------

* GitLab 17.0 or later
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

   # .gitlab/jobs/lassen.yml (no changes needed)
   gcc-build:
     extends: .job_on_lassen
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
     LASSEN_JOB_ALLOC: "1 -W 30"
     DANE_SHARED_ALLOC: "-N 1 -p pdebug -t 30"
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
* ``examples/example-custom-jobs.yml`` - Optinoal job templates (child pipelines)
* ``examples/example-jobs-lassen.yml`` - Machine-specific jobs

Component Reference
-------------------

Base Pipeline Component
^^^^^^^^^^^^^^^^^^^^^^^

**Component:** ``base-pipeline``

Provides orchestration templates for multi-machine pipelines.

.. important::
   This component does NOT define stages. You must define the required stages
   in your ``.gitlab-ci.yml`` to allow customization:

   .. code-block:: yaml

      stages:
        - prerequisites
        - build-and-test
        - performance-measurements
        # Add your custom stages here

**Inputs:**

* ``github_project_name`` (required) - GitHub project name
* ``github_project_org`` (required) - GitHub organization
* ``github_token`` (optional) - GitHub API token for status reporting

**Exported Templates:**

* ``.machine-check`` - Machine availability verification
* ``.build-and-test`` - Child pipeline trigger template
* ``.custom_job`` - Base template for job customization
* ``.custom_perf`` - Base template for performance jobs

Machine Pipeline Components
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Components:**
``lassen-pipeline``, ``dane-pipeline``, ``matrix-pipeline``,
``corona-pipeline``, ``tioga-pipeline``, ``tuolumne-pipeline``

Each provides machine-specific CI templates.

**Common Inputs:**

* ``job_cmd`` (required) - Build and test command
* ``github_project_name`` (required) - GitHub project name
* ``github_project_org`` (required) - GitHub organization
* ``llnl_service_user`` (optional) - LLNL service account

**LSF-specific (Lassen):**

* ``job_alloc`` - LSF allocation arguments (e.g., "1 -W 30")

**SLURM/ Flux-specific (Dane, Matrix, Corona, Tioga, Tuolumne):**

* ``shared_alloc`` - Shared allocation args or "OFF"
* ``job_alloc`` - Per-job allocation args
* ``alloc_name`` - Name for shared allocation

**Exported Templates:**

* ``.custom_jobs`` - Customization template for every jobs
* ``.job_on_<machine>`` - Main job template for the machine
* ``.on_<machine>`` - Machine-specific rules
* ``.<machine>_reproducer_init`` - Reproducer initialization
* ``.<machine>_reproducer_vars`` - Reproducer variables (overridable)
* ``.<machine>_reproducer_job`` - Reproducer job command

Utility Components
^^^^^^^^^^^^^^^^^^

**utility-draft-pr-filter**

Skips CI on draft pull requests.

**Inputs:**

* ``github_token`` (required)
* ``github_project_name`` (required)
* ``github_project_org`` (required)
* ``always_run_pattern`` (optional) - Regex for branches that always run

**utility-branch-skip**

Skips CI on branches not associated with a PR.

**Inputs:** Same as draft-pr-filter

Performance Pipeline Component
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Component:** ``performance-pipeline``

Provides templates for performance measurements across machines.

**Inputs:**

* ``job_cmd`` - Performance measurement command
* ``<machine>_perf_alloc`` (optional) - Per job allocation args, specify for the machine(s) you want to use
* ``perf_processing_cmd`` - Results processing command
* ``perf_artifact_dir`` (optional) - Artifacts directory
* ``perf_results_file`` (optional) - Name of raw results file
* ``perf_processed_file`` (optional) - Name of processed results file
* ``github_token`` (optional) - For GitHub reporting
* ``github_project_name`` (optional)
* ``github_project_org`` (optional)

**Exported Templates:**

* ``.custom_perf`` - Customization template for every perf jobs
* ``.perf_on_<machine>`` - Performance job for each machine
* ``.results_processing`` - Processing job template
* ``.results_reporting`` - Reporting job template
* ``.convert_to_gh_benchmark`` - GitHub benchmark conversion
* ``.report_to_gh_benchmark`` - GitHub reporting

Troubleshooting
---------------

Component Not Found
^^^^^^^^^^^^^^^^^^^

**Error:** ``Component 'radiuss/radiuss-shared-ci/lassen-pipeline' not found``

**Solution:** Ensure you're using GitLab 17.0+ and the components have been
published to the CI/CD Catalog on your GitLab instance.

Input Validation Errors
^^^^^^^^^^^^^^^^^^^^^^^

**Error:** ``Required input 'github_project_name' not provided``

**Solution:** Check that all required inputs are specified in the ``inputs:`` section
of your component include.

Template Not Found in Child Pipeline
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Error:** ``Job extends template that doesn't exist: .job_on_lassen``

**Solution:** Ensure the machine pipeline component is included in the child pipeline's
``trigger: include:`` section, not in the parent pipeline.

Legacy Compatibility
--------------------

The legacy include-based approach (using ``pipelines/*.yml`` and ``utilities/*.yml``)
remains available and supported. Projects can continue using this approach if:

* Using GitLab version < 17.0
* Not ready to migrate yet
* Require features not yet available in components

Both approaches will only be maintained in parallel for this release.

Getting Help
------------

* Check the `examples/` directory for working configurations
* Review component documentation in the GitLab CI/CD Catalog
* Consult the main documentation at https://radiuss-shared-ci.readthedocs.io
* Open an issue at https://github.com/LLNL/radiuss-shared-ci/issues
