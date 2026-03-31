.. ##
.. ## Copyright (c) 2022-2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _setup_ci:

************
Setup the CI
************

Choose your setup path based on your GitLab version and needs.

=================
Component-Based
=================

**Recommended for GitLab 17.0+**

Use GitLab CI Components for type-safe, versioned CI configuration.

→ **Continue to** :doc:`setup-with-components`

**Benefits:**

- Type-checked inputs
- Clear versioning (@v2026.02.2)
- Self-documenting in GitLab catalog
- No template file copying

**Requirements:**

- GitLab 17.0 or later

==================
Legacy (Include-Based)
==================

**For GitLab < 17.0 or gradual migration**

Use the traditional include-based approach with project/file references.

→ **Continue to** :doc:`setup-legacy`

**When to use:**

- GitLab version < 17.0
- Not ready to migrate existing setup
- Need features not yet in components

.. note::
   Both approaches are currently supported, but components are recommended
   for new projects on GitLab 17.0+.

================
Quick Start
================

**New to RADIUSS Shared CI?**

Start with the quick start guide for fastest setup:

→ :doc:`../getting-started/five-minute-setup`

Then customize using the full setup guides above.

================
Need Help Choosing?
================

See :doc:`../getting-started/choosing-your-path` for guidance on:

- Which machines to enable
- Whether to use components or legacy approach
- When to add performance testing
- Utility component decisions
