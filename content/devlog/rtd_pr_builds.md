---
title: "Building documentation on Read the Docs for GitHub pull requests"
date: "2023-10-05"
---

Even though you may deploy your documentation on services other than Read the Docs (RTD), such as GitHub pages, those services may not offer a convenient way to build documentation for pull requests (PR).

Building Sphinx documentation on RTD only requires a minimal configuration file as well as an environment file.
In this case, our RTD configuration file will match the one found in the [MDAnalysis MDAKit cookiecutter](https://github.com/MDAnalysis/cookiecutter-mdakit/blob/16e01026b417ef06b9d05a3b60aa39c3cf12dece/readthedocs.yaml), which also references the mentioned environment file used with conda.
The contents of the `.readthedocs.yaml`, which must be in the root of the git repository, are shown below.

```
version: 2

build:
  os: ubuntu-22.04
  tools:
    python: "mambaforge-4.10"

python:
  install:
    - method: pip
      path: .

conda:
  environment: docs/requirements.yaml
```

The environment file should look something like the following, lifted from the [PathSimAnalysis](https://github.com/MDAnalysis/PathSimAnalysis) MDAKit.

```yaml
name: pathsimanalysis-test
channels:
  - conda-forge
  - defaults
dependencies:
  # Base depends
  - python
  - pip

  # MDAnalysis
  - MDAnalysis

  # Testing
  - MDAnalysisTests
  - pytest
  - pytest-cov
  - pytest-xdist
  - codecov
```

Building on a PR can be enabled in the advanced settings in the Read the Docs interface for the project.
Check the "Build pull requests for this project" box.

New PRs will trigger a RTD build.
If you have a previously opened PR, you will have to push to the branch again to trigger the build.
The least intrusive way to do this without editing any files is by adding an empty commit.

```bash
git commit --allow-empty -m "Trigger RTD"
git push origin rtd_pr_build
```

If everything was done correctly, your PRs should have their docs built automatically on RTD.
