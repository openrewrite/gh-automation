# gh-automation

Shared GitHub Actions workflows which are used by multiple repositories under the OpenRewrite GitHub organization.

## Repository layout

Everything lives under `.github`, split by what GitHub does with each folder.

### `.github/workflows`

Reusable workflows: entire jobs that another repository calls with `uses:` from its own workflow file. They all trigger on `workflow_call` only, so nothing here runs on this repository itself.

| Workflow | Purpose |
| --- | --- |
| `ci-gradle.yml` | Standard Gradle CI: build on `ubuntu-latest`, publish snapshots from `main`, notify Slack on scheduled-build failure. |
| `publish-gradle.yml` | Release publishing from a tag: `candidate` for `-rc.` tags, `final` otherwise, closing and releasing the Sonatype staging repository. |
| `receive-pr-runner.yml` | Runs a Moderne CLI recipe against an incoming pull request and uploads the resulting diff as an artifact. Handles untrusted code, so it gets no secrets. |
| `comment-pr-runner.yml` | Picks up that artifact from a `workflow_run` and posts the diff back as pull request review suggestions. Has write permissions, so it must not execute untrusted code. |
| `repository-backup.yml` | Mirrors the repository to an S3-compatible object storage bucket. |
| `stale.yml` | Closes unanswered `question` issues and pull requests without activity for 90 days. |

The pull request pair follows the [preventing pwn requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/) pattern: `receive-pr-runner` runs untrusted code without secrets, `comment-pr-runner` holds the write token but only applies a patch.

### `.github/actions`

Composite actions: individual steps shared between the workflows above (and callable directly from a downstream repository's own job).

| Action | Purpose |
| --- | --- |
| `setup` | Checkout with full history, install the requested JDK(s) and uv, configure Gradle with optional Develocity access. Expected to run first. |
| `build` | Run `./gradlew build` with the standard CI switches. |
| `publish-snapshots` | Publish snapshots with signing credentials wired in as Gradle project properties. |
| `slack-failure` | Post a CI failure notification to a Slack webhook; the caller supplies the `if:` guard. |

## Usage

Reference a workflow from a downstream repository, passing along the secrets it needs:

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:
jobs:
  build:
    uses: openrewrite/gh-automation/.github/workflows/ci-gradle.yml@main
    secrets: inherit
```
