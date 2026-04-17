.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

**********************
Component Reference
**********************

Complete reference documentation for all RADIUSS Shared CI components.

===================
Component Overview
===================

RADIUSS Shared CI provides 9 reusable components organized into four categories:

**Core Orchestration:**
  - :doc:`base-pipeline` - Required for all pipelines

**Machine-Specific:**
  - :doc:`machine-pipelines` - Build/test on LC machines (5 components)

**Performance Testing:**
  - :doc:`performance-pipeline` - Benchmarking and GitHub reporting

**Utilities:**
  - :doc:`utility-components` - Draft PR filter, branch skip (2 components)

========================
How to Use This Reference
========================

Each component page includes:

- **Purpose** - What the component does and when to use it
- **Inputs** - Complete table of all inputs (required/optional)
- **Exported Templates** - Job templates you can extend
- **Variables** - Environment variables set by the component
- **Examples** - Working code snippets
- **Common Issues** - Troubleshooting tips

===================
Component Pages
===================

.. toctree::
   :maxdepth: 2

   base-pipeline
   machine-pipelines
   performance-pipeline
   utility-components

========================
Quick Input Reference
========================

Common Inputs Across Components
================================

Many components share these inputs:

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Input
     - Type
     - Used By
   * - ``github_project_name``
     - string
     - base-pipeline, all machine pipelines, utilities
   * - ``github_project_org``
     - string
     - base-pipeline, all machine pipelines, utilities
   * - ``github_token``
     - string
     - base-pipeline, utilities, performance-pipeline
   * - ``job_cmd``
     - string
     - All machine pipelines, performance-pipeline
   * - ``llnl_service_user``
     - string
     - All machine pipelines

See individual component pages for complete input specifications.

=======================
Version Compatibility
=======================

All components in a pipeline should use the **same version**:

.. code-block:: yaml

   # Good: All @v2026.02.2
   include:
     - component: .../base-pipeline@v2026.02.2
     - component: .../dane-pipeline@v2026.02.2
     - component: .../utility-draft-pr-filter@v2026.02.2

   # Bad: Mixed versions
   include:
     - component: .../base-pipeline@v2026.02.2
     - component: .../dane-pipeline@v2025.12.0  # Don't mix!

**Reason:** Components share template names and variable conventions. Mixing
versions can cause template conflicts or missing definitions.

======================
See Also
======================

- :doc:`../../user_guide/quick-reference` - Quick lookup for common patterns
- :doc:`../../getting_started/five-minute-setup` - Get started quickly
- :doc:`../../user_guide/components_migration` - Migrate from legacy setup
