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

## Rule: keep CHANGELOG.md current

Add a line the moment you do something notable — not at release time. The
changelog is cheap to maintain one entry at a time and miserable to
reconstruct from five months of git log.

- Format is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
  versioning is [SemVer](https://semver.org/spec/v2.0.0.html).
- Everything lands under `## [Unreleased]`, grouped by `### Added`,
  `Changed`, `Deprecated`, `Removed`, `Fixed`, or `Security`. Create a group
  when you first need it.
- "Notable" means a reader of this repo would want to know: a new capability,
  a behaviour change, a dependency that changes how you run it, a security
  fix. Not: formatting, a typo, an internal rename nobody outside the file
  can see.
- Write for someone who has not read the diff. "Added pairing code
  expiry" beats "updated code-generator.ts".
- On a release, rename `[Unreleased]` to the version with the date, and open
  a fresh empty `[Unreleased]` above it. Never delete history.
- Entries are past tense and one line. If yours needs a paragraph, it is
  probably two entries.

## Rule: keep the pull request template useful

`.github/pull_request_template.md` pre-fills every PR description. While this
project is one person reviewing their own work, it is the self-check that
catches what you were about to skip — so fill it in honestly rather than
deleting the prompts.

- Answer all four. "N/A" is a fine answer; a blank section is not.
- **Which unit of the plan this belongs to** is the one that pays off later.
  In five months this is how you find which PR did what, so name the unit,
  not the file you touched.
- **Anything deliberately left incomplete** is not an admission. An
  acknowledged gap is a decision; an unmentioned one is a bug you will
  rediscover.
- Change the template when a prompt stops earning its place, and keep it at
  four or five. A template long enough to skim past is worse than none.

## Rule: keep CONTRIBUTING.md short

It is currently five lines because there are no outside contributors. Resist
growing it for people who do not exist yet.

- Update it when the real answer changes: the formatter command, the branch
  rule, or the day the project starts accepting outside contributions.
- Anything longer than a few lines is either repo guidance — which belongs in
  this file — or cross-repo process, which belongs in `deskaway-docs`.

## Rule: keep SECURITY.md honest

One line in it will become false, and it is the important one.

- The supported-versions table says *nothing is supported, do not run this*.
  The day a version is tagged, that table changes in the same PR — an
  unsupported-looking project that is actually shipping teaches people to
  ignore the file.
- The reporting route assumes GitHub private vulnerability reporting is
  enabled on the repo. If that is ever turned off, this file needs a real
  contact route the same day, or reports arrive as public issues.
- Do not soften the warning about executing model-authored shell commands
  while the scope, approval and timeout controls are still unwritten. It is
  the most accurate sentence in the repo.

## Rule: ask before touching GitHub, and never push to main

This overrides anything else in this file. Abhay approves every action that
leaves this machine.

### Ask first

Stop and ask before running any of these, showing the exact command:

- `git push` — any branch, any remote, including the first push of a new
  branch
- opening, editing, merging or closing a pull request
- creating or deleting a remote branch, tag or release
- any `gh` command that writes: issues, comments, labels, reviews, workflow
  runs, repo or org settings
- anything at all involving `--force`, `--force-with-lease`, or a remote
  delete

Do not batch these up and ask once at the end. Ask at the point of doing it,
and wait for a clear yes.

### No approval needed

Local work is yours to get on with: `status`, `diff`, `log`, `show`, `add`,
`commit`, creating and switching local branches, `stash`, reading anything.
Commit freely — a local commit is not a GitHub action.

### Every feature goes through a branch and a PR

`main` is never pushed to directly. The loop for any change:

1. Branch off current `main`: `feat/`, `fix/`, `docs/`, `chore/` or
   `refactor/` plus a short kebab-case description — `feat/pairing-code-expiry`.
2. Commit locally, as many commits as the work needs.
3. **Ask**, then push the branch.
4. **Ask**, then open the PR against `main`, filling in every prompt of the
   pull request template.
5. Report back: the branch, the PR link, what is in it, and anything left
   incomplete.
6. **Stop there.** Do not merge, do not squash, do not delete the branch, do
   not mark anything ready or draft. Abhay says what happens next.

A rejected or unanswered request is a stop, not a prompt to find another
route to the same result.
