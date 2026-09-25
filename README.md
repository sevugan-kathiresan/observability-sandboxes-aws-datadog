# observability-sandboxes-aws-datadog

A series of hands-on sandboxes to build practical experience in setting up **Observability
for AWS resources deployed via Terraform, using Datadog**. Datadog resources are deployed
via Terraform wherever possible, using the [Datadog Terraform Provider](https://registry.terraform.io/providers/DataDog/datadog/latest/docs).

## Project Goal

Each sandbox is a **standalone, self-contained architecture** - combining multiple AWS
services into a realistic, commonly-used pattern (three-tier web apps, serverless APIs,
event-driven systems, containerized microservices, and more), with its own compute,
its own Terraform state, and its own Datadog observability setup layered on top.

- **No sandbox depends on another sandbox's Terraform state or resources.** Each one can
  be deployed and destroyed independently.
- **Services repeat across sandboxes on purpose.** EC2, ECS Fargate, Lambda, and other
  services appear in multiple architectures - each occurrence is a fresh, isolated
  deployment.
- **Topics are ordered by increasing complexity**, so working through them in order
  builds observability concepts progressively - starting with simple managed-service
  patterns and ending with the most infrastructure-heavy architectures (Kubernetes,
  multi-AZ high availability).
- **Every architecture combines the same three observability pillars**:
  - **Infrastructure metrics** — AWS-integration metrics via `datadog_integration_aws`
  - **Logs** - Agent-based or CloudWatch-forwarded log collection, via a custom Datadog log pipeline
  - **APM** - distributed tracing, or log-based monitors where tracing doesn't apply

## Repository Structure

Each architecture lives in its own top-level folder, named `a<n>-<short-description>`,
and is a complete, independently-deployable Terraform root module:

```
observability-sandboxes-aws-datadog/
├── a1-static-site-cdn/
│   └── ...
├── a2-three-tier-web-app/        # (and so on, through a10)
```

> **For instructions on how to deploy, configure, or destroy an individual sandbox,
> refer to the README inside that architecture's own folder** (e.g.,
> `a1-static-site-cdn/README.md`). This top-level README only covers the project as a
> whole; each architecture's specific setup steps, prerequisites, and teardown
> instructions live alongside its own Terraform code.

## Architectures

### a1 - Static Site + Global CDN Delivery

**AWS Services:** S3 (static assets), CloudFront, Route53, ACM (TLS cert), WAF

A CDN-fronted static site with no VPC and no compute to manage — the simplest sandbox in
the series, and a good first topic for learning the Datadog AWS-integration and
Synthetics setup pattern without any infrastructure overhead.

**Datadog Scope:**
- `datadog_integration_aws` for CloudFront edge metrics (cache hit ratio, origin latency)
- CloudFront access logs → S3 → Datadog log pipeline (edge location, status code, cache status)
- `datadog_synthetics_test` (Browser + API) for availability checks
- WAF rule-block metrics/monitors
- Dashboard: cache performance + edge latency + synthetic uptime

See `a1-static-site-cdn/README.md` for setup instructions specific to this sandbox.

## Global Conventions

- **One root module, one state, per architecture** — fully deployable and destroyable on
  its own.
- **Remote state backend:** this project uses [HCP Terraform](https://developer.hashicorp.com/terraform/cloud-docs)
  (Terraform Cloud), with one workspace per architecture (`ddaws-sandbox-<topicID>`).
  This is my choice for convenience (free-tier remote state, locking, run history) -
  **it isn't required to use this project.** Each topic's `backend.tf` can be pointed at
  whichever remote backend you prefer instead (S3 + DynamoDB, Azure Storage, GCS,
  Terraform Enterprise, or local state) with no other changes needed.
- **Unified Service Tagging** (`env`, `service`, `version`) applied consistently across
  AWS and Datadog resources.
- **Internal tags (optional):** if you want to apply your own extra tracking tags (e.g.
  owner, team, cost center) without hardcoding tag names into the Terraform code, each
  topic exposes a generic, optional map variable:
  ```hcl
  variable "internal_tags" {
    type    = map(string)
    default = {}
  }
  ```
  It's merged into the common tag set alongside Unified Service Tagging:
  ```hcl
  locals {
    common_tags = merge(
      { env = var.env, service = var.service_name, version = var.service_version },
      var.internal_tags
    )
  }
  ```
  To use it, create a `<preferred_file_name>.auto.tfvars` file in that topic's folder (Terraform loads it
  automatically - no flag needed) with whatever keys/values you want:
  ```hcl
  # tags.auto.tfvars
  internal_tags = {
    owner = "your-name"
    team  = "your-team"
  }
  ```
  This file is gitignored, so your tag names and values never get committed. Leaving it
  out entirely causes no errors — `internal_tags` defaults to an empty map, and
  Unified Service Tagging still applies as normal.
- **Naming convention:** all resources prefixed with `ddaws-sandbox-<topicID>-*` for easy
  identification in both the AWS and Datadog UIs.
