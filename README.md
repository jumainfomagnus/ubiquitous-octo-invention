# Lab 2: Concurrency Control

**Trainer:** GitHub Actions Intermediate Training for Wells Fargo  
**Duration:** 45-60 minutes  
**Prerequisites:** Understanding of GitHub Actions workflow triggers

---

## Overview

This lab demonstrates how to prevent overlapping workflow runs using concurrency controls. You'll observe the problem of simultaneous deployments, then incrementally add concurrency groups and cancellation strategies to manage workflow execution efficiently.

No local setup required—you'll create workflows directly in GitHub and trigger them multiple times to see how concurrency changes behavior.

---

## Step-by-Step

### Create your repository

1. Use an existing GitHub repository or create a new one.

2. Make sure you're working on the **main** branch.

---

## Exercise 1: Observe the Problem (10 minutes)

### See what happens without concurrency control

1. In GitHub, navigate to **Code** tab → **Add file** → **Create new file**.

2. Name it `.github/workflows/concurrency.yml`.

3. Paste this initial workflow:

```yaml
name: Concurrency Demo

on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Simulate deployment
        run: |
          echo "Starting deployment at $(date)"
          echo "Run ID: ${{ github.run_id }}"
          sleep 30
          echo "Deployment complete at $(date)"
```

4. Commit the file.

5. Go to **Actions** → **Concurrency Demo** → **Run workflow** (click it 3 times quickly to queue 3 runs).

6. **Watch**: All 3 workflows run simultaneously, potentially causing deployment conflicts.

### What just happened?

Without concurrency control, GitHub Actions runs every triggered workflow immediately (subject to runner availability). For deployments, this means:
- Multiple deployments overwrite each other
- Resource conflicts in production
- Unpredictable final state

This is the problem we'll solve.

---

## Exercise 2: Add Basic Concurrency Group (10 minutes)

### Queue runs instead of running simultaneously

1. Edit `.github/workflows/concurrency.yml`.

2. Add a `concurrency:` section at the workflow level (right after the `on:` section):

```yaml
name: Concurrency Demo

on:
  workflow_dispatch:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Simulate deployment
        run: |
          echo "Starting deployment at $(date)"
          echo "Run ID: ${{ github.run_id }}"
          echo "Concurrency group: ${{ github.workflow }}-${{ github.ref }}"
          sleep 30
          echo "Deployment complete at $(date)"
```

3. Commit the changes.

4. Go to **Actions** → **Concurrency Demo** → run the workflow 3 times quickly again.

5. **Watch**: Runs now queue up—only one runs at a time, the others wait.

### What just happened?

The `concurrency` key creates a "group" that identifies related workflow runs:
- `group: ${{ github.workflow }}-${{ github.ref }}` creates a unique ID like "Concurrency Demo-refs/heads/main"
- Only one run per group executes at a time
- New runs wait in a pending state until the current one finishes
- `cancel-in-progress: false` means we queue (don't cancel) waiting runs

---

## Exercise 3: Enable Auto-Cancellation (10 minutes)

### Cancel old runs when new commits arrive

1. Edit `.github/workflows/concurrency.yml`.

2. Change `cancel-in-progress: false` to `cancel-in-progress: true`:

```yaml
name: Concurrency Demo

on:
  workflow_dispatch:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Simulate deployment
        run: |
          echo "Starting deployment at $(date)"
          echo "Run ID: ${{ github.run_id }}"
          echo "Concurrency group: ${{ github.workflow }}-${{ github.ref }}"
          sleep 30
          echo "Deployment complete at $(date)"
```

3. Commit the changes.

4. Trigger the workflow 3 times quickly using **Run workflow** or by pushing commits.

5. **Watch**: The first run starts. As soon as the second run queues, the first is cancelled. When the third run queues, the second is cancelled. Only the last one completes.

### What just happened?

`cancel-in-progress: true` changes the behavior from "queue and wait" to "cancel and replace":
- When a new run enters the concurrency group, any currently running workflow in that group is immediately cancelled
- This is useful for development branches where only the latest code matters
- Saves CI minutes by not running outdated workflows
- Use for: feature branches, PR checks, preview deployments
- Avoid for: production deployments, workflows with side effects

---

## Exercise 4: Use Branch-Specific Concurrency (10 minutes)

### Different behavior for different branches

1. Edit `.github/workflows/concurrency.yml`.

2. Update the `on:` section to trigger on both main and develop branches, and adjust the concurrency to be conditional:

```yaml
name: Concurrency Demo

on:
  workflow_dispatch:
  push:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Simulate deployment
        run: |
          echo "Starting deployment at $(date)"
          echo "Branch: ${{ github.ref }}"
          echo "Run ID: ${{ github.run_id }}"
          echo "Cancel enabled: ${{ github.ref != 'refs/heads/main' }}"
          sleep 30
          echo "Deployment complete at $(date)"
```

3. Commit the changes.

4. Create a `develop` branch (if it doesn't exist):
   - Go to **Code** → branch dropdown → type "develop" → **Create branch: develop**

5. Push commits to `develop` multiple times quickly, then push to `main` multiple times quickly.

6. **Watch**: Develop runs cancel each other. Main runs queue and all complete.

### What just happened?

You can use expressions in concurrency settings:
- `${{ github.ref != 'refs/heads/main' }}` evaluates to `true` for non-main branches and `false` for main
- Main branch: `cancel-in-progress: false` → runs queue and all complete (safe for production)
- Develop branch: `cancel-in-progress: true` → older runs cancelled (faster feedback)
- This pattern lets you protect production while staying fast on feature work

---

## Exercise 5: Add Job-Level Concurrency (10 minutes)

### Control concurrency for individual jobs

1. Edit `.github/workflows/concurrency.yml`.

2. Add a second job with its own concurrency settings:

```yaml
name: Concurrency Demo

on:
  workflow_dispatch:
  push:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Simulate deployment
        run: |
          echo "Starting deployment at $(date)"
          echo "Branch: ${{ github.ref }}"
          echo "Run ID: ${{ github.run_id }}"
          sleep 30
          echo "Deployment complete at $(date)"
  
  cleanup:
    runs-on: ubuntu-latest
    concurrency:
      group: cleanup-${{ github.ref }}
      cancel-in-progress: true
    
    steps:
      - name: Cleanup old artifacts
        run: |
          echo "Cleaning up artifacts at $(date)"
          echo "Branch: ${{ github.ref }}"
          sleep 20
          echo "Cleanup complete at $(date)"
```

3. Commit the changes.

4. Trigger multiple runs quickly.

5. **Watch**: Both jobs run in parallel within each workflow run. But the cleanup job has its own concurrency group, so if you trigger runs quickly, cleanup jobs can cancel independently from deploy jobs.

### What just happened?

Job-level concurrency works alongside workflow-level concurrency:
- Workflow-level concurrency controls entire workflow runs
- Job-level concurrency adds an additional layer of control per job
- Each job can have its own concurrency settings (group and cancel-in-progress)
- Useful when different jobs have different cancellation requirements
- Example: allow test jobs to cancel but not deployment jobs

---

## Exercise 6: Use Dynamic Concurrency Groups (10 minutes)

### One concurrency group per pull request

1. Edit `.github/workflows/concurrency.yml`.

2. Add `pull_request` to the triggers and update the concurrency group to handle both pushes and PRs:

```yaml
name: Concurrency Demo

on:
  workflow_dispatch:
  push:
    branches: [main, develop]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Simulate deployment
        run: |
          echo "Starting deployment at $(date)"
          echo "Branch: ${{ github.ref }}"
          echo "PR: ${{ github.event.pull_request.number || 'N/A' }}"
          echo "Run ID: ${{ github.run_id }}"
          sleep 30
          echo "Deployment complete at $(date)"
  
  cleanup:
    runs-on: ubuntu-latest
    concurrency:
      group: cleanup-${{ github.ref }}
      cancel-in-progress: true
    
    steps:
      - name: Cleanup old artifacts
        run: |
          echo "Cleaning up artifacts at $(date)"
          sleep 20
          echo "Cleanup complete at $(date)"
```

3. Commit the changes.

4. Create a pull request, then push multiple commits to that PR branch.

5. **Watch**: Each PR gets its own concurrency group (pr-123, pr-124, etc.). Commits to the same PR cancel previous runs. Different PRs don't interfere with each other.

### What just happened?

The expression `${{ github.event.pull_request.number || github.ref }}` creates flexible concurrency groups:
- For pull requests: uses PR number (pr-42, pr-43, etc.)
- For pushes: uses branch ref (main, develop, etc.)
- `||` is the "or" operator: use PR number if available, otherwise use ref
- Each PR has isolated concurrency—multiple PRs can run simultaneously
- New commits to the same PR cancel old checks

---

## What You've Learned

### Concurrency Fundamentals

**Concurrency Groups**: A unique identifier that groups related workflow runs. Only one run per group executes at a time (or cancels previous runs based on settings).

**cancel-in-progress Setting**:
- `false`: New runs wait in queue until current run completes (safe for production, deployments with side effects)
- `true`: New runs cancel currently running workflows in the same group (fast feedback for development, saves CI minutes)

### Strategic Patterns

**Workflow-Level Concurrency**: Applied at the top level, controls entire workflow runs across all jobs.

**Job-Level Concurrency**: Applied to individual jobs, adds granular control independent of workflow-level settings.

**Dynamic Groups**: Use expressions like `${{ github.event.pull_request.number || github.ref }}` to create context-aware concurrency groups.

**Branch-Specific Behavior**: Use conditional expressions `${{ github.ref != 'refs/heads/main' }}` to apply different strategies to different branches.

### Real-World Applications

**Development/Feature Branches**: 
```yaml
cancel-in-progress: true
```
Cancels old runs when new commits arrive. Developers get faster feedback on latest code.

**Pull Requests**: 
```yaml
group: pr-${{ github.event.pull_request.number }}
```
Each PR gets its own concurrency group. Multiple PRs run in parallel, but commits to the same PR cancel previous checks.

**Production Deployments**: 
```yaml
cancel-in-progress: false
```
Runs queue and all complete. Never cancel deployments that might leave systems in inconsistent states.

**Mixed Strategies**: Combine workflow-level and job-level concurrency for complex scenarios—tests can cancel, deployments queue.
- Concurrency groups identify which runs should be managed together
- `cancel-in-progress: true` for dev/PR workflows saves time and resources
- `cancel-in-progress: false` for production ensures deployments complete
- Job-level concurrency provides fine-grained control
- Use `${{ github.ref }}` for branch-specific concurrency
- Use PR numbers for PR-specific concurrency

---

## Resources

- [GitHub Actions Concurrency Documentation](https://docs.github.com/en/actions/using-jobs/using-concurrency)
- [Workflow syntax for concurrency](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#concurrency)

---

**End of Lab 2**
