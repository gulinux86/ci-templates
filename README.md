# ci-templates

Shared GitHub Actions building blocks for Terraform on AWS.

## `aws-terraform-setup`

A composite action for the three steps every AWS Terraform job starts with: select the
environment's role, authenticate through OIDC, install Terraform.

```yaml
- uses: gulinux86/ci-templates/.github/actions/aws-terraform-setup@master
  with:
    environment: hml
    role: deploy                    # or plan
    deploy-role-arn: ${{ secrets.HML_DEPLOY_ROLE_ARN }}
```

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

Pinned to `@master` while the first consumer validates it. Tag `v1` once a full
deploy/destroy cycle has passed through it, and move consumers to the tag — `@master`
propagates a mistake to every consumer at once with no way to pin out.
