# 🚀 From Merge to Production — A Complete Detailed Guide

## How Code Actually Moves from PR Merge → Staging → Approval → Production

---

## Table of Contents

1. [Overview — The Big Picture](#1-overview--the-big-picture)
2. [Step 1: Merge to Main Branch](#step-1-merge-main)
3. [Step 2: CD Pipeline Triggers Automatically](#step-2-cd-triggers)
4. [Step 3: Deploy to Staging Environment](#step-3-deploy-staging)
5. [Step 4: Automated Testing on Staging](#step-4-automated-testing)
6. [Step 5: QA & Exploratory Testing](#step-5-qa)
7. [Step 6: Approval Gate](#step-6-approval-gate)
8. [Step 7: Deploy to Production](#step-7-deploy-production)
9. [Step 8: Post-Deployment Verification](#step-8-post-deploy)
10. [Step 9: Rollback Strategy](#step-9-rollback)
11. [Real-World Pipeline Examples](#real-world-pipeline-examples)
12. [Tools Involved at Each Step](#tools-involved)
13. [Common Patterns & Best Practices](#best-practices)

---

<a id="1-overview--the-big-picture"></a>
<a id="overview"></a>
## 1. Overview — The Big Picture

```
Developer's PR Approved
        │
        ▼
┌─────────────────┐
│  Merge to Main  │ ◄── Squash merge / merge commit
└────────┬────────┘
         │  (auto-trigger)
         ▼
┌─────────────────┐
│   CD Pipeline   │ ◄── Webhook detects new commit on main
│   Kicks Off     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Build & Push   │ ◄── Build Docker image, tag with Git SHA
│  Docker Image   │     Push to container registry (ECR/GCR/DockerHub)
└────────┬────────┘
         │
         ▼
┌─────────────────────┐
│  Deploy to Staging  │ ◄── Kubernetes/Helm/ArgoCD updates staging cluster
└────────┬────────────┘
         │
         ▼
┌────────────────────────┐
│ Automated Tests Run    │ ◄── Integration, E2E, performance, security tests
│ on Staging Environment │
└────────┬───────────────┘
         │
         ▼
┌──────────────────────┐
│  QA Exploratory      │ ◄── Manual QA testing, visual checks
│  Testing (Optional)  │
└────────┬─────────────┘
         │
         ▼
┌─────────────────┐
│  Approval Gate  │ ◄── Tech Lead / Release Manager reviews & approves
└────────┬────────┘
         │  (manual click OR automated after checks pass)
         ▼
┌──────────────────────┐
│ Deploy to Production │ ◄── Canary: 5% → 25% → 50% → 100%
│ (Canary Rollout)     │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Post-Deploy Monitor  │ ◄── Watch error rates, latency, dashboards
│ & Verification       │
└──────────────────────┘
```

### Why This Flow Exists

| Concern | How This Flow Addresses It |
|---------|---------------------------|
| **Bad code reaching production** | Multiple test stages catch bugs before prod |
| **Downtime during deployment** | Canary/rolling deployments = zero downtime |
| **Human error** | Automation handles deployment, not humans |
| **Accountability** | Approval gates create an audit trail |
| **Fast rollback** | Canary approach allows instant rollback |
| **Consistency** | Same pipeline runs every time — no "works on my machine" |

---

<a id="step-1-merge-main"></a>
## 2. Step 1: Merge to Main Branch

### What Actually Happens

When a Pull Request is approved and all CI checks pass, the developer (or the reviewer) clicks **"Merge"** on the PR.

### Merge Strategies

```
┌──────────────────────────────────────────────────────────┐
│                    MERGE STRATEGIES                       │
├──────────────────┬───────────────────────────────────────┤
│                  │                                       │
│  Merge Commit    │  Creates a merge commit that joins    │
│  (default)       │  the feature branch into main.        │
│                  │  Preserves full branch history.       │
│                  │                                       │
│  git merge       │  main: A─B─C─────M                   │
│                  │             \   /                     │
│                  │  feature:    D─E                      │
│                  │                                       │
├──────────────────┼───────────────────────────────────────┤
│                  │                                       │
│  Squash & Merge  │  Combines ALL feature branch commits  │
│  (most common)   │  into ONE single commit on main.      │
│                  │  Clean, linear history.               │
│                  │                                       │
│  git merge       │  main: A─B─C─S                       │
│  --squash        │  (S = squashed D+E into one)          │
│                  │                                       │
├──────────────────┼───────────────────────────────────────┤
│                  │                                       │
│  Rebase & Merge  │  Replays feature commits on top of    │
│                  │  main. Linear history, no merge       │
│                  │  commit.                              │
│                  │                                       │
│  git rebase      │  main: A─B─C─D'─E'                   │
│                  │                                       │
└──────────────────┴───────────────────────────────────────┘
```

### What Happens Behind the Scenes

```
STEP-BY-STEP: MERGE TO MAIN

1. Developer clicks "Merge Pull Request" on GitHub/GitLab
       │
2. GitHub/GitLab performs the merge on the server
       │
3. A new commit SHA is created on the `main` branch
       │  Example: commit abc123def456
       │
4. The feature branch is optionally deleted
       │
5. A WEBHOOK fires to notify external systems
       │  POST https://ci-server.com/webhook
       │  Payload: { "ref": "refs/heads/main", "after": "abc123def456", ... }
       │
6. CI/CD server (Jenkins/GitHub Actions/GitLab CI) receives the webhook
       │
7. CI/CD server detects: "New commit on `main` branch → trigger CD pipeline"
```

### GitHub Webhook Payload (Simplified)

```json
{
  "ref": "refs/heads/main",
  "before": "previous_commit_sha",
  "after": "abc123def456789",
  "repository": {
    "full_name": "mycompany/myapp"
  },
  "pusher": {
    "name": "developer-name",
    "email": "dev@company.com"
  },
  "head_commit": {
    "id": "abc123def456789",
    "message": "feat: add user dashboard (#1234)",
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

### Branch Protection Rules (Pre-Merge Guards)

Before the merge can even happen, these rules must be satisfied:

```yaml
# GitHub Branch Protection Rules for `main`
branch_protection:
  required_reviews: 2                    # At least 2 approvals
  dismiss_stale_reviews: true            # New commits invalidate old approvals
  require_code_owner_review: true        # Code owners must approve
  required_status_checks:
    - "ci/build"                         # Build must pass
    - "ci/unit-tests"                    # Unit tests must pass
    - "ci/lint"                          # Linting must pass
    - "security/snyk"                    # Security scan must pass
    - "quality/sonarqube"                # Code quality must pass
  require_linear_history: true           # Squash merges only
  restrict_pushes: true                  # No direct pushes to main
  require_signed_commits: false          # Optional
```

**Why this matters:**
- Nobody can push directly to `main` — everything goes through PRs
- Code must be reviewed by at least 2 people
- All automated checks must pass — no exceptions
- Creates an audit trail of who approved what

---

<a id="step-2-cd-triggers"></a>
## 3. Step 2: CD Pipeline Triggers Automatically

### How the Pipeline Knows to Start

```yaml
# GitHub Actions — Trigger on push to main
name: CD Pipeline

on:
  push:
    branches:
      - main        # ← Only triggers when code lands on main

# GitLab CI equivalent
# rules:
#   - if: '$CI_COMMIT_BRANCH == "main"'
```

### Pipeline Trigger Mechanisms

```
┌─────────────────────────────────────────────────────┐
│              HOW PIPELINES GET TRIGGERED             │
├────────────────┬────────────────────────────────────┤
│ Webhook        │ Git server sends HTTP POST to CI   │
│ (most common)  │ server when push event happens     │
├────────────────┼────────────────────────────────────┤
│ Polling        │ CI server periodically checks Git  │
│ (legacy)       │ for new commits (every 1-5 min)    │
├────────────────┼────────────────────────────────────┤
│ Manual         │ Someone clicks "Run Pipeline"      │
│                │ in the CI/CD UI                     │
├────────────────┼────────────────────────────────────┤
│ API Call       │ External system triggers via REST   │
│                │ API call to CI/CD server            │
├────────────────┼────────────────────────────────────┤
│ Scheduled      │ Cron-based triggers (nightly        │
│ (cron)         │ builds, periodic deployments)       │
└────────────────┴────────────────────────────────────┘
```

### What the CD Pipeline Does (Step by Step)

```
CD PIPELINE STAGES:

┌──────────────────────────────────────────────────────────┐
│ STAGE 1: CHECKOUT & PREPARE                              │
│                                                          │
│  • Clone the repository at the specific commit SHA       │
│  • Set environment variables                             │
│  • Determine the version/tag for this build              │
│  • Example: v2.4.1 or commit SHA abc123d                 │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│ STAGE 2: BUILD DOCKER IMAGE                              │
│                                                          │
│  • Run `docker build` using the Dockerfile               │
│  • Tag the image with:                                   │
│    - Git commit SHA:  myapp:abc123d                       │
│    - Semantic version: myapp:v2.4.1                       │
│    - Branch name:     myapp:main                          │
│  • Multi-stage build to minimize image size              │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│ STAGE 3: SECURITY SCAN THE IMAGE                         │
│                                                          │
│  • Scan Docker image for CVEs (Common Vulnerabilities)   │
│  • Tool: Trivy, Snyk Container, Aqua                     │
│  • Fail pipeline if CRITICAL/HIGH vulnerabilities found  │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│ STAGE 4: PUSH IMAGE TO CONTAINER REGISTRY                │
│                                                          │
│  • Push to ECR/GCR/Docker Hub/GitLab Registry            │
│  • Image is now available for any environment to pull    │
│  • Registry URL: 123456789.dkr.ecr.us-east-1.amazonaws  │
│    .com/myapp:abc123d                                    │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│ STAGE 5: UPDATE DEPLOYMENT MANIFEST                      │
│                                                          │
│  • Update Kubernetes YAML / Helm values with new image   │
│  • If using GitOps (ArgoCD): commit updated manifest     │
│    to the GitOps repo                                    │
│  • If using direct deploy: kubectl apply or helm upgrade │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
              DEPLOY TO STAGING
```

### Actual Pipeline Code (GitHub Actions)

```yaml
name: CD Pipeline

on:
  push:
    branches: [main]

env:
  REGISTRY: 123456789.dkr.ecr.us-east-1.amazonaws.com
  IMAGE_NAME: myapp
  CLUSTER_NAME: my-eks-cluster
  REGION: us-east-1

jobs:
  # ──────────────────────────────────────────
  # STAGE 1: Build & Push Docker Image
  # ──────────────────────────────────────────
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.REGION }}

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Generate image tag
        id: meta
        run: |
          SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)
          echo "tags=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${SHORT_SHA}" >> $GITHUB_OUTPUT
          echo "short-sha=${SHORT_SHA}" >> $GITHUB_OUTPUT

      - name: Build Docker image
        run: |
          docker build -t ${{ steps.meta.outputs.tags }} .

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.meta.outputs.tags }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'  # Fail pipeline if vulnerabilities found

      - name: Push image to ECR
        run: |
          docker push ${{ steps.meta.outputs.tags }}

  # ──────────────────────────────────────────
  # STAGE 2: Deploy to Staging
  # ──────────────────────────────────────────
  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging   # ← GitHub Environment (can have rules)
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure kubectl
        uses: aws-actions/amazon-eks@v1
        with:
          cluster-name: ${{ env.CLUSTER_NAME }}-staging

      - name: Deploy to staging via Helm
        run: |
          helm upgrade --install myapp ./helm-chart \
            --namespace staging \
            --set image.tag=${{ needs.build-and-push.outputs.image-tag }} \
            --set environment=staging \
            --wait \
            --timeout 300s

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/myapp -n staging --timeout=120s

  # ──────────────────────────────────────────
  # STAGE 3: Run Integration Tests on Staging
  # ──────────────────────────────────────────
  integration-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run API integration tests
        run: |
          npm run test:integration -- --base-url=https://staging.myapp.com

      - name: Run E2E tests with Cypress
        uses: cypress-io/github-action@v6
        with:
          config: baseUrl=https://staging.myapp.com

      - name: Run performance tests
        run: |
          k6 run tests/performance/load-test.js \
            --env BASE_URL=https://staging.myapp.com

  # ──────────────────────────────────────────
  # STAGE 4: Approval Gate (manual)
  # ──────────────────────────────────────────
  # (Handled by the `production` environment configuration)

  # ──────────────────────────────────────────
  # STAGE 5: Deploy to Production (Canary)
  # ──────────────────────────────────────────
  deploy-production:
    needs: integration-tests
    runs-on: ubuntu-latest
    environment: production  # ← Has required reviewers configured
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure kubectl for production
        uses: aws-actions/amazon-eks@v1
        with:
          cluster-name: ${{ env.CLUSTER_NAME }}-production

      - name: Canary deployment - 5%
        run: |
          helm upgrade --install myapp ./helm-chart \
            --namespace production \
            --set image.tag=${{ needs.build-and-push.outputs.image-tag }} \
            --set canary.enabled=true \
            --set canary.weight=5

      - name: Monitor canary (5 minutes)
        run: |
          sleep 300
          # Check error rate
          ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status=~'5..'}[5m])" | jq '.data.result[0].value[1]')
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high: $ERROR_RATE"
            exit 1
          fi

      - name: Promote canary to 50%
        run: |
          helm upgrade myapp ./helm-chart \
            --namespace production \
            --set canary.weight=50 \
            --reuse-values

      - name: Monitor canary (5 minutes)
        run: |
          sleep 300
          # Same health checks...

      - name: Full rollout - 100%
        run: |
          helm upgrade myapp ./helm-chart \
            --namespace production \
            --set canary.enabled=false \
            --set image.tag=${{ needs.build-and-push.outputs.image-tag }}

      - name: Verify production deployment
        run: |
          kubectl rollout status deployment/myapp -n production --timeout=180s

      - name: Notify team
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ myapp deployed to production: ${{ github.sha }}"
            }
```

---

<a id="step-3-deploy-staging"></a>
## 4. Step 3: Deploy to Staging Environment

### What is the Staging Environment?

```
┌─────────────────────────────────────────────────────────┐
│                STAGING ENVIRONMENT                       │
│                                                         │
│  A near-exact replica of production that is used for    │
│  final validation before code reaches real users.       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Same Kubernetes cluster configuration           │    │
│  │  Same database schema (with anonymized data)     │    │
│  │  Same environment variables (staging values)     │    │
│  │  Same networking setup (VPC, load balancers)     │    │
│  │  Same monitoring & logging stack                 │    │
│  │  Same third-party service integrations (sandbox) │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  Differences from Production:                           │
│  • Smaller scale (fewer replicas/instances)              │
│  • Anonymized/synthetic data (no real user data)         │
│  • Third-party services in sandbox/test mode             │
│  • Lower resource limits (cost savings)                  │
│  • Not publicly accessible                               │
└─────────────────────────────────────────────────────────┘
```

### How Deployment to Staging Works

There are several approaches. Here are the most common:

#### Approach 1: Direct Deployment (kubectl / Helm)

```
Pipeline                  Kubernetes Staging Cluster
   │                              │
   │  helm upgrade --install      │
   │  myapp ./chart               │
   │  --set image.tag=abc123d     │
   │ ────────────────────────────►│
   │                              │
   │                    ┌─────────┴──────────┐
   │                    │ K8s performs:       │
   │                    │ 1. Pull new image   │
   │                    │ 2. Create new pods  │
   │                    │ 3. Health checks    │
   │                    │ 4. Route traffic    │
   │                    │ 5. Kill old pods    │
   │                    └─────────┬──────────┘
   │                              │
   │  rollout status ✅           │
   │ ◄────────────────────────────│
```

```bash
# Helm deployment command
helm upgrade --install myapp ./helm-chart \
  --namespace staging \
  --set image.repository=123456789.dkr.ecr.us-east-1.amazonaws.com/myapp \
  --set image.tag=abc123d \
  --set replicaCount=2 \
  --set environment=staging \
  --set resources.limits.cpu=500m \
  --set resources.limits.memory=512Mi \
  --values ./helm-chart/values-staging.yaml \
  --wait \
  --timeout 300s

# Verify
kubectl rollout status deployment/myapp -n staging --timeout=120s
kubectl get pods -n staging -l app=myapp
```

#### Approach 2: GitOps with ArgoCD

```
Pipeline                    GitOps Repo               ArgoCD              K8s Cluster
   │                            │                       │                     │
   │  1. Update image tag       │                       │                     │
   │  in values.yaml            │                       │                     │
   │ ──────────────────────────►│                       │                     │
   │                            │                       │                     │
   │  2. Commit & push          │  3. ArgoCD detects    │                     │
   │                            │  new commit (polls    │                     │
   │                            │  every 3 min or       │                     │
   │                            │  webhook)             │                     │
   │                            │ ─────────────────────►│                     │
   │                            │                       │                     │
   │                            │                       │  4. Syncs desired   │
   │                            │                       │  state to cluster   │
   │                            │                       │ ───────────────────►│
   │                            │                       │                     │
   │                            │                       │  5. Reports status  │
   │                            │                       │ ◄────────────────── │
   │                            │                       │     Healthy ✅      │
```

```bash
# In the CD pipeline, update the GitOps repo:
git clone https://github.com/mycompany/gitops-manifests.git
cd gitops-manifests

# Update the image tag in the staging values file
yq eval '.image.tag = "abc123d"' -i apps/myapp/staging/values.yaml

# Commit and push
git add .
git commit -m "deploy: myapp staging abc123d"
git push origin main

# ArgoCD automatically detects this change and deploys to staging
```

#### What Happens Inside Kubernetes During Deployment

```
KUBERNETES ROLLING UPDATE PROCESS:

Before deployment:
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Pod v1.0│ │ Pod v1.0│ │ Pod v1.0│   ← 3 replicas running v1.0
└─────────┘ └─────────┘ └─────────┘

Step 1: K8s creates new pod with v2.0
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Pod v1.0│ │ Pod v1.0│ │ Pod v1.0│ │ Pod v2.0│  ← New pod starting
└─────────┘ └─────────┘ └─────────┘ └────┬────┘
                                          │
                                    Readiness probe
                                    checks health
                                          │
Step 2: v2.0 pod is healthy → terminate one v1.0 pod
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Pod v1.0│ │ Pod v1.0│ │ Pod v2.0│  ← 1 old pod removed
└─────────┘ └─────────┘ └─────────┘

Step 3: Repeat until all pods are v2.0
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Pod v2.0│ │ Pod v2.0│ │ Pod v2.0│  ← All pods now v2.0 ✅
└─────────┘ └─────────┘ └─────────┘
```

---

<a id="step-4-automated-testing"></a>
## 5. Step 4: Automated Testing on Staging

### Types of Tests That Run on Staging

```
┌──────────────────────────────────────────────────────────────┐
│              AUTOMATED TESTS ON STAGING                       │
├──────────────────┬───────────────────────────────────────────┤
│                  │                                           │
│  Integration     │  Test how different services/components   │
│  Tests           │  work together (API calls, DB queries,    │
│                  │  message queues)                          │
│                  │                                           │
│  Example:        │  "When user creates an order via API,     │
│                  │   does it save to DB AND send email AND   │
│                  │   update inventory service?"              │
│                  │                                           │
├──────────────────┼───────────────────────────────────────────┤
│                  │                                           │
│  End-to-End      │  Simulate real user workflows in a real   │
│  (E2E) Tests     │  browser — click buttons, fill forms,    │
│                  │  verify UI behavior                      │
│                  │                                           │
│  Example:        │  "Login → Add item to cart → Checkout →   │
│                  │   Verify order confirmation page"         │
│                  │                                           │
│  Tools:          │  Cypress, Playwright, Selenium            │
│                  │                                           │
├──────────────────┼───────────────────────────────────────────┤
│                  │                                           │
│  API Contract    │  Verify API responses match expected      │
│  Tests           │  schemas/contracts between services       │
│                  │                                           │
│  Tools:          │  Pact, Postman/Newman, REST Assured       │
│                  │                                           │
├──────────────────┼───────────────────────────────────────────┤
│                  │                                           │
│  Performance     │  Load testing — can the app handle        │
│  Tests           │  expected traffic without degradation?    │
│                  │                                           │
│  Tools:          │  k6, JMeter, Gatling, Locust              │
│                  │                                           │
├──────────────────┼───────────────────────────────────────────┤
│                  │                                           │
│  Security        │  DAST scanning — test running app for     │
│  Tests           │  security vulnerabilities (XSS, SQL      │
│                  │  injection, etc.)                         │
│                  │                                           │
│  Tools:          │  OWASP ZAP, Burp Suite                    │
│                  │                                           │
├──────────────────┼───────────────────────────────────────────┤
│                  │                                           │
│  Smoke Tests     │  Quick sanity checks — "Is the app       │
│                  │  alive? Can users login? Do main          │
│                  │  features work?"                          │
│                  │                                           │
└──────────────────┴───────────────────────────────────────────┘
```

### Test Execution Flow

```
Staging Deployment Complete
         │
         ▼
┌─────────────────┐
│   Smoke Tests   │ ← Quick (30 seconds) — Is the app alive?
│   (health check)│   GET /health → 200 OK?
└────────┬────────┘
         │ PASS ✅
         ▼
┌─────────────────┐
│  API Integration│ ← Medium (5 minutes) — Do APIs work correctly?
│  Tests          │   POST /api/users → 201 Created?
└────────┬────────┘
         │ PASS ✅
         ▼
┌─────────────────┐
│  E2E Browser    │ ← Slower (10-20 minutes) — Full user flows
│  Tests (Cypress)│   Login → Navigate → Perform actions → Verify
└────────┬────────┘
         │ PASS ✅
         ▼
┌─────────────────┐
│  Performance    │ ← Slow (10-15 minutes) — Load test
│  Tests (k6)    │   Simulate 1000 concurrent users
└────────┬────────┘
         │ PASS ✅
         ▼
┌─────────────────┐
│  Security Scan  │ ← Slow (15-30 minutes) — DAST scanning
│  (OWASP ZAP)   │   Automated vulnerability scan
└────────┬────────┘
         │ PASS ✅
         ▼
   ALL TESTS PASSED
   → Notify team
   → Ready for approval
```

### Example: Test Results Report

```
╔══════════════════════════════════════════════════════════╗
║           STAGING TEST RESULTS — Build abc123d           ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  🟢 Smoke Tests           4/4 passed    (0.3 min)       ║
║  🟢 Unit Tests          127/127 passed  (2.1 min)       ║
║  🟢 Integration Tests    42/42 passed   (4.7 min)       ║
║  🟢 E2E Tests            18/18 passed   (12.3 min)      ║
║  🟢 Performance Tests     PASS          (8.5 min)       ║
║     ├─ Avg Response Time:  142ms (threshold: <500ms)     ║
║     ├─ P99 Latency:        380ms (threshold: <1000ms)    ║
║     ├─ Error Rate:         0.02% (threshold: <1%)        ║
║     └─ Throughput:         2,340 req/sec                  ║
║  🟢 Security Scan          PASS          (18.2 min)      ║
║     ├─ Critical:           0                              ║
║     ├─ High:               0                              ║
║     ├─ Medium:             2 (accepted)                   ║
║     └─ Low:                5                              ║
║                                                          ║
║  Total Time: 46.1 minutes                                ║
║  Status: ✅ ALL CHECKS PASSED                            ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

<a id="step-5-qa"></a>
## 6. Step 5: QA & Exploratory Testing

### What QA Does on Staging

```
┌──────────────────────────────────────────────────────────┐
│              QA EXPLORATORY TESTING                       │
│                                                          │
│  While automated tests catch known scenarios, QA testers │
│  explore the application to find issues that automated   │
│  tests might miss.                                       │
│                                                          │
│  Activities:                                             │
│  ┌─────────────────────────────────────────────────┐     │
│  │ • Test edge cases and unusual user behaviors     │     │
│  │ • Verify UI/UX looks correct (visual testing)    │     │
│  │ • Test on different browsers/devices             │     │
│  │ • Check accessibility (screen readers, etc.)     │     │
│  │ • Test error handling (invalid inputs)           │     │
│  │ • Verify feature matches acceptance criteria     │     │
│  │ • Check for regression in existing features      │     │
│  │ • Test third-party integrations                  │     │
│  └─────────────────────────────────────────────────┘     │
│                                                          │
│  If issues found:                                        │
│  → Minor: Log bug ticket, proceed with deployment        │
│  → Major: BLOCK deployment, fix required                 │
│  → Critical: BLOCK deployment, hotfix immediately        │
└──────────────────────────────────────────────────────────┘
```

> **Note:** Many teams skip manual QA for every deployment and rely heavily on automated tests. Manual QA is often reserved for major feature releases or critical changes.

---

<a id="step-6-approval-gate"></a>
## 7. Step 6: Approval Gate

### What is an Approval Gate?

An approval gate is a **manual or policy-based checkpoint** in the pipeline that requires human authorization before code can proceed to production.

```
┌──────────────────────────────────────────────────────────┐
│                    APPROVAL GATE                          │
│                                                          │
│  All staging tests passed ✅                             │
│  QA testing completed ✅                                 │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │                                                    │  │
│  │  🔔 Notification sent to approvers:                │  │
│  │                                                    │  │
│  │  "Build abc123d is ready for production            │  │
│  │   deployment. All 191 tests passed.                │  │
│  │   Please review and approve."                      │  │
│  │                                                    │  │
│  │  [✅ Approve]  [❌ Reject]  [📋 View Report]      │  │
│  │                                                    │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Approvers:                                              │
│  • Tech Lead / Engineering Manager                       │
│  • Release Manager                                       │
│  • QA Lead (for major features)                          │
│                                                          │
│  Required approvals: At least 1 of the designated        │
│  reviewers must approve before the pipeline continues.   │
└──────────────────────────────────────────────────────────┘
```

### How Approval Gates Work in Different Tools

#### GitHub Actions — Environment Protection Rules

```yaml
# In GitHub Repository Settings:
# Settings → Environments → production
#
# Configuration:
#   Required reviewers: 
#     - @tech-lead
#     - @release-manager
#   Wait timer: 0 minutes (or set delay)
#   Deployment branches: main only

# In the pipeline YAML:
deploy-production:
  needs: integration-tests
  runs-on: ubuntu-latest
  environment: production     # ← This triggers the approval gate
  #
  # When the pipeline reaches this job:
  # 1. GitHub pauses the pipeline
  # 2. Sends notification to required reviewers
  # 3. Reviewer sees "Review deployments" in GitHub UI
  # 4. Reviewer clicks "Approve" or "Reject"
  # 5. If approved → pipeline continues
  # 6. If rejected → pipeline fails
```

#### GitLab CI — Manual Job

```yaml
deploy-production:
  stage: deploy-prod
  when: manual                # ← Creates a "play" button in GitLab UI
  allow_failure: false
  only:
    - main
  environment:
    name: production
    url: https://myapp.com
  script:
    - helm upgrade myapp ./chart --namespace production
```

#### Jenkins — Input Step

```groovy
// Jenkinsfile
pipeline {
    stages {
        stage('Approval Gate') {
            steps {
                // Pipeline pauses here and waits for human input
                input message: 'Deploy to Production?', 
                      ok: 'Deploy',
                      submitter: 'tech-lead,release-manager',
                      parameters: [
                          string(name: 'REASON', 
                                 description: 'Reason for deployment')
                      ]
            }
        }
        stage('Deploy to Production') {
            steps {
                sh 'helm upgrade myapp ./chart --namespace production'
            }
        }
    }
}
```

### What the Approver Reviews Before Clicking "Approve"

```
APPROVER'S CHECKLIST:

□  All automated tests passed (unit, integration, E2E, performance)
□  Security scan shows no critical/high vulnerabilities
□  Code coverage hasn't decreased
□  Staging deployment is healthy and stable
□  No open critical bugs related to this release
□  Database migrations (if any) are backward compatible
□  Feature flags are properly configured
□  Rollback plan is documented and ready
□  On-call engineer is available
□  Not deploying during a maintenance window or high-traffic period
□  Change request / deployment ticket is approved (for regulated environments)
□  Communication sent to stakeholders (if major feature)
```

### Automated Approval (No Human Needed)

Some mature teams eliminate manual approval entirely and use **automated quality gates**:

```yaml
# Automated approval gate — no human needed
# Pipeline auto-approves if ALL conditions are met:

automated_gate:
  conditions:
    - all_tests_passed: true
    - code_coverage: ">= 80%"
    - security_scan: "no_critical_or_high"
    - performance_test: "p99_latency < 500ms"
    - error_rate: "< 0.1%"
    - change_risk_score: "< 3"    # Calculated based on files changed
    - deploy_window: "weekday AND 9am-4pm"
    - active_incidents: 0

  if_all_pass: auto_deploy_to_production
  if_any_fail: require_manual_approval
```

---

<a id="step-7-deploy-production"></a>
<a id="8-step-7-deploy-to-production"></a>
## 8. Step 7: Deploy to Production

### Canary Deployment — Detailed Breakdown

Canary deployment is the safest production deployment strategy. It gradually shifts traffic from the old version to the new version while monitoring for issues.

```
CANARY DEPLOYMENT PHASES:

Time 0:
┌──────────────────────────────────────────────────────────┐
│                     LOAD BALANCER                         │
│                    ┌──────────┐                           │
│        ┌──────────┤ 100%     ├──────────┐                │
│        │          └──────────┘          │                │
│        ▼                                ▼                │
│  ┌──────────┐                    ┌──────────┐            │
│  │ v1.0     │ (Stable)           │ v2.0     │ (Canary)   │
│  │ 5 pods   │                    │ 0 pods   │            │
│  └──────────┘                    └──────────┘            │
└──────────────────────────────────────────────────────────┘

Phase 1 — 5% Traffic (t+0 minutes):
┌──────────────────────────────────────────────────────────┐
│                     LOAD BALANCER                         │
│              ┌─────┐      ┌─────┐                        │
│        ┌─────┤ 95% ├──┐ ┌─┤ 5%  ├────┐                  │
│        │     └─────┘  │ │ └─────┘    │                  │
│        ▼              │ │            ▼                  │
│  ┌──────────┐         │ │     ┌──────────┐              │
│  │ v1.0     │         │ │     │ v2.0     │              │
│  │ 5 pods   │         │ │     │ 1 pod    │              │
│  └──────────┘         │ │     └──────────┘              │
│                       │ │                               │
│  📊 MONITORING:       │ │  Check metrics for 5 min:     │
│  Error rate: 0.01% ✅ │ │  Latency p99: 180ms ✅        │
│  CPU: 45% ✅          │ │  Memory: 62% ✅               │
└──────────────────────────────────────────────────────────┘

Phase 2 — 25% Traffic (t+15 minutes):
┌──────────────────────────────────────────────────────────┐
│                     LOAD BALANCER                         │
│              ┌─────┐      ┌──────┐                       │
│        ┌─────┤ 75% ├──┐ ┌─┤ 25%  ├───┐                  │
│        ▼     └─────┘  │ │ └──────┘   ▼                  │
│  ┌──────────┐         │ │     ┌──────────┐              │
│  │ v1.0     │         │ │     │ v2.0     │              │
│  │ 4 pods   │         │ │     │ 2 pods   │              │
│  └──────────┘         │ │     └──────────┘              │
│                       │ │                               │
│  📊 MONITORING:       │ │  All metrics still healthy ✅  │
└──────────────────────────────────────────────────────────┘

Phase 3 — 50% Traffic (t+30 minutes):
┌──────────────────────────────────────────────────────────┐
│                     LOAD BALANCER                         │
│              ┌─────┐      ┌──────┐                       │
│        ┌─────┤ 50% ├──┐ ┌─┤ 50%  ├───┐                  │
│        ▼     └─────┘  │ │ └──────┘   ▼                  │
│  ┌──────────┐         │ │     ┌──────────┐              │
│  │ v1.0     │         │ │     │ v2.0     │              │
│  │ 3 pods   │         │ │     │ 3 pods   │              │
│  └──────────┘         │ │     └──────────┘              │
│                       │ │                               │
│  📊 MONITORING:       │ │  All metrics still healthy ✅  │
└──────────────────────────────────────────────────────────┘

Phase 4 — 100% Traffic (t+45 minutes):
┌──────────────────────────────────────────────────────────┐
│                     LOAD BALANCER                         │
│                    ┌──────┐                               │
│               ┌────┤ 100% ├────┐                         │
│               │    └──────┘    │                         │
│               ▼                ▼                         │
│        ┌──────────┐     ┌──────────┐                     │
│        │ v1.0     │     │ v2.0     │                     │
│        │ 0 pods   │     │ 5 pods   │                     │
│        │ REMOVED  │     │ ✅ LIVE  │                     │
│        └──────────┘     └──────────┘                     │
│                                                          │
│  🎉 DEPLOYMENT COMPLETE!                                 │
└──────────────────────────────────────────────────────────┘
```

### How Canary is Implemented in Kubernetes

#### Option 1: Using Kubernetes Native (Simple)

```yaml
# stable-deployment.yaml (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9           # 90% of total pods
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0

---
# canary-deployment.yaml (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1            # 10% of total pods (1 out of 10)
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0

---
# service.yaml — routes to BOTH stable and canary
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp         # ← Matches BOTH stable and canary pods
  ports:
  - port: 80
    targetPort: 3000
```

#### Option 2: Using Istio Service Mesh (Precise Traffic Control)

```yaml
# Istio VirtualService for precise traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  http:
  - route:
    - destination:
        host: myapp-stable
        port:
          number: 80
      weight: 95           # ← 95% to stable
    - destination:
        host: myapp-canary
        port:
          number: 80
      weight: 5            # ← 5% to canary
```

#### Option 3: Using Argo Rollouts (Most Popular)

```yaml
# Argo Rollouts — Built-in canary support
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      # Step 1: Send 5% traffic to canary
      - setWeight: 5
      # Step 2: Wait 5 minutes and check metrics
      - pause: { duration: 5m }
      # Step 3: Automated analysis (check Prometheus metrics)
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: myapp
      # Step 4: Increase to 25%
      - setWeight: 25
      - pause: { duration: 5m }
      # Step 5: Increase to 50%
      - setWeight: 50
      - pause: { duration: 5m }
      - analysis:
          templates:
          - templateName: success-rate
      # Step 6: Increase to 80%
      - setWeight: 80
      - pause: { duration: 5m }
      # Step 7: Full rollout (100%)
      # (automatic when all steps pass)

      # Automatic rollback if analysis fails
      abortScaleDownDelaySeconds: 30

  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0
        ports:
        - containerPort: 3000

---
# AnalysisTemplate — Define what "healthy" means
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 60s
    count: 5
    successCondition: result[0] >= 0.99    # 99% success rate required
    failureLimit: 2
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: |
          sum(rate(http_requests_total{
            service="{{args.service-name}}",
            status!~"5.."
          }[5m])) / 
          sum(rate(http_requests_total{
            service="{{args.service-name}}"
          }[5m]))
```

### Monitoring During Canary Deployment

```
WHAT TO MONITOR DURING CANARY ROLLOUT:

┌──────────────────────────────────────────────────────────┐
│                  KEY METRICS TO WATCH                    │
├──────────────────┬───────────────────────────────────────┤
│                  │                                       │
│  Error Rate      │  % of requests returning 5xx errors   │
│  (most critical) │  Threshold: < 0.1%                    │
│                  │  Alert if: > 1%                       │
│                  │                                       │
├──────────────────┼───────────────────────────────────────┤
│                  │                                       │
│  Latency (P99)   │  99th percentile response time        │
│                  │  Threshold: < 500ms                   │
│                  │  Alert if: > 1000ms                   │
│                  │                                       │
├──────────────────┼───────────────────────────────────────┤
│                  │                                       │
│  Request Rate    │  Requests per second                  │
│  (throughput)    │  Should remain consistent             │
│                  │  Alert if: drops > 20%                │
│                  │                                       │
├──────────────────┼───────────────────────────────────────┤
│                  │                                       │
│  CPU / Memory    │  Resource usage of new pods           │
│                  │  Compare canary vs stable pods        │
│                  │  Alert if: > 80% utilization          │
│                  │                                       │
├──────────────────┼───────────────────────────────────────┤
│                  │                                       │
│  Pod Restarts    │  Number of container restarts         │
│                  │  Should be 0                          │
│                  │  Alert if: any restarts               │
│                  │                                       │
├──────────────────┼───────────────────────────────────────┤
│                  │                                       │
│  Business        │  Conversion rate, checkout success,   │
│  Metrics         │  user signups, revenue                │
│                  │  Should not decrease                  │
│                  │                                       │
└──────────────────┴───────────────────────────────────────┘
```

### Prometheus Queries for Canary Monitoring

```promql
# Error rate for canary pods
sum(rate(http_requests_total{track="canary",status=~"5.."}[5m]))
/
sum(rate(http_requests_total{track="canary"}[5m]))

# Compare latency: canary vs stable
# Canary P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{track="canary"}[5m]))

# Stable P99 latency (for comparison)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{track="stable"}[5m]))

# Pod restart count for canary
kube_pod_container_status_restarts_total{pod=~"myapp-canary.*"}
```

---

<a id="step-8-post-deploy"></a>
## 9. Step 8: Post-Deployment Verification

```
AFTER PRODUCTION DEPLOYMENT IS COMPLETE:

┌──────────────────────────────────────────────────────────┐
│              POST-DEPLOYMENT CHECKLIST                   │
│                                                          │
│  Immediate (0-5 minutes):                                │
│  □ All pods running and healthy                          │
│  □ Health check endpoints returning 200                  │
│  □ No error spikes in logs                               │
│  □ Grafana dashboards show normal metrics                │
│                                                          │
│  Short-term (5-30 minutes):                              │
│  □ Error rate stable and within threshold                │
│  □ Response times normal                                 │
│  □ No increase in customer support tickets               │
│  □ Key user flows working (login, checkout, etc.)        │
│                                                          │
│  Medium-term (30 min - 2 hours):                         │
│  □ Business metrics stable (revenue, conversions)        │
│  □ No memory leaks (watch memory over time)              │
│  □ Database performance normal                           │
│  □ Third-party integrations functioning                  │
│                                                          │
│  Actions:                                                │
│  ✉ Send deployment notification to Slack/Teams           │
│  📝 Update deployment log / changelog                    │
│  🎫 Close Jira ticket → mark as "Deployed"               │
│  📊 Take screenshot of key dashboards for records        │
└──────────────────────────────────────────────────────────┘
```

### Automated Smoke Test After Production Deploy

```bash
#!/bin/bash
# post-deploy-smoke-test.sh

BASE_URL="https://myapp.com"
FAILURES=0

echo "🔍 Running production smoke tests..."

# Test 1: Health check
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/health")
if [ "$STATUS" != "200" ]; then
  echo "❌ Health check failed: $STATUS"
  FAILURES=$((FAILURES + 1))
else
  echo "✅ Health check: OK"
fi

# Test 2: Homepage loads
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL")
if [ "$STATUS" != "200" ]; then
  echo "❌ Homepage failed: $STATUS"
  FAILURES=$((FAILURES + 1))
else
  echo "✅ Homepage: OK"
fi

# Test 3: API endpoint
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/api/v1/status")
if [ "$STATUS" != "200" ]; then
  echo "❌ API status failed: $STATUS"
  FAILURES=$((FAILURES + 1))
else
  echo "✅ API status: OK"
fi

# Test 4: Response time
RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" "$BASE_URL")
if (( $(echo "$RESPONSE_TIME > 2.0" | bc -l) )); then
  echo "❌ Response time too slow: ${RESPONSE_TIME}s"
  FAILURES=$((FAILURES + 1))
else
  echo "✅ Response time: ${RESPONSE_TIME}s"
fi

# Result
if [ $FAILURES -gt 0 ]; then
  echo "🚨 $FAILURES smoke tests failed! Initiating rollback..."
  exit 1
else
  echo "🎉 All smoke tests passed!"
  exit 0
fi
```

---

<a id="step-9-rollback"></a>
## 10. Step 9: Rollback Strategy

### When to Rollback

```
ROLLBACK TRIGGERS:

🚨 AUTOMATIC ROLLBACK (no human needed):
  • Canary analysis fails (error rate > threshold)
  • Pods failing health checks / crash-looping
  • Argo Rollouts automatic abort

⚠️ MANUAL ROLLBACK (human decides):
  • Customer complaints spike
  • Business metrics drop (revenue, conversions)
  • Performance degradation detected
  • Security vulnerability discovered
  • Unexpected behavior in production
```

### How to Rollback

```bash
# ─────────────────────────────────────────────
# METHOD 1: Kubernetes native rollback
# ─────────────────────────────────────────────
# See deployment history
kubectl rollout history deployment/myapp -n production

# Rollback to previous version
kubectl rollout undo deployment/myapp -n production

# Rollback to a specific revision
kubectl rollout undo deployment/myapp -n production --to-revision=3

# Verify rollback
kubectl rollout status deployment/myapp -n production


# ─────────────────────────────────────────────
# METHOD 2: Helm rollback
# ─────────────────────────────────────────────
# See release history
helm history myapp -n production

# Rollback to previous release
helm rollback myapp -n production

# Rollback to specific revision
helm rollback myapp 5 -n production


# ─────────────────────────────────────────────
# METHOD 3: Argo Rollouts abort
# ─────────────────────────────────────────────
# Abort canary (immediately routes all traffic back to stable)
kubectl argo rollouts abort myapp -n production

# Or via ArgoCD UI: click "Abort" button


# ─────────────────────────────────────────────
# METHOD 4: GitOps rollback (revert the commit)
# ─────────────────────────────────────────────
# In the GitOps repo, revert the image tag change
git revert HEAD
git push origin main
# ArgoCD will auto-sync back to previous version
```

### Rollback Timeline

```
INCIDENT DETECTED (t=0)
    │
    ├── t+0: Alert fires → On-call engineer paged
    │
    ├── t+2min: Engineer acknowledges alert, starts investigating
    │
    ├── t+5min: Confirms issue is related to new deployment
    │
    ├── t+6min: Initiates rollback command
    │         kubectl rollout undo deployment/myapp -n production
    │
    ├── t+8min: Old pods starting, new pods terminating
    │
    ├── t+10min: Rollback complete, traffic back on stable version
    │
    ├── t+15min: Verify all metrics back to normal
    │
    ├── t+30min: Send incident report to team
    │
    └── Next day: Blameless postmortem meeting
```

---

<a id="real-world-pipeline-examples"></a>
## 11. Real-World Pipeline Examples

### Example: Medium-Sized SaaS Company

```
COMPLETE PIPELINE — REAL-WORLD EXAMPLE

Company: E-commerce SaaS (50 developers, 20 microservices)
Deploy frequency: 10-15 times per day
Tools: GitHub, GitHub Actions, Docker, EKS (Kubernetes on AWS), 
       ArgoCD, Terraform, Prometheus, Grafana

┌─────────────────────────────────────────────────┐
│ PR Merged to Main (any of 20 microservices)     │
└─────────────────────┬───────────────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ STAGE: Build (2 min)              │
    │ • Checkout code                   │
    │ • Run unit tests                  │
    │ • Build Docker image              │
    │ • Tag: ECR/myapp:sha-abc123d     │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ STAGE: Security (3 min)           │
    │ • Trivy scan Docker image         │
    │ • Snyk dependency check           │
    │ • Fail if CRITICAL CVEs found     │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ STAGE: Push Image (1 min)         │
    │ • Push to Amazon ECR              │
    │ • Sign image with cosign          │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ STAGE: Update GitOps Repo (1 min) │
    │ • Clone gitops-manifests repo     │
    │ • Update image tag in             │
    │   apps/myapp/staging/values.yaml  │
    │ • Commit & push                   │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ ArgoCD Auto-Sync to Staging       │
    │ (2-3 min)                         │
    │ • Detects new commit              │
    │ • Syncs Helm chart to staging     │
    │ • Rolling update in K8s           │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ STAGE: Staging Tests (12 min)     │
    │ • Smoke tests (30 sec)            │
    │ • API integration tests (4 min)   │
    │ • E2E browser tests (7 min)       │
    │ • Performance baseline (3 min)    │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ APPROVAL GATE                     │
    │ • Slack notification sent         │
    │ • Tech lead reviews test results  │
    │ • Clicks "Approve" in GitHub UI   │
    │                                   │
    │ (For low-risk changes: auto-      │
    │  approved based on risk score)    │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ STAGE: Production Deploy (15 min) │
    │ • Update gitops-manifests for     │
    │   apps/myapp/production/values    │
    │ • ArgoCD syncs with canary:       │
    │   - 5% traffic (5 min monitor)    │
    │   - 25% traffic (5 min monitor)   │
    │   - 100% traffic                  │
    │ • Argo Rollouts auto-analysis     │
    │   checks Prometheus metrics       │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────▼─────────────────┐
    │ POST-DEPLOY                       │
    │ • Smoke tests on production       │
    │ • Slack notification: "myapp      │
    │   v2.4.1 deployed to production"  │
    │ • Grafana dashboard monitoring    │
    └───────────────────────────────────┘

Total time: ~35-40 minutes (merge to fully live)
```

---

<a id="tools-involved"></a>
## 12. Tools Involved at Each Step

```
STEP                        TOOLS USED
─────────────────────────── ──────────────────────────────────
Merge to Main               GitHub, GitLab, Bitbucket
Pipeline Trigger             GitHub Actions, Jenkins, GitLab CI
Build Docker Image           Docker, Buildah, Kaniko
Security Scanning            Trivy, Snyk, Aqua, Clair
Push to Registry             ECR, GCR, Docker Hub, JFrog
Update Manifests             Helm, Kustomize, yq
Deploy to K8s                ArgoCD, FluxCD, Helm, kubectl
Traffic Management           Istio, Nginx Ingress, Traefik
Canary Analysis              Argo Rollouts, Flagger, Spinnaker
Integration Tests            Postman/Newman, pytest, Jest
E2E Tests                    Cypress, Playwright, Selenium
Performance Tests            k6, JMeter, Gatling, Locust
Monitoring                   Prometheus, Grafana, Datadog
Logging                      ELK Stack, Loki, Fluentd
Alerting                     PagerDuty, Opsgenie, Alertmanager
Notifications                Slack, MS Teams, Email
Approval Gates               GitHub Environments, Jenkins Input
Rollback                     kubectl, Helm, ArgoCD, git revert
```

---

<a id="best-practices"></a>
## 13. Common Patterns & Best Practices

### Do's ✅

```
• Always tag Docker images with Git SHA (not just "latest")
• Run smoke tests immediately after every deployment
• Have automated rollback triggers based on metrics
• Keep canary monitoring duration proportional to traffic volume
• Document your rollback procedure before you need it
• Deploy during business hours when the team is available
• Start with small canary percentages (1-5%)
• Use separate namespaces/clusters for staging and production
• Make deployments idempotent (running twice = same result)
• Practice rollbacks regularly (even when nothing is wrong)
```

### Don'ts ❌

```
• Don't deploy on Friday evenings (seriously)
• Don't skip staging — "it worked in dev" is not enough
• Don't make manual changes to production (use pipelines)
• Don't ignore canary metrics — automation should catch issues
• Don't deploy without a rollback plan
• Don't deploy during peak traffic hours (if avoidable)
• Don't deploy multiple unrelated changes together
• Don't skip security scans to "save time"
• Don't give everyone production deploy access
• Don't forget to communicate deployments to the team
```

### Deployment Windows Best Practices

```
WHEN TO DEPLOY:

✅ GOOD:
  Monday - Thursday, 9 AM - 3 PM (local business hours)
  When on-call engineer is available
  When deployment is small and well-tested
  After a successful staging validation

❌ AVOID:
  Friday afternoon / evening
  Weekends (unless critical hotfix)
  During peak traffic hours
  During holidays or company events
  When the team is understaffed
  Right before a major product launch
  When there are active production incidents

EXCEPTION: 
  Hotfixes for production outages → deploy anytime
  Companies with mature automation → deploy anytime 
  (Netflix deploys thousands of times per day)
```

---

## Summary Cheat Sheet

```
┌─────────────────────────────────────────────────────────────┐
│           MERGE → PRODUCTION QUICK REFERENCE                 │
├──────────┬──────────────────────────────────────────────────┤
│ MERGE    │ PR approved → squash merge → webhook fires        │
│          │ Branch protection ensures quality gates pass       │
├──────────┼──────────────────────────────────────────────────┤
│ BUILD    │ Docker build → tag with SHA → security scan →     │
│          │ push to container registry                        │
├──────────┼──────────────────────────────────────────────────┤
│ STAGING  │ Update manifests → K8s rolling update →           │
│          │ run integration + E2E + perf tests                │
├──────────┼──────────────────────────────────────────────────┤
│ APPROVAL │ Human review OR automated quality gate →          │
│          │ check test results, security, metrics             │
├──────────┼──────────────────────────────────────────────────┤
│ CANARY   │ 5% → monitor → 25% → monitor → 50% → 100%       │
│          │ Auto-rollback if error rate/latency too high      │
├──────────┼──────────────────────────────────────────────────┤
│ VERIFY   │ Smoke tests → monitor dashboards →                │
│          │ notify team → update deployment log               │
├──────────┼──────────────────────────────────────────────────┤
│ ROLLBACK │ kubectl rollout undo OR helm rollback OR          │
│          │ git revert in GitOps repo                         │
└──────────┴──────────────────────────────────────────────────┘
```

---

*📚 These notes are part of the DevOps Complete Reference Guide.*
*Keep learning, keep deploying, keep automating! 🚀*

