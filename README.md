# Module 4 - Edric's Coaching Week 2

## Agenda

1. Case Study Analysis

   1. [CircleCI Incident Report (Jan 4, 2023)](https://circleci.com/blog/jan-4-2023-incident-report/) - Review some of the best practices recommended in the report
   2. [GitHub Actions Supply Chain Attack](https://unit42.paloaltonetworks.com/github-actions-supply-chain-attack/) - Security implications and lessons

2. CircleCI Concepts and Commands

   1. Best practices for CI/CD implementation
   2. Key commands and configuration options

3. CircleCI Checks Implementation

   1. Setting up automated checks
   2. Configuration and customization

4. GitHub Actions Overview
   1. Comparison with CircleCI
   2. Key differences and use cases

## Case Study Analysis

### CircleCI Incident Report (Jan 4, 2023)

I'll extract the key information from the CircleCI incident report and provide a summarized version covering the requested points.

### 1. What Happened?

In January 2023, CircleCI experienced a security breach where an unauthorized third party gained access to customer data. The timeline:

- December 16, 2022: A CircleCI engineer's laptop was compromised with undetected malware
- December 19, 2022: The attacker conducted reconnaissance (scouting/spying) activities
- December 22, 2022: Customer data was exfiltrated, including encrypted environment variables, tokens, and keys
- December 29, 2022: CircleCI was alerted to suspicious GitHub OAuth activity
- January 4, 2023: CircleCI disclosed the incident and began remediation

### 2. Root Cause

The root cause was malware deployed on a CircleCI engineer's laptop that:

- Stole a valid 2FA-backed SSO session
- Allowed the attacker to impersonate the employee remotely
- Enabled the attacker to escalate access to production systems
- Gave access to generate production tokens (part of the employee's regular duties)
- Allowed extraction of encryption keys from running processes, enabling decryption of the stolen data

### 3. Learning and Takeaway

From the incident, CircleCI learned:

1. **Systems Over Individuals**: While one employee's device was compromised, the incident reflects a systems failure rather than individual error. Organizations must design security systems that prevent escalation even when individual endpoints are compromised.

2. **Limit Privileged Access**: The extensive damage was possible because the compromised account had broad production access. More restrictive and ephemeral permissions, along with additional authentication steps for production access, could have limited the impact.

3. **Monitor and Respond to Unusual Activity**: Better monitoring and alerting for suspicious behavior patterns could have detected the unauthorized access sooner, particularly during the 3-day gap between reconnaissance and data exfiltration.

### 4. Recommended Actions from CircleCI

For customers:

- Rotate all secrets (OAuth tokens, API tokens, SSH keys, etc.)
- Use OIDC tokens where possible to avoid storing long-lived credentials
- Take advantage of IP ranges to limit inbound connections
- Use [Contexts](https://circleci.com/docs/contexts/) to consolidate shared secrets and restrict access
- Consider using [runners](https://circleci.com/docs/runner-overview/#circleci-runner-use-cases) for privileged access and additional controls

For CircleCI's internal improvements:

- Implement periodic automatic OAuth token rotation
- Make system permissions more ephemeral
- Shift from OAuth to GitHub apps for more granular permissions
- Reduce session trust and add additional authentication factors

### Supply Chain Attack on GitHub Actions - (March 2025)

#### 1. What Happened?

1. Attackers compromised `tj-actions/changed-files` and reviewdog actions, affecting 23,000+ repositories
2. The attack extracted CI/CD secrets from memory, exposing them in workflow logs
3. This sophisticated attack spanned from November 2024 to March 2025

#### 2. Root Cause

1. Initial exploitation of a `pull_request_target` trigger leaked a maintainer's PAT
2. Attackers used a chain of compromised tokens to move through multiple repositories
3. Techniques included commit impersonation, git tag manipulation, and deleted accounts

#### 3. Key Learnings

1. Dependencies between actions create dangerous compromise chains
2. Unpinned GitHub actions (by tag instead of commit hash) are vulnerable
3. Git tag changes don't appear in audit logs for free-tier GitHub accounts

**Unpinned (Unsafe)**:

```yaml
- uses: tj-actions/changed-files@v39 # References a tag that can be modified
- uses: tj-actions/changed-files@main # References a branch that can change
```

**Pinned (Safe)**:

```yaml
- uses: tj-actions/changed-files@a084acbbf7109ce39622e9eaaafafc31a5a6b7ef # Full commit hash
```

Why pinning matters:

- Tags and branches are mutable references that can be modified to point to different commits
- In this attack, the hackers changed the `v39` tag to point to malicious code
- Even repositories that hadn't updated their workflows were affected since the tag itself was changed
- When pinned to a full commit hash, the exact version is immutable and cannot be changed
- Had repositories pinned to specific commit hashes, they would have been protected from this attack

#### 4. Recommended Actions

1. Pin actions to specific commit hashes and implement strict pipeline access controls
2. Replace long-term credentials with short-lived tokens and rotate secrets regularly
3. Avoid using `pull_request_target` triggers and implement monitoring for suspicious activity

## Recap of CircleCI Concepts and Commands

| Concept       | Description                                                           |
| ------------- | --------------------------------------------------------------------- |
| **Jobs**      | Individual units of work that run commands in a specified environment |
| **Workflows** | Orchestration of jobs, defining their order and dependencies          |
| **Orbs**      | Reusable packages of CircleCI configuration (like plugins)            |
| **Executors** | Environments where jobs run (Docker, machine, macos)                  |
| **Steps**     | Commands or actions executed within a job                             |
| **Pipelines** | Complete set of workflows triggered by a code change                  |
| **Contexts**  | Secure environment variable storage across projects                   |
| **Filters**   | Conditions that determine when jobs run (branches, tags)              |

```yaml
version: 2.1

orbs:
  node: circleci/node@5.0.1
  docker: circleci/docker@2.1.4
  snyk: snyk/snyk@1.5.0

jobs:
  build:
    docker:
      - image: cimg/node:16.10
    steps:
      - checkout
      - node/install-packages
      - run: echo "Simulate build"

  test:
    docker:
      - image: cimg/node:16.10
    steps:
      - checkout
      - node/install-packages
      - run: npm run test

  publish:
    docker:
      - image: cimg/node:16.10
    steps:
      - setup_remote_docker
      - checkout
      - docker/build:
          image: huai99/education-space
      - docker/push:
          image: huai99/education-space

  security-scan:
    docker:
      - image: cimg/node:16.10
    steps:
      - checkout
      - setup_remote_docker
      - run: docker build -t huai99/education-space .
      - snyk/scan:
          docker-image-name: huai99/education-space
          fail-on-issues: false

workflows:
  main-workflow:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          requires:
            - test
      - security-scan:
          requires:
            - build
          filters:
            branches:
              only: main
```

## GitHub Actions Overview

| Concept         | Description                                                                                                              |
| --------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Workflow**    | A configurable automated process made up of one or more jobs, defined in YAML files in the `.github/workflows` directory |
| **Events**      | Triggers that start workflows (e.g., push, pull request, schedule)                                                       |
| **Jobs**        | A set of steps executed on the same runner; jobs can run in parallel or sequentially                                     |
| **Steps**       | Individual tasks within a job that can run commands or actions                                                           |
| **Actions**     | Reusable, modular units that perform specific tasks                                                                      |
| **Runners**     | Servers that run workflows (GitHub-hosted or self-hosted)                                                                |
| **Environment** | A specific deployment target with protection rules and secrets                                                           |
| **Secrets**     | Encrypted environment variables for sensitive information                                                                |
| **Artifacts**   | Files produced during a workflow that can be shared between jobs                                                         |

## Workflow Flow

GitHub Actions follows this general flow:

1. **Trigger**: An event (push, PR, etc.) initiates the workflow
2. **Workflow Execution**: GitHub creates the environment specified in the workflow
3. **Job Execution**: Jobs run in parallel by default, or sequentially if dependencies are specified
4. **Step Execution**: Each step in a job runs sequentially
5. **Completion**: The workflow succeeds if all jobs succeed

## Explanation of the Example Workflow

### Workflow Triggers

```yaml
on:
  workflow_dispatch: # Manual trigger via GitHub UI
  push: # Automatically triggered on push to main branch
    branches: [main]
  pull_request: # Automatically triggered on PR to main branch
    branches: [main]
```

### Build Job

```yaml
build:
  runs-on: ubuntu-latest # Uses GitHub-hosted Ubuntu runner
  steps:
    - uses: actions/checkout@v3 # Checks out repository code
    - name: Set up Node.js
      uses: actions/setup-node@v3 # Sets up Node.js environment
      with:
        node-version: "16.x"
        cache: "npm" # Caches npm dependencies
    - name: Install dependencies
      run: npm ci # Installs dependencies via npm
    - name: Simulate build
      run: echo "Simulate build" # Placeholder for actual build command
```

### Test Job

```yaml
test:
  needs: build # This job runs after the build job completes
  runs-on: ubuntu-latest
  steps:
    # Similar setup to build job
    - uses: actions/checkout@v3
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: "16.x"
        cache: "npm"
    - name: Install dependencies
      run: npm ci
    - name: Run tests
      run: |
        echo "Running tests..."
        npm run test  # Runs the test script from package.json
```

### Publish Job

```yaml
publish:
  needs: test # This job runs after the test job completes
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2 # Sets up Docker build system
    - name: Login to Docker Hub
      uses: docker/login-action@v2 # Logs in to Docker Hub
      with:
        username: ${{ secrets.DOCKER_HUB_USERNAME }} # Uses repository secrets
        password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
    - name: Build and push
      uses: docker/build-push-action@v4 # Builds and pushes Docker image
      with:
        context: .
        push: true
        tags: huai99/education-space:latest # Tags the Docker image
```

## Key Differences from CircleCI

1. **File Location**: GitHub Actions workflows are stored in `.github/workflows/` directory, while CircleCI uses `.circleci/config.yml`
2. **Actions vs Orbs**: GitHub uses "Actions" while CircleCI uses "Orbs" for reusable components
3. **Runners vs Executors**: GitHub calls execution environments "Runners", CircleCI calls them "Executors"
4. **Syntax**: The YAML structure differs slightly between platforms
5. **Default Behavior**: GitHub Actions jobs run in parallel by default, while CircleCI workflows need explicit parallelism configuration

## Best Practices

1. **Use Dependency Caching**: Speed up workflows by caching dependencies (as shown with `cache: "npm"`)
2. **Store Secrets Securely**: Use GitHub's repository secrets for sensitive information
3. **Minimize Job Dependencies**: Only use `needs` when a job truly depends on another
5. **Use Community Actions**: Leverage existing actions from the GitHub Marketplace
7. **Set Timeout Limits**: Prevent workflows from running too long by setting timeout limits
8. **Use Artifacts**: Share data between jobs by uploading and downloading artifacts
