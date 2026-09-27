# AGENT.md — deskaway-infra

The Terraform that runs DeskAway's cloud side: network, load balancer, the
ECS services for `deskaway-relay` and `deskaway-agent`, Postgres, Redis, the
recordings bucket, secrets, WAF and DNS. Plus the runbooks for when it
misbehaves.

Nothing in the product is deployed by hand. If a resource exists in AWS and
is not described here, that is a defect.

## Folder structure

```
.github/workflows/
  plan.yml                # plan on every PR, comment the diff
  apply.yml               # apply on merge, per environment
envs/
  dev/  staging/  prod/   # one root module each; composes modules/
global/
  backend/                # state bucket and lock table, bootstrapped once
  ecr/                    # image registries
  iam/                    # org-wide roles, CI deploy identity
modules/
  network/                # vpc, subnets, routing
  alb/                    # public entry, listeners, target groups
  ecs-service/            # reusable service: task def, autoscaling, logs
  rds-postgres/           # relay's database
  elasticache-redis/      # pub/sub and ephemeral state
  s3-recordings/          # run transcripts, lifecycle and encryption
  secrets/                # secret definitions, never values
  waf/                    # rate limiting, managed rules
  dns-certs/              # zones, records, ACM
  observability/          # dashboards, alarms, log retention
policies/
  checkov/                # IaC scanning config and suppressions
  iam/                    # policy documents and boundaries
runbooks/
  deploy.md
  rollback.md
  incident-relay-down.md
  restore-from-backup.md
  rotate-model-key.md
docs/
  environments.md         # what differs between dev, staging and prod
  cost-notes.md           # what actually costs money here
```

Every directory under `envs/`, `global/`, `modules/` and `policies/` holds
only a `.gitkeep` today — the layout is agreed, the Terraform is not written.

## Conventions

- `modules/` are reusable and environment-agnostic: no hardcoded account id,
  region, or environment name. Everything varies by input variable.
- `envs/` are composition only — module calls and values, no inline
  `resource` blocks.
- Never apply from a laptop. `plan.yml` and `apply.yml` are the only paths to
  an environment, so that state and audit trail stay in one place.
- State lives in the remote backend from `global/backend/`, which is the one
  thing bootstrapped by hand. Commit `.terraform.lock.hcl`.
- Secrets: this repo declares that a secret exists and who can read it. The
  value is set out of band and never appears in a variable, a tfvars file, or
  a plan. `.gitignore` blocks `*.tfvars` and `*.tfstate*` — keep it that way.
- A plan output can contain secrets in plaintext. Never paste a raw plan into
  an issue or a PR comment.
- Changing a module means planning every environment that consumes it, not
  just the one you care about.
- A new alarm needs a runbook entry saying what to do when it fires.

## Rule: keep README.md current

The README is the one file a newcomer is guaranteed to read. Revisit it
whenever this repo's answer to any of the four questions below changes — not
on a schedule.

Every DeskAway README answers four things, in this order:

1. **What this one repo is**, in two lines, and where it sits in the whole
   system.
2. **Its current status**, stated honestly. Right now that is *early
   development, nothing works yet.*
3. **How to run it locally**, aiming for under ten minutes.
4. **A link back** to the org or to `deskaway-docs`, so someone landing here
   can find the rest.

How to apply it:

- Keep those four as the first four sections, in that order. Anything else
  goes after them.
- Status rots fastest. The moment the first thing in this repo actually
  runs, that line changes in the same PR. "Nothing works yet" is honest
  only until it isn't.
- If a setup step breaks, or creeps past ten minutes, fix the README in the
  PR that caused it. A stale run section is worse than no run section.
- Never write intent as if it were fact. Anything not yet true is either
  labelled as planned or left out entirely.
- Two lines means two lines. If section 1 needs a third paragraph, that
  content belongs in `deskaway-docs`.
