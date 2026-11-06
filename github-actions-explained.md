# GitHub Actions and CI Triggers, Explained Simply

A baby-steps guide for Claude Code or any agent. If you can understand a doorbell, you can understand this.

## Contents

1. [What is CI?](#1-what-is-ci)
2. [What is GitHub Actions?](#2-what-is-github-actions)
3. [What is a trigger?](#3-what-is-a-trigger)
4. [The parts of a workflow file](#4-the-parts-of-a-workflow-file)
5. [Your first workflow](#5-your-first-workflow)
6. [Common triggers](#6-common-triggers)
7. [Filtering triggers](#7-filtering-triggers)
8. [How Claude Code uses triggers](#8-how-claude-code-uses-triggers)
9. [Safety rules](#9-safety-rules)
10. [Cheat sheet](#10-cheat-sheet)

---

## 1. What is CI?

**CI** means **Continuous Integration**.

Imagine you and your friends build one big LEGO castle together. Every time someone adds a piece, a helper checks: "Does the castle still stand? Are all pieces the right color?"

That helper is **CI**. Every time someone changes the code, CI automatically checks that nothing broke.

## 2. What is GitHub Actions?

**GitHub Actions** is a robot helper that lives inside GitHub.

- You write a to-do list for the robot.
- GitHub lends the robot a temporary computer.
- The robot does your list, then the computer disappears.

You don't have to press any buttons. The robot starts by itself when something happens.

## 3. What is a trigger?

A **trigger** is the "something happens" that wakes the robot up.

Think of a doorbell:

| Real life | GitHub |
|-----------|--------|
| Someone presses the doorbell | Someone opens a pull request |
| You go to the door | The robot starts a job |

In GitHub Actions, triggers are written under the word `on:`.

Read `on:` as "**when this happens...**"

## 4. The parts of a workflow file

A **workflow** is one to-do list for the robot. It is a small file in `.github/workflows/` ending in `.yml`.

Five words to learn:

| Word | Kid-friendly meaning |
|------|----------------------|
| `name` | The title of the to-do list |
| `on` | The trigger: when should the robot wake up? |
| `jobs` | The big chores on the list |
| `runs-on` | Which kind of temporary computer to borrow (e.g. `ubuntu-latest`) |
| `steps` | The small tasks inside a chore, done in order |

Picture it like this:

```
Workflow (the to-do list)
 ├── on: when to start
 └── jobs (chores)
      └── job
           ├── runs-on: which computer
           └── steps (small tasks, in order)
```

## 5. Your first workflow

Make a file called `.github/workflows/hello.yml`:

```yaml
name: Say Hello

on: push            # when someone pushes code...

jobs:
  hello:
    runs-on: ubuntu-latest      # borrow a computer
    steps:
      - name: Say hi
        run: echo "Hello from the robot!"
```

What happens, step by step:

1. You push code to GitHub. (The doorbell rings.)
2. GitHub sees `on: push` and wakes the robot.
3. The robot borrows an Ubuntu computer.
4. It runs `echo "Hello from the robot!"`.
5. A green check mark appears if all went well. A red X appears if not.

You can watch it in the **Actions** tab of your repository.

## 6. Common triggers

| Trigger | Plain English | Example |
|---------|---------------|---------|
| `push` | Someone uploads new commits | Run tests after each push |
| `pull_request` | A pull request is opened or updated | Check the proposed change |
| `issues` | An issue is opened, edited, assigned... | Greet new issues |
| `issue_comment` | Someone comments on an issue or PR | Reply when someone types `@claude` |
| `pull_request_review` | Someone submits a review | React to review feedback |
| `pull_request_review_comment` | Someone comments on a line of code in a PR | Answer a code question |
| `schedule` | A clock time (uses cron) | Run every night |
| `workflow_dispatch` | A person clicks "Run workflow" | Manual button |

### Many triggers at once

```yaml
on:
  push:
  pull_request:
  workflow_dispatch:
```

The robot wakes up when **any** of them happens.

### Schedule example

```yaml
on:
  schedule:
    - cron: '0 9 * * 1'   # every Monday at 9:00 UTC
```

A cron has 5 parts: `minute hour day-of-month month day-of-week`.

## 7. Filtering triggers

A doorbell that rings for everyone is annoying. You can filter.

### Only certain branches

```yaml
on:
  push:
    branches: [main]
```

"Only wake up when someone pushes to `main`."

### Only certain actions (types)

```yaml
on:
  issues:
    types: [opened, assigned]
```

"Only wake up when an issue is *opened* or *assigned*, not when it is edited."

### Only certain files

```yaml
on:
  push:
    paths:
      - '**.md'
```

"Only wake up if a Markdown file changed."

### Only if a condition is true (`if`)

Filters on `on:` are coarse. For finer control, add `if:` to a job:

```yaml
jobs:
  reply:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Someone called Claude!"
```

"The doorbell rang, but only get up if the visitor said the magic word."

## 8. How Claude Code uses triggers

Claude Code runs in GitHub through an action called `anthropics/claude-code-action`. It is just another step in a robot's to-do list.

This repository's `.github/workflows/claude.yml` works like this (shortened):

```yaml
on:
  issue_comment:
    types: [created]        # someone wrote a comment
  issues:
    types: [opened, assigned]

jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')   # only if they said @claude
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # copy the repo onto the computer
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The story, step by step:

1. You write a comment: `@claude please fix this`. (Doorbell!)
2. The `issue_comment` trigger fires.
3. The `if:` checks for the magic word `@claude`. Found it.
4. The robot borrows a computer and copies the repo onto it.
5. Claude reads your comment, does the work, and pushes a branch.
6. Claude replies in a comment. Then the computer disappears.

There is also an automatic review workflow (`claude-code-review.yml`) that wakes up on `pull_request` so Claude can review changes without anyone asking.

### Same idea for any agent

Any agent (not just Claude) follows the same recipe:

1. **Pick a trigger** (comment, PR, schedule, button).
2. **Add an `if:`** so it only runs when it should.
3. **Check out the code.**
4. **Run the agent** with its secret key.
5. **Let it report back** (comment, PR, or check result).

## 9. Safety rules

Robots with power need rules.

- **Never write secrets in the file.** Store keys in *Settings → Secrets* and use `${{ secrets.NAME }}`.
- **Give the minimum permissions.** The `permissions:` block says what the robot may touch. Start with `contents: read`.
- **Be careful with who can trigger it.** Only people with write access should be able to wake an agent that can change code.
- **Treat outside text as untrusted.** A stranger's issue or comment could contain sneaky instructions. Limit what tools the agent may use (for example with `claude_args: '--allowed-tools ...'`).
- **Watch for loops.** If a robot's own push triggers the same robot again, it can run forever. Use filters so it ignores itself.
- **Mind the cost.** Every run uses minutes and, for agents, tokens.

## 10. Cheat sheet

| I want to... | Use |
|--------------|-----|
| Run when code is pushed | `on: push` |
| Check pull requests | `on: pull_request` |
| React to comments | `on: issue_comment` |
| Run on a timer | `on: schedule` with `cron` |
| Run with a button | `on: workflow_dispatch` |
| Limit to a branch | `branches: [main]` |
| Limit to an event kind | `types: [opened]` |
| Limit to some files | `paths: ['**.md']` |
| Run only if a word appears | `if: contains(..., '@claude')` |

**One-sentence summary:** GitHub Actions is a robot, a trigger is the doorbell that wakes it, and the workflow file is the to-do list it follows.
