# CI Notes: Claude Code as a CI Worker

Exam prep notes on using Claude Code in continuous integration (CI) with GitHub.

## Contents

1. [Get the GitHub CLI (`gh`)](#1-get-the-github-cli-gh)
2. [CI concept using a GitHub project](#2-ci-concept-using-a-github-project)
3. [Live demo: Claude as a CI worker](#3-live-demo-claude-as-a-ci-worker)
4. [In-depth configuration of Claude Code as a CI worker](#4-in-depth-configuration-of-claude-code-as-a-ci-worker)

---

## 1. Get the GitHub CLI (`gh`)

`gh` is GitHub's official command-line tool. It lets you work with issues, pull requests, workflow runs, and secrets from the terminal. Claude Code also uses it to talk to GitHub.

### Install

| Platform | Command |
|----------|---------|
| macOS (Homebrew) | `brew install gh` |
| Windows (winget) | `winget install --id GitHub.cli` |
| Debian/Ubuntu | See the official instructions at https://github.com/cli/cli/blob/trunk/docs/install_linux.md |
| Fedora/RHEL | `sudo dnf install gh` |

Check the install:

```bash
gh --version
```

### Authenticate

```bash
gh auth login      # follow the prompts (browser or token)
gh auth status     # confirm who you are logged in as
```

### Commands worth knowing

```bash
gh repo view                      # info about the current repo
gh issue list                     # open issues
gh issue view 5                   # read one issue
gh pr create --fill               # open a PR from the current branch
gh pr checks                      # CI status of the current PR
gh run list                       # recent workflow runs
gh run view <run-id> --log-failed # logs of the failed steps only
gh secret set NAME                # store a repository secret
```

> Exam tip: in CI, `gh` reads the `GH_TOKEN` (or `GITHUB_TOKEN`) environment variable instead of an interactive login.

---

## 2. CI concept using a GitHub project

### What is CI?

Continuous Integration means every change is automatically built and checked as soon as it is pushed. Problems show up early, on the pull request, instead of after merging.

### Core pieces in GitHub Actions

| Piece | Meaning |
|-------|---------|
| Workflow | A YAML file in `.github/workflows/` |
| Trigger (`on:`) | The event that starts the workflow (`pull_request`, `push`, `issue_comment`, ...) |
| Job | A group of steps that runs on one runner |
| Runner | The temporary machine (e.g. `ubuntu-latest`) |
| Step | One command (`run:`) or one reusable action (`uses:`) |
| Secret | An encrypted value such as an API token, referenced as `${{ secrets.NAME }}` |

### Example in this repo

This repo is Markdown only, and CI enforces it. `.github/workflows/markdown-only.yml`:

- **Trigger:** every pull request, and every push to `main`.
- **Steps:** check out the repo, list tracked files with `git ls-files`, and fail if any file is not `.md` (small allowed exceptions: `.gitignore`, `LICENSE`, workflow YAML).
- **Result:** a green or red check on each PR. A red check blocks a careless merge, if branch protection requires it.

### The flow

```
push / open PR  ->  workflow triggered  ->  runner starts
        ->  checkout + checks run  ->  pass/fail shown on the PR
```

### Where Claude fits

A normal CI job runs fixed scripts. A Claude job runs an agent that can read code, reason about it, and act: review a PR, answer a question, or implement a change. It is still just a workflow, with a trigger, a runner, and a secret.

---

## 3. Live demo: Claude as a CI worker

This repo already uses Claude as a CI worker through `.github/workflows/claude.yml`.

### Demo script

1. **Open an issue** describing a task (e.g. "Add CI notes for Exam Prep", issue #5).
2. **Comment `@claude work on this issue and create a pr to main`.**
3. **Watch the Actions tab.** The `Claude Code` workflow starts because the comment contains `@claude`.
4. **Watch Claude's single tracking comment.** It shows a checklist that is updated as work progresses.
5. **Claude works on its own branch** (`claude/issue-<n>-<timestamp>`), commits, and pushes.
6. **Open the PR** from the link Claude posts. The other checks (e.g. `Markdown only`) run on it.
7. **Review and merge**, like any human contributor's PR.

### What to point out

- Claude never pushes to `main` directly. It pushes a branch and you review.
- All communication goes through one GitHub comment.
- CI checks apply to Claude's PR the same way as to anyone else's. Here, adding a non-Markdown file would fail `Markdown only`.
- Follow-up: comment `@claude` again with feedback on the PR and it will push more commits.

---

## 4. In-depth configuration of Claude Code as a CI worker

### 4.1 Setup

- Easiest: run `/install-github-app` inside Claude Code. It installs the GitHub App and sets up the workflow and secret.
- Manual: install the Claude GitHub App on the repo, add the workflow file, and add the auth secret.

### 4.2 Authentication

| Method | Secret | Notes |
|--------|--------|-------|
| Claude subscription OAuth token | `CLAUDE_CODE_OAUTH_TOKEN` | Used by this repo |
| Anthropic API key | `ANTHROPIC_API_KEY` | Pay per token |
| Amazon Bedrock / Google Vertex AI | Cloud credentials (e.g. OIDC) | For enterprise cloud setups |

Never hard-code a key in the workflow. Always use `${{ secrets.NAME }}`.

### 4.3 The workflow file (annotated)

```yaml
name: Claude Code

on:
  issue_comment:            # comments on issues and PRs
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    # Only run when someone mentions @claude
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    permissions:            # least privilege for GITHUB_TOKEN
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: read         # lets Claude read CI results
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # prompt: '...'          # optional fixed instructions
          # claude_args: '...'     # CLI flags, see below
```

(Simplified: the real file's `if:` also covers the other event types.)

### 4.4 Key action inputs

| Input | Purpose |
|-------|---------|
| `claude_code_oauth_token` / `anthropic_api_key` | Authentication |
| `prompt` | Fixed instructions. Without it, Claude follows the `@claude` comment (interactive mode). With it, the action runs automatically (automation mode). |
| `claude_args` | Pass Claude Code CLI flags |
| `additional_permissions` | Extra GitHub permissions, e.g. `actions: read` |
| `trigger_phrase` | Change the default `@claude` |

### 4.5 Useful `claude_args`

```yaml
claude_args: |
  --model <model-id>
  --max-turns 10
  --allowed-tools "Bash(gh pr *),Bash(npm test)"
  --append-system-prompt "Follow the style guide in CLAUDE.md"
```

- `--allowed-tools` restricts what Claude may run. Allow only what the job needs.
- `--max-turns` caps the agent loop, which bounds cost and runaway behaviour.
- `--model` picks the model, trading speed and cost against capability.

### 4.6 `CLAUDE.md`: repo instructions

Claude reads `CLAUDE.md` at the repo root. Put build/test commands, conventions, and rules there (e.g. "this repo is Markdown only"). This is how you steer a CI worker without touching the workflow.

### 4.7 Automation mode: review every PR

```yaml
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: "Review this PR for correctness and clarity."
```

### 4.8 Security checklist

- Keep secrets in GitHub Secrets, never in the repo or the prompt.
- Give the workflow minimal `permissions:`.
- Limit tools with `--allowed-tools`.
- Keep humans in the loop: Claude opens PRs, people merge them. Use branch protection and required checks.
- Be careful with untrusted input (issue text, PRs from forks): it can contain prompt injection. Do not give the job broad permissions when it processes such input.
- Only users with write access to the repo can trigger the action by default.
- Workflow files (`.github/workflows/`) cannot be modified by the Claude GitHub App. Change them yourself.

### 4.9 Cost and reliability

- Each run consumes tokens or subscription usage. Use `--max-turns`, narrow triggers, and `if:` conditions.
- Set `timeout-minutes` on the job.
- Use `concurrency:` to avoid overlapping runs on the same PR.

### 4.10 Quick recap for the exam

- Claude Code in CI = a GitHub Actions workflow + `anthropics/claude-code-action` + an auth secret.
- Two modes: interactive (`@claude` mention) and automation (`prompt:` set).
- Control it with `claude_args`, `CLAUDE.md`, and `permissions:`.
- Safety = least privilege, allowed tools, human review, protected branches.
