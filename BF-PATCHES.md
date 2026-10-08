# BF patches

This is a fork of [Azure/terraform-azurerm-avm-ptn-aiml-ai-foundry][upstream],
carrying fixes that Business Finland needs before upstream ships them. It is a
stopgap, not a product: every patch here is also an upstream pull request, and
each one is removed as soon as that pull request is released.

If a change cannot go upstream, it does not belong in this fork. The moment
that rule is broken this stops being a fork and becomes a copy nobody can
merge back.

[upstream]: https://github.com/Azure/terraform-azurerm-avm-ptn-aiml-ai-foundry

## How this fork is laid out

- `main` is an untouched mirror of upstream. Never commit to it.
- `fix/BI-2499-avm-capability-host-output` holds the patches, rebased onto an
  upstream tag. If a second patch is ever added, rename this to something
  role-based: consumers pin to tags, never to a branch, so the name is free to
  change.
- Tags are named `<upstream tag>-bf.<n>`, so `v0.11.0-bf.1` is upstream
  `v0.11.0` plus the first patch set. Consumers pin to a tag, never a branch.
- GitHub Actions are disabled. The upstream workflows run end-to-end tests
  against real Azure subscriptions and have no credentials here, so they would
  fail on every push.

## Current patches

### 1. BI-2499 — output condition must match the capability host count expression

`modules/ai-foundry-project/outputs.tf`

The output guards on `create_ai_agent_service` alone while the resource counts
on `create_ai_agent_service && create_project_connections`. Setting
`create_project_connections = false` for one project in an agent-enabled
account drives the count to 0, but the output still indexes `[0]`, and the plan
fails.

The practical consequence is that `create_project_connections` cannot be varied
per project at all: every project in an agent-enabled account must carry an
agent capability host whether or not it ever runs an agent. Each of those hosts
provisions Cosmos containers with an autoscale floor billed around the clock.
Measured on the BF platform: about 16 EUR per unused project per environment,
roughly 130 EUR a month across dev, test and prod.

- Upstream: [#128](https://github.com/Azure/terraform-azurerm-avm-ptn-aiml-ai-foundry/pull/128)
  (closed), continued as [#167](https://github.com/Azure/terraform-azurerm-avm-ptn-aiml-ai-foundry/pull/167)
- **Remove when** #167 is merged and released, then point `source` back at the
  registry and drop this patch.

## Updating to a new upstream release

```bash
git fetch upstream --tags
git rebase v0.11.4 bf-patches      # whatever the new tag is
git tag v0.11.4-bf.1
git push --force-with-lease origin bf-patches
git push origin --tags
```

Then bump `?ref=` in the consuming repository and read the plan before applying.
A module version change is its own change: do not combine it with anything else.
