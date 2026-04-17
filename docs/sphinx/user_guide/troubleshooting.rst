.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
..
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _troubleshooting:

***************
Troubleshooting
***************

This guide covers common issues and their solutions.

====================
Component-Based Setup
====================

Component Not Found
===================

**Error:**

.. code-block:: text

   Component 'radiuss/radiuss-shared-ci/dane-pipeline' not found

**Possible Causes:**

1. Using GitLab version < 17.0
2. Component not published to CI/CD Catalog
3. Incorrect component path
4. Network/permissions issue

**Solutions:**

1. Check GitLab version at ``https://lc.llnl.gov/gitlab/help``
2. Verify component exists in catalog: ``https://lc.llnl.gov/gitlab/explore/catalog``
3. Check component path format: ``$CI_SERVER_FQDN/radiuss/radiuss-shared-ci/component-name@version``
4. Try using full path instead of ``$CI_SERVER_FQDN``

Input Validation Errors
========================

**Error:**

.. code-block:: text

   Required input 'github_project_name' not provided

**Solution:**

Add missing input to component's ``inputs:`` section:

.. code-block:: yaml

   - component: .../base-pipeline@v2026.02.2
     inputs:
       github_project_name: "my-project"  # Add this
       github_project_org: "LLNL"
       github_token: $GITHUB_STATUS_TOKEN

**Error:**

.. code-block:: text

   Input 'job_cmd' has invalid type

**Solution:**

Check that input matches expected type. For ``job_cmd``, provide a string:

.. code-block:: yaml

   inputs:
     job_cmd: $JOB_CMD              # Correct: variable reference
     job_cmd: "./script.sh"         # Correct: string
     job_cmd:                       # Incorrect: structured data
       value: "./script.sh"

Version Mismatch
================

**Issue:** Different component versions in same pipeline

**Solution:**

Use consistent version across all components:

.. code-block:: yaml

   include:
     - component: .../base-pipeline@v2026.02.2
     - component: .../utility-draft-pr-filter@v2026.02.2  # Same version

   dane-build-and-test:
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2  # Same version

====================
Pipeline Issues
====================

Pipeline Doesn't Start
======================

**Possible Causes:**

1. **Mirror not set up**: Verify your GitHub project is mirrored to LC GitLab
2. **YAML syntax errors**: Validate with the CI Lint tool at ``https://lc.llnl.gov/gitlab/<org>/<project>/-/ci/lint``
3. **Wrong project name**: Ensure ``GITHUB_PROJECT_NAME`` matches your GitHub repository name exactly

Machine Check Failures
======================

**Machine is down**: Expected — the pipeline skips that machine gracefully, no action required.

**Lorenz file missing**: The machine availability check depends on the Lorenz status file. Contact LC support if this persists.

Template Not Found in Child Pipeline
=====================================

**Error:**

.. code-block:: text

   Job extends template that doesn't exist: .job_on_dane

**Cause:**

Machine pipeline component not included in child pipeline trigger.

**Solution:**

Include machine component in trigger section, not parent pipeline:

.. code-block:: yaml

   # Parent pipeline (.gitlab-ci.yml)
   dane-build-and-test:
     trigger:
       include:
         - component: .../dane-pipeline@v2026.02.2  # Include here
         - local: '.gitlab/jobs/dane.yml'

   # Job file (.gitlab/jobs/dane.yml)
   my-job:
     extends: .job_on_dane  # Template now available

Stage Not Defined
==================

**Error:**

.. code-block:: text

   'prerequisites' is not a defined stage

**Cause:**

Components don't define stages to allow customization.

**Solution:**

Define required stages in ``.gitlab-ci.yml``:

.. code-block:: yaml

   stages:
     - prerequisites           # Required for machine checks
     - build-and-test          # Required for build/test jobs
     - performance-measurements # Required if using perf pipeline

Missing ASSOCIATED_CHILD_PIPELINE
==================================

**Error:**

.. code-block:: text

   Machine check failure status is never updated on GitHub after machine is back up.

**Cause:**

Machine check requires ``ASSOCIATED_CHILD_PIPELINE`` variable (added in v2026.02.0).

**Solution:**

Add variable to machine check job:

.. code-block:: yaml

   dane-up-check:
     extends: [.dane, .machine-check]
     variables:
       ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"  # Add this

==========================
Machine/Allocation Issues
==========================

Allocation Failures
===================

**Error:**

.. code-block:: text

   sbatch: error: Batch job submission failed

**Common Causes:**

1. Invalid allocation parameters
2. No access to specified partition/queue/reservation
3. Resource unavailable
4. Time limit exceeded

**Solutions:**

Check allocation parameters:

.. code-block:: yaml

   # SLURM (Dane, Matrix)
   inputs:
     shared_alloc: "--reservation=ci --exclusive --nodes=1 --time=30"
     job_alloc: "--reservation=ci --nodes=1"

   # Flux (Tioga, Tuolumne, Corona)
   inputs:
     shared_alloc: "--queue=pci --exclusive --nodes=1 --time-limit=30m"
     job_alloc: "--nodes=1 --begin-time=+5s"

Verify access to CI resources:

.. code-block:: bash

   # SLURM
   sinfo -p partition_name
   scontrol show reservation=ci

   # Flux
   flux resource list

Service User Issues
===================

**Issue:** Jobs fail with service user permissions

**Solutions:**

1. Verify service user configured: ``LLNL_SERVICE_USER`` variable set
2. Check service user has machine access
3. Check allocation permissions for service user

**Issue:** Working directory creation fails

**Solution:**

Service user needs write access to default build directory or set custom path:

.. code-block:: yaml

   variables:
     CUSTOM_CI_BUILD_DIR: "/usr/workspace/your_project/ci"

========================
GitHub Integration Issues
========================

Token Not Working
=================

**Issue:** GitHub status not updating

**Solutions:**

1. **Verify token scope:** Requires ``repo:status`` permission
2. **Check token validity:** Token may have expired
3. **Verify variable name:** Should be consistent in UI and accross configuration

To test token:

.. code-block:: bash

   curl -H "Authorization: token YOUR_TOKEN" \
     https://api.github.com/repos/LLNL/your-project/commits/main/status

====================
Job Execution Issues
====================

Job Command Fails
==================

**Issue:** ``JOB_CMD`` execution fails

**Debugging Steps:**

1. **Check command syntax:**

   .. code-block:: yaml

      JOB_CMD:
        value: "./scripts/build-and-test.sh"
        expand: false  # Prevents variable expansion

2. **Verify script exists and is executable:**

   .. code-block:: bash

      ls -l scripts/build-and-test.sh
      # Should show: -rwxr-xr-x

3. **Test script locally:**

   .. code-block:: bash

      ./scripts/build-and-test.sh

4. **Check script dependencies:**
   - Required modules loaded
   - Environment variables set
   - Paths correct

5. **Review job logs:** Check GitLab job output for error messages

Shared Allocation Issues
========================

**Issue:** Jobs waiting indefinitely

**Possible Causes:**

1. Shared allocation failed but jobs still queued
2. Allocation time limit too short and expired before jobs start

**Solutions:**

1. **Check allocation status in logs**
2. **Increase time limit:**

   .. code-block:: yaml

      shared_alloc: "--nodes=1 --time=60"  # Increase from 30

3. **Disable shared allocation:**

   .. code-block:: yaml

      shared_alloc: "OFF"
      job_alloc: "--exclusive --nodes=1 --time=20"

=================
Legacy Setup Issues
=================

Include File Not Found
=======================

**Error:**

.. code-block:: text

   Include file 'pipelines/dane.yml' not found

**Solutions:**

1. Check ``ref:`` points to valid version tag
2. Verify file path correct (``pipelines/`` not ``pipeline/``)

====================
Common Error Patterns
====================

This section lists error messages and their typical causes.

GitLab YAML Errors
==================

**Error:** ``jobs:job_name config should implement a script: or a trigger: keyword``

**Cause:** Job missing required execution method.

**Solution:** Add ``extends:`` or define ``script:``/``trigger:``

.. code-block:: yaml

   my-job:
     extends: .job_on_dane  # Provides script from template

**Error:** ``Included file could not be parsed``

**Cause:** YAML syntax error in included file.

**Solution:** Check YAML formatting:

- Consistent indentation (spaces, not tabs)
- Proper quoting of special characters
- Valid structure

**Error:** ``This GitLab CI configuration is invalid``

**Cause:** Various YAML validation errors.

**Solution:** Check CI Lint: ``https://lc.llnl.gov/gitlab/<org>/<project>/-/ci/lint``

Variable Errors
===============

**Error:** ``variable is not defined``

**Cause:** Referenced variable not set.

**Solution:** Define variable in ``.gitlab-ci.yml`` or custom-variables.yml:

.. code-block:: yaml

   variables:
     MY_VARIABLE: "value"

**Issue:** Variable shows as literal ``$VAR`` in job

**Cause:** Variable not available in child pipeline context.

**Solution:** Pass explicitly through component inputs or job variables.

**Issue:** Variable expands too early

**Cause:** ``expand: true`` (default) expands in parent pipeline.

**Solution:** Use ``expand: false`` for late expansion:

.. code-block:: yaml

   variables:
     JOB_CMD:
       value: "./script.sh"
       expand: false

Machine-Specific Patterns
==========================

**Dane/Matrix (SLURM):**

.. code-block:: text

   sbatch: error: invalid partition specified
   sbatch: error: invalid account name specified
   sbatch: error: invalid reservation name specified

**Solution:** Check SLURM allocation parameters, verify access to resources.

**Tioga/Tuolumne/Corona (Flux):**

.. code-block:: text

   flux-submit: Invalid --queue specified
   flux-submit: Unknown option --time

**Solution:** Use Flux syntax: ``--time-limit=`` not ``--time=``

===================
Getting More Help
===================

If your issue isn't covered here:

1. **Check logs:** Review complete GitLab job output
2. **Check examples:** See ``examples/`` directory in repository
3. **Search issues:** https://github.com/LLNL/radiuss-shared-ci/issues
4. **Ask for help:** Open new issue with:
   - GitLab version
   - RADIUSS Shared CI version
   - Minimal reproduction example
   - Complete error message
   - Relevant configuration files

See Also
========

- :doc:`../getting-started/five-minute-setup` - Quick start
- :doc:`setup-with-components` - Component setup guide
- :doc:`components_migration` - Migration guide
- :doc:`../reference/components/index` - Component reference
