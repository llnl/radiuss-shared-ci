.. ##
.. ## Copyright (c) 2022-2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _dev_how_to-label:

*******************
Developer How-To
*******************

Step-by-step procedures for common development tasks.

.. _add-a-new-machine:

==================
Add a New Machine
==================

Complete procedure to add support for a new machine.

Step 1: Create Component
=========================

1. **Create component directory:**

   .. code-block:: bash

      mkdir -p templates/newmachine-pipeline

2. **Choose template base:**

   - For SLURM machines: Copy ``templates/dane-pipeline/template.yml``
   - For Flux machines: Copy ``templates/tioga-pipeline/template.yml``

3. **Create** ``templates/newmachine-pipeline/template.yml``:

   Replace all instances of source machine name with new machine name:

   .. code-block:: bash

      # Example: If copying from dane
      sed 's/dane/newmachine/g; s/DANE/NEWMACHINE/g' \
        templates/dane-pipeline/template.yml > \
        templates/newmachine-pipeline/template.yml

4. **Update machine-specific details:**

   - Runner tags
   - Scheduler commands (if different)
   - Default allocation parameters
   - Environment setup

Step 2: Update Base Pipeline
=============================

Add machine rules to ``templates/base-pipeline/template.yml``:

.. code-block:: yaml

   # Add after other machine rules
   .newmachine:
     rules:
       - if: '$ON_NEWMACHINE == "OFF"'
         when: never
       - when: on_success

Step 3: Create Legacy Implementation
=====================================

For backward compatibility:

1. **Create** ``pipelines/newmachine.yml`` by copying similar machine
2. **Replace** machine names throughout
3. **Test** legacy include syntax works

Step 4: Update Customization Templates
=======================================

1. **Add to** ``customization/subscribed-pipelines.yml``:

   .. code-block:: yaml

      # NewMachine
      newmachine-up-check:
        extends: [.newmachine, .machine-check]
        variables:
          ASSOCIATED_CHILD_PIPELINE: "newmachine-build-and-test"

      newmachine-build-and-test:
        needs: [newmachine-up-check]
        extends: [.newmachine, .build-and-test]
        variables:
          CI_MACHINE: "newmachine"
        trigger:
          include:
            - local: '.gitlab/custom-jobs.yml'
            - project: 'radiuss/radiuss-shared-ci'
              ref: '${SHARED_CI_REF}'
              file: 'pipelines/newmachine.yml'
            - local: '.gitlab/jobs/newmachine.yml'

2. **Add to** ``customization/custom-jobs-and-variables.yml``:

   .. code-block:: yaml

      # NewMachine allocation parameters
      NEWMACHINE_SHARED_ALLOC: "--nodes=1 --time=30"
      NEWMACHINE_JOB_ALLOC: "--nodes=1"

Step 5: Update Documentation
=============================

1. **Update** ``README.md`` **at repository root:**

   Add subsection under ``## Components``:

   .. code-block:: markdown

      ### NewMachine Pipeline

      Provides CI templates for NewMachine (SLURM, Intel CPUs).

      **Usage:**

      ```yaml
      include:
        - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/newmachine-pipeline@v2026.02.2
          inputs:
            job_cmd: "./build-and-test.sh"
      ```

      [View component inputs →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/newmachine-pipeline)

   **Do not list inputs** - they appear automatically on component page

2. **Add to** ``docs/sphinx/reference/components/machine-pipelines.rst``:

   Add section describing the new machine with:

   - Hardware description (CPU/GPU)
   - Scheduler type
   - Common allocation parameters
   - Example usage

3. **Update machine lists in:**

   - ``docs/sphinx/getting-started/choosing-your-path.rst``
   - ``docs/sphinx/user_guide/quick-reference.rst``
   - ``docs/sphinx/getting-started/five-minute-setup.rst`` (if appropriate)

4. **Add** ``ON_NEWMACHINE`` **to variable tables**

Step 6: Test Implementation
============================

1. **Commit changes to development branch:**

   .. code-block:: bash

      git checkout -b add-newmachine-support
      git add templates/newmachine-pipeline/ pipelines/newmachine.yml
      git add customization/ examples/ docs/
      git commit -m "Add NewMachine support"
      git push origin add-newmachine-support

2. **Wait for GitHub to LC GitLab mirror sync** (5-30 minutes)

3. **Test in project:**

   .. code-block:: yaml

      include:
        - component: .../newmachine-pipeline@add-newmachine-support
          inputs:
            job_cmd: "./test.sh"
            shared_alloc: "--nodes=1"
            job_alloc: "--nodes=1"

4. **Verify:**

   - Machine check executes
   - Jobs run successfully
   - Allocations work correctly
   - GitHub status reports
   - Reproducer prints correctly

Step 7: Submit Changes
=======================

1. **Update** ``CHANGELOG.md``:

   .. code-block:: markdown

      ## [Unreleased]

      ### Added
      - NewMachine support (SLURM/Flux, CPU/GPU description)

2. **Open pull request** on GitHub with:

   - Description of new machine
   - Hardware/scheduler details
   - Test results
   - Links to test pipeline runs

======================
Add a Utility Component
======================

Procedure for adding non-machine components (like draft-pr-filter).

Step 1: Create Component
=========================

1. **Create directory:**

   .. code-block:: bash

      mkdir -p templates/utility-feature-name

2. **Create** ``templates/utility-feature-name/template.yml``:

   .. code-block:: yaml

      spec:
        inputs:
          required_input:
            type: string
            description: "What this input does"
      ---
      # Component implementation
      feature-job:
        stage: prerequisites
        tags: [shell, oslic]
        script:
          - echo "Doing something with ${{ inputs.required_input }}"

Step 2: Add Documentation
==========================

1. **Add to** ``docs/sphinx/reference/components/utility-components.rst``

2. **Update** ``README.md`` **at repository root:**

   Add subsection under ``## Components`` with:

   - What the component does
   - YAML usage example
   - Link to published component (do not list inputs)

   Example:

   .. code-block:: markdown

      ### Utility: Feature Name

      Brief description of what this utility does.

      **Usage:**

      ```yaml
      include:
        - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/utility-feature-name@v2026.02.2
          inputs:
            required_input: "value"
      ```

      [View component inputs →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/utility-feature-name)

3. **Update examples** if needed

Step 3: Test and Submit
========================

1. Test in development branch
2. Update CHANGELOG.md
3. Submit pull request

==========================
Update Component Version
==========================

Procedure for making version updates.

For Breaking Changes
====================

1. **Update component spec:**

   .. code-block:: yaml

      spec:
        inputs:
          new_required_input:  # New required input is breaking
            type: string

2. **Update legacy implementation** to match

3. **Document in** ``CHANGELOG.md``:

   .. code-block:: markdown

      ### Breaking Changes
      - Component now requires `new_required_input`

      ### Migration
      ```yaml
      inputs:
        new_required_input: "value"  # Add this
      ```

4. **Update all examples** and documentation

For Non-Breaking Changes
=========================

1. **Add optional inputs** (with defaults):

   .. code-block:: yaml

      spec:
        inputs:
          new_optional_input:
            type: string
            default: "default_value"

2. **Update** ``CHANGELOG.md`` **under Added or Changed**

3. **Update examples** to show new functionality

================
Update Examples
================

When updating example files for new version.

Procedure
=========

1. **Update version references** in:

   - ``examples/example-gitlab-ci.yml``
   - ``examples/example-custom-jobs.yml``
   - ``examples/example-jobs-*.yml``

2. **Find and replace:**

   .. code-block:: bash

      cd examples/
      sed -i 's/@v2026.02.2/@v2026.03.0/g' *.yml

3. **Update** ``customization/*.yml`` **ref values:**

   .. code-block:: bash

      cd customization/
      sed -i 's/v2026.02.2/v2026.03.0/g' *.yml

4. **Verify changes** and test

=====================
Build Documentation
=====================

Procedure to build docs locally.

Setup
=====

.. code-block:: bash

   cd docs

   # Create virtual environment (one time)
   python3 -m venv venv
   source venv/bin/activate

   # Install dependencies
   pip install -r requirements.txt

Build
=====

.. code-block:: bash

   # Build documentation
   sphinx-build -b html sphinx _build/html

   # Open in browser
   open _build/html/index.html

Rebuild After Changes
=====================

.. code-block:: bash

   # Clean previous build
   rm -rf _build

   # Rebuild
   sphinx-build -b html sphinx _build/html

Check for Warnings
==================

Look for Sphinx warnings during build:

- Broken cross-references
- Missing files in toctree
- Invalid RST syntax

Fix warnings before submitting changes.

===============
Troubleshooting
===============

For development issues, see :doc:`troubleshooting`.

==================
Additional Resources
==================

- :doc:`radiuss_ci_explained` - Architecture and design
- :doc:`contributing` - Contribution guidelines
- :doc:`troubleshooting` - Common development issues
