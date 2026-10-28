# CI/CD syntax cheatsheet – Jenkins, GitHub Actions, GitLab CI, CodeBuild

[← Cheatsheet index](README.md)

## Concept map

| Concept | Jenkins | GitHub Actions | GitLab CI | CodePipeline/CodeBuild |
|---|---|---|---|---|
| Definition file | `Jenkinsfile` | `.github/workflows/*.yml` | `.gitlab-ci.yml` | pipeline JSON/CDK + `buildspec.yml` |
| Executor | agent/node | runner (`runs-on`) | runner (`tags`, executor) | CodeBuild project |
| Unit of work | stage → steps | job → steps | job (in stage) → script | action → build phases |
| Parallel | `parallel {}` | separate jobs, `matrix` | same stage, `parallel:` | parallel actions in a stage |
| Dependencies | sequential stages | `needs:` | `stages:` order, `needs:` DAG | stages |
| Secrets | Credentials store, `withCredentials` | `${{ secrets.X }}`, environments | CI/CD variables (masked/protected) | Secrets Manager / SSM in `env.secrets-manager` |
| Approval | `input` step | environment required reviewers | `when: manual`, protected env | Manual approval action |
| Artifacts | `archiveArtifacts`, `stash/unstash` | `upload-artifact`/`download-artifact` | `artifacts:` paths, `expire_in` | S3 artifact store |
| Cache | plugins / workspace | `actions/cache` | `cache:` key/paths | S3 or local cache |
| Reuse | Shared Libraries | reusable workflows (`workflow_call`), composite actions | `include:`, `extends:`, anchors, components | CDK constructs |
| Conditions | `when {}` | `if:` | `rules:` | — |
| Triggers | webhook, `pollSCM`, `cron` | `on: push/pull_request/schedule/workflow_dispatch` | push, MR, `schedule`, `trigger` | CodeConnections, EventBridge, schedule |
| AWS auth | `withAWS(role:)`, IRSA on K8s agents | OIDC `aws-actions/configure-aws-credentials` | `id_tokens` + `assume-role-with-web-identity` | service role |

## Jenkinsfile (declarative)

```groovy
@Library('platform-lib@v2') _
pipeline {
  agent none
  options { timeout(time: 60, unit: 'MINUTES'); timestamps(); disableConcurrentBuilds(); ansiColor('xterm') }
  parameters { choice(name: 'ENV', choices: ['dev','prod']); booleanParam(name: 'DESTROY', defaultValue: false) }
  environment { AWS_REGION = 'ap-south-1'; TF_IN_AUTOMATION = '1' }
  triggers { cron('H 2 * * 1-5') }
  stages {
    stage('Checkout') { agent { label 'linux' } steps { checkout scm; stash name: 'src', includes: '**' } }
    stage('Validate') {
      agent { docker { image 'hashicorp/terraform:1.9'; args '-u root' } }
      steps { unstash 'src'; sh 'terraform fmt -check && terraform init -backend=false && terraform validate' }
    }
    stage('Plan') {
      agent { kubernetes { yamlFile 'ci/agent.yaml'; defaultContainer 'terraform' } }
      steps {
        unstash 'src'
        withCredentials([usernamePassword(credentialsId: 'x', usernameVariable: 'U', passwordVariable: 'P')]) { sh 'echo $U' }
        withAWS(role: "arn:aws:iam::123:role/jenkins-${params.ENV}", roleSessionName: "jenkins-${env.BUILD_NUMBER}") {
          sh "terraform init -backend-config=envs/${params.ENV}.backend.hcl && terraform plan -var-file=envs/${params.ENV}.tfvars -out=tfplan"
        }
        stash name: 'plan', includes: 'tfplan,.terraform.lock.hcl'
        script { env.PLAN_SUMMARY = sh(script: 'terraform show -no-color tfplan | tail -1', returnStdout: true).trim() }
      }
    }
    stage('Approve') {
      when { expression { params.ENV == 'prod' } }
      steps { timeout(time: 4, unit: 'HOURS') { input message: "Apply? ${env.PLAN_SUMMARY}", submitter: 'platform-leads', parameters: [string(name: 'TICKET')] } }
    }
    stage('Apply') {
      agent { label 'linux' }
      steps { unstash 'src'; unstash 'plan'; withAWS(role: "...") { sh 'terraform apply -input=false tfplan' } }
    }
  }
  post {
    always { junit allowEmptyResults: true, testResults: 'reports/*.xml'; archiveArtifacts artifacts: 'tfplan.txt', allowEmptyArchive: true; cleanWs() }
    success { slackSend channel: '#deploys', color: 'good', message: "${env.JOB_NAME} #${env.BUILD_NUMBER} OK" }
    failure { slackSend channel: '#deploys', color: 'danger', message: "${env.JOB_NAME} #${env.BUILD_NUMBER} FAILED ${env.BUILD_URL}" }
  }
}
```

Shared library `vars/terraformDeploy.groovy`: `def call(Map cfg) { pipeline { ... } }` → `terraformDeploy(env: 'prod')`. Scripted: `node('linux') { stage('x') { sh '...' } }`. Useful steps: `sh(returnStatus:true)`, `readJSON`, `writeFile`, `fileExists`, `retry(3)`, `waitUntil`, `lock('resource')`, `milestone`, `catchError`, `error 'msg'`, `currentBuild.result`, `build job: 'other', parameters: [...]`. Jenkins CLI: `java -jar jenkins-cli.jar -s URL -auth u:t list-jobs | build job -p K=V | console job | reload-configuration | safe-restart`. JCasC: `JENKINS_HOME/casc_configs/jenkins.yaml`.

## GitHub Actions

```yaml
name: deploy
on:
  push: { branches: [main], paths: ['infra/**'] }
  pull_request:
  workflow_dispatch: { inputs: { env: { type: choice, options: [dev, prod], default: dev } } }
  schedule: [{ cron: '0 2 * * 1-5' }]
permissions: { id-token: write, contents: read, pull-requests: write }
concurrency: { group: deploy-${{ github.ref }}, cancel-in-progress: false }
defaults: { run: { shell: bash, working-directory: infra } }
env: { AWS_REGION: ap-south-1, TF_VERSION: 1.9.5 }
jobs:
  plan:
    runs-on: ubuntu-latest           # or [self-hosted, linux, vpc]
    strategy: { fail-fast: false, matrix: { env: [dev, staging] } }
    environment: ${{ matrix.env }}
    timeout-minutes: 30
    outputs: { exitcode: ${{ steps.plan.outputs.exitcode }} }
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: ${{ vars.PLAN_ROLE_ARN }}, aws-region: ${{ env.AWS_REGION }}, role-session-name: gha-${{ github.run_id }} }
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: ${{ env.TF_VERSION }}, terraform_wrapper: false }
      - uses: actions/cache@v4
        with: { path: ~/.terraform.d/plugin-cache, key: tf-${{ hashFiles('**/.terraform.lock.hcl') }} }
      - run: terraform init -input=false -backend-config=envs/${{ matrix.env }}.backend.hcl
      - id: plan
        run: |
          set +e
          terraform plan -input=false -no-color -detailed-exitcode -var-file=envs/${{ matrix.env }}.tfvars -out=tfplan | tee plan.txt
          echo "exitcode=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"
      - if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs'); const plan = fs.readFileSync('infra/plan.txt','utf8').slice(-60000);
            github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body: '```\n' + plan + '\n```' });
      - uses: actions/upload-artifact@v4
        with: { name: tfplan-${{ matrix.env }}, path: infra/tfplan, retention-days: 1 }
  apply:
    needs: plan
    if: github.ref == 'refs/heads/main' && needs.plan.outputs.exitcode == '2'
    runs-on: ubuntu-latest
    environment: { name: prod, url: https://app.example.com }      # required reviewers = approval gate
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: tfplan-dev, path: infra }
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: ${{ vars.APPLY_ROLE_ARN }}, aws-region: ${{ env.AWS_REGION }} }
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -input=false -backend-config=envs/prod.backend.hcl && terraform apply -input=false tfplan
      - if: failure()
        run: curl -X POST -H 'Content-type: application/json' --data '{"text":"deploy failed ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}' ${{ secrets.SLACK_WEBHOOK }}
```

Reusable workflow: `on: workflow_call: { inputs: {...}, secrets: {...} }` → caller `jobs: x: uses: org/repo/.github/workflows/tf.yml@v1 with: {...} secrets: inherit`. Composite action: `action.yml` with `runs: { using: composite, steps: [...] }`. Contexts: `github.*`, `env.*`, `vars.*`, `secrets.*`, `needs.*`, `steps.*.outputs`, `runner.*`, `matrix.*`. Functions: `contains() startsWith() format() join() toJSON() fromJSON() hashFiles() success() failure() always() cancelled()`. Step outputs: `echo "k=v" >> $GITHUB_OUTPUT`; env: `>> $GITHUB_ENV`; summary: `>> $GITHUB_STEP_SUMMARY`; `::add-mask::`, `::warning::`, `::group::`. Security: pin actions by SHA, `permissions` minimal, no `pull_request_target` with checkout of PR head, Dependabot for actions. `gh run list|view|watch|rerun`, `gh workflow run deploy -f env=prod`, `act` for local.

## GitLab CI

```yaml
stages: [validate, plan, apply]
default:
  image: hashicorp/terraform:1.9
  tags: [vpc-runner]
  interruptible: true
  retry: { max: 1, when: [runner_system_failure] }
variables:
  TF_ROOT: ${CI_PROJECT_DIR}/infra
  TF_IN_AUTOMATION: "1"
  AWS_REGION: ap-south-1
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
.aws: &aws
  id_tokens: { GITLAB_OIDC_TOKEN: { aud: "https://gitlab.example.com" } }
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
  rules: [{ changes: ["infra/**/*"] }]
plan:
  stage: plan
  <<: *aws
  parallel: { matrix: [{ ENV: [dev, staging] }] }
  script:
    - terraform plan -input=false -var-file=envs/$ENV.tfvars -out=tfplan
    - terraform show -json tfplan > plan.json
  artifacts:
    paths: [$TF_ROOT/tfplan]
    reports: { terraform: $TF_ROOT/plan.json }     # MR widget
    expire_in: 1 day
  cache: { key: "$CI_COMMIT_REF_SLUG", paths: [$TF_ROOT/.terraform] }
apply:prod:
  stage: apply
  <<: *aws
  variables: { ENV: prod }
  script: [terraform apply -input=false tfplan]
  needs: [{ job: plan, artifacts: true }]
  environment: { name: production, url: https://app.example.com, deployment_tier: production }
  when: manual
  allow_failure: false
  rules: [{ if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH' }]
  resource_group: prod                                  # serialize deploys
include:
  - project: platform/ci-templates
    ref: v2
    file: /templates/security-scan.yml
  - component: gitlab.example.com/platform/components/tf-plan@1.0
```

Predefined vars: `CI_COMMIT_SHA CI_COMMIT_SHORT_SHA CI_COMMIT_REF_SLUG CI_PIPELINE_ID CI_JOB_TOKEN CI_REGISTRY CI_REGISTRY_IMAGE CI_MERGE_REQUEST_IID CI_ENVIRONMENT_NAME CI_PROJECT_PATH`. `rules:` with `if / changes / exists / when / allow_failure`; `needs:` for DAG; `extends:`; `!reference [.job, script]`; `trigger:` child/multi-project pipelines; `services:` (docker:dind, postgres); `dependencies:`; protected variables only on protected branches; `gitlab-runner register --executor kubernetes|docker|shell`; GitLab-managed Terraform state: `backend "http" {}` with `TF_HTTP_ADDRESS=${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/${ENV}`.

## CodeBuild `buildspec.yml`

```yaml
version: 0.2
env:
  variables: { AWS_REGION: ap-south-1 }
  parameter-store: { TF_VAR_db_pass: /app/db_pass }
  secrets-manager: { DOCKERHUB_TOKEN: dockerhub:token }
  exported-variables: [IMAGE_TAG]
phases:
  install:
    runtime-versions: { python: 3.12 }
    commands: [curl -sLo tf.zip https://releases.hashicorp.com/terraform/1.9.5/terraform_1.9.5_linux_amd64.zip && unzip -o tf.zip -d /usr/local/bin]
  pre_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR
      - export IMAGE_TAG=${CODEBUILD_RESOLVED_SOURCE_VERSION:0:7}
  build:
    commands:
      - docker build -t $ECR/app:$IMAGE_TAG .
      - trivy image --exit-code 1 --severity CRITICAL $ECR/app:$IMAGE_TAG
  post_build:
    commands:
      - docker push $ECR/app:$IMAGE_TAG
      - printf '[{"name":"app","imageUri":"%s"}]' $ECR/app:$IMAGE_TAG > imagedefinitions.json
reports:
  pytest: { files: ['reports/*.xml'], file-format: JUNITXML }
artifacts:
  files: [imagedefinitions.json, tfplan]
cache:
  paths: ['/root/.cache/pip/**/*', '/root/.terraform.d/plugin-cache/**/*']
```

CodeBuild needs `privilegedMode: true` for Docker builds; VPC config for private resources (needs NAT or endpoints); `aws codebuild start-build --project-name p --source-version main`; `aws codebuild batch-get-builds --ids ID --query 'builds[].phases[?phaseStatus==`FAILED`]'`; logs in CloudWatch `/aws/codebuild/p`. CodePipeline: `aws codepipeline get-pipeline-state --name p`, `start-pipeline-execution`, `put-approval-result`, `retry-stage-execution`; V2 pipelines: variables, triggers with path filters, `ROLLBACK` on failure.

## Universal pipeline hygiene checklist

Build once, deploy the same artifact · immutable tags (SHA) · pinned tool versions · OIDC not keys · least-privilege role per environment · plan/preview before apply · manual approval for prod with ticket ID · artifacts retained for audit · timeouts and concurrency locks · fail fast on lint/scan · notify on failure to a channel humans watch · rollback path tested.
