# Local setup — deskaway-infra

**There is nothing to apply yet.** No Terraform is written, and `global/backend/`
— the remote state bucket and lock table — does not exist, so `terraform init`
has nothing to point at. This page describes the setup as it is intended to work.
Correct it in the same pull request that makes it true.

## What you need

- Terraform, at the version CI uses. Keep the two in step.
- AWS credentials with **read** access. You do not need write access to work on
  this repo, and you should not use write credentials for local work.
- `checkov`, if you want to run policy scans before pushing.

## Steps

The only local command anyone should normally need is `plan`, which is read-only:

```sh
git clone https://github.com/away-desk/deskaway-infra.git
cd deskaway-infra/envs/dev

terraform init            # remote state, from global/backend
terraform validate
terraform plan            # read-only; safe
```

Under ten minutes on a cold clone once the backend exists, most of it provider
download.

To validate without any credentials at all — useful for a syntax check on a
module you are writing:

```sh
terraform -chdir=envs/dev init -backend=false
terraform -chdir=envs/dev validate
```

That is what CI does, and it needs neither state nor AWS access.

## Formatting and policy checks

```sh
terraform fmt -recursive            # apply
terraform fmt -check -recursive     # what CI checks
checkov -d .                        # config in policies/checkov
```

## Applying

**Do not apply from your machine.** `plan` runs on every pull request and `apply`
runs on merge, and that is the only path to an environment — so that state, the
audit trail and the order of changes all live in one place. A local apply with a
different Terraform version or a stale state lock is how two people's changes get
lost.

See `runbooks/deploy.md`, and `runbooks/rollback.md` for when it goes wrong.

## Adding a module

- Modules under `modules/` are environment-agnostic: no hardcoded account ID,
  region, or environment name. Everything varies by input variable.
- `envs/` is composition only — module calls and values, no resource blocks.
- Changing a shared module means planning **every** environment that consumes it,
  not just the one you care about. `ecs-service` is used by both the relay and the
  agent in all three environments.
- A new alarm needs a runbook entry saying what to do when it fires. An alarm
  nobody knows how to respond to gets muted.

## Secrets

This repo declares that a secret exists and who can read it. The value is set out
of band and must never appear in a variable, a tfvars file, or the repo in any
form. `.gitignore` blocks `*.tfvars` and `*.tfstate*`; keep it that way.

**A plan output can contain secret values in plaintext.** Never paste a raw plan
into an issue, a pull request comment, or a chat. That is the most likely way a
credential leaks out of this repo.

## When it will not work

- **`init` cannot find the backend** — expected today; `global/backend/` has not
  been bootstrapped. It is the one thing created by hand, because state has to
  live somewhere before Terraform can manage it.
- **State lock errors** — someone else is applying, or a previous CI run died
  holding the lock. Check the runs before forcing anything; force-unlocking a live
  apply corrupts state.
- **`plan` wants to destroy something you did not touch** — stop. That is usually
  a module input that changed meaning, or drift from a hand-made change. Work out
  which before proceeding.
