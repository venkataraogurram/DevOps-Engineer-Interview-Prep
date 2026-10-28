# 19. CI/CD – advanced and scenario questions

[← Back to index](README.md)

**Q184. Design a pipeline that satisfies segregation of duties for a regulated bank.**
Developers can merge code after review but cannot deploy to prod directly. The pipeline is the only identity with prod write access, running under a role no human can assume. Prod deployment requires an approval by a different person from the author (enforced by environment protection rules or a Jenkins `submitter` group that excludes the committer, checked in code), tied to a change ticket ID that the pipeline validates against the ITSM API (ServiceNow/Jira) for an approved window. Artifacts are built once and signed; the deploy step verifies the signature. Every run stores plan output, approver, ticket, and commit as retained artifacts. Audit can reconstruct any prod change from git plus pipeline logs plus CloudTrail.

---

**Q185. How do you make a pipeline reproducible?**
Pin everything: base images by digest, tool versions (Terraform, kubectl, Helm, Node) via version files, actions and plugins by SHA or version, dependencies via lock files, Terraform provider lock file committed. Build in clean ephemeral runners. Build once and promote the same artifact. Record the build's inputs as metadata (git SHA, dependencies, builder image) in an SBOM or provenance attestation (SLSA). Then a rebuild from the same commit yields the same image digest, or at least the same behaviour.

---

**Q186. Explain OIDC federation from CI to AWS in detail.**
The CI provider (GitHub, GitLab, Jenkins with an OIDC plugin) issues a short-lived JWT per job with claims like `sub` (repo, branch or environment), `aud`, and `job_workflow_ref`. In AWS you create an IAM OIDC identity provider for the issuer URL. The role's trust policy allows `sts:AssumeRoleWithWebIdentity` for that provider with conditions on `aud` and `sub`, for example only `repo:org/infra:environment:prod`. The job calls STS with the token and receives temporary credentials, typically for one hour. Nothing is stored. Scoping the `sub` claim to branch or environment is the control; a wildcard `sub` means any branch, including a fork PR if the workflow allows it, can assume the role.

---

**Q187. How would you implement canary deployments for a Kubernetes service with automatic rollback?**
Argo Rollouts canary strategy with steps: 5% for 5 minutes, analysis, 25%, analysis, 100%. Traffic shaping through the AWS LB Controller (weighted target groups) or a mesh. AnalysisTemplates query Prometheus or CloudWatch for error rate and p99 latency of the canary subset compared to the stable subset; a failed analysis aborts and shifts traffic back automatically. Alternatively, Lambda or ECS with CodeDeploy canary configs and CloudWatch alarms as rollback triggers. The requirement underneath is metrics labelled by version, which means the app must emit them.

---

**Q188. Your Jenkins controller is a snowflake with 400 jobs. Modernisation plan?**
Inventory jobs and classify: active, dead, duplicate. Convert freestyle jobs to Jenkinsfiles in the repos they build, standardised through a shared library. Move the controller to Configuration as Code (JCasC) in git with plugins pinned in `plugins.txt`, built as a container image, so the controller is rebuildable. Agents become ephemeral pods via the Kubernetes plugin; credentials move to Secrets Manager or Vault. Put the controller behind SSO and RBAC. Then evaluate whether GitHub Actions or GitLab CI should replace it for new work, and migrate team by team, keeping Jenkins only for what truly needs it.

---

**Q189. How do you handle secrets rotation without downtime in a pipeline-driven system?**
Applications read secrets from Secrets Manager or SSM at startup or through a sidecar/CSI driver with caching and periodic refresh; rotation uses dual-secret versions (`AWSCURRENT`/`AWSPREVIOUS`) with a rotation Lambda that creates the new credential, tests it, then flips. Database credentials via IAM database authentication avoid passwords entirely. For Kubernetes, External Secrets refreshes the Secret and a Reloader annotation triggers a rolling restart, or the app watches the mounted file. Pipelines themselves hold no long-lived secrets (OIDC), so there's nothing to rotate there.

---

**Q190. The pipeline takes 45 minutes. How do you get it under 10?**
Measure stage durations first. Typical wins: run lint, unit tests, and scans in parallel jobs; cache dependencies and Docker layers (registry cache or BuildKit cache mounts); split the monorepo pipeline with path filters so unrelated changes don't rebuild everything; use larger or more runners; shard tests; move slow end-to-end tests post-merge or nightly with a fast smoke subset in the PR; pre-built builder images with tools installed; incremental Terraform plans per changed directory; fail fast on cheap checks before expensive ones.

---

**Q191. How do you promote the same artifact through dev → staging → prod when configs differ?**
The artifact (container image or zip) is environment-agnostic; configuration is injected at deploy time: Helm values per environment, Kustomize overlays, SSM parameters by path `/app/<env>/`, environment variables from ConfigMaps/Secrets. Promotion is a metadata operation: the pipeline records the image digest that passed staging and the prod deploy uses exactly that digest, never a rebuild. Tag immutability in ECR enforces it; a promotion step can add a `prod-approved` tag to the same digest for traceability.

---

**Q192. Describe a safe database migration strategy in continuous delivery.**
Expand/contract: release 1 adds the new column or table and writes to both old and new; release 2 reads from new; release 3 drops the old. Each step is backward-compatible with the previous app version so rollback never needs a schema rollback. Migrations run as a versioned, idempotent step before the app rollout (Flyway/Liquibase with a lock), tested against a production-like snapshot in staging, with a backup or snapshot taken immediately before prod. Destructive changes are their own change ticket, never bundled with features.

---

**Q193. How do you test infrastructure changes before they hit prod?**
Static analysis and policy checks on plan; `terraform test` or Terratest applying modules into an ephemeral sandbox account and asserting on real resources; a non-prod environment deployed from the same code and pipeline; drift detection to ensure non-prod really mirrors prod; integration smoke tests after deploy (can the app reach the DB, does the ALB return 200). For Kubernetes, kind clusters in CI for manifest and Helm chart tests, `kubectl apply --dry-run=server` against a real cluster, and conftest on rendered manifests.

---

**Q194. What is a software supply chain attack and how does your pipeline defend against it?**
Compromise of a dependency, build tool, base image, or CI action that gets into your artifact: typosquatted packages, hijacked GitHub Actions tags, poisoned base images. Defences: pin dependencies and actions by hash, private registries with pull-through caches and allowlists, SCA scanning (Dependabot, Snyk, Trivy), SBOM generation, image signing with cosign and admission policies that only allow signed images from your registry, provenance attestations (SLSA), minimal and ephemeral build environments, and least-privilege tokens in CI so a compromised job can't push to main or prod.

---

**Q195. How do you manage feature flags and what do they change about deployment?**
A flag service (LaunchDarkly, AWS AppConfig, Unleash) decouples deploy from release: code ships dark, is enabled per cohort or percentage, and rolled back by flipping the flag without a deploy. Pipelines get simpler and faster because every merge can deploy. Costs: flag debt (remove flags after rollout), testing combinatorics, and the flag service becomes a dependency with its own availability requirements; default to safe values when it's unreachable.

---

**Q196. How do you handle a long-running pipeline that must deploy across 8 AWS regions?**
Fan out with a matrix job per region after a single build; make each region deploy idempotent and independently retryable; deploy in waves (one canary region, then the rest) with a health gate between waves; region-specific config from parameters, not branches; a global lock so two pipelines don't deploy the same service concurrently; aggregate results and only mark the run successful when all regions pass, with per-region rollback on failure.

[← Back to index](README.md)
