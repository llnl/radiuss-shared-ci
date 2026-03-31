.. ##
.. ## Copyright (c) 2026, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for
.. ## details.
.. ##
.. ## SPDX-License-Identifier: (MIT)
.. ##

.. _performance-pipeline-reference:

********************
performance-pipeline
********************

Component providing templates for running performance benchmarks across multiple
machines with result processing and optional GitHub benchmark reporting.

=======
Purpose
=======

The ``performance-pipeline`` component provides:

- **Multi-Machine Templates**: Job templates for performance tests on selected machines
- **Processing Job Template**: Template for processing raw benchmark results
- **GitHub Reporting Templates**: Optional templates for posting results to GitHub

Use this component when you have performance benchmarks and want to run them
consistently across machines.

===========
When to Use
===========

Enable performance pipeline when:

- You have performance-critical code
- You want to track performance trends over time
- You need to detect performance regressions in PRs
- You have dedicated performance benchmarks

**Consider skipping if**:

- You're just getting started with CI (add later)
- You don't have performance tests yet
- CI time is already constrained

.. tip::
   Run performance tests on 1-2 machines (not all) to balance coverage and CI time.
   Recommended: **Tioga** for AMD GPU, **Matrix** for NVIDIA GPU, or **Dane** for CPU.

======
Inputs
======

Core Inputs
===========

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Input
     - Required
     - Description
   * - ``job_cmd``
     - Yes
     - Command to run performance benchmarks
   * - ``perf_processing_cmd``
     - Yes
     - Command to process raw results into standardized format

GitHub Reporting (Optional)
============================

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Input
     - Required
     - Description
   * - ``github_token``
     - No
     - GitHub token with ``repo`` and ``workflow`` scopes
   * - ``github_project_name``
     - No
     - GitHub repository name (for reporting)
   * - ``github_project_org``
     - No
     - GitHub organization name (for reporting)

Machine-Specific Allocations
=============================

Specify allocation for each machine you want to test on:

.. list-table::
   :header-rows: 1
   :widths: 30 15 55

   * - Input
     - Required
     - Description
   * - ``dane_perf_alloc``
     - No
     - Dane allocation args (if omitted, no Dane perf jobs)
   * - ``matrix_perf_alloc``
     - No
     - Matrix allocation args
   * - ``corona_perf_alloc``
     - No
     - Corona allocation args
   * - ``tioga_perf_alloc``
     - No
     - Tioga allocation args
   * - ``tuolumne_perf_alloc``
     - No
     - Tuolumne allocation args

.. note::
   Only machines with ``*_perf_alloc`` specified will run performance jobs.
   This allows selective testing (e.g., only Tioga and Matrix).

Artifact Configuration
======================

.. list-table::
   :header-rows: 1
   :widths: 30 15 55

   * - Input
     - Required
     - Description
   * - ``perf_artifact_dir``
     - No
     - Directory for artifacts (default: ".")
   * - ``perf_results_file``
     - No
     - Raw results filename (default: "results.txt")
   * - ``perf_processed_file``
     - No
     - Processed results filename (default: "processed.json")

==================
Exported Templates
==================

.custom_perf
============

**Purpose**: Base customization template for all performance jobs.

**Usage**: Override in ``.gitlab/custom-jobs.yml`` for project specific perf setup.

.. code-block:: yaml

   # In .gitlab/custom-jobs.yml
   .custom_perf:
     before_script:
       - echo "Performance environment setup"
       - export OMP_NUM_THREADS=1
       - export CUDA_VISIBLE_DEVICES=0

.perf_on_<machine>
==================

**Purpose**: Performance job template for each machine.

**Usage**: Extend this template to create performance jobs.

.. code-block:: yaml

   # In .gitlab/jobs/performances.yml
   tioga-perf:
     extends: .perf_on_tioga
     variables:
       BENCHMARK_CONFIG: "large"

Available templates:

- ``.perf_on_dane``
- ``.perf_on_matrix``
- ``.perf_on_corona``
- ``.perf_on_tioga``
- ``.perf_on_tuolumne``

.results_processing
===================

**Purpose**: Template for processing raw benchmark results.

**Usage**: Automatically used by the component (typically not extended directly).

Runs after all performance jobs complete to process raw results.

.results_reporting
==================

**Purpose**: Template for GitHub benchmark reporting.

**Usage**: Automatically used when GitHub inputs provided.

Reports processed results to GitHub as benchmark comments.

.convert_to_gh_benchmark
=========================

**Purpose**: Converts processed results to GitHub benchmark format.

**Usage**: Internal template (not typically extended).

.report_to_gh_benchmark
=======================

**Purpose**: Posts benchmark data to GitHub.

**Usage**: Internal template (not typically extended).

========
Workflow
========

When configured, the performance pipeline creates this job flow:

.. code-block:: text

   1. Performance Jobs (parallel across machines)
      ├─ .perf_on_dane (if dane_perf_alloc set)
      ├─ .perf_on_matrix (if matrix_perf_alloc set)
      └─ .perf_on_tioga (if tioga_perf_alloc set)
      │
      ↓ (each produces raw results)
      │
   2. Results Processing
      └─ .results_processing
         Runs: perf_processing_cmd
         Input: perf_results_file from all jobs
         Output: perf_processed_file (JSON)
      │
      ↓
      │
   3. GitHub Reporting (if token provided)
      ├─ .convert_to_gh_benchmark
      │  Converts JSON to GitHub benchmark format
      │
      └─ .report_to_gh_benchmark
         Posts results to GitHub PR/commit

=============
Data Formats
=============

The component works with user-provided data formats:

Raw Results
===========

Your ``job_cmd`` produces results in whatever format your ``perf_processing_cmd``
expects. Common formats:

- CSV
- JSON
- Custom text format

Processed Results (JSON)
=========================

Your ``perf_processing_cmd`` should output JSON in GitHub benchmark format:

.. code-block:: json

   {
     "benchmarks": [
       {
         "name": "MatrixMultiply",
         "unit": "GFLOPS",
         "value": 234.5
       },
       {
         "name": "VectorAdd",
         "unit": "ms",
         "value": 12.3
       }
     ],
     "commit_hash": "<sha>",
     "machine": "<machine-name>"
   }

.. note::
   As of v2026.02.2, the component makes ``commit_hash`` available to your
   processing script for inclusion in exported data.

========
Examples
========

Minimal Example
===============

Basic performance testing on one machine:

.. code-block:: yaml

   # .gitlab-ci.yml
   stages:
     - prerequisites
     - build-and-test
     - performance-measurements

   performance-measurements:
     extends: [.performance-measurements]
     trigger:
       include:
         - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/performance-pipeline@v2026.02.2
           inputs:
             job_cmd: "./scripts/run-benchmarks.sh"
             tioga_perf_alloc: "--nodes=1 --exclusive --time-limit=15m"
             perf_processing_cmd: "python scripts/process-results.py"
         - local: '.gitlab/jobs/performances.yml'

   # .gitlab/jobs/performances.yml
   tioga-benchmark:
     extends: .perf_on_tioga

Multi-Machine Example
======================

Performance testing on multiple machines:

.. code-block:: yaml

   performance-measurements:
     extends: [.performance-measurements]
     trigger:
       include:
         - component: .../performance-pipeline@v2026.02.2
           inputs:
             job_cmd: "./scripts/run-benchmarks.sh"
             # CPU testing
             dane_perf_alloc: "--nodes=1 --exclusive --time=15"
             # NVIDIA GPU testing
             matrix_perf_alloc: "--nodes=1 --exclusive --time=15 --gres=gpu:1"
             # AMD GPU testing
             tioga_perf_alloc: "--nodes=1 --exclusive --time-limit=15m -g 1"
             perf_processing_cmd: "python scripts/process-results.py"
             github_token: $GITHUB_TOKEN
             github_project_name: $GITHUB_PROJECT_NAME
             github_project_org: $GITHUB_PROJECT_ORG
         - local: '.gitlab/jobs/performances.yml'

   # .gitlab/jobs/performances.yml
   dane-perf:
     extends: .perf_on_dane
     variables:
       CONFIG: "cpu"

   matrix-perf:
     extends: .perf_on_matrix
     variables:
       CONFIG: "cuda"

   tioga-perf:
     extends: .perf_on_tioga
     variables:
       CONFIG: "rocm"

Conditional Execution
=====================

Run performance tests only on specific branches:

.. code-block:: yaml

   performance-measurements:
     extends: [.performance-measurements]
     rules:
       # Auto-run on main/develop
       - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"'
         when: on_success
       # Manual on PRs
       - if: '$CI_MERGE_REQUEST_ID'
         when: manual
       # Never on other branches
       - when: never
     trigger:
       include:
         - component: .../performance-pipeline@v2026.02.2
           inputs: { ... }

With Custom Artifacts
=====================

Customize artifact handling:

.. code-block:: yaml

   performance-measurements:
     trigger:
       include:
         - component: .../performance-pipeline@v2026.02.2
           inputs:
             job_cmd: "./run-benchmarks.sh"
             perf_processing_cmd: "./process-results.sh"
             perf_artifact_dir: "benchmark-results"
             perf_results_file: "raw-data.csv"
             perf_processed_file: "summary.json"
             tioga_perf_alloc: "--nodes=1 --exclusive --time-limit=30m"

===================
Script Examples
===================

Benchmark Script
================

Example ``run-benchmarks.sh``:

.. code-block:: bash

   #!/bin/bash
   set -e

   # Run your benchmarks
   ./build/benchmarks/matrix_multiply --iterations=100 > results.txt
   ./build/benchmarks/vector_add --size=1000000 >> results.txt

   echo "Benchmarks complete"

Processing Script
=================

Example ``process-results.py``:

.. code-block:: python

   #!/usr/bin/env python3
   import json
   import sys

   # Read raw results
   with open('results.txt', 'r') as f:
       lines = f.readlines()

   # Parse and convert to GitHub benchmark format
   benchmarks = []
   for line in lines:
       if 'GFLOPS' in line:
           name, value = line.split(':')
           benchmarks.append({
               'name': name.strip(),
               'unit': 'GFLOPS',
               'value': float(value.strip())
           })

   # Output JSON
   output = {'benchmarks': benchmarks}
   with open('processed.json', 'w') as f:
       json.dump(output, f, indent=2)

   print(f"Processed {len(benchmarks)} benchmarks")

==============
Common Issues
==============

Missing Machine Allocation
===========================

**Symptom**: No performance jobs run for a machine

**Solution**: Ensure machine-specific allocation input is provided:

.. code-block:: yaml

   inputs:
     tioga_perf_alloc: "--nodes=1 --exclusive --time-limit=15m"

Machines without ``*_perf_alloc`` are skipped.

Processing Command Fails
=========================

**Symptom**: Results processing job fails

**Possible causes**:

1. **Script not found**: Verify ``perf_processing_cmd`` path
2. **Wrong format**: Check output format matches expected JSON
3. **Missing dependencies**: Script may need Python/tools not loaded
4. **File not found**: Check ``perf_results_file`` name matches output

GitHub Reporting Not Working
=============================

**Symptom**: Results don't appear on GitHub

**Checklist**:

1. ``github_token`` provided and valid
2. Token has ``repo`` and ``workflow`` permissions
3. ``github_project_name`` and ``github_project_org`` correct
4. Processing produced valid JSON in correct format

Long Running Benchmarks
========================

**Symptom**: Performance jobs timeout

**Solutions**:

1. Increase ``*_perf_alloc`` time limit
2. Reduce benchmark iterations
3. Run benchmarks on fewer configurations

Result File Not Found
======================

**Error**: "results.txt not found" in processing step

**Solution**: Ensure your benchmark script produces the expected filename:

.. code-block:: yaml

   inputs:
     perf_results_file: "results.txt"  # Must match your script output

Or adjust the input to match your actual filename.

========
See Also
========

- :doc:`../getting-started/choosing-your-path` - When to use performance testing
- :doc:`../user_guide/quick-reference` - Quick performance setup reference
- :doc:`machine-pipelines` - Machine allocation details

**Related Topics**:

- GitHub benchmark format specification
- Performance regression detection
- CI time optimization strategies
