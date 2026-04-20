.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _contributing:

************
Contributing
************

This guide covers how to contribute to RADIUSS Shared CI.

===================
Getting Started
===================

Fork and Clone
==============

1. **Fork repository** on GitHub: https://github.com/LLNL/radiuss-shared-ci
2. **Clone your fork:**

   .. code-block:: bash

      git clone https://github.com/YOUR_USERNAME/radiuss-shared-ci.git
      cd radiuss-shared-ci

3. **Add upstream remote:**

   .. code-block:: bash

      git remote add upstream https://github.com/LLNL/radiuss-shared-ci.git

Development Workflow
====================

1. **Create feature branch** from ``main``:

   .. code-block:: bash

      git checkout main
      git pull upstream main
      git checkout -b feature/my-feature

2. **Make changes** with clear commits
3. **Test changes** (see Testing section below)
4. **Push to your fork:**

   .. code-block:: bash

      git push origin feature/my-feature

5. **Open pull request** on GitHub

====================
Testing Changes
====================

Component Testing
=================

Test component changes in actual project:

1. **Push branch to GitHub:**

   .. code-block:: bash

      git push origin feature/my-feature

2. **Wait for mirror sync** to LC GitLab (5-30 minutes)

3. **Update test project** to use development branch:

   .. code-block:: yaml

      include:
        - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@feature/my-feature
          inputs:
            job_cmd: "./test.sh"
            shared_alloc: "--nodes=1"
            job_alloc: "--nodes=1"

4. **Run full pipeline** and verify:

   - All machines work correctly
   - Jobs execute successfully
   - GitHub status updates appear
   - Reproducer instructions print correctly

Documentation Testing
=====================

Build documentation locally:

.. code-block:: bash

   cd docs

   # Create virtual environment
   python3 -m venv venv
   source venv/bin/activate

   # Install dependencies
   pip install -r requirements.txt

   # Build documentation
   sphinx-build -b html sphinx _build/html

   # Open in browser
   open _build/html/index.html  # macOS
   # Or: xdg-open _build/html/index.html  # Linux

   deactivate

Check for:

- Sphinx warnings in build output
- Broken cross-references
- Proper formatting and rendering
- Valid code examples

Legacy Testing
==============

If changes affect legacy implementation, test with include-based setup:

.. code-block:: yaml

   include:
     - project: 'radiuss/radiuss-shared-ci'
       ref: 'feature/my-feature'
       file: 'pipelines/dane.yml'

===================
Code Guidelines
===================

YAML Style
==========

- Use 2-space indentation
- Consistent naming: ``job_on_<machine>``, ``ON_<MACHINE>``
- Document component inputs with descriptions
- Add comments for complex logic

Component Structure
===================

Components should follow this structure:

.. code-block:: yaml

   # templates/component-name/template.yml
   spec:
     inputs:
       input_name:
         type: string
         description: "Clear description of what this input does"
         default: "value"  # Optional
   ---
   # Component implementation
   .template_name:
     # Template definition

Documentation Style
===================

- Use clear, concise language
- Provide working code examples
- Use cross-references: ``:doc:`file``` and ``:ref:`label```
- Follow existing documentation structure

=====================
Pull Request Process
=====================

Before Submitting
=================

- [ ] Changes tested in actual project
- [ ] Documentation updated if needed
- [ ] CHANGELOG.md updated for user-facing changes
- [ ] Code follows style guidelines

Submitting PR
=============

1. **Write clear PR description:**

   - What problem does this solve?
   - What changes were made?
   - How was it tested?
   - Any breaking changes?

2. **Link related issues** if applicable

3. **Request review** from maintainers

Review Process
==============

Maintainers will:

- Review code and documentation
- Test changes if needed
- Provide feedback
- Approve when ready

Address feedback by:

- Making requested changes
- Pushing to same branch (PR updates automatically)
- Responding to comments

Merging
=======

Maintainers will merge when:

- All feedback addressed
- Tests pass
- Documentation complete
- At least one approval

=============================
Component Catalog and README
=============================

The repository ``README.md`` serves as the catalog entry for GitLab CI/CD Catalog.

README Structure (GitLab Guidelines)
=====================================

Following GitLab's recommendations, the README should include:

**Summary**
  Start with summary of capabilities that components provide

**Table of Contents** (for multiple components)
  Help users quickly jump to specific component details

**## Components Section**
  With subsections like ``### Component Name`` for each component

  For each component:

  - Describe what the component does
  - Add at least one YAML example showing usage
  - Link to published component (inputs appear automatically on component page)
  - **Do not duplicate input documentation** - use ``spec:inputs:description`` in component

**## Contribute Section**
  Include if contributions are welcome

**Additional Documentation**
  For complex components, create Markdown file in component directory and link from README

When Adding Components
======================

Add new component to README with:

1. **Subsection** under ``## Components``
2. **Clear description** of what it does
3. **YAML usage example**
4. **Link to published component** in catalog
5. **Do not list inputs** (they appear automatically)

Example structure:

.. code-block:: markdown

   ### NewMachine Pipeline

   Provides CI templates for NewMachine (SLURM scheduler, Intel CPUs).

   **Usage:**

   ```yaml
   include:
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/newmachine-pipeline@v2026.02.2
       inputs:
         job_cmd: "./build-and-test.sh"
   ```

   [View component inputs →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/newmachine-pipeline)

========================
Release Process
========================

Version Updates
===============

When preparing release, update version references in:

- ``docs/sphinx/conf.py`` - Set ``release`` variable
- ``README.md`` - Update example version tags and version references
- ``examples/*.yml`` - Update component versions
- ``customization/*.yml`` - Update ref versions

Changelog
=========

Update ``CHANGELOG.md`` with:

**Added**
  New features and capabilities

**Changed**
  Changes to existing functionality

**Fixed**
  Bug fixes

**Breaking Changes**
  Changes requiring user action

**Migration Notes**
  How to update from previous version

Creating Release
================

Maintainers create releases:

.. code-block:: bash

   # Tag and push
   git tag v2026.03.0
   git push origin v2026.03.0

Components automatically publish to GitLab CI/CD Catalog when tagged.

Announcement
============

After release:

1. Create GitHub release with changelog
2. Notify projects using RADIUSS Shared CI
3. Update any external documentation

==================
Getting Help
==================

Questions About Contributing
============================

- **Open discussion** in issue before major changes
- **Ask in PR** if uncertain about implementation
- **Check existing code** for examples and patterns

Reporting Issues
================

When reporting issues, include:

- Clear description of problem
- Steps to reproduce
- Expected vs actual behavior
- Relevant configuration files
- Error messages and logs
- GitLab and RADIUSS Shared CI versions

Feature Requests
================

When requesting features, describe:

- Use case and motivation
- Proposed solution (if any)
- Alternative approaches considered
- Impact on existing functionality

==================
Code of Conduct
==================

- Be respectful and professional
- Provide constructive feedback
- Focus on code and ideas, not people
- Welcome newcomers

====================
Additional Resources
====================

- :doc:`how_to` - Step-by-step procedures
- :doc:`radiuss_ci_explained` - Architecture overview
- :doc:`troubleshooting` - Common development issues
- `RADIUSS Shared CI Repository <https://github.com/LLNL/radiuss-shared-ci>`_
- `GitLab CI Components <https://docs.gitlab.com/ee/ci/components/>`_
