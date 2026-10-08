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

- `main` is an untouched mirror of upstream. Never commit to it, and never
  merge `bf-patches` into it: the whole point is that
  `git merge --ff-only upstream/main` always works.
- `bf-patches` is the integration branch. It holds one commit per fix, in a
  straight line, and every released tag is cut from it.
- Tags are named `<upstream tag>-bf.<n>`, so `v0.11.0-bf.1` is upstream
  `v0.11.0` plus the first patch set. Consumers pin to a tag, never a branch,
  and a published tag is never moved — new content means a new tag.
- GitHub Actions are disabled. The upstream workflows run end-to-end tests
  against real Azure subscriptions and have no credentials here, so they would
  fail on every push.

## Adding a patch

Each fix gets its own short-lived branch and its own pull request, so it is
reviewed on its own and carries its ticket. The branch is named for the ticket,
the commit scope repeats it:

```bash
git switch -c fix/BI-XXXX-short-description bf-patches
# ...one commit: fix(BI-XXXX): what it does
git push -u origin fix/BI-XXXX-short-description
```

Open the pull request against `bf-patches` and use **Rebase and merge**, never
"Create a merge commit". `bf-patches` has to stay linear or the rebase onto the
next upstream release stops being a rebase and becomes a merge conflict to
untangle.

Then cut a new tag and bump `?ref=` in the consuming repository.

## Removing a patch

When an upstream pull request is released, drop that single commit and re-tag:

```bash
git fetch upstream --tags
git rebase --onto <commit before it> <the commit> bf-patches
git tag v0.11.4-bf.2
```

When the last patch goes, point `source` back at the registry, restore the
`version` argument and archive this fork.

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
