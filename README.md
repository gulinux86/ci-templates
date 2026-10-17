# ci-templates

Shared GitHub Actions building blocks for Terraform on AWS.

## `aws-terraform-setup`

A composite action for the three steps every AWS Terraform job starts with: select the
environment's role, authenticate through OIDC, install Terraform.

```yaml
- uses: gulinux86/ci-templates/.github/actions/aws-terraform-setup@<sha>  # pre-v1
  with:
    environment: hml
    role: deploy                    # or plan
    deploy-role-arn: ${{ secrets.HML_DEPLOY_ROLE_ARN }}
```

**Pin by commit SHA, not by tag or branch.** A tag is mutable — whoever controls this
repository can repoint `v1` at different code, and a consumer would run it with no
diff anywhere. That is not hypothetical: `tj-actions/changed-files` was compromised
in 2025 by exactly that, with tags repointed at code that leaked secrets. A SHA
cannot move.

The version goes in a trailing comment so a human can read what the SHA is, which is
the convention the consuming workflows already follow for `actions/checkout` and the
rest.

**Why a composite action and not a reusable workflow.** It is steps, not a job: it
runs on the caller's runner, in the caller's workspace, and the credentials it exports
are there for the steps that follow. A reusable workflow would get its own runner and
the `terraform` command after it would find nothing.

It also cannot declare `secrets:` — composite actions have no such block — so the role
ARNs arrive as inputs and the caller resolves `secrets.X` before passing them.

**Why `role` is an input rather than one ARN.** The read/write split is the point: a
pull-request job runs the workflow file from its own branch, so the job that only
needs to read must never hold a credential that can write. Passing `role: plan`
against a read-only ARN keeps that explicit at every call site, and an empty ARN fails
loudly rather than authenticating as something unintended.

## What is deliberately not here

Reusable workflows. They were tried here and moved back to the consuming repository,
because the ones worth sharing turned out to carry project-specific decisions — layer
ordering, a cross-layer output contract, a teardown shaped by nine corrections
particular to one platform. A workflow that needs six inputs to describe someone
else's architecture is not shared logic.

This action is generic because its job is small: assume a role, install a binary.

## Consumers

- [`eks-terraform`](https://github.com/gulinux86/eks-terraform) — plan, deploy, destroy

### Deliberately out of scope

`aks-terraform` and `gcp-serverless-platform-terraform` are Azure and GCP. Different
auth entirely, so this action is useless to them — sharing CI across clouds would mean
an action whose only shared step is installing Terraform.

Two sibling AWS projects (`aws-plataform-terraform`, `aws-platform-apps`) could adopt
this. Neither has been read, so neither is claimed as a consumer: a template justified
by projects nobody checked is shared infrastructure for an assumption.

## What stayed with the consumer, and why

Reusable workflows were tried here and moved back. The ones worth sharing turned out
to carry decisions specific to one platform — layer ordering, a cross-layer output
contract, a teardown shaped by nine corrections — and a workflow needing six inputs to
describe someone else's architecture is not shared logic.

What is left is small enough to stay generic: assume a role, install a binary.

Consumers pin the current commit while this is validated. Tag `v1` once a full
deploy/destroy cycle has passed through it — the tag is for humans reading the
comment; the SHA is what is resolved.
