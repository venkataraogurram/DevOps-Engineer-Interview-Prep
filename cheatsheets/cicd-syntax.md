# CI/CD syntax cheatsheet (annotated) – Jenkins, GitHub Actions, GitLab CI, CodeBuild

[← Cheatsheet index](README.md) · Concepts and strategies: [../06-refresher-cicd-git.md](../06-refresher-cicd-git.md)

Format: every construct is annotated with what it does and why it's there.

## Concept map (same idea, four names)

| Concept | Jenkins | GitHub Actions | GitLab CI | CodePipeline / CodeBuild |
|---|---|---|---|---|
| Definition file | `Jenkinsfile` in repo | `.github/workflows/*.yml` | `.gitlab-ci.yml` | pipeline JSON/CDK + `buildspec.yml` |
| Where it runs | agent / node | runner (`runs-on`) | runner (`tags`, executor) | CodeBuild project |
| Unit of work | stage → steps | job → steps | job (in a stage) → script | action → build phases |
| Parallelism | `parallel {}` | separate jobs, `strategy.matrix` | jobs in the same stage, `parallel:` | parallel actions in a stage |
| Ordering | sequential stages | `needs:` | `stages:` order, `needs:` DAG | stages |
| Secrets | Credentials store, `withCredentials` | `${{ secrets.X }}`, environment secrets | CI/CD variables (masked, protected) | Secrets Manager / SSM via `env` |
| Approval gate | `input` step | environment "required reviewers" | `when: manual` + protected environment | Manual approval action |
| Artifacts | `archiveArtifacts`, `stash/unstash` | `upload-artifact` / `download-artifact` | `artifacts:` paths, `expire_in` | S3 artifact store |
| Cache | plugins / persistent workspace | `actions/cache` | `cache:` key/paths | S3 or local cache |
| Reuse | Shared Libraries | reusable workflows, composite actions | `include:`, `extends:`, components | CDK constructs |
| Conditions | `when {}` | `if:` | `rules:` | – |
| Triggers | webhook, `pollSCM`, `cron` | `on:` push / pull_request / schedule / workflow_dispatch | push, MR, `schedule`, `trigger` | CodeConnections, EventBridge, schedule |
| AWS auth | `withAWS(role:)`, IRSA on K8s agents | OIDC via `configure-aws-credentials` | `id_tokens` + `assume-role-with-web-identity` | CodeBuild service role |

## Jenkinsfile (declarative, annotated)

```groovy
@Library('platform-lib@v2') _                     // load the shared library at a pinned version
pipeline {
  agent none                                      // choose agents per stage instead of one for all
  options {
    timeout(time: 60, unit: 'MINUTES')            // kill runaway builds
    timestamps()                                  // prefix log lines with time
    disableConcurrentBuilds()                     // one run per branch at a time (protects Terraform state)
    ansiColor('xterm')
  }
  parameters {                                    // appear as a form on "Build with Parameters"
    choice(name: 'ENV', choices: ['dev','prod'])
    booleanParam(name: 'DESTROY', defaultValue: false)
  }
  environment {                                   // env vars for every stage
    AWS_REGION = 'ap-south-1'
    TF_IN_AUTOMATION = '1'
  }
  triggers { cron('H 2 * * 1-5') }                // nightly drift check; 'H' spreads load
  stages {
    stage('Checkout') {
      agent { label 'linux' }
      steps { checkout scm; stash name: 'src', includes: '**' }      // stash: hand files to later stages on other agents
    }
    stage('Validate') {
      agent { docker { image 'hashicorp/terraform:1.9'; args '-u root' } }   // tool pinned via container
      steps { unstash 'src'; sh 'terraform fmt -check && terraform init -backend=false && terraform validate' }
    }
    stage('Plan') {
      agent { kubernetes { yamlFile 'ci/agent.yaml'; defaultContainer 'terraform' } }   // ephemeral pod agent
      steps {
        unstash 'src'
        withCredentials([usernamePassword(credentialsId: 'x', usernameVariable: 'U', passwordVariable: 'P')]) { sh 'echo $U' }   // secrets injected and masked
        withAWS(role: "arn:aws:iam::123:role/jenkins-${params.ENV}", roleSessionName: "jenkins-${env.BUILD_NUMBER}") {        // assume a per-env role; session name shows in CloudTrail
          sh "terraform init -backend-config=envs/${params.ENV}.backend.hcl && terraform plan -var-file=envs/${params.ENV}.tfvars -out=tfplan"
        }
        stash name: 'plan', includes: 'tfplan,.terraform.lock.hcl'      // carry the exact plan to the apply stage
        script { env.PLAN_SUMMARY = sh(script: 'terraform show -no-color tfplan | tail -1', returnStdout: true).trim() }   // capture output for the approval prompt
      }
    }
    stage('Approve') {
      when { expression { params.ENV == 'prod' } }                      // gate only prod
      steps {
        timeout(time: 4, unit: 'HOURS') {                               // don't hold an agent forever
          input message: "Apply? ${env.PLAN_SUMMARY}", submitter: 'platform-leads', parameters: [string(name: 'TICKET')]   // who may approve + change ticket captured
        }
      }
    }
    stage('Apply') {
      agent { label 'linux' }
      steps { unstash 'src'; unstash 'plan'; withAWS(role: "...") { sh 'terraform apply -input=false tfplan' } }   // apply the reviewed plan, nothing else
    }
  }
  post {                                                                // always runs, regardless of result
    always  { junit allowEmptyResults: true, testResults: 'reports/*.xml'; archiveArtifacts artifacts: 'tfplan.txt', allowEmptyArchive: true; cleanWs() }
    success { slackSend channel: '#deploys', color: 'good',   message: "${env.JOB_NAME} #${env.BUILD_NUMBER} OK" }
    failure { slackSend channel: '#deploys', color: 'danger', message: "${env.JOB_NAME} #${env.BUILD_NUMBER} FAILED ${env.BUILD_URL}" }
  }
}
```

- **Shared library `vars/terraformDeploy.groovy` with `def call(Map cfg) { pipeline { ... } }`**

  Teams write `terraformDeploy(env: 'prod')` and get the standard pipeline. How you enforce org standards without copying Jenkinsfiles.

- **Scripted pipeline `node('linux') { stage('x') { sh '...' } }`**

  Freeform Groovy; use only when declarative can't express it.

- `sh(returnStatus: true)` / `sh(returnStdout: true)`

  Capture exit code / output instead of failing.

- `readJSON` / `writeFile` / `fileExists`

  File helpers (Pipeline Utility Steps plugin).

- `retry(3) { ... }` / `waitUntil { ... }`

  Retry flaky steps / poll for a condition.

- `lock('tf-state-prod') { ... }`

  Mutex across jobs (Lockable Resources): prevents two pipelines applying the same state.

- `milestone`

  Cancel older runs when a newer one passes the same point.

- `catchError` / `error 'msg'` / `currentBuild.result`

  Handle or raise failures; set the build result manually.

- `build job: 'other', parameters: [...]`

  Trigger another job.

- **CLI `java -jar jenkins-cli.jar -s URL -auth u:t list-jobs | build job -p K=V | console job | reload-configuration | safe-restart`**

  Script the controller.

- **JCasC `JENKINS_HOME/casc_configs/jenkins.yaml` + `plugins.txt`**

  Controller configuration as code: rebuildable Jenkins.

## GitHub Actions (annotated)

```yaml
name: deploy
on:
  push: { branches: [main], paths: ['infra/**'] }      # only when infra changes land on main
  pull_request:                                        # plan on PRs
  workflow_dispatch:                                   # manual run button with inputs
    inputs: { env: { type: choice, options: [dev, prod], default: dev } }
  schedule: [{ cron: '0 2 * * 1-5' }]                  # nightly drift detection
permissions: { id-token: write, contents: read, pull-requests: write }   # id-token for OIDC; least privilege otherwise
concurrency: { group: deploy-${{ github.ref }}, cancel-in-progress: false }   # serialise runs per branch; don't cancel an apply
defaults: { run: { shell: bash, working-directory: infra } }
env: { AWS_REGION: ap-south-1, TF_VERSION: 1.9.5 }
jobs:
  plan:
    runs-on: ubuntu-latest                             # or [self-hosted, linux, vpc] for private resources
    strategy: { fail-fast: false, matrix: { env: [dev, staging] } }   # one job per env; keep going if one fails
    environment: ${{ matrix.env }}                     # pulls env-scoped secrets/vars
    timeout-minutes: 30
    outputs: { exitcode: ${{ steps.plan.outputs.exitcode }} }   # pass data to downstream jobs
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }                       # full history (needed for tags/changelogs)
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: ${{ vars.PLAN_ROLE_ARN }}, aws-region: ${{ env.AWS_REGION }}, role-session-name: gha-${{ github.run_id }} }   # OIDC → STS; no stored keys; session name traceable in CloudTrail
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: ${{ env.TF_VERSION }}, terraform_wrapper: false }   # wrapper off so exit codes/outputs are raw
      - uses: actions/cache@v4
        with: { path: ~/.terraform.d/plugin-cache, key: tf-${{ hashFiles('**/.terraform.lock.hcl') }} }   # provider cache keyed on lock file
      - run: terraform init -input=false -backend-config=envs/${{ matrix.env }}.backend.hcl
      - id: plan
        run: |
          set +e                                       # don't fail the step on exit code 2
          terraform plan -input=false -no-color -detailed-exitcode -var-file=envs/${{ matrix.env }}.tfvars -out=tfplan | tee plan.txt
          echo "exitcode=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"   # 0 none, 1 error, 2 changes
      - if: github.event_name == 'pull_request'        # post the plan to the PR for review
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs'); const plan = fs.readFileSync('infra/plan.txt','utf8').slice(-60000);
            github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body: '```\n' + plan + '\n```' });
      - uses: actions/upload-artifact@v4
        with: { name: tfplan-${{ matrix.env }}, path: infra/tfplan, retention-days: 1 }   # hand the exact plan to apply
  apply:
    needs: plan
    if: github.ref == 'refs/heads/main' && needs.plan.outputs.exitcode == '2'   # only on main and only if there are changes
    runs-on: ubuntu-latest
    environment: { name: prod, url: https://app.example.com }   # "required reviewers" on this environment = the approval gate
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: tfplan-dev, path: infra }
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: ${{ vars.APPLY_ROLE_ARN }}, aws-region: ${{ env.AWS_REGION }} }   # separate, more privileged role
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -input=false -backend-config=envs/prod.backend.hcl && terraform apply -input=false tfplan
      - if: failure()
        run: curl -X POST -H 'Content-type: application/json' --data '{"text":"deploy failed ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}' ${{ secrets.SLACK_WEBHOOK }}
```

- **Reusable workflow: `on: workflow_call: { inputs, secrets }`; caller `jobs: x: uses: org/repo/.github/workflows/tf.yml@v1 with: {...} secrets: inherit`**

  One pipeline definition shared by many repos. The platform-team pattern.

- **Composite action: `action.yml` with `runs: { using: composite, steps: [...] }`**

  Bundle several steps into one `uses:`.

- **Contexts `github.*`, `env.*`, `vars.*`, `secrets.*`, `needs.*`, `steps.*.outputs`, `runner.*`, `matrix.*`**

  Data available to expressions. `vars` = non-secret config per repo/env.

- **Functions `contains() startsWith() format() join() toJSON() fromJSON() hashFiles() success() failure() always() cancelled()`**

  In `if:` and `with:` expressions.

- `echo "k=v" >> $GITHUB_OUTPUT` / `>> $GITHUB_ENV` / `>> $GITHUB_STEP_SUMMARY`

  Set a step output / an env var for later steps / write a Markdown summary on the run page.

- `::add-mask::VALUE` / `::warning::msg` / `::group::title`

  Mask a value in logs / annotate / collapse log sections.

- **Pin actions by SHA (`actions/checkout@<sha>`)**

  Supply-chain safety: a tag can be moved, a SHA can't. Dependabot keeps them updated.

- **Never checkout PR head with `pull_request_target`**

  That trigger runs with write permissions; untrusted code from a fork could steal secrets.

- `gh run list | view | watch | rerun`, `gh workflow run deploy -f env=prod`

  CLI control. `act` runs workflows locally.

## GitLab CI (annotated)

```yaml
stages: [validate, plan, apply]                 # execution order of stages
default:
  image: hashicorp/terraform:1.9                # default container for jobs
  tags: [vpc-runner]                            # pick runners inside the VPC
  interruptible: true                           # cancel when a newer pipeline starts on the branch
  retry: { max: 1, when: [runner_system_failure] }   # retry infra flakes, not real failures
variables:
  TF_ROOT: ${CI_PROJECT_DIR}/infra
  TF_IN_AUTOMATION: "1"
  AWS_REGION: ap-south-1
workflow:
  rules:                                        # when the whole pipeline runs
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
.aws: &aws                                      # hidden job used as a template (YAML anchor)
  id_tokens: { GITLAB_OIDC_TOKEN: { aud: "https://gitlab.example.com" } }   # request an OIDC token for AWS
  before_script:
    - apk add --no-cache aws-cli jq
    - >
      export $(aws sts assume-role-with-web-identity --role-arn "$ROLE_ARN" --role-session-name "gl-$CI_JOB_ID"
      --web-identity-token "$GITLAB_OIDC_TOKEN" --duration-seconds 3600
      --query 'Credentials.[`AWS_ACCESS_KEY_ID=`,AccessKeyId,`AWS_SECRET_ACCESS_KEY=`,SecretAccessKey,`AWS_SESSION_TOKEN=`,SessionToken]' --output text | tr -d '\t' | sed 's/= /=/g')
    - cd $TF_ROOT && terraform init -input=false -backend-config=envs/$ENV.backend.hcl
validate:
  stage: validate
  script: [terraform fmt -check -recursive, terraform init -backend=false, terraform validate]
  rules: [{ changes: ["infra/**/*"] }]          # only when infra files changed
plan:
  stage: plan
  <<: *aws                                      # inherit the AWS auth template
  parallel: { matrix: [{ ENV: [dev, staging] }] }   # one job per env
  script:
    - terraform plan -input=false -var-file=envs/$ENV.tfvars -out=tfplan
    - terraform show -json tfplan > plan.json
  artifacts:
    paths: [$TF_ROOT/tfplan]                    # pass the plan to apply
    reports: { terraform: $TF_ROOT/plan.json }  # renders a plan summary widget in the MR
    expire_in: 1 day
  cache: { key: "$CI_COMMIT_REF_SLUG", paths: [$TF_ROOT/.terraform] }   # provider cache per branch
apply:prod:
  stage: apply
  <<: *aws
  variables: { ENV: prod }
  script: [terraform apply -input=false tfplan]
  needs: [{ job: plan, artifacts: true }]       # DAG dependency + fetch its artifacts
  environment: { name: production, url: https://app.example.com, deployment_tier: production }   # protected environment = approvers
  when: manual                                  # someone must click
  allow_failure: false                          # pipeline shows as blocked, not passed, until run
  rules: [{ if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH' }]
  resource_group: prod                          # one prod deploy at a time across pipelines
include:
  - project: platform/ci-templates              # shared templates from another repo at a pinned ref
    ref: v2
    file: /templates/security-scan.yml
  - component: gitlab.example.com/platform/components/tf-plan@1.0   # CI/CD component (versioned)
```

- **Predefined vars `CI_COMMIT_SHA`, `CI_COMMIT_SHORT_SHA`, `CI_COMMIT_REF_SLUG`, `CI_PIPELINE_ID`, `CI_JOB_TOKEN`, `CI_REGISTRY_IMAGE`, `CI_MERGE_REQUEST_IID`, `CI_ENVIRONMENT_NAME`**

  Image tags, cache keys, registry auth, MR comments.

- `rules:` with `if` / `changes` / `exists` / `when` / `allow_failure`

  Per-job conditions; replaces `only/except`.

- `needs:`

  Run as soon as dependencies finish, ignoring stage order (DAG pipelines).

- `extends:` / `!reference [.job, script]`

  Inherit a hidden job / reuse one section of it.

- `trigger:`

  Child pipelines (`include` a generated file) or multi-project pipelines.

- `services: [docker:dind, postgres:16]`

  Sidecar containers for the job (Docker builds, integration tests).

- **Protected variables**

  Only exposed on protected branches/tags, so a feature branch can't read prod secrets.

- `gitlab-runner register --executor kubernetes|docker|shell`

  Register a runner; Kubernetes executor gives ephemeral pods.

- **GitLab-managed Terraform state: `backend "http" {}` + `TF_HTTP_ADDRESS=${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/${ENV}`**

  State and locking stored in GitLab; alternative to S3 for small teams.

## CodeBuild `buildspec.yml` (annotated)

```yaml
version: 0.2
env:
  variables: { AWS_REGION: ap-south-1 }                 # plain env vars
  parameter-store: { TF_VAR_db_pass: /app/db_pass }     # pulled from SSM at start, masked in logs
  secrets-manager: { DOCKERHUB_TOKEN: dockerhub:token } # secret:json-key from Secrets Manager
  exported-variables: [IMAGE_TAG]                       # make available to later CodePipeline actions
phases:
  install:
    runtime-versions: { python: 3.12 }
    commands: [curl -sLo tf.zip https://releases.hashicorp.com/terraform/1.9.5/terraform_1.9.5_linux_amd64.zip && unzip -o tf.zip -d /usr/local/bin]   # pin tools
  pre_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR   # service role auth, no keys
      - export IMAGE_TAG=${CODEBUILD_RESOLVED_SOURCE_VERSION:0:7}                       # git SHA as the tag
  build:
    commands:
      - docker build -t $ECR/app:$IMAGE_TAG .
      - trivy image --exit-code 1 --severity CRITICAL $ECR/app:$IMAGE_TAG            # fail on critical CVEs
  post_build:
    commands:
      - docker push $ECR/app:$IMAGE_TAG
      - printf '[{"name":"app","imageUri":"%s"}]' $ECR/app:$IMAGE_TAG > imagedefinitions.json   # what the ECS deploy action expects
reports:
  pytest: { files: ['reports/*.xml'], file-format: JUNITXML }   # test report tab
artifacts:
  files: [imagedefinitions.json, tfplan]                 # passed to the next pipeline stage via S3
cache:
  paths: ['/root/.cache/pip/**/*', '/root/.terraform.d/plugin-cache/**/*']   # S3/local cache between builds
```

- `privilegedMode: true` on the project

  Required to run Docker builds inside CodeBuild.

- **VPC config on the project**

  Lets the build reach private resources; then it needs NAT or VPC endpoints for AWS APIs and ECR.

- `aws codebuild start-build --project-name p --source-version main`

  Trigger a build manually.

- `aws codebuild batch-get-builds --ids ID --query 'builds[].phases[?phaseStatus==`FAILED`]'`

  Which phase failed and why.

- **Logs in CloudWatch `/aws/codebuild/<project>`**

  Full build output.

- `aws codepipeline get-pipeline-state --name p` / `start-pipeline-execution` / `put-approval-result` / `retry-stage-execution`

  Inspect / run / approve / retry a stage.

- **CodePipeline V2**

  Pipeline-level variables, triggers with branch/path filters, execution modes (queued/parallel/superseded), automatic rollback on failure.

## Pipeline hygiene checklist

Build once, deploy the same artifact everywhere · immutable tags (git SHA) · pinned tool/action versions · OIDC roles, never static keys · least-privilege role per environment · plan/preview before apply · manual approval for prod tied to a ticket · artifacts and approvals retained for audit · timeouts and concurrency locks · fail fast on lint/scan · failure notifications to a channel humans watch · tested rollback path.
