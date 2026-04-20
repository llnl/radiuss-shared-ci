# RADIUSS Shared CI

**Shared GitLab CI/CD framework for LLNL open-source projects to run tests on LC machines.**

RADIUSS Shared CI provides reusable GitLab CI Components for testing HPC
software across multiple LC supercomputers. Projects define their build/test
command once and run it across multiple machines with different compilers,
architectures, and configurations.

**Current Version:** v2026.02.2 | **Requires:** GitLab 17.0+

📚 **[Full Documentation](https://radiuss-shared-ci.readthedocs.io/)** | 🚀 **[5-Minute Setup](https://radiuss-shared-ci.readthedocs.io/en/latest/getting-started/five-minute-setup.html)** | 📖 **[Migration Guide](https://radiuss-shared-ci.readthedocs.io/en/latest/user_guide/components_migration.html)**

## Table of Contents

- [Quick Start](#quick-start)
- [Components](#components)
  - [Base Pipeline](#base-pipeline)
  - [Machine Pipelines](#machine-pipelines)
    - [Dane Pipeline](#dane-pipeline)
    - [Matrix Pipeline](#matrix-pipeline)
    - [Tioga Pipeline](#tioga-pipeline)
    - [Tuolumne Pipeline](#tuolumne-pipeline)
    - [Corona Pipeline](#corona-pipeline)
  - [Performance Pipeline](#performance-pipeline)
  - [Utilities](#utilities)
    - [Draft PR Filter](#draft-pr-filter)
    - [Branch Skip](#branch-skip)
- [Examples](#examples)
- [Legacy Approach](#legacy-approach)
- [Contribute](#contribute)
- [Versioning](#versioning)
- [Users](#users)
- [License](#license)

## Quick Start

Define stages and include components in your `.gitlab-ci.yml`:

```yaml
# Define required stages (components don't define them)
stages:
  - prerequisites
  - build-and-test

# Include base pipeline
include:
  - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
    inputs:
      github_project_name: "my-project"
      github_project_org: "LLNL"
      github_token: $GITHUB_STATUS_TOKEN

# Machine availability check
dane-up-check:
  extends: [.dane, .machine-check]
  variables:
    ASSOCIATED_CHILD_PIPELINE: "dane-build-and-test"

# Machine pipeline trigger
dane-build-and-test:
  needs: [dane-up-check]
  extends: [.dane, .build-and-test]
  trigger:
    include:
      - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@v2026.02.2
        inputs:
          job_cmd: "./scripts/build-and-test.sh"
          shared_alloc: "--nodes=1 --time=30"
          job_alloc: "--nodes=1"
          github_project_name: "my-project"
          github_project_org: "LLNL"
      - local: '.gitlab/jobs/dane.yml'
```

Create `.gitlab/jobs/dane.yml` with your jobs:

```yaml
gcc-build:
  extends: .job_on_dane
  variables:
    COMPILER: "gcc"
```

See [5-Minute Setup Guide](https://radiuss-shared-ci.readthedocs.io/en/latest/getting-started/five-minute-setup.html) for complete walkthrough.

## Components

### Base Pipeline

Provides orchestration templates for multi-machine pipelines and machine availability checks.

**Usage:**

```yaml
include:
  - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
    inputs:
      github_project_name: "my-project"
      github_project_org: "LLNL"
      github_token: $GITHUB_STATUS_TOKEN
```

**Exports:** `.machine-check`, `.build-and-test`, `.custom_job`, `.custom_perf`, machine rule templates

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/base-pipeline)

### Machine Pipelines

#### Dane Pipeline

CI templates for Dane (SLURM scheduler, Intel Xeon CPUs).

**Usage:**

```yaml
dane-build-and-test:
  trigger:
    include:
      - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@v2026.02.2
        inputs:
          job_cmd: "./build-and-test.sh"
          shared_alloc: "--nodes=1 --time=30"
          job_alloc: "--nodes=1"
          github_project_name: "my-project"
          github_project_org: "LLNL"
      - local: '.gitlab/jobs/dane.yml'
```

**Exports:** `.job_on_dane`, reproducer templates

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/dane-pipeline)

#### Matrix Pipeline

CI templates for Matrix (SLURM scheduler, Intel Xeon CPUs, NVIDIA H100 GPUs).

**Usage:**

```yaml
matrix-build-and-test:
  trigger:
    include:
      - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/matrix-pipeline@v2026.02.2
        inputs:
          job_cmd: "./build-and-test.sh"
          shared_alloc: "--partition=pci --nodes=1 --time=30"
          job_alloc: "--partition=pci --nodes=1"
          github_project_name: "my-project"
          github_project_org: "LLNL"
      - local: '.gitlab/jobs/matrix.yml'
```

**Exports:** `.job_on_matrix`, reproducer templates

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/matrix-pipeline)

#### Tioga Pipeline

CI templates for Tioga (Flux scheduler, AMD EPYC CPUs, AMD MI250X GPUs).

**Usage:**

```yaml
tioga-build-and-test:
  trigger:
    include:
      - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/tioga-pipeline@v2026.02.2
        inputs:
          job_cmd: "./build-and-test.sh"
          shared_alloc: "--queue=pci --nodes=1 --time-limit=30m"
          job_alloc: "--nodes=1 --begin-time=+5s"
          github_project_name: "my-project"
          github_project_org: "LLNL"
      - local: '.gitlab/jobs/tioga.yml'
```

**Exports:** `.job_on_tioga`, reproducer templates

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/tioga-pipeline)

#### Tuolumne Pipeline

CI templates for Tuolumne (Flux scheduler, AMD EPYC CPUs, AMD MI300A GPUs, production-like environment).

**Usage:**

```yaml
tuolumne-build-and-test:
  trigger:
    include:
      - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/tuolumne-pipeline@v2026.02.2
        inputs:
          job_cmd: "./build-and-test.sh"
          shared_alloc: "--queue=pci --nodes=1 --time-limit=30m"
          job_alloc: "--nodes=1 --begin-time=+5s"
          github_project_name: "my-project"
          github_project_org: "LLNL"
      - local: '.gitlab/jobs/tuolumne.yml'
```

**Exports:** `.job_on_tuolumne`, reproducer templates

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/tuolumne-pipeline)

#### Corona Pipeline

CI templates for Corona (Flux scheduler, AMD EPYC CPUs, AMD MI50 GPUs).

**Usage:**

```yaml
corona-build-and-test:
  trigger:
    include:
      - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/corona-pipeline@v2026.02.2
        inputs:
          job_cmd: "./build-and-test.sh"
          shared_alloc: "--nodes=1 --time-limit=30m"
          job_alloc: "--nodes=1 --begin-time=+5s"
          github_project_name: "my-project"
          github_project_org: "LLNL"
      - local: '.gitlab/jobs/corona.yml'
```

**Exports:** `.job_on_corona`, reproducer templates

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/corona-pipeline)

### Performance Pipeline

Templates for performance measurements across machines with GitHub benchmark reporting.

**Usage:**

```yaml
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
          tioga_perf_alloc: "--queue=pci --nodes=1 --time-limit=15m"
          perf_processing_cmd: "./scripts/process-results.py"
          github_token: $GITHUB_STATUS_TOKEN
          github_project_name: "my-project"
          github_project_org: "LLNL"
      - local: '.gitlab/jobs/performances.yml'
```

**Exports:** `.perf_on_<machine>`, `.results_processing`, `.results_reporting`, GitHub benchmark templates

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/performance-pipeline)

### Utilities

#### Draft PR Filter

Skips CI pipelines on draft pull requests.

**Usage:**

```yaml
include:
  - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/utility-draft-pr-filter@v2026.02.2
    inputs:
      github_token: $GITHUB_STATUS_TOKEN
      github_project_name: "my-project"
      github_project_org: "LLNL"
```

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/utility-draft-pr-filter)

#### Branch Skip

Skips CI on branches not associated with a pull request.

**Usage:**

```yaml
include:
  - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/utility-branch-skip@v2026.02.2
    inputs:
      github_token: $GITHUB_STATUS_TOKEN
      github_project_name: "my-project"
      github_project_org: "LLNL"
```

[View component →](https://lc.llnl.gov/gitlab/radiuss/radiuss-shared-ci/-/components/utility-branch-skip)

## Examples

Complete configuration examples in [`examples/`](examples/):

- [`example-gitlab-ci.yml`](examples/example-gitlab-ci.yml) - Complete component-based setup
- [`example-custom-jobs.yml`](examples/example-custom-jobs.yml) - Job customization templates
- [`example-jobs-dane.yml`](examples/example-jobs-dane.yml) - Machine-specific jobs

## Legacy Approach

The include-based approach (using `pipelines/*.yml` and `utilities/*.yml`) remains available for GitLab < 17.0 or projects not yet migrated.

See [Migration Guide](https://radiuss-shared-ci.readthedocs.io/en/latest/user_guide/components_migration.html) for migration instructions.

## Contribute

Contributions are welcome! See [Contributing Guide](https://radiuss-shared-ci.readthedocs.io/en/latest/dev_guide/contributing.html) for:

- Development workflow
- Testing guidelines
- Code style
- Pull request process

Please read our [Code of Conduct](https://github.com/LLNL/.github/blob/main/CODE_OF_CONDUCT.md).

## Versioning

**Current Version:** v2026.02.2

Version format: `vYYYY.MM.PATCH`

- `YYYY.MM` - Release year and month
- `PATCH` - Hotfix index

See [CHANGELOG.md](CHANGELOG.md) for release history.

## Users

Projects using RADIUSS Shared CI:

- [AMS](https://github.com/LLNL/AMS)
- [Caliper](https://github.com/LLNL/Caliper)
- [CARE](https://github.com/LLNL/CARE)
- [CHAI](https://github.com/LLNL/CHAI)
- [mfem](https://github.com/mfem/mfem)
- [RAJA](https://github.com/LLNL/RAJA)
- [RAJAPerf](https://github.com/LLNL/RAJAPerf)
- [SAMRAI](https://github.com/LLNL/SAMRAI)
- [sundials](https://github.com/LLNL/sundials)
- [Umpire](https://github.com/LLNL/Umpire)

## Authors

Adrien M. Bernede

See [contributors](https://github.com/LLNL/radiuss-shared-ci/contributors) for all participants.

## License

MIT License - All new contributions must be made under the MIT License.

See [LICENSE](https://github.com/LLNL/radiuss-shared-ci/blob/master/LICENSE),
[COPYRIGHT](https://github.com/LLNL/radiuss-shared-ci/blob/master/COPYRIGHT), and
[NOTICE](https://github.com/LLNL/radiuss-shared-ci/blob/master/NOTICE) for details.


SPDX-License-Identifier: (MIT)

LLNL-CODE-793462

---

**About RADIUSS:** The RADIUSS project promotes and supports key HPC open-source software developed at LLNL. These tools and libraries cover a wide range of features teams would need to develop modern simulation codes targeting HPC platforms.
