.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

*********
Reference
*********

This section provides detailed reference documentation for RADIUSS Shared CI
components, inputs, outputs, and technical specifications.

=================
Component Catalog
=================

RADIUSS Shared CI provides reusable GitLab CI Components for building
multi-machine CI pipelines. Each component is versioned, type-checked, and
documented.

Browse Components
=================

.. toctree::
   :maxdepth: 2

   components/index
   glossary

==================
Quick Component List
==================

**Core Components:**

- :doc:`components/base-pipeline` - Orchestration and GitHub integration
- :doc:`components/machine-pipelines` - Machine-specific CI (5 machines)
- :doc:`components/performance-pipeline` - Performance testing and reporting
- :doc:`components/utility-components` - Draft PR filter, branch skip

**Machine Components:**

- dane-pipeline (SLURM/Intel Sapphire Rapids)
- matrix-pipeline (SLURM/Intel Sapphire Rapids/NVIDIA H100)
- corona-pipeline (Flux/AMD Rome/AMD MI50)
- tioga-pipeline (Flux/AMD Trento/AMD MI250X)
- tuolumne-pipeline (Flux/AMD EPYC/AMD MI300A)

See :doc:`components/machine-pipelines` for details.

===================
Usage Pattern
===================

All components follow this usage pattern:

.. code-block:: yaml

   include:
     - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/<name>@<version>
       inputs:
         required_input: "value"
         optional_input: "value"

Components export templates that your jobs can extend:

.. code-block:: yaml

   my-job:
     extends: .<template-name>
     variables:
       CUSTOM_VAR: "value"

====================
Version Information
====================

**Current Version:** v2026.02.2

**Version History:**

- v2026.02.2 - Performance pipeline: add commit hash to exported data
- v2026.02.1 - Add RSCI_ prefix to internal variables
- v2026.02.0 - Machine checks use same GitHub context as child pipelines
- v2025.12.1 - Performance pipeline rule fixes
- v2025.12.0 - Initial component release

See :doc:`../CHANGELOG` for complete version history.

==================
For New Users
==================

If you're new to RADIUSS Shared CI components:

1. **Start here**: :doc:`../getting_started/index`
2. **Understand concepts**: :doc:`../user_guide/concepts`
3. **Quick reference**: :doc:`../user_guide/quick-reference`
4. **Then explore**: Component reference documentation below

==================
For Developers
==================

Contributing to RADIUSS Shared CI components:

- **Architecture**: :doc:`../dev_guide/radiuss_ci_explained`
- **Development**: :doc:`../dev_guide/how_to`
- **Troubleshooting**: :doc:`../dev_guide/troubleshooting`
