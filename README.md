# RADIUSS Shared CI

The RADIUSS project promotes and supports key High Performance Computing (HPC)
open-source software developed at the LLNL. These tools and libraries cover a
wide range of features a team would need to develop a modern simulation code
targeting HPC platforms.

Radiuss Shared CI allows project to share CI configuration for GitLab.

## Getting Started

This project is meant to be hosted in a GitLab instance so that other projects
can use it to configure their CI pipelines.

**As of v2025.12.0**, RADIUSS Shared CI is available as **GitLab CI Components** (requires GitLab 17.0+). Current version: **v2026.02.2**. This provides:

- **Better versioning** through the GitLab CI/CD Catalog
- **Type-safe inputs** with validated parameters
- **Cleaner syntax** using `component:` instead of `include: project:`
- **No more copy-pasting** - all templates are reusable components

User documentation is located here: [**RADIUSS Shared CI Docs**](https://radiuss-shared-ci.readthedocs.io/en/latest/).

### Quick Start with Components

Add components to your `.gitlab-ci.yml`:

```yaml
# You must define stages (components don't define them to allow customization)
stages:
  - prerequisites
  - build-and-test
  - performance-measurements

include:
  # Base pipeline orchestration
  - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/base-pipeline@v2026.02.2
    inputs:
      github_project_name: "my-project"
      github_project_org: "LLNL"
      github_token: $GITHUB_TOKEN

  # Machine-specific pipeline (choose the machines you need)
  - component: $CI_SERVER_FQDN/radiuss/radiuss-shared-ci/dane-pipeline@v2026.02.2
    inputs:
      job_cmd: "./scripts/build-and-test.sh"
      shared_alloc: "--reservation=ci --exclusive --nodes=1 --time=30"
      job_alloc: "--reservation=ci --overlap --nodes=1"
      github_project_name: "my-project"
      github_project_org: "LLNL"
```

See the [examples/](examples/) directory for complete configuration examples.

### Available Components

- **base-pipeline** - Orchestration templates and machine availability checks
- **dane-pipeline** - Dane supercomputer (SLURM scheduler)
- **matrix-pipeline** - Matrix supercomputer (SLURM scheduler)
- **corona-pipeline** - Corona supercomputer (flux scheduler)
- **tioga-pipeline** - Tioga supercomputer (flux scheduler)
- **tuolumne-pipeline** - Tuolumne supercomputer (flux scheduler)
- **performance-pipeline** - Performance measurement and GitHub reporting
- **utility-draft-pr-filter** - Skip CI on draft pull requests
- **utility-branch-skip** - Skip CI on non-PR branches

### Legacy Include-Based Approach

The traditional include-based approach (using `pipelines/*.yml` and `utilities/*.yml`) is still available for GitLab versions < 17.0 or for projects that haven't yet migrated. See the documentation for migration guidance.

### Installing

This project requires no installation. Components are consumed directly from the GitLab instance.

## Contributing

Please read [CONTRIBUTING.md](https://github.com/LLNL/radiuss-shared-ci/CONTRIBUTING.md) for details on our code of
conduct, and the process for submitting pull requests to us.

## Versioning

version: v2026.02.2

When a new version of RADIUSS Shared CI is released, we use the current year and month to set the version number,
followed by a "minor" index which we increase in case we need a hotfix.

This is coherent with how most projects using RADIUSS Shared CI deal with versioning and helps syncing with other
utilities, like [RADIUSS Spack configs](https://github.com/LLNL/radiuss-spack-configs).

## List of users
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

See also the list of [contributors](https://github.com/LLNL/radiuss-shared-ci/contributors) who participated in this
project.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

All new contributions must be made under the MIT License.

See [LICENSE](https://github.com/LLNL/radiuss-shared-ci/blob/master/LICENSE),
[COPYRIGHT](https://github.com/LLNL/radiuss-shared-ci/blob/master/COPYRIGHT), and
[NOTICE](https://github.com/LLNL/radiuss-shared-ci/blob/master/NOTICE) for details.

SPDX-License-Identifier: (MIT)

LLNL-CODE-793462

## Acknowledgments


