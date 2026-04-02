.. ##
.. ## Copyright (c) 2022-2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _dev_common_issues-label:

*****************************
Developer Troubleshooting
*****************************

This guide covers issues developers may encounter when working on RADIUSS Shared CI.

==========================
Component Development
==========================

Component Validation Errors
============================

**Error:** ``Component has invalid spec``

**When this occurs:** When publishing component to catalog (at tag push time)

**Causes:**

- Invalid input type
- Missing required spec fields
- YAML syntax errors in spec section

**Solution:**

Validate component structure:

.. code-block:: yaml

   # templates/component-name/template.yml
   spec:
     inputs:
       input_name:
         type: string          # Valid: string, number, boolean
         description: "..."    # Required
         default: "value"      # Optional
   ---
   # Component implementation below

**Error:** ``Required input 'input_name' not provided``

**When this occurs:** When user includes component in their pipeline (before pipeline execution)

**Cause:**

User didn't provide required input or provided wrong type.

**Solution:**

Check component usage matches spec. GitLab validates inputs against component spec before running pipeline.

Component Not in Catalog
=========================

**Issue:** Component doesn't appear in CI/CD Catalog after tagging

**Checklist:**

1. Tag was synced to GitLab (after being pushed to GitHub):
2. Repository has catalog metadata in README
3. Component ``template.yml`` is valid
4. GitLab 17.0+ instance

**Solution:**

Check catalog status at: ``https://lc.llnl.gov/gitlab/explore/catalog``

If missing, verify tag exists on LC GitLab and re-push if needed.

================================
Repository and Mirroring
================================

Branch Not Mirrored to LC GitLab
=================================

**Error:** ``Project 'radiuss/radiuss-shared-ci' reference 'feature-branch' does not exist``

.. image:: images/error-mirrored-reference.png
   :scale: 45 %
   :alt: Reference not found on LC GitLab
   :align: center

**Cause:**

Development branch does not exist on GitHub or hasn't been mirrored to LC GitLab yet.

**Solution:**

1. **Push branch on GitHub**
2. **Wait for mirror sync** (usually a few minutes), alternatively trigger manual sync if you have access
3. **Verify on LC GitLab:** ``https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/branches``

Mirror Sync Delays
==================

**Issue:** Changes not appearing in LC GitLab

**Solution:**

Check mirror status and force sync if needed (requires maintainer access).

Typical sync time: 5-30 minutes after GitHub push. If delays persist, contact maintainers to check mirror health (mirroring can be paused after inactivity).

==================
Legacy Compatibility
==================

Legacy Include Not Found
=========================

**Error:** ``Include file 'pipelines/dane.yml' not found``

**Causes:**

1. Invalid ``ref:`` value
2. File path typo
3. Version doesn't exist on LC GitLab

**Solution:**

Check file exists at ref:

``https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/blob/<ref>/pipelines/dane.yml``

Empty Custom Directory Error
=============================

**Error:** ``Invalid character \x1b looking for beginning of value``

.. image:: images/error-empty-custom-dir.png
   :scale: 30 %
   :alt: Empty CUSTOM_CI_BUILDS_DIR error
   :align: center

**Cause:**

``CUSTOM_CI_BUILDS_DIR`` set to empty string in ``.gitlab/custom-jobs-and-variables.yml``.

**Solution:**

Comment out the line or provide valid path:

.. code-block:: yaml

   variables:
     # CUSTOM_CI_BUILDS_DIR: ""              # Comment out
     CUSTOM_CI_BUILDS_DIR: "/path/to/dir"    # Or set valid path

========================
Getting Help
========================

If issue isn't covered here:

1. **Check user troubleshooting:** :doc:`../user_guide/troubleshooting`
2. **Search existing issues:** https://github.com/LLNL/radiuss-shared-ci/issues
3. **Ask in PR:** If working on contribution, ask in pull request
4. **Open new issue:** Provide:

   - Development environment (GitLab version, branch)
   - Component or feature being developed
   - Error messages and logs
   - Steps to reproduce

==================
See Also
==================

- :doc:`how_to` - Developer how-to guide
- :doc:`radiuss_ci_explained` - Architecture overview
- :doc:`../user_guide/troubleshooting` - User troubleshooting
