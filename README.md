# deskaway-infra

The Terraform for DeskAway's cloud side: network, load balancer, ECS
services, Postgres, Redis, the recordings bucket, secrets, WAF and DNS.

It deploys `deskaway-relay` and `deskaway-agent`. The desktop and phone
clients are distributed, not deployed, so nothing here touches them.

## Status

**Early development, nothing works yet.**

The module and environment layout is settled, and the runbook filenames name
the operations we expect to need. No Terraform is written — every directory
under `envs/`, `global/`, `modules/` and `policies/` is an empty placeholder,
and the runbooks and docs are empty files. Nothing has ever been applied;
there is no state and no AWS footprint.

## Running locally

There is nothing to apply. Until `global/backend/` is written the remote
backend does not exist, so even `terraform init` has nothing to point at.

You will need Terraform and AWS credentials with read access. The intended
path once the root modules exist, with `dev` as the example:

```sh
git clone https://github.com/away-desk/deskaway-infra.git
cd deskaway-infra/envs/dev

terraform init          # remote state from global/backend
terraform validate
terraform plan          # read-only; safe to run
```

`plan` is the only local command anyone should need, and it is under ten
minutes on a cold clone once the backend exists. **Applies happen in CI
only** — see `runbooks/deploy.md`. `docs/environments.md` will explain what
differs between dev, staging and prod; both files are currently empty.

## The rest of DeskAway

Cross-repo docs and architecture decisions live in
**[deskaway-docs](https://github.com/away-desk/deskaway-docs)**. All
components are under the **[away-desk](https://github.com/away-desk)** org.

Contributor guidance for this repo, including the rule for maintaining this
README, is in [AGENT.md](./AGENT.md).
