<h1> <img src="https://rawcdn.githack.com/michidk/run-komac/86f4bad6701cb130ce85c4505ad39d9bbcd3d919/.github/images/github-actions-logo.png" width="32" height="32" alt="Logo" /> Run Komac (GitHub Action) <a href="https://github.com/russellbanks/Komac"> <img src="https://rawcdn.githack.com/michidk/run-komac/86f4bad6701cb130ce85c4505ad39d9bbcd3d919/.github/images/komac-logo.svg" height="24px" style="vertical-align:bottom" alt="Komac logo" /> </a></h1>

![GitHub issues][github-issues-badge]
![GitHub release (latest by date)][github-release-badge]
![GitHub Repo stars][github-repo-stars-badge]
![GitHub][github-license-badge]

This GitHub Action sets up Java, installs Komac, and then runs a user-specified command with Komac. It's designed to be efficient by checking if Java and Komac are already installed before attempting to install them.

## 📖 Example Usage

```yaml
name: My Workflow
on: workflow_dispatch

jobs:
  my_job:
    runs-on: ubuntu-latest
    steps:
    - name: Run Komac
      uses: michidk/run-komac@v1
      with:
        args: '--version'
```

## ⚒️ Configuration Options

- `komac-version`: Specifies which version of Komac to use.
  - **Required**: ❌
  - **Default**: `1.11.0`

- `command`: The command to run with Komac.
  - **Required**: ✅

- `java-version`: Specifies which version of Java to use.
  - **Required**: ❌
  - **Default**: `17`

[github-issues-badge]: https://img.shields.io/github/issues/michidk/run-komac?logo=target
[github-release-badge]: https://img.shields.io/github/v/release/michidk/run-komac?logo=github
[github-repo-stars-badge]: https://img.shields.io/github/stars/michidk/run-komac?logo=githubsponsors
[github-license-badge]: https://img.shields.io/github/license/michidk/run-komac?logo=gnu
