# ci-templates

Shared GitHub Actions building blocks: Terraform on AWS, and releasing a service to
the EKS platform.

| | Kind | For |
|---|---|---|
| [`aws-terraform-setup`](#aws-terraform-setup) | composite action | Terraform jobs: OIDC role + Terraform |
| [`service-release`](#service-release) | reusable workflow | application repositories: image to ECR, digest to app-gitops |

## `aws-terraform-setup`

A composite action for the three steps every AWS Terraform job starts with: select the
environment's role, authenticate through OIDC, install Terraform.

```yaml
- uses: gulinux86/ci-templates/.github/actions/aws-terraform-setup@<sha>  # pre-v1
  with:
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

## `service-release`

Takes a service from a commit on `main` to a pull request in
[`app-gitops`](https://github.com/gulinux86/app-gitops) that deploys it:

```
pull request   build → scan (Trivy)                                   nothing pushed
push to main   build → scan → push to ECR apps/<app>[/<service>]
               → PR in app-gitops: image: <registry>/apps/<app>:<sha>@sha256:… in
                 apps/<app>[/<service>]/values.yaml
               → app-gitops CI validates it → merge (or auto-merge) → Argo CD deploys
```

In the application's repository, `.github/workflows/release.yml`:

```yaml
name: release
on:
  push:
    branches: [main]
  pull_request:
permissions:
  contents: read
jobs:
  release:
    uses: gulinux86/ci-templates/.github/workflows/service-release.yml@<sha>  # pre-v1
    permissions:
      contents: read
      id-token: write
    with:
      app: orders            # apps/orders/ in app-gitops, ECR apps/orders
      # service: api         # apps/orders/api/ for a multi-service application
      # auto-merge: true     # merge the app-gitops PR when its checks pass
    secrets:
      ecr-push-role-arn: ${{ secrets.ECR_PUSH_ROLE_ARN }}
      gitops-app-client-id: ${{ secrets.GITOPS_APP_CLIENT_ID }}
      gitops-app-private-key: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}
```

The application must already be onboarded — `apps/<app>/values.yaml` in app-gitops and
its tenant in platform-gitops. The workflow refuses to invent a values file: a first
push before onboarding builds and pushes the image, then fails at the pull request with
"onboard the application there first". Onboard with that image's digest, and every
push after opens the release pull request.

**What it decides, and why:**

- **The tag is the commit** (12 characters), and ECR tags are immutable, so a tag names
  one build forever. The pull request pins the digest as well; the tag is for humans.
- **The ECR repository is created on the first push**, with immutable tags, scan on
  push and a lifecycle policy keeping the newest 200 images. Not by Terraform: images
  must outlive the cluster, and admitting an application should not need an apply.
- **Scan before push.** A HIGH or CRITICAL vulnerability with a fix available fails the
  build before the image exists anywhere. Unfixed ones do not: they would block every
  release with nothing the team can do. Trivy is installed from its release asset with
  a pinned checksum, not through `setup-trivy`, whose tag lookup has failed for hours
  at a time.
- **Two jobs, two credentials.** `build` holds the ECR push role and no token for
  app-gitops; `promote` holds that token and no AWS credentials. A pull request runs
  neither the push nor the promote.
- **A GitHub App, not `GITHUB_TOKEN` or a personal token.** The workflow's own token
  cannot write to another repository, and a pull request it opened would not start
  app-gitops' CI. An App token is scoped to app-gitops alone and expires in an hour.
- **Releases are serialized per service, and supersede each other.** A newer release
  closes the older open release PRs of the same service; merging one of those after
  would roll the service back.

### One-time setup

1. **AWS roles** — `terraform -chdir=bootstrap apply` in
   [`eks-terraform`](https://github.com/gulinux86/eks-terraform) creates
   `github-actions-ecr-push` (assumable from the `main` branch of your repositories) and
   `github-actions-ecr-read` (for app-gitops' CI). Outputs `ecr_push_role_arn` and
   `ecr_read_role_arn`. Their trust accepts both OIDC `sub` formats GitHub issues —
   newer repositories get the immutable `repo:<owner>@<id>/<repo>@<id>` one, and a
   trust without it fails with `Not authorized to perform
   sts:AssumeRoleWithWebIdentity`.
2. **GitHub App** — Settings → Developer settings → GitHub Apps → New:
   - no webhook; repository permissions **Contents: Read and write** and **Pull
     requests: Read and write**, nothing else;
   - install it on **app-gitops only**;
   - note its **Client ID** and generate a **private key**.
3. **Secrets**
   - each application repository: `ECR_PUSH_ROLE_ARN`, `GITOPS_APP_CLIENT_ID`,
     `GITOPS_APP_PRIVATE_KEY`;
   - app-gitops: `ECR_READ_ROLE_ARN`.
4. **For `auto-merge: true`** — in app-gitops, enable *Allow auto-merge* and protect
   `main` with its CI jobs as required checks. Without required checks, auto-merge
   would merge before CI has said anything.

## What is deliberately not here

Terraform pipelines as reusable workflows. They were tried here and moved back to the
consuming repository, because the ones worth sharing turned out to carry
project-specific decisions — layer ordering, a cross-layer output contract, a teardown
shaped by nine corrections particular to one platform. A workflow that needs six inputs
to describe someone else's architecture is not shared logic.

`service-release` is the opposite case: every application repository runs the same
pipeline, and what differs between them is one name.

## Consumers

- [`eks-terraform`](https://github.com/gulinux86/eks-terraform) — plan, deploy, destroy
  (`aws-terraform-setup`)
- application repositories deployed through app-gitops (`service-release`) —
  [`hello-api`](https://github.com/gulinux86/hello-api) is the reference: its
  `.github/workflows/release.yml` is the caller above, nothing more

### Deliberately out of scope

`aks-terraform` and `gcp-serverless-platform-terraform` are Azure and GCP. Different
auth entirely, so this action is useless to them — sharing CI across clouds would mean
an action whose only shared step is installing Terraform.

Two sibling AWS projects (`aws-plataform-terraform`, `aws-platform-apps`) could adopt
this. Neither has been read, so neither is claimed as a consumer: a template justified
by projects nobody checked is shared infrastructure for an assumption.

## What stayed with the consumer, and why

Terraform reusable workflows were tried here and moved back. The ones worth sharing
turned out to carry decisions specific to one platform — layer ordering, a cross-layer output
contract, a teardown shaped by nine corrections — and a workflow needing six inputs to
describe someone else's architecture is not shared logic.

What is left of the Terraform side is small enough to stay generic: assume a role,
install a binary.

Consumers pin the current commit while this is validated. Tag `v1` once a full
deploy/destroy cycle has passed through it — the tag is for humans reading the
comment; the SHA is what is resolved.
