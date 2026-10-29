# Hambisa — DevOps and Platform Engineering Specification

29 October 2026 · Musa

## 1. Purpose and scope

Hambisa runs its C# services and Angular portals on managed Kubernetes. Argo CD deploys every change from Git, and no person or pipeline holds production credentials. This document turns the product specification's platform lines into requirements engineering can estimate. It covers sections 9.1, 10, 11, 13.3 and 14 of the specification, which name Docker, Terraform, CI/CD and a managed secret store but stop short of a delivery design.

| In scope | Out of scope |
| --- | --- |
| Cloud accounts, networks and Kubernetes clusters | Application business logic and APIs |
| Container build standards | Payment, map and identity provider selection |
| CI pipelines for backend and Angular; Expo EAS pipelines for the React Native apps | App store listing content |
| GitOps delivery with Argo CD, promotion and rollouts | Corporate IT, laptops and office networks |
| Secret management, supply chain and cluster security |  |
| Observability, backup, disaster recovery and platform launch gates |  |

Requirements marked **Must** block release, as in the product specification. **Should** items are expected for the MVP unless the platform lead records a deferral.

## 2. Principles and key decisions

Git is the single source of truth for infrastructure, cluster configuration and deployed versions. Everything below follows from that.

1. **Everything as code.** Terraform for cloud resources, Helm or Kustomize for workloads, Argo CD for delivery. Manual console changes are treated as drift and reverted.
2. **Pull, don't push.** CI builds and publishes artefacts but never deploys. Argo CD inside each cluster pulls the desired state from Git.
3. **Build once, promote the same image.** One immutable image digest moves from dev to staging to production. Environments differ only in configuration.
4. **No long-lived credentials.** Pipelines and workloads use workload identity (OIDC federation). Secrets live in the cloud secret store, never in Git or CI variables.
5. **Small blast radius.** Production is its own cloud account or subscription and its own cluster.
6. **Boring and managed first.** Managed Kubernetes, PostgreSQL, Redis, broker and secret store before self-hosted options, matching the specification's "modular before many services" stance.

| Decision | Choice | Why |
| --- | --- | --- |
| Cloud | AWS, af-south-1 (Cape Town) | Local region for POPIA data residency and low latency to the Cape Town launch area |
| Accounts | AWS Organizations with Control Tower; IAM Identity Center for staff sign-in with MFA | One account per environment, central guardrails |
| Kubernetes | Amazon EKS with Karpenter | Managed control plane, fast right-sized node scaling |
| Workload identity | EKS Pod Identity (IRSA where an add-on needs it) | No static AWS keys in pods |
| Container registry | Amazon ECR, immutable tags, scan on push | Private, works with Cosign signatures |
| Secret store | AWS Secrets Manager, encrypted with KMS | Managed rotation, CloudTrail audit |
| Database | Amazon RDS for PostgreSQL, Multi-AZ | Spec section 6.4 |
| Cache and location snapshots | Amazon ElastiCache for Redis | Spec section 6.4 |
| Messaging | Amazon SQS and SNS | Listed in spec section 11; no Kafka without a throughput need |
| Real time | ASP.NET Core SignalR on EKS with an ElastiCache Redis backplane | Azure SignalR Service is not on AWS; spec section 11 allows a Redis backplane |
| GitOps | Argo CD | Web UI for operators, ApplicationSets, mature RBAC |
| Progressive delivery | Argo Rollouts with ALB traffic splitting | Canary with metric analysis |
| CI | GitHub Actions with OIDC to IAM roles | Code lives in GitHub (musa-zulu); no stored AWS keys |
| IaC | Terraform, S3 remote state with native locking | Named in spec section 11 |
| Ingress and edge | AWS Load Balancer Controller (ALB), AWS WAF, Route 53 | Managed TLS termination with ACM certificates |
| In-cluster TLS | cert-manager | Automatic renewal |
| Policy | Kyverno | Policies written as Kubernetes YAML, simpler than Rego |

AWS is the confirmed cloud. af-south-1 is an opt-in region, so it must be enabled in every account before Terraform runs.

### Mobile and client stack

The customer and rider apps use React Native on Expo, with Clerk for sign-in, NativeWind for styling, Sanity for editable content and PostHog for product analytics and feature flags.

| Layer | Choice | Platform impact |
| --- | --- | --- |
| Framework | React Native with TypeScript on Expo (development builds, config plugins) | EAS Build, Submit and Update replace self-managed mobile build machines (section 8) |
| Authentication | Clerk (`@clerk/clerk-expo`) | Identity provider for customers and riders. Services verify Clerk session tokens against Clerk's JWKS; a Clerk webhook keeps the Identity/Profile service in sync |
| Styling | NativeWind (Tailwind CSS for React Native) | One shared Tailwind config holds the colour, type and spacing tokens from the UI foundations board |
| Content | Sanity | Help articles, FAQs, onboarding and promo copy, legal text versions. Never orders, prices or other operational state |
| Analytics and flags | PostHog | Funnels for the spec's booking completion rate (section 1.4), feature flags for staged releases and kill switches, session replay off by default |

Clerk, Sanity and PostHog are hosted outside South Africa, so each needs a POPIA cross-border review and an operator agreement before launch (section 16).

## 3. Environments and cloud accounts

Four environments, each in its own AWS account under AWS Organizations, so a mistake in one cannot reach production data.

| Environment | Purpose | Cluster | Data | Deploys | Access |
| --- | --- | --- | --- | --- | --- |
| Preview | One namespace per open pull request | Shared dev cluster | Seeded fixtures, destroyed on PR close | Automatic per PR | Engineers |
| Dev | Integration of merged work | Dev cluster | Synthetic only | Automatic on merge to main | Engineers |
| Staging | Release candidate, load and payment-sandbox tests | Staging cluster, production-like size ratio | Synthetic plus anonymised samples, never real personal data | Automatic after dev checks pass | Engineers read, platform team write |
| Production | Customers, riders, staff | Production cluster, zone-redundant | Real | Manual promotion approval | Break-glass only, logged |

**Must**

- Separate AWS accounts for non-production and production, a shared platform account for ECR, Route 53 and Terraform state, and a log archive account for CloudTrail.
- Production personal data never copied to lower environments (POPIA, spec section 9.2).
- Separate payment, map, SMS and push provider credentials per environment (spec section 9.1).
- Environment name visible in every back-office page header outside production, so staff never act on the wrong system.

**Should**

- Preview environments for Angular and backend pull requests, deleted automatically after 72 hours or on merge.
- Non-production clusters scale to a minimum node count outside South African working hours to control cost.

## 4. Containers and Docker

Every deployable is one container image built from a multi-stage Dockerfile, running as a non-root user on a pinned minimal base.

| Workload | Build stage | Runtime base | Notes |
| --- | --- | --- | --- |
| C# services (Orders, Pricing, Dispatch, Riders, Payments, Tracking, Notifications, Support, Reporting, Identity/Profile) | `mcr.microsoft.com/dotnet/sdk:10.0` | `mcr.microsoft.com/dotnet/aspnet:10.0` chiseled (distroless) | `dotnet publish -c Release`, no SDK in runtime image |
| Angular back office and merchant portal | `node:22-alpine` running `ng build` | `nginxinc/nginx-unprivileged` | Static files only; runtime config injected at start, not baked in |
| Database migration jobs | .NET SDK | aspnet chiseled | One per service, runs `ef migrations bundle` output |
| React Native apps | Not containerised | — | Built and signed by Expo EAS Build (section 8) |

**Must**

- Base images pinned by digest and updated by Renovate or Dependabot pull requests.
- Process runs as a non-root UID with a read-only root filesystem; writable paths mounted as `emptyDir`.
- No secrets, `.env` files or connection strings in images or build arguments.
- Image tag = Git commit SHA; deployments reference the immutable digest. The tag `latest` is never deployed.
- `HEALTHCHECK` replaced by Kubernetes probes: `/health/live` and `/health/ready` on every service, `ready` failing when PostgreSQL, Redis or the broker is unreachable.
- Graceful shutdown: services stop taking traffic on SIGTERM and finish in-flight requests within 30 seconds.

**Should**

- Runtime images under 150 MB for services and 50 MB for Angular.
- `docker compose` file in each repo for local development with PostgreSQL, Redis and a broker emulator.

## 5. Kubernetes platform

Each environment gets one managed cluster with a private API server, three availability zones in production, and separate node pools for system and application workloads.

### Cluster layout

| Node pool | Runs | Production sizing (starting point) | Scaling |
| --- | --- | --- | --- |
| system | Argo CD, ingress, cert-manager, External Secrets, Kyverno, monitoring agents | 3 nodes, 4 vCPU / 16 GB, one per zone | Fixed |
| apps | Hambisa services and portals | 3–9 nodes, 4 vCPU / 16 GB | Karpenter |
| jobs | Migrations, exports, reconciliation batch | 0–3 nodes, EC2 Spot where the job can retry | Scale to zero |

Sizing is a placeholder until the peak volume forecast in spec section 15 exists; load tests set the final figures.

### Namespaces

| Namespace | Contents |
| --- | --- |
| `platform-*` | Argo CD, ingress, cert-manager, external-secrets, kyverno, observability |
| `hambisa-core` | Identity/Profile, Orders, Pricing, Dispatch, Riders, Tracking |
| `hambisa-money` | Payments and reconciliation, with stricter network and RBAC rules |
| `hambisa-ops` | Notifications, Support, Reporting, back-office and merchant web |
| `pr-<number>` | Preview environments (dev cluster only) |

### Workload standards

**Must**

- CPU and memory requests and limits on every container; a namespace `LimitRange` and `ResourceQuota` back this up.
- Liveness, readiness and startup probes on every service.
- At least 2 replicas and a `PodDisruptionBudget` for every user-facing service in production, spread across zones with topology spread constraints.
- Horizontal Pod Autoscaler on CPU for APIs; KEDA on queue length for broker consumers (dispatch, notifications).
- Default-deny `NetworkPolicy` per namespace; only declared service-to-service paths allowed. Payments reachable only from Orders and the webhook ingress route.
- Pod Security Standard `restricted` enforced on all application namespaces.
- EKS Pod Identity per service account to reach Secrets Manager, S3, SQS and SNS; no AWS access keys in pods.
- Live tracking runs ASP.NET Core SignalR on EKS with an ElastiCache Redis backplane and ALB sticky sessions, sized from connection count (spec section 11).

**Should**

- Ingress exposes three hostnames per environment: `api.`, `ops.` (back office, behind the identity provider and allowed IP ranges) and `business.` (merchant portal).
- Webhook endpoints for payment providers on a dedicated ingress path with request size limits and provider IP allow-lists where the provider publishes them.
- Cluster upgrades one minor version at a time, staging first, within 60 days of a new supported version.

## 6. Infrastructure as code with Terraform

Terraform owns every cloud resource below the cluster; Argo CD owns everything inside it. Nothing is created by hand.

| Terraform layer | Contains | Change rate | Applied by |
| --- | --- | --- | --- |
| `bootstrap` | S3 state bucket, GitHub OIDC provider and IAM roles, AWS Organizations accounts | Rare | Platform lead, locally, two-person review |
| `foundation` | VPCs across 3 AZs, Route 53 zones, Secrets Manager, KMS keys, ECR, CloudWatch log groups | Monthly | Pipeline after approval |
| `data` | RDS for PostgreSQL, ElastiCache, S3 buckets, SQS queues and SNS topics | Monthly | Pipeline after approval, `prevent_destroy` on |
| `cluster` | EKS, Karpenter, Pod Identity associations, Argo CD bootstrap | Monthly | Pipeline after approval |

**Must**

- Remote state in S3 per environment and layer, KMS-encrypted, versioned and locked; state access limited to the CI identity and platform team.
- Every change through a pull request that posts `terraform plan` output; `apply` runs only from the pipeline, after approval for staging and production.
- `prevent_destroy` on RDS, S3 and Secrets Manager; RDS deletion protection and S3 Object Lock enabled on the cloud side as well.
- Modules versioned and shared across environments; environment folders hold variables only.
- Nightly `terraform plan` against production; any drift opens an issue for the platform team.
- `tflint`, `checkov` or `tfsec`, and `terraform fmt` checks on every pull request.

**Should**

- Infracost comment on pull requests that change compute or database tiers.
- Argo CD installed by Terraform once (Helm release), then Argo CD manages its own upgrades from Git ("app of apps").

## 7. Repositories, branching and versioning

Application code and deployment configuration live in separate repositories, so a deploy is a reviewed commit and CI never needs cluster access.

| Repository | Holds | Who merges |
| --- | --- | --- |
| `hambisa` (existing) | Product spec, UI designs, architecture decisions | Product and tech leads |
| `hambisa-services` | C# services as a monorepo, one folder per service, shared contracts | Backend engineers |
| `hambisa-web` | Angular back office and merchant portal | Web engineers |
| `hambisa-mobile` | React Native customer and rider apps | Mobile engineers |
| `hambisa-infra` | Terraform modules and environment variables | Platform team |
| `hambisa-gitops` | Helm charts or Kustomize bases, per-environment values, Argo CD Applications | CI bot for dev/staging; platform team approves production |

A monorepo per platform keeps the specification's modular-service approach cheap to run; services can move to their own repositories later without changing the delivery model.

### Branching

- Trunk-based: short-lived feature branches into `main`, squash-merged after review and green checks.
- `main` is always deployable; unfinished work is hidden behind PostHog feature flags, which also cover the spec's "rollback or disablement plan for each major feature".
- Protected `main`: one approving review (two for `hambisa-money` paths and `hambisa-infra`), signed commits, no force-push, required status checks.
- `CODEOWNERS` routes payment, dispatch and security-sensitive paths to named reviewers.

### Versioning

- Images tagged with the Git SHA; human-readable releases use SemVer tags (`orders-v1.4.0`) generated from Conventional Commits.
- Mobile apps: SemVer marketing version plus a monotonically increasing build number per store.
- Event schemas carry their version in the event type (`delivery.order.ready-for-dispatch.v1`, spec section 6.3); contract tests block breaking changes.

## 8. CI pipelines

GitHub Actions builds, tests, scans, signs and publishes; its last step is a commit to `hambisa-gitops`, never a `kubectl` or `helm` call.

| Stage | C# services | Angular portals | React Native apps |
| --- | --- | --- | --- |
| 1. Trigger | PR and merge to `main`, only for changed service folders | PR and merge to `main` | PR, merge to `main`, release tag |
| 2. Static checks | `dotnet format`, analyzers as errors, secret scan (gitleaks) | ESLint, Prettier, `tsc --noEmit`, secret scan | ESLint, `tsc`, secret scan |
| 3. Unit tests | NUnit + NSubstitute + FluentAssertions, coverage reported | Jest or Vitest | Jest + React Native Testing Library |
| 4. Integration tests | Testcontainers: PostgreSQL, Redis, broker; outbox and idempotency cases | — | — |
| 5. Contract tests | Consumer/provider API and event schema checks | Against API OpenAPI spec | Against API OpenAPI spec |
| 6. Dependency scan | `dotnet list package --vulnerable`, SCA tool | `npm audit` or SCA tool | Same |
| 7. Build artefact | Docker image, tag = SHA | Docker image (nginx), tag = SHA | EAS Build: Android AAB and iOS IPA, one build profile per environment |
| 8. Image scan + SBOM | Trivy or Grype; fail on Critical/High with a fix | Same | — |
| 9. Sign + push | Cosign signature and SBOM attestation, push to registry | Same | EAS-managed signing credentials |
| 10. E2E | Playwright against the PR's preview environment | Playwright | Maestro flows on EAS simulator and emulator builds |
| 11. Hand-off | Bot opens or merges a commit bumping the image digest in `hambisa-gitops` dev values | Same | EAS Submit to TestFlight and the Play internal track |

**Must**

- CI authenticates to the cloud with OIDC federation only; no stored cloud keys in GitHub secrets.
- The spec's release-blocking tests (section 12.1) run in the pipeline: idempotent order create, expired quote, concurrent offer accept, unapproved rider, delivery without proof, forged webhook, cross-customer access, stale location, refund self-approval, duplicate event.
- A failing check blocks merge; no admin override without a logged reason.
- Pipelines pinned to action versions by commit SHA.
- Build time under 15 minutes for a single-service change.

**Should**

- Reusable workflow templates in a shared `.github` repository so every service gets the same gates.
- EAS Update (over-the-air) limited to JavaScript and asset changes on a matching runtime version, with the same review and staging rules as store builds.

### Mobile delivery with Expo

EAS builds, signs and ships the apps; GitHub Actions starts the EAS jobs and runs the checks around them. Each environment maps to its own EAS profile, update channel and hosted-service project.

| Environment | EAS build profile | EAS Update channel | Clerk | Sanity dataset | PostHog project | Distribution |
| --- | --- | --- | --- | --- | --- | --- |
| Dev | `development` | `development` | Development instance | `development` | Hambisa Dev | Development client, engineers only |
| Staging | `preview` | `preview` | Separate staging application | `staging` | Hambisa Staging | TestFlight and Play internal track |
| Production | `production` | `production` | Production instance | `production` | Hambisa | App Store and Google Play |

**Must**

- `runtimeVersion` follows the app version policy, so an over-the-air update never reaches a binary it cannot run.
- EAS Update releases to production start at 10% of users and widen after crash and error checks; rollback republishes the previous update.
- Only public identifiers ship in the app bundle: Clerk publishable key, PostHog project key, Sanity project ID and dataset name. Everything else stays in Secrets Manager or EAS secrets (section 10).
- EAS authenticates with a robot access token kept as a protected GitHub environment secret for the mobile repository only, rotated every 90 days.
- Config plugins for background location (`expo-location`, `expo-task-manager`), maps and push notifications are pinned and tested on the release device matrix before each Expo SDK upgrade.
- Expo SDK upgrades one version at a time, staging first, within 3 months of a new SDK.

**Should**

- Sanity Studio and its schemas live in Git and deploy through CI; content editors publish to the `staging` dataset first, then promote to `production`.
- PostHog session replay only on screens without personal data, with all inputs masked; never on payment, OTP, delivery code or document upload screens.
- PostHog person profiles keyed by the internal user ID, never by phone number or email.

## 9. GitOps delivery with Argo CD

Argo CD in each cluster pulls the desired state from `hambisa-gitops`; promotion is a pull request that copies an image digest from one environment's values to the next.

```
Pull request merged --> GitHub Actions (build, test, scan, sign) --> ECR (signed image + SBOM)
                              |
                              v  bot bumps image digest
hambisa-gitops:  envs/dev  --promote-->  envs/staging  --promote (PR + approval)-->  envs/prod
                    |                        |                                        |
              Argo CD pulls            Argo CD pulls                            Argo CD pulls
                    v                        v                                        v
               Dev cluster            Staging cluster                     Production cluster
           (every merge + PR       (E2E, load, payment               (Argo Rollouts canary,
              previews)               sandbox tests)                 auto-abort on SLO breach)

AWS Secrets Manager --> External Secrets Operator --> Kubernetes Secrets in each cluster

Mobile: GitHub Actions --> EAS Build --> EAS Submit (TestFlight / Play) and EAS Update channels
```

Clusters pull images from the registry by digest. Rolling back production is a `git revert` on `prod values`, which Argo CD applies within its sync interval.

### GitOps repository layout

```
hambisa-gitops/
  apps/                     # one Helm chart (or Kustomize base) per service
    orders/  dispatch/  payments/  ...  backoffice-web/  merchant-web/
  platform/                 # ingress, cert-manager, external-secrets, kyverno, monitoring
  envs/
    dev/      values per app, image digests
    staging/  values per app, image digests
    prod/     values per app, image digests
  argocd/
    projects/               # AppProjects: platform, core, money, ops
    applicationsets/        # generate one Application per app per env
```

### Argo CD

**Must**

- One Argo CD per cluster ("pull per cluster"), so production holds no credentials for other clusters and vice versa.
- `AppProject`s restrict each team to its namespaces and repositories; Payments in its own project.
- Automated sync with self-heal and prune in dev and staging. Production: automated sync of the approved commit, self-heal on, prune requiring manual confirmation.
- Argo CD UI behind the identity provider with MFA (spec section 9.1); local admin disabled after bootstrap.
- Sync waves: namespaces and secrets first, migration job second, workloads third.
- Sync and health notifications to the team channel; a failed or degraded production sync pages the on-call engineer.

### Promotion

1. Merge to `main` → CI bot commits the new digest to `envs/dev`.
2. Dev sync healthy and smoke tests green → bot opens a pull request to `envs/staging`, auto-merged when checks pass.
3. Staging E2E, contract and load suites green → bot opens a pull request to `envs/prod` with the change list.
4. Release approver (not the author) approves; changes to Payments need a second approver.
5. Argo Rollouts runs the canary in production.

### Progressive delivery

| Step | Traffic to new version | Hold | Analysis that must pass |
| --- | --- | --- | --- |
| 1 | 10% | 5 min | 5xx rate below baseline + 0.5 pp; p95 latency within spec budget (300 ms reads, 700 ms writes) |
| 2 | 50% | 10 min | Same, plus no rise in payment webhook failures or assignment conflicts |
| 3 | 100% | — | Rollout complete; previous ReplicaSet kept for 1 hour |

A failed analysis aborts the rollout and shifts traffic back automatically. Broker consumers (dispatch, notifications) use blue-green instead of canary, so two versions never compete for the same messages with different logic.

**Should**

- Production deploy window Monday to Thursday, 09:00–15:00 SAST, outside lunch and evening delivery peaks; hotfixes exempt with on-call approval.
- Argo CD Image Updater not used: digests change only through reviewed commits.

## 10. Secret management

Secrets live only in AWS Secrets Manager, encrypted with KMS; External Secrets Operator copies them into Kubernetes, and Git holds only references to them.

| Secret | Store | Consumed by | Rotation |
| --- | --- | --- | --- |
| PostgreSQL access | RDS IAM authentication through Pod Identity where supported; otherwise per-service database users with passwords in Secrets Manager | Each service, own schema only | Passwordless; or 90 days, automated |
| Redis and broker access | Pod Identity for SQS and SNS; ElastiCache auth token in Secrets Manager | Services | 90 days |
| Payment provider API keys and webhook signing secret | Secrets Manager, `hambisa-money` resource policy allows only the Payments role | Payments | On provider schedule or 180 days; immediately on suspected leak |
| Map, SMS, email and push keys | Secrets Manager | Tracking, Notifications, Pricing | 180 days |
| Identity provider client secrets | Secrets Manager | API gateway, Angular backend-for-frontend | 180 days, or certificate credentials |
| TLS certificates | ACM on the ALB; cert-manager inside the cluster | Ingress | Renewed 30 days before expiry |
| Mobile signing keys, store API keys | Secrets Manager, CI identity only | Mobile pipeline | Per store rules; upload key separate from app signing key |
| Terraform state access | Cloud RBAC, no shared keys | CI identity | — |

Mobile and hosted-service credentials:

| Secret | Store | Consumed by | Rotation |
| --- | --- | --- | --- |
| Clerk secret key and webhook signing secret | Secrets Manager | Identity/Profile service, API gateway | 180 days |
| Clerk publishable key | App config, public by design | Mobile apps | When the Clerk instance changes |
| Sanity write and draft-preview tokens | Secrets Manager; Studio users sign in through Sanity | CI content scripts, preview tooling | 90 days |
| PostHog project API key | App config, public by design | Mobile apps and web portals | — |
| PostHog personal API key | Secrets Manager | Flag management and data export scripts | 90 days |
| EAS robot access token | GitHub environment secret, mobile repository only | Mobile CI | 90 days |
| APNs key and FCM service account | Secrets Manager, uploaded to EAS credentials | Notifications service, EAS | Yearly review |

**Must**

- No secret in Git, container images, Helm values, CI logs or GitHub repository secrets, other than the OIDC trust configuration and the EAS robot token (section 8).
- External Secrets Operator with one `SecretStore` per namespace, each bound to a Pod Identity role that can read only that namespace's secrets.
- Secrets kept in separate AWS accounts for non-production and production; production access requires MFA through IAM Identity Center and is logged in CloudTrail (spec section 9.1).
- Services read secrets at start-up and on change notification; a rotated secret reaches pods without a rebuild (Reloader or native reload).
- gitleaks pre-commit hook and CI step; GitHub secret scanning and push protection enabled on all repositories.
- Leak runbook: revoke, rotate, redeploy, review access logs, record as an incident.

**Should**

- Customer-managed KMS keys for RDS and the S3 buckets holding evidence photos and identity documents (spec section 9.2).
- Expiry alerts 30 days before any secret or certificate expires.

## 11. Supply chain and platform security

Only signed images built by Hambisa's own pipeline, scanned clean of fixable critical vulnerabilities, can run in staging or production.

**Must**

- Kyverno admission policies in staging and production:
  - verify the Cosign signature and SBOM attestation against the CI identity;
  - allow images only from the Hambisa registry;
  - reject `latest` tags, privileged containers, host networking and host paths;
  - require resource limits, probes and the `restricted` security context.
- ECR enhanced scanning (Amazon Inspector) on push and continuously for deployed images; a new Critical finding with a fix opens a pull request within 24 hours.
- Private EKS API endpoint; access through IAM Identity Center with MFA and EKS access entries and just-in-time elevation for production. No shared kubeconfig files.
- Kubernetes RBAC: engineers read-only in production; write only through Argo CD; break-glass role time-boxed to 4 hours and alerting the platform lead.
- CloudTrail and EKS audit logs shipped to the log archive account and kept at least 1 year (final period set by the retention matrix, spec section 15).
- AWS WAF on the public ALBs, with rate limits on sign-in, OTP and booking endpoints (spec section 9.3).
- Egress from application namespaces limited to known provider hosts through AWS Network Firewall or an egress proxy.

**Should**

- Runtime threat detection (Amazon GuardDuty EKS Protection and Runtime Monitoring).
- SLSA build level 2 provenance attached to every image.
- Annual external penetration test before launch and after major architecture changes, covering the spec's security test list (section 12).

## 12. Database migrations and data operations

Each service migrates its own PostgreSQL schema through a Kubernetes Job that Argo CD runs before the new pods start, and every migration must work with both the old and new application version.

**Must**

- Expand-and-contract changes only: add columns and tables first, deploy code that uses them, remove old ones in a later release. No rename or drop in the same release as the code change.
- Migration Job runs in sync wave 2 with a dedicated, higher-privilege database identity; application pods use a least-privilege identity.
- A failed migration stops the sync; the previous version keeps serving.
- Migrations tested in CI against a PostgreSQL container seeded with a production-sized anonymised dataset for long-running changes.
- No service reads another service's schema (spec section 6.4); Kyverno and network policies plus database grants enforce it.
- Outbox tables cleaned on a schedule (spec section 8) by a CronJob, with metrics on backlog size.

**Should**

- Changes expected to lock large tables flagged in the pull request and scheduled in the low-traffic window.
- Point-in-time restore to a scratch server rehearsed for any migration that rewrites data.

## 13. Observability, SLOs and alerting

Every service emits OpenTelemetry traces, metrics and structured logs carrying the request correlation ID, and alerts fire on SLO burn rate rather than on raw thresholds.

| Signal | Tooling (default) | Retention |
| --- | --- | --- |
| Metrics | kube-prometheus-stack + Grafana, long-term metrics in S3 (Thanos) | 13 months, downsampled after 30 days |
| Traces | OpenTelemetry Collector → AWS X-Ray or Grafana Tempo on S3 | 14 days |
| Logs | OpenTelemetry Collector → CloudWatch Logs | 30 days hot; audit logs per retention matrix |
| Uptime | CloudWatch Synthetics canaries, plus one external check outside AWS, every minute | 13 months |
| Mobile | Crash reporting tagged with EAS build and update IDs; PostHog for funnels, retention and flag exposure. No personal data in events | 90 days |

### Initial SLOs

| Journey | SLI | Target (30 days) | Source |
| --- | --- | --- | --- |
| Booking and order APIs | Successful (non-5xx) requests | 99.9% | Spec section 10 |
| Order reads | p95 latency | under 300 ms | Spec section 10 |
| Simple writes | p95 latency | under 700 ms | Spec section 10 |
| Live tracking | Rider positions younger than the freshness threshold for active trips | \[TARGET\]% | Set after pilot (spec section 10) |
| Dispatch | Ready-to-assigned time, median and p90 | \[TARGET\] | Set after pilot |
| Payment webhooks | Verified and processed within 60 s | 99.9% | Proposed |

**Must**

- Dashboards for every alert named in spec section 14: API errors, payment failures, message backlog, assignment delays, location freshness, crash rates.
- Multi-window burn-rate alerts on each SLO; pages go to the on-call rota, warnings to the team channel.
- Every alert links to a runbook in the `hambisa` repository.
- No secrets, full card data, OTPs or delivery codes in logs or traces (spec section 10); a log-scrubbing processor in the collector as a backstop.
- Kubernetes, Argo CD, broker and database metrics collected alongside application metrics.

**Should**

- Deploy markers on dashboards from Argo CD notifications.
- Monthly SLO review with operations, feeding the error budget into release pace.

## 14. Backup, disaster recovery and resilience

The platform meets the specification's proposed RPO of 15 minutes and RTO of 4 hours for transactional data by rebuilding clusters from Git and restoring databases from managed point-in-time backups.

| What | Protection | Recovery path | Target |
| --- | --- | --- | --- |
| PostgreSQL | RDS automated backups with point-in-time restore, Multi-AZ, cross-region snapshot copies through AWS Backup | Restore to a new RDS instance, update the secret | RPO 15 min, RTO 4 h |
| Object storage (evidence, documents) | S3 versioning, Object Lock on evidence, cross-region replication | Restore versions | RPO 15 min |
| Redis | Rebuilt from source of truth; not backed up as a record (spec section 6.4) | Recreate, warm from services | Minutes |
| Broker | SQS and SNS are regional and span AZs; dead-letter queues kept | Replay from dead-letter after fix | Per runbook |
| Cluster state | Git (Terraform + `hambisa-gitops`) | Terraform apply, Argo CD resync | Under 2 h to a serving cluster |
| Persistent volumes, if any | Velero to S3, daily | Velero restore | 24 h |
| Secrets Manager | 7–30 day recovery window on deletion; replicas in the backup region for critical secrets | Restore the secret within the window | Minutes |

**Must**

- A full restore drill in staging before launch and every 6 months after: new cluster from Git, database point-in-time restore, smoke tests green (spec section 14 checklist).
- Fault-injection tests for broker outage, provider timeouts and node loss (spec section 10 resilience).
- Documented manual dispatch fallback when the platform is degraded (spec section 14).

**Should**

- A second region only if the business case in spec section 11 justifies it; until then, cross-region backup copies give a cold-standby path. The privacy owner approves the backup region under POPIA's cross-border rules.

## 15. Release-blocking platform acceptance criteria

These ten tests join the specification's section 12.1 list; production launch waits until all pass in staging.

- [ ] Given a merged change, when CI finishes, then no pipeline credential can reach any cluster API, and the change reaches dev only through a commit to `hambisa-gitops`.
- [ ] Given an unsigned image or one from another registry, when it is applied to staging, then admission rejects it.
- [ ] Given a manual `kubectl` change in production, then Argo CD reports drift and restores the Git state within one sync interval.
- [ ] Given a canary whose 5xx rate exceeds the analysis threshold, then the rollout aborts and traffic returns to the previous version without human action.
- [ ] Given a rotated payment provider key in Secrets Manager, then Payments uses the new key within 5 minutes with no redeploy.
- [ ] Given a search of all repositories, images and CI logs, then no secret is found (gitleaks and registry scan clean).
- [ ] Given a pod in `hambisa-ops`, when it calls the Payments database or service directly, then the network policy blocks it.
- [ ] Given the loss of the staging cluster, then a new cluster built from Terraform and Argo CD serves smoke tests within 2 hours.
- [ ] Given a point-in-time restore request, then the Orders database is restored to within 15 minutes of the chosen time and verified.
- [ ] Given an engineer without break-glass approval, when they try to write to production, then access is denied and the attempt is logged.

## 16. Delivery plan and decisions required

The platform is built in four steps that line up with the specification's P0 "Platform foundation" and "Release assurance" epics; each step ends with a gate before the next starts.

1. **Foundation** — AWS accounts with af-south-1 enabled, Terraform state, VPCs, ECR, Secrets Manager, GitHub OIDC, dev EKS cluster with Argo CD. *Gate:* a hello-world service deploys to dev through Git only.
2. **Delivery path** — reusable CI workflows, image signing, Kyverno, External Secrets, staging cluster, promotion bot. *Gate:* one real service promoted dev → staging with no manual step.
3. **Production readiness** — production cluster, Argo Rollouts, observability stack, SLO alerts, WAF, backup policies. *Gate:* first nine acceptance tests in section 15 pass.
4. **Launch assurance** — load tests at the agreed forecast, restore drill, penetration test, runbooks, on-call rota. *Gate:* all of section 15 and spec section 14 checklist signed by named owners.

| Decision | Why it matters | Suggested default | Owner |
| --- | --- | --- | --- |
| Cloud provider | Fixes every managed service name and cost | Decided: AWS, af-south-1 (Cape Town) | Tech lead + finance |
| Helm or Kustomize | One packaging style across all apps | Helm charts from a shared library chart | Platform lead |
| Observability: managed or self-hosted | Cost versus operating effort | Self-hosted Prometheus and Grafana; CloudWatch Logs | Platform lead |
| Ingress controller | TLS, WAF integration, cost | AWS Load Balancer Controller with AWS WAF | Platform lead |
| Production deploy window | Risk around delivery peaks | Mon–Thu 09:00–15:00 SAST | Operations + tech lead |
| Log and audit retention periods | POPIA and cost | Set in the spec's retention matrix | Legal/privacy owner |
| On-call model | Who answers pages outside office hours | Weekly rota, primary + backup | Engineering manager |
| Peak volume forecast | Sizes node pools, autoscaling and load tests | Required from the business (spec section 15) | Product owner |

Decisions raised by the mobile stack:

| Decision | Why it matters | Suggested default | Owner |
| --- | --- | --- | --- |
| Data residency for Clerk, Sanity and PostHog | All three hold or receive personal data outside South Africa | POPIA cross-border assessment and operator agreements before launch; PostHog Cloud EU region | Legal/privacy owner |
| Staff identity provider | Clerk covers customers and riders; back-office staff need MFA and role management | Clerk organisations for staff, or a separate workforce identity provider | Tech lead |
| Push notification route | Expo's push service is simplest; direct APNs and FCM gives the Notifications service full control | Expo push service for the MVP | Mobile lead |
