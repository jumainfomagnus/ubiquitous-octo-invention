# Lab 3: Job Dependencies

**Trainer:** GitHub Actions Intermediate Training for Wells Fargo  
**Duration:** 45-60 minutes  
**Prerequisites:** Understanding of GitHub Actions jobs and workflows

---

## Learning Objectives

By the end of this lab, you will be able to:
- Chain jobs using the `needs` keyword
- Build multi-stage CI/CD pipelines
- Pass data between jobs using outputs
- Handle job failures and conditional dependencies
- Design complex job dependency graphs

---

## Exercise 1: Basic Job Chaining (10 minutes)

### Scenario
You need to create a basic CI/CD pipeline: Build → Test → Deploy. Each stage should only run if the previous stage succeeds.

### Task
Create a workflow with three dependent jobs.

### Steps

1. Create `.github/workflows/basic-dependencies.yml`

```yaml
name: Basic Job Dependencies

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build application
        run: |
          echo "Building application..."
          mkdir -p build
          echo "Build artifact" > build/app.txt
          sleep 5
          echo "Build complete!"
      
      - name: Show build info
        run: echo "Build job completed at $(date)"
  
  test:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Run tests
        run: |
          echo "Running tests..."
          sleep 5
          echo "All tests passed!"
      
      - name: Show test info
        run: echo "Test job completed at $(date)"
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy application
        run: |
          echo "Deploying application..."
          sleep 5
          echo "Deployment complete!"
      
      - name: Show deploy info
        run: echo "Deploy job completed at $(date)"
```

2. Push to trigger the workflow

3. Observe the job execution order in the Actions UI

### Questions to Answer
1. What happens if the build job fails?
2. Can test and deploy jobs run in parallel with this configuration?
3. How does GitHub Actions visualize job dependencies?

### Expected Outcome
- Jobs run sequentially: build → test → deploy
- Each job waits for its dependency to complete successfully

---

## Exercise 2: Multiple Dependencies (15 minutes)

### Scenario
Your application needs both frontend and backend to be built and tested before deployment.

### Task
Create a workflow where the deploy job depends on multiple jobs completing successfully.

### Steps

1. Create `.github/workflows/multiple-dependencies.yml`

```yaml
name: Multiple Dependencies

on:
  push:
    branches: [main]

jobs:
  build-frontend:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build frontend
        run: |
          echo "Building frontend..."
          sleep 10
          echo "Frontend build complete!"
  
  build-backend:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build backend
        run: |
          echo "Building backend..."
          sleep 10
          echo "Backend build complete!"
  
  test-frontend:
    needs: build-frontend
    runs-on: ubuntu-latest
    
    steps:
      - name: Test frontend
        run: |
          echo "Testing frontend..."
          sleep 8
          echo "Frontend tests passed!"
  
  test-backend:
    needs: build-backend
    runs-on: ubuntu-latest
    
    steps:
      - name: Test backend
        run: |
          echo "Testing backend..."
          sleep 8
          echo "Backend tests passed!"
  
  deploy:
    needs: [test-frontend, test-backend]
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy application
        run: |
          echo "Deploying full application..."
          echo "Frontend and backend both ready"
          sleep 5
          echo "Deployment successful!"
```

### Questions to Answer
1. Which jobs can run in parallel?
2. What happens if only the frontend tests fail?
3. How long does the entire workflow take to complete?

### Visualization Exercise
Draw the dependency graph for this workflow.

---

## Exercise 3: Passing Data Between Jobs (20 minutes)

### Scenario
Your build job creates a version number that needs to be used by both test and deploy jobs.

### Task
Use job outputs to pass data between jobs.

### Steps

1. Create `.github/workflows/job-outputs.yml`

```yaml
name: Job Outputs

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      build-time: ${{ steps.version.outputs.build-time }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate version
        id: version
        run: |
          VERSION="1.0.${{ github.run_number }}"
          BUILD_TIME=$(date +'%Y-%m-%d %H:%M:%S')
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "build-time=$BUILD_TIME" >> $GITHUB_OUTPUT
          echo "Generated version: $VERSION"
      
      - name: Build with version
        run: |
          echo "Building version ${{ steps.version.outputs.version }}"
          sleep 5
  
  test:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Test version
        run: |
          echo "Testing version: ${{ needs.build.outputs.version }}"
          echo "Built at: ${{ needs.build.outputs.build-time }}"
          sleep 5
          echo "Tests passed!"
  
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy version
        run: |
          echo "Deploying version: ${{ needs.build.outputs.version }}"
          echo "Built at: ${{ needs.build.outputs.build-time }}"
          sleep 5
          echo "Deployment complete!"
      
      - name: Create release tag
        run: |
          echo "Would tag release: v${{ needs.build.outputs.version }}"
```

### Questions to Answer
1. How do you define outputs in a job?
2. How do you access outputs from a dependent job?
3. What types of data can be passed between jobs?

### Challenge
Add another output for the commit SHA and use it in the deploy job.

---

## Exercise 4: Conditional Dependencies (15 minutes)

### Scenario
You want to deploy to staging automatically, but deploy to production only on tags or manual approval.

### Task
Create conditional job execution based on dependencies and context.

### Steps

1. Create `.github/workflows/conditional-dependencies.yml`

```yaml
name: Conditional Dependencies

on:
  push:
    branches: [main, develop]
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set version
        id: version
        run: |
          if [[ $GITHUB_REF == refs/tags/* ]]; then
            VERSION=${GITHUB_REF#refs/tags/}
          else
            VERSION="snapshot-${{ github.run_number }}"
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Version: $VERSION"
      
      - name: Build
        run: echo "Building ${{ steps.version.outputs.version }}"
  
  test:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Run tests
        run: |
          echo "Testing version ${{ needs.build.outputs.version }}"
          sleep 5
  
  deploy-staging:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying ${{ needs.build.outputs.version }} to staging"
          sleep 5
  
  deploy-production:
    needs: test
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying ${{ needs.build.outputs.version }} to production"
          sleep 5
          echo "Production deployment complete!"
```

### Questions to Answer
1. When does the staging deployment run?
2. When does the production deployment run?
3. Can staging and production deploy in parallel?

### Test Scenarios
1. Push to main branch - which jobs run?
2. Push to develop branch - which jobs run?
3. Create a tag v1.0.0 - which jobs run?

---

## Exercise 5: Handling Job Failures (15 minutes)

### Scenario
You want a notification job that runs whether the pipeline succeeds or fails, and a cleanup job that always runs.

### Task
Use special conditionals to handle different job outcomes.

### Steps

1. Create `.github/workflows/failure-handling.yml`

```yaml
name: Failure Handling

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Build
        run: |
          echo "Building..."
          # Uncomment to simulate failure
          # exit 1
          sleep 5
  
  test:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Test
        run: |
          echo "Testing..."
          # Uncomment to simulate failure
          # exit 1
          sleep 5
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy
        run: |
          echo "Deploying..."
          sleep 5
  
  notify:
    needs: [build, test, deploy]
    if: always()
    runs-on: ubuntu-latest
    
    steps:
      - name: Check results
        run: |
          echo "Build: ${{ needs.build.result }}"
          echo "Test: ${{ needs.test.result }}"
          echo "Deploy: ${{ needs.deploy.result }}"
      
      - name: Notify on success
        if: needs.deploy.result == 'success'
        run: echo "✅ Pipeline succeeded!"
      
      - name: Notify on failure
        if: contains(needs.*.result, 'failure')
        run: echo "❌ Pipeline failed!"
  
  cleanup:
    needs: [build, test, deploy]
    if: always()
    runs-on: ubuntu-latest
    
    steps:
      - name: Cleanup resources
        run: |
          echo "Cleaning up temporary resources..."
          echo "Cleanup complete"
```

### Questions to Answer
1. What does `if: always()` do?
2. How can you check if any dependency failed?
3. What are the possible values of `needs.<job>.result`?

### Test It
1. Run with all jobs succeeding
2. Uncomment the `exit 1` in test job and observe
3. Check what the notify job reports

### Possible Result Values
- `success` - Job completed successfully
- `failure` - Job failed
- `cancelled` - Job was cancelled
- `skipped` - Job was skipped

---

## Exercise 6: Complex Dependency Graph (20 minutes)

### Scenario
Build a realistic CI/CD pipeline with:
- Parallel linting and unit tests
- Integration tests that need both
- Separate staging and production deployments
- Security scanning in parallel with tests
- Final verification after deployment

### Task
Create a complex workflow with multiple dependency paths.

### Steps

1. Create `.github/workflows/complex-dependencies.yml`

```yaml
name: Complex Dependencies

on:
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      - name: Lint code
        run: |
          echo "Linting code..."
          sleep 5
          echo "Linting complete"
  
  unit-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      - name: Unit tests
        run: |
          echo "Running unit tests..."
          sleep 10
          echo "Unit tests passed"
  
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      - name: Security scan
        run: |
          echo "Running security scan..."
          sleep 8
          echo "No vulnerabilities found"
  
  build:
    needs: [lint, unit-test]
    runs-on: ubuntu-latest
    outputs:
      artifact-id: ${{ steps.build.outputs.artifact-id }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build application
        id: build
        run: |
          ARTIFACT_ID="build-${{ github.run_number }}"
          echo "artifact-id=$ARTIFACT_ID" >> $GITHUB_OUTPUT
          echo "Building $ARTIFACT_ID..."
          sleep 10
          echo "Build complete"
  
  integration-test:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Integration tests
        run: |
          echo "Running integration tests for ${{ needs.build.outputs.artifact-id }}"
          sleep 15
          echo "Integration tests passed"
  
  deploy-staging:
    needs: [build, integration-test, security-scan]
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying ${{ needs.build.outputs.artifact-id }} to staging"
          sleep 10
          echo "Staging deployment complete"
  
  smoke-test-staging:
    needs: deploy-staging
    runs-on: ubuntu-latest
    
    steps:
      - name: Smoke tests on staging
        run: |
          echo "Running smoke tests on staging..."
          sleep 5
          echo "Smoke tests passed"
  
  deploy-production:
    needs: smoke-test-staging
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          sleep 10
          echo "Production deployment complete"
  
  smoke-test-production:
    needs: deploy-production
    runs-on: ubuntu-latest
    
    steps:
      - name: Smoke tests on production
        run: |
          echo "Running smoke tests on production..."
          sleep 5
          echo "Smoke tests passed"
  
  notify-success:
    needs: smoke-test-production
    runs-on: ubuntu-latest
    
    steps:
      - name: Send success notification
        run: echo "🎉 Deployment successful!"
```

### Questions to Answer
1. Which jobs can run in parallel?
2. What's the critical path (longest dependency chain)?
3. How would you add a manual approval before production?

### Deliverables
1. Draw the complete dependency graph
2. Calculate minimum workflow duration (assuming times as given)
3. Identify optimization opportunities

---

## Exercise 7: Reusable Job Dependencies (15 minutes)

### Scenario
Create a reusable workflow pattern where calling workflows can depend on the reusable workflow's completion.

### Task
Create a reusable workflow and call it with dependencies.

### Steps

1. Create `.github/workflows/reusable-build.yml`

```yaml
name: Reusable Build

on:
  workflow_call:
    outputs:
      build-version:
        description: "The build version"
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate version
        id: version
        run: |
          VERSION="1.0.${{ github.run_number }}"
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Building version $VERSION"
      
      - name: Build
        run: |
          echo "Build complete"
          sleep 5
```

2. Create `.github/workflows/caller-workflow.yml`

```yaml
name: Caller Workflow

on:
  push:
    branches: [main]

jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
  
  test:
    needs: call-build
    runs-on: ubuntu-latest
    
    steps:
      - name: Test with version
        run: |
          echo "Testing version: ${{ needs.call-build.outputs.build-version }}"
          sleep 5
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy
        run: |
          echo "Deploying version: ${{ needs.call-build.outputs.build-version }}"
          sleep 5
```

### Questions to Answer
1. How do reusable workflows integrate with job dependencies?
2. How do you access outputs from reusable workflows?
3. What are the benefits of this pattern?

---

## Best Practices Summary

### ✅ DO
- Use clear, descriptive job names
- Chain jobs only when truly dependent
- Pass data through outputs, not artifacts (for small data)
- Use `if: always()` for cleanup and notification jobs
- Document complex dependency graphs
- Keep dependency chains as flat as possible

### ❌ DON'T
- Create unnecessary dependencies (slows pipeline)
- Make all jobs depend on one another (no parallelism)
- Forget to handle failure cases
- Create circular dependencies (impossible)
- Over-complicate with too many dependency levels

---

## Verification Checklist

- [ ] Jobs execute in correct order
- [ ] Parallel jobs run simultaneously when possible
- [ ] Outputs pass correctly between jobs
- [ ] Failed dependencies prevent downstream jobs
- [ ] Conditional dependencies work as expected
- [ ] Cleanup jobs run regardless of success/failure

---

## Troubleshooting Guide

**Problem:** Job doesn't wait for dependency  
**Solution:** Verify `needs:` is properly configured

**Problem:** Can't access output from previous job  
**Solution:** Check output is defined in job outputs section

**Problem:** Jobs running when they shouldn't  
**Solution:** Verify conditionals and use `needs.<job>.result` checks

**Problem:** Pipeline too slow  
**Solution:** Review dependencies and parallelize independent jobs

---

## Key Takeaways

- `needs:` creates job dependencies and execution order
- Jobs without dependencies run in parallel
- Use job outputs to pass data between jobs
- `if: always()` ensures jobs run regardless of failures
- Multiple dependencies with `needs: [job1, job2]` creates AND logic
- Job results can be checked with `needs.<job>.result`
- Complex pipelines need careful design to balance speed and safety

---

## Resources

- [Using jobs in GitHub Actions](https://docs.github.com/en/actions/using-jobs)
- [Defining outputs for jobs](https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs)
- [Job status check functions](https://docs.github.com/en/actions/learn-github-actions/expressions#status-check-functions)

---

**End of Lab 3**
