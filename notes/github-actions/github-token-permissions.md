# GitHub Actions: the vault and the visitor's badge

Added: 2026-11-19  
Topic: GitHub Actions security  
Status: to-review

## What I learned

A workflow file lives in the repository, but the commands it describes run in a **job on a runner**. To access GitHub, that job can use `GITHUB_TOKEN`: a short-lived GitHub App installation token created for the job and scoped to the repository containing the workflow. The `permissions` key controls what that token may do; it does not change your personal GitHub permissions.

Think of the repository as a vault, the workflow as instructions, and the token as a visitor's badge. The runner uses the badge when it asks GitHub for repository access. Activity may appear as `github-actions[bot]`; there is no literal contractor or separate person.

A GitHub-hosted runner normally starts on a fresh VM with tools already installed, and GitHub disposes of it after the job. It is not an empty machine. Self-hosted runners can persist between jobs and require their own isolation and cleanup. A job token expires when the job finishes or reaches its maximum lifetime; it is not a single-use token.

## A read-only badge

```yaml
name: Test
on: [push, pull_request]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - run: echo "Run your tests here"
```

`contents: read` allows the token to read repository contents, which is the recommended permission for a normal checkout. Once you specify any permissions, unspecified permissions are set to `none`: for example, the token cannot write contents, issues, or pull requests. A workflow-level declaration applies to all jobs unless a job declares its own permissions; that job's declaration defines its own set, so include every permission it needs.

`persist-credentials: false` prevents checkout from leaving its authentication token in local Git configuration for later Git commands. Actions can still access `github.token` even if the workflow does not explicitly pass it, so use trusted actions and grant the token only what the job needs.

## When write access is needed

| Task | Typical permission | Why |
| --- | --- | --- |
| Check out and test code | `contents: read` | Fetch repository contents. |
| Push a commit or create a release | `contents: write` | Modify repository contents or releases. Branch protection and rulesets can still block a push. |
| Comment on an issue | `issues: write` | Create an issue comment. |
| Create or update a pull request | `pull-requests: write` | Modify pull requests; creating one can also require repository settings to allow Actions to create pull requests. |
| Publish a GitHub package | `packages: write` | Publish packages, subject to package access settings. |

Grant write access only to the job that needs it, for example:

```yaml
permissions: {}

jobs:
  inspect:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4

  comment:
    runs-on: ubuntu-latest
    permissions:
      issues: write
    steps:
      - run: echo "A trusted issue-commenting step would run here"
```

These jobs have different token permissions. This example illustrates the permissions; it does not actually post a comment. Forked pull request workflows normally receive a read-only token even if write access is requested, subject to repository settings. Treat `pull_request_target` with special care: it runs in the base repository's trusted context; do not check out and execute untrusted pull request code in such a privileged job.

## What the badge does not protect

If malicious code steals a read-only token, it can still read and potentially leak anything that token can access, including private repository contents. It may also access other credentials, secrets, files, network resources, or privileges available to the job or runner. `permissions` limits `GITHUB_TOKEN`, not a separately supplied personal access token, cloud credentials, or the runner's operating-system permissions. Use least privilege, pin third-party actions to reviewed full commit SHAs, and avoid running untrusted code with secrets or elevated credentials.

## Verification and follow-up

The examples are illustrative and have not been run. In a test repository, try a checkout with `contents: read`, then attempt an authenticated write and observe it fail. Test write access separately on a disposable branch, accounting for branch protection and repository policy. Never print a token in logs.

## Sources and related notes

- [GitHub: automatic token authentication](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication)
- [GitHub: workflow syntax and permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#permissions)
- [GitHub: securely using pull_request_target](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target)
- [actions/checkout documentation](https://github.com/actions/checkout)
- [Self-hosted runners](self-hosted-runners/README.md)
