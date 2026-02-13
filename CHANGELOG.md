# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a
Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

Added RSCI_ environment variable prefix in for internal variables that come from component inputs to prevent collisions with GitLab globals.
Note that component input names are unchanged.

Switch performance pipeline to batch executor for automatic job cancellation.

## [v2025.12.1 - 2026-01-12 ]

Patch release:

In the performance pipeline the input values used in rules had to be protected
with quotes.

## [v2025.12.0 - 2025-12-16 ]

Major Migration: GitLab CI Components Architecture (v2025.12.0).

Key Transformation:
The repository has been completely restructured from a traditional
include-based CI configuration to GitLab CI Components (requires GitLab 17.0+),
representing a fundamental modernization of the CI infrastructure.

What's New:

1. GitLab CI Components Catalog (47 commits)
- Added catalog-info.yml defining 9 reusable components
- Creates a browsable CI/CD catalog in GitLab's interface
- Versioned components with type-safe inputs

2. Component Templates Added (~2,580 lines of new code)
- Base Pipeline (templates/base-pipeline/) - Main orchestration component
- Machine-Specific Pipelines (7 components):
  - corona-pipeline, tioga-pipeline, tuolumne-pipeline (Flux scheduler)
  - dane-pipeline, matrix-pipeline (SLURM scheduler)
  - lassen-pipeline (LSF scheduler)
- Performance Pipeline (templates/performance-pipeline/) - 338 lines
- Utility Components:
  - utility-draft-pr-filter - Skip CI on draft PRs
  - utility-branch-skip - Skip CI on non-PR branches

3. Documentation & Examples
- New migration guide: docs/sphinx/user_guide/components_migration.rst (380 lines)
- Example configurations:
  - examples/example-gitlab-ci.yml - Complete component usage
  - examples/example-custom-jobs.yml - Custom job definitions
  - examples/example-jobs-lassen.yml - Machine-specific examples

Benefits of the New Architecture:

- Better versioning - @v2025.12.0 syntax instead of ref:
- Type safety - Validated component inputs
- Cleaner syntax - component: instead of include: project:
- No duplication - Reusable, parameterized templates
- Discoverable - Browse components in GitLab CI/CD Catalog

## [v2025.09.1 - 2025-10-01 ]

- Improved handling of curl commands errors.
- Add support for matrix machine.

## [v2025.09.0 - 2025-09-16 ]

- Add performance pipeline for any machine.
- Improve mechanism to turn off CI on a given machine by variable (e.g. during outage).
- Add possibility to not share allocation by setting corresponding variable to OFF.

## [v2025.06.0 - 2025-07-16 ]

- Add pipelines for Dane and Tuolumne.
- Remove Poodle (not supposed to be used for CI).

## [v2024.12.0 - 2024-12-19 ]

- Add new utility that prevent to run CI on branches that are not PRs.
- Document sharing jobs (e.g. GH llnl/radiuss-spack-configs/gitlab/radiuss-jobs).
- Point at GitLab documentation for mirroring setup.
- Improve support for complex commands passed to scheduler using xargs.

## [v2024.07.0 - 2024-07-15 ]

- Fix documentation template.
- Do not report child pipeline statuses to github (now handled by GitLab directly).

## [v2024.06.0 - 2024-06-07 ]

- Releasing an allocation will not fail anymore when the allocation did not
  exist. This happened when restarting a single allocation (no shared alloc)
  or when the allocation completed (job timeout).
- Comment and improve the templates.

## [v2024.04.0 - 2024-04-17 ]

- Add changes required by LC to set ID tokens appropriately.
- Suggest use of dedicated CI queues.
- Do not suggest specifying allocation duration in sub-allocations.

## [v2023.12.3 - 2024-03-06 ]

- Fix draft PR filter mechanism: override failures with a success status when
  PR is not a draft anymore, support pipelines running on tags.

## [v2023.12.2 - 2024-02-12 ]

- Fix reproducer logic: apply anchors to references migration everywhere

## [v2023.12.1 - 2023-12-19 ]

- Fix reproducer logic: move from YAML anchors to YAML references

## [v2023.12.0 - 2023-12-11 ]

- Update flux commands to allow controlled overlapping and MPI tests
- BREAKING: Update reproducer logic to allow for customization. The reproducer
  was tied to the "classic" RADIUSS workflow, implying Spack. Now, projects
  can customize the reproducer so that it better reflect what they actually
  run. In particular, there is no way in GitLab to automatically capture the
  custom variable your JOB_CMD requires, .<machine_reproducer_vars> is meant to
  remedy this.

## [v2023.10.0 - never-released ]

- Add support for poodle

## [v2023.09.0 - 2023-10-02 ]

- BREAKING: Shared jobs have moved to RADIUSS Spack Configs.
- BREAKING: Multiple changes in the naming of files, jobs, variables.

- Adapted documentation

## [v2023.08.0 - 2023-08-22 ]

- BREAKING: Use a regex/pattern to describe the branches that should always be
  ignored by the draft pr filter job.
  -> ALWAYS_RUN_LIST is now ALWAYS_RUN_PATTERN and takes a regex for string
     comparison in bash test.typical example:
     `^develop$|^main$|^master$|^v[0-9.]*$|^releases/`
     This will always run the CI for references "develop" "main" "master" and
     release tags or branches.

- Deprecated: flux mini run -> flux run
- Update rocm to 5.6.0 on Tioga and Corona
- Update cce to 16.0.0 on Tioga
- Updates gcc to 10.3.1 when using intel 19.1.2
- Remove unused xl 2022 combined with cuda 11.7
- Switch to IBM clang 12 in one of the lassen jobs

## [v2023.06.0 - 2023-06-23]

- Update rocm to 5.5.1 on Tioga
- Update rocm to 5.5.0 on Corona

## [v2023.03.1 - 2023-05-25]

- Improved documentation
- Update to RHEL 8 on Ruby
  - Replace gcc 8.3.1 with gcc 8.5.0 or remove it.
  - Update gcc to 10.3.1
  - Deactivate oneapi

## [v2022.03.0 - 2023-04-12]

### Added

- Support module specifications in the CI:
  -> Update you build_and_test script to load modules in $MODULE_LIST.

### Changed

- BREAKING: Set the build-and-test script command with a variable.
  -> Specify the build and test command in your .gitlab-ci.yml (see customization/gitlab-ci.yml).
- BREAKING: Machine checks jobs moved to parent pipeline:
  -> Update your .gitlab/subscribed-pipelines.yml w.r.t. customization/subscribed-pipelines.yml.
  -> Add the "machine-checks" stage to your .gitlab-ci.yml (see customization/gitlab-ci.yml).
- BREAKING: Updated shared specs with current toolchains
  -> Update your extra jobs: local overrides will be outdated.

### Fixed

- Allow selected branches to skip the draft test: allow to run CI on develop.
- Restrict tioga CI to tioga11 runner, tioga10 has issues.
- Shorter working dir path in reproducer prevents issues when using local spack.

## [v2022.12.0 - 2022-12-14]

### Added

- Tioga machine.
- A job that tests whether the machine is up, and reports the failure to GitHub otherwise:
  machine is therefore skipped when down. (Assumes oslic is always up).
- Print a complete reproducer of the job.

### Changed

- Speed-up CI by not cloning the project in jobs only reporting to GitHub.
- Improved documentation, fixed links and syntax.
- BREAKING: now require to specify GitHub project name and organization for status reports.
- BREAKING: now require spack@v0.19.0, propagate flags using "==" syntax.

### Fixed

## [v2022.09.0 - 2022-09-09]

### Added

- New HowTo articles in documentation
- build-and-test file for Tioga

### Changed

- Modified file layout to work around issue with variable override
- Update xl compiler
- Update cuda is some xl specs

### Fixed

- Coherent job naming

## [v2022.08.1 - 2022-08-xx]

### Added

- Documentation regarding known issues.
- Report status for each sub-pipeline.
- Project can now specify global (custom) variants and dependencies.
- A How-To about global variants and dependencies.
- A How-To about choosing a spack reference in Uberenv.

### Changed

- Slurm allocation now uses command line option on project side (--overlap)
  instead of environment variable (SLURM_OVERLAP) in shared CI.
- Corona now uses Flux scheduler.

### Fix

- Wrong defaults in custom-variables file.
- Sub-pipeline status report works around Gitlab issue 216629.
- Corona used to include ruby allocation options.

## [v2022.08.0 - 2022-08-03]

### Changed

- Use protected double quotes in Spack specs, see https://github.com/LLNL/uberenv/pull/84.
- Dropped intel 18 compiler.

## [v2022.06.0 - 2022-06-13]

### Added

- Provide configuration for ruby, lassen and corona.

### Documentation

- Provide a first version of the documentation with user and developer guides.
