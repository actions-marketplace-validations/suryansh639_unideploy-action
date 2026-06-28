# UniDeploy AI DevOps Agent — GitHub Action

Run [UniDeploy](https://unideploy.com) — an AI-powered DevOps agent — directly in
your CI/CD pipeline. Auto-diagnose failed builds, audit your AWS / Azure / GCP
infrastructure, and post a root-cause analysis or fix as a pull-request comment.

> **Bring Your Own Model (BYOM)** — UniDeploy uses *your* LLM (AWS Bedrock,
> Anthropic, OpenAI, Gemini). It never charges for inference; token usage is
> billed to your provider account.

## Quick start

```yaml
- uses: suryansh639/unideploy-action@v1
  with:
    api-key: ${{ secrets.UNIDEPLOY_API_KEY }}
    provider: amazon-bedrock
    region: us-east-1
    prompt: "List my S3 buckets and flag any with public access."
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

Get a free API key at **https://unideploy.com** (10 free requests, then $15/month).

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api-key` | ✅ | — | UniDeploy API key (store as a GitHub secret) |
| `prompt` | * | — | Natural-language task. Required unless `playbook` is set. |
| `playbook` | * | — | Name of a saved playbook to run (e.g. `security-audit`). |
| `playbook-vars` | | `""` | Space-separated `key=value` overrides for the playbook. |
| `provider` | | `amazon-bedrock` | Model provider: `amazon-bedrock`, `anthropic`, `openai`, `gemini`. |
| `region` | | `us-east-1` | AWS region for Bedrock / AWS operations. |
| `model` | | `""` | Optional model override. |
| `max-steps` | | `25` | Max agent steps before stopping. |
| `comment-on-pr` | | `false` | If `true` on a `pull_request`, post output as a PR comment. |
| `github-token` | | `${{ github.token }}` | Token used to post PR comments. |

\* Provide either `prompt` or `playbook`.

## Outputs

| Output | Description |
|--------|-------------|
| `result` | The agent's full output text (use in later steps). |

## Built-in playbooks

UniDeploy ships with ready-to-run playbooks you can call via `playbook:`:

| Playbook | What it does |
|----------|--------------|
| `security-audit` | Public S3, open security groups, IAM without MFA, old keys |
| `cost-report` | Unused Elastic IPs, unattached EBS, stopped instances, old snapshots |
| `lambda-health` | Lambda functions with recent errors/throttles |
| `ssl-expiry` | ACM certificates nearing expiry |
| `untagged-resources` | Resources missing required cost/ownership tags |

## Example 1 — Auto-diagnose a failed CI run

```yaml
name: UniDeploy CI Auto-Fix
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]

jobs:
  autofix:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: suryansh639/unideploy-action@v1
        with:
          api-key: ${{ secrets.UNIDEPLOY_API_KEY }}
          comment-on-pr: "true"
          prompt: >
            The CI workflow just failed. Look at the most recent run logs,
            find the root cause, and explain the fix in clear steps.
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Example 2 — Nightly security + cost audit

```yaml
name: UniDeploy Nightly Audit
on:
  schedule:
    - cron: "0 2 * * *"
  workflow_dispatch: {}

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: suryansh639/unideploy-action@v1
        with:
          api-key: ${{ secrets.UNIDEPLOY_API_KEY }}
          playbook: security-audit
          playbook-vars: "region=us-east-1"
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Provider credentials

This action does not ship any model credentials (BYOM). Pass yours via `env:`:

- **AWS Bedrock / AWS ops**: `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`, or
  (recommended) use [`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials)
  with GitHub OIDC for short-lived credentials.
- **Anthropic / OpenAI / Gemini**: set the matching provider API-key env var.

## Permissions

To post PR comments the job needs:

```yaml
permissions:
  contents: read
  pull-requests: write
```

## How it works

The action is a thin wrapper: it installs the UniDeploy CLI from
`https://dl2.unideploy.com/install.sh`, configures your key + provider, and runs
the agent. All execution happens on the GitHub runner using your own model.

## Links

- Website: https://unideploy.com
- Docs: https://unideploy.com/api/docs

## License

MIT — for this action wrapper. The UniDeploy CLI is distributed under its own terms.
