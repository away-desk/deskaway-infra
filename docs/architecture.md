# Architecture — deskaway-infra

## Responsibility

Describes DeskAway's cloud side as code: the network it runs in, the services that
run there, the data stores they depend on, and the alarms that say when something is
wrong. It deploys `deskaway-relay` and `deskaway-agent`.

Nothing in the cloud is created by hand. A resource existing in AWS but not
described here is a defect, not a shortcut.

## What it deliberately does not do

- **It holds no secret values.** This repo declares that a secret exists and who may
  read it. Values are set out of band and must never appear in a variable, a tfvars
  file, a plan output, or a chat message. **A plan can print secrets in plaintext**,
  which is the most likely way a credential escapes this repo.
- **It is never applied from a laptop.** `plan` on pull request, `apply` on merge,
  and that is the only path to an environment — so state, audit trail and ordering
  live in one place. A local apply with a different Terraform version or a stale
  lock is how two people's changes get lost.
- **It does not deploy the clients.** `deskaway-desktop` and `deskaway-android` are
  distributed to users, not deployed to servers.
- **It does not grant the relay a model provider key.** Only the agent's task role
  can read that secret. The separation exists so that compromising the relay does
  not yield a model credential.
- **Modules do not know where they are.** No hardcoded account ID, region, or
  environment name. A module that only works in prod is not a module.
- **`envs/` contains no resources of its own.** If something is worth creating, it is
  worth being a module.
- **It does not adopt hand-made resources.** Importing an ad-hoc resource
  legitimises the shortcut; delete it and describe it instead.

## Internal pieces, and how a change flows

**`envs/dev`, `envs/staging`, `envs/prod`** — one root module each, composing shared
modules with environment-specific inputs.

**`global/`** — what exists once rather than per environment. `backend` is the state
bucket and lock table, and is the one thing bootstrapped by hand, because state must
live somewhere before Terraform can manage it. `ecr` holds image registries; `iam`
the org-wide roles including the CI deploy identity.

**`modules/`** — `network` (VPC, subnets, routing), `alb` (public entry, listeners,
target groups), `ecs-service` (the one reusable service definition both the relay
and the agent deploy through), `rds-postgres`, `elasticache-redis`,
`s3-recordings` (lifecycle and encryption), `secrets` (declarations only), `waf`,
`dns-certs`, `observability` (dashboards, alarms, retention).

**`policies/`** — `checkov` config for scanning this Terraform, `iam` policy
documents and permission boundaries. **`runbooks/`** — deploy, rollback,
incident-relay-down, restore-from-backup, rotate-model-key.

### Flow of one infrastructure change

1. A branch edits a module or an environment's inputs.
2. The pull request triggers `plan.yml`, which runs `terraform plan` per affected
   environment and posts the diff. **Changing a shared module means planning every
   environment that consumes it** — `ecs-service` is used by both services in all
   three.
3. `checkov` scans the Terraform itself; a policy failure blocks the merge.
4. A human reads the plan. This is the actual review step: the diff is the change,
   not the code.
5. On merge, `apply.yml` applies per environment, taking the state lock from
   `global/backend`.
6. `observability` alarms are how you learn whether it worked after CI goes green.

### Flow of a service deployment

Code merging in `deskaway-relay` or `deskaway-agent` builds an image, pushes it to
the registry from `global/ecr`, and updates the `ecs-service` task definition. The
service rolls with health checks in front of the `alb` target group. This repo owns
the shape of that; the service repos own what is inside the image.

## Layering rules

**`modules/` are environment-agnostic; `envs/` are composition only.** Modules take
inputs and know nothing about where they run. Environments call modules and contain
no `resource` blocks.

The reason: dev, staging and prod must be able to differ in size and cost without
differing in *shape*. The moment an environment declares its own resource, prod has
something staging cannot test, and "it worked in staging" stops meaning anything —
which is the entire reason for having a staging environment. Keeping the divergence
to input values means the only difference between environments is a number you can
read in one file.

Two supporting rules:

- **`global/` is bootstrapped once and depended on by everything.** Nothing in
  `envs/` or `modules/` may create state backends, registries, or org-level roles.
- **A new alarm needs a runbook entry.** An alarm nobody knows how to respond to
  gets muted, and a muted alarm is worse than none.

## What it talks to, and in which direction

| Direction | Peer | How |
| --- | --- | --- |
| **outbound** | AWS | Terraform provider, from CI only |
| **inbound** | `deskaway-relay` CI | pushes an image, triggers a service update |
| **inbound** | `deskaway-agent` CI | pushes an image, triggers a service update |
| **manages** | Postgres, Redis, S3, ALB, WAF, DNS | as declared in `modules/` |
| — | `deskaway-desktop`, `deskaway-android` | nothing, in either direction |

The clients are absent from this table on purpose. They are distributed to end users
and never touch this repo, which is why an infrastructure incident cannot break an
already-running desktop mid-task — only its ability to reach the relay.
