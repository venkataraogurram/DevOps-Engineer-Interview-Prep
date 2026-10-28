# Refresher – CI/CD and Git

## 1. Pipeline design

Standard stages and what each proves:

| Stage | What runs | Gate |
|---|---|---|
| Source | webhook on push/PR | branch protection, signed commits (banks) |
| Lint/format | eslint/ruff/`terraform fmt -check`/hadolint/yamllint | fail fast |
| Unit tests | language test runner, coverage threshold | ≥ X% coverage |
| SAST / secrets scan | SonarQube, Semgrep, gitleaks, Checkov/tfsec for IaC | no high/critical |
| Build | compile, `docker build` once, tag `:<git-sha>` and `:<semver>` | reproducible |
| Image scan / SBOM | Trivy, ECR enhanced scanning (Inspector), Syft | no critical CVE; sign with cosign |
| Publish | push to ECR / artifact repo (Nexus/Artifactory/CodeArtifact) | immutable tags (ECR tag immutability on) |
| Deploy dev | Terraform apply / Helm upgrade / `kubectl apply` | `rollout status` |
| Integration/API tests | Postman/newman, pytest, k6 smoke | pass |
| Deploy staging | same artifact, prod-like config | DAST (ZAP) optional |
| Approval | manual, ticket ID (ServiceNow/Jira CAB) attached | segregation of duties |
| Deploy prod | rolling / blue-green / canary | health alarms |
| Post-deploy | smoke tests, synthetic canaries, alarm watch window | auto-rollback on breach |

Principles: build once deploy many; environment differences via config only; no humans with prod write access (pipeline role only); every deploy traceable to commit + ticket; pipelines as code in the repo.

## 2. Rollback strategies

- **Redeploy previous artifact**: simplest; works when DB schema compatible. Keep last N images.
- **Rolling update undo**: `kubectl rollout undo`; ASG instance refresh back to previous launch template version; Beanstalk previous app version.
- **Blue/green**: two full environments; switch ALB listener rule / Route 53 weighted record / Service selector. Rollback = switch back. Cost 2x during cutover. CodeDeploy supports for ECS/Lambda/EC2.
- **Canary**: 5% → 25% → 100% with metric gates (error rate, p99). Abort = shift to 0. Lambda alias weights, ALB weighted target groups, Argo Rollouts/Flagger.
- **Feature flags**: deploy dark, enable per cohort; rollback is a flag flip (LaunchDarkly, AppConfig).
- **Database**: expand/contract (add column → deploy code using both → remove old); never destructive migration in the same release; backups/snapshot before migration; Aurora Blue/Green for engine upgrades.
- **IaC rollback**: `git revert` + apply; keep state versioned (S3 versioning) for disaster recovery; `prevent_destroy` on data.

## 3. Jenkins (likely their standard – know the shape)

```groovy
// Jenkinsfile (declarative)
pipeline {
  agent { label 'linux-docker' }                // or kubernetes { yaml ... } via Kubernetes plugin
  options { timestamps(); disableConcurrentBuilds(); buildDiscarder(logRotator(numToKeepStr: '30')) }
  environment {
    AWS_REGION = 'ap-south-1'
    ECR        = '123456789012.dkr.ecr.ap-south-1.amazonaws.com/payments-api'
    IMAGE_TAG  = "${env.GIT_COMMIT.take(7)}"
  }
  parameters { choice(name: 'ENV', choices: ['dev','staging','prod']) }
  triggers { pollSCM('H/5 * * * *') }           // or GitHub/GitLab webhook
  stages {
    stage('Lint & Test') {
      parallel {
        stage('Lint') { steps { sh 'make lint' } }
        stage('Unit') { steps { sh 'make test'; junit 'reports/*.xml' } }
      }
    }
    stage('Build & Push') {
      steps {
        withAWS(role: 'arn:aws:iam::123456789012:role/jenkins-ci', region: env.AWS_REGION) {   // or withCredentials
          sh '''
            aws ecr get-login-password | docker login --username AWS --password-stdin $ECR
            docker build -t $ECR:$IMAGE_TAG .
            trivy image --exit-code 1 --severity CRITICAL $ECR:$IMAGE_TAG
            docker push $ECR:$IMAGE_TAG
          '''
        }
      }
    }
    stage('Terraform Plan') {
      steps { dir("infra/envs/${params.ENV}") { sh 'terraform init -input=false && terraform plan -out=tfplan'; archiveArtifacts 'tfplan' } }
    }
    stage('Approve') {
      when { expression { params.ENV == 'prod' } }
      steps { timeout(time: 2, unit: 'HOURS') { input message: 'Apply to prod?', submitter: 'platform-leads', parameters: [string(name: 'CHANGE_TICKET')] } }
    }
    stage('Deploy') {
      steps {
        dir("infra/envs/${params.ENV}") { sh 'terraform apply -input=false tfplan' }
        sh "helm upgrade --install payments ./charts/payments -n payments --set image.tag=$IMAGE_TAG --atomic --timeout 5m"
        sh 'kubectl rollout status deploy/payments-api -n payments --timeout=180s'
      }
    }
  }
  post {
    failure { slackSend channel: '#platform-alerts', message: "FAILED ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
    always  { cleanWs() }
  }
}
```

Concepts: controller + agents (static VMs, or ephemeral pods via Kubernetes plugin), Jenkinsfile in repo (Multibranch Pipeline / Organization folder), Shared Libraries (`vars/standardBuild.groovy`) for org-wide standards, credentials store + `withCredentials`, RBAC via Role Strategy plugin, Configuration as Code (JCasC) for the controller itself, backups of `JENKINS_HOME`.

Scripted vs declarative: declarative is structured and lintable; scripted (`node { ... }`) is Groovy freeform.

## 4. GitHub Actions (what you actually use)

```yaml
name: ci-cd
on:
  pull_request: { branches: [main] }
  push: { branches: [main], tags: ['v*'] }
permissions: { id-token: write, contents: read, pull-requests: write }
concurrency: { group: ${{ github.workflow }}-${{ github.ref }}, cancel-in-progress: true }
env: { AWS_REGION: ap-south-1 }
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12', cache: pip }
      - run: pip install -r requirements.txt && pytest -q
  terraform-plan:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: arn:aws:iam::123456789012:role/gha-plan, aws-region: ${{ env.AWS_REGION }} }
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.9.5 }
      - run: terraform -chdir=infra/envs/dev init -input=false
      - run: terraform -chdir=infra/envs/dev plan -no-color -out=tfplan
      - uses: actions/upload-artifact@v4
        with: { name: tfplan, path: infra/envs/dev/tfplan }
  build-push:
    if: github.event_name == 'push'
    needs: test
    runs-on: ubuntu-latest
    outputs: { tag: ${{ steps.meta.outputs.tag }} }
    steps:
      - uses: actions/checkout@v4
      - id: meta
        run: echo "tag=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: arn:aws:iam::123456789012:role/gha-deploy, aws-region: ${{ env.AWS_REGION }} }
      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr
      - run: |
          docker build -t ${{ steps.ecr.outputs.registry }}/payments-api:${{ steps.meta.outputs.tag }} .
          docker push ${{ steps.ecr.outputs.registry }}/payments-api:${{ steps.meta.outputs.tag }}
  deploy-prod:
    needs: build-push
    runs-on: ubuntu-latest
    environment: production          # required reviewers configured on the environment = approval gate
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: arn:aws:iam::123456789012:role/gha-deploy, aws-region: ${{ env.AWS_REGION }} }
      - run: aws eks update-kubeconfig --name prod
      - run: kubectl set image deploy/payments-api api=…/payments-api:${{ needs.build-push.outputs.tag }} -n payments
      - run: kubectl rollout status deploy/payments-api -n payments --timeout=180s
```

OIDC trust policy for the role (the part people get wrong):

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
    "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:idfc/payments-api:ref:refs/heads/main" }
  }
}
```

Other: reusable workflows (`workflow_call`), composite actions, self-hosted runners (in VPC for private resources; banks usually require), secrets at repo/env/org level, `GITHUB_TOKEN` permissions least privilege, pin actions by SHA for supply chain.

## 5. GitLab CI (second most likely in Indian banks)

```yaml
stages: [test, build, plan, deploy]
variables: { TF_ROOT: infra/envs/dev, AWS_REGION: ap-south-1 }
default: { image: hashicorp/terraform:1.9 }
.aws_oidc: &aws_oidc
  id_tokens: { GITLAB_OIDC_TOKEN: { aud: https://gitlab.idfc.internal } }
  before_script:
    - export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s" $(aws sts assume-role-with-web-identity --role-arn $ROLE_ARN --role-session-name gitlab-$CI_JOB_ID --web-identity-token $GITLAB_OIDC_TOKEN --duration-seconds 3600 --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' --output text))
test:
  stage: test
  image: python:3.12
  script: [pip install -r requirements.txt, pytest -q]
  cache: { key: "$CI_COMMIT_REF_SLUG", paths: [.cache/pip] }
plan:
  stage: plan
  <<: *aws_oidc
  script:
    - cd $TF_ROOT && terraform init -input=false && terraform plan -out=tfplan
  artifacts: { paths: [$TF_ROOT/tfplan], expire_in: 1 day }
apply:
  stage: deploy
  <<: *aws_oidc
  script: [cd $TF_ROOT && terraform apply -input=false tfplan]
  when: manual                    # approval
  environment: { name: production }
  rules: [{ if: '$CI_COMMIT_BRANCH == "main"' }]
```

Concepts: runners (shared/group/project; Kubernetes executor), `rules`/`only`, `needs` for DAG, `extends`/anchors, protected branches + protected variables, environments + approvals, merge request pipelines, Terraform-managed state backend built into GitLab (`http` backend), Auto DevOps.

## 6. AWS-native (your home turf; mention when relevant)

CodePipeline (V2: triggers, variables, stages) → CodeBuild (buildspec.yml, VPC, caching) → CodeDeploy (EC2 in-place/blue-green via appspec, ECS blue-green, Lambda canary/linear). CodeConnections for GitHub/GitLab source. CodeArtifact for packages. Approval action with SNS. EventBridge for pipeline state events.

## 7. Git

**Branching**

- **GitFlow**: `main` (prod), `develop`, `feature/*`, `release/*`, `hotfix/*`. Fits versioned release trains and formal QA sign-off; long-lived branches → merge conflicts, slower feedback.
- **GitHub Flow**: `main` + short feature branches + PRs; deploy from main.
- **Trunk-based**: everyone commits to `main` (or branches <1–2 days), feature flags hide incomplete work, release branches optional and cut from trunk. Enables CD. Sell to a bank: "trunk-based with release branches for CAB evidence".
- **Environment branches** (dev/staging/prod branches) = anti-pattern: promotes by merge not by artifact. Promote the same artifact with environment config instead.

**Commands you may be asked about**

```bash
git fetch --all --prune
git rebase -i HEAD~3                      # squash/reword local commits (never on shared branches)
git merge --no-ff feature/x               # keep merge commit (GitFlow)
git cherry-pick <sha>                     # hotfix to release branch
git revert <sha>                          # safe undo on shared branch (new commit)
git reset --soft HEAD~1                   # undo commit keep changes (local only)
git stash / git stash pop
git bisect start; git bisect bad; git bisect good v1.3   # find regressing commit
git log --oneline --graph --decorate --all
git tag -a v1.4.2 -m "release" && git push origin v1.4.2  # semver tags drive releases
git blame -L 10,20 file
```

Merge vs rebase: merge preserves history/merge commits; rebase linearises, rewrites SHAs → only on your own unpushed branches. Squash-merge PRs for clean `main`.

Hygiene: branch protection (reviews, status checks, no force push), CODEOWNERS (platform team owns `infra/`), signed commits/tags, conventional commits + semantic-release for changelogs, pre-commit hooks (`terraform fmt`, gitleaks), `.gitignore` for state/plan/`.env`.

Monorepo vs polyrepo for IaC: monorepo for modules + envs with path-filtered pipelines; polyrepo when teams need independent release cadence. Module registry (private) decouples either way.

## 8. Questions you may get

- "Pipeline deployed a bad build to prod at 2 am; what happens?" → alarm → auto rollback (CodeDeploy/ Argo) or on-call runs `rollout undo`/redeploy previous tag; incident channel; post-mortem; add the missing test/canary gate.
- "How do you keep secrets out of pipelines?" → OIDC roles, vault/Secrets Manager at runtime, masked variables, gitleaks pre-commit, rotate on leak.
- "How do you test Terraform in CI?" → fmt/validate/tflint/checkov, `plan` review, `terraform test`/Terratest in ephemeral account, policy-as-code on plan JSON.
- "Monorepo pipeline is slow" → path filters, caching (`actions/cache`, Docker layer cache, `.terraform` plugin cache), parallel jobs, remote build cache, split pipelines per component.
- "How do you handle DB migrations in CD?" → migration job as a pre-deploy step (K8s Job / init container with lock), backward-compatible schema, run once, fail pipeline on error.
- "Approval but auditors want evidence" → pipeline stores plan output, approver identity, ticket ID as artifacts; CloudTrail shows the role that applied.
