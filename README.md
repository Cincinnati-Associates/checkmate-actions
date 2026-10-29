# checkmate-actions

Reusable GitHub Actions workflows for Checkmate:

- `.github/workflows/review.yml`: the Lindamar code review of a pull request, and the required check `review / Lindamar review`.
- `.github/workflows/agent-fix.yml`: an agent fix the Checkmate console dispatches. It calls `agent-fix-round.yml` once per pass.

These files are public so that repositories in any organisation can call them. The reviewer's skills and scripts are not here. They stay in a private repository, and the workflows fetch them at runtime. Each job that needs them gets a short-lived token from your Checkmate tenant through GitHub Actions OIDC. Your repository holds no GitHub App key.

## Caller

```yaml
name: Lindamar review

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  workflow_dispatch:
    inputs:
      pr:
        description: Pull request number to review
        required: false
        type: string
      agent_fix:
        description: Set by the Checkmate console only, the suggestion an agent fixes as JSON
        required: false
        type: string

permissions: {}

concurrency:
  group: lindamar-review-caller-${{ github.event.pull_request.number || github.event.inputs.pr || github.run_id }}-${{ github.event_name }}
  cancel-in-progress: true

jobs:
  review:
    if: ${{ github.event_name == 'workflow_dispatch' && github.event.inputs.pr != '' || github.event_name == 'pull_request' && !github.event.pull_request.draft }}
    permissions:
      id-token: write      # required: the token exchange with Checkmate
      contents: read
      pull-requests: write
      issues: write        # the "could not run" notice
      statuses: write      # hand-started runs only
      checks: write        # hand-started runs only
    uses: Cincinnati-Associates/checkmate-actions/.github/workflows/review.yml@v1
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      LINDAMAR_SLACK_BOT_TOKEN: ${{ secrets.LINDAMAR_SLACK_BOT_TOKEN }}
      LINDAMAR_SLACK_CHANNEL: ${{ secrets.LINDAMAR_SLACK_CHANNEL }}
      LINDAMAR_SLACK_WEBHOOK: ${{ secrets.LINDAMAR_SLACK_WEBHOOK }}
      LINDAMAR_APPROVER_PAT: ${{ secrets.LINDAMAR_APPROVER_PAT }}
      LINDAMAR_FINDINGS_KEY: ${{ secrets.LINDAMAR_FINDINGS_KEY }}
    with:
      # Your Checkmate console URL: https and a bare host, no trailing slash.
      checkmate-url: https://checkmate.example.com
      claude-credential: api-key
      pr: ${{ github.event.inputs.pr }}
  agent-fix:
    if: ${{ github.event_name == 'workflow_dispatch' && github.event.inputs.agent_fix != '' }}
    permissions:
      id-token: write
      contents: read
    uses: Cincinnati-Associates/checkmate-actions/.github/workflows/agent-fix.yml@v1
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    with:
      checkmate-url: https://checkmate.example.com
      task: ${{ github.event.inputs.agent_fix }}
```

`id-token: write` must be granted by the calling job. A reusable workflow cannot raise its caller's permissions. Without it, the token step fails with an error that says so.

Only one of `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` is needed. Every other secret is optional, and each is described in the workflow's `secrets:` block.

## Tokens

Each job asks Checkmate (`POST <checkmate-url>/v1/actions/token`) for only the tokens it uses:

| Workflow | Job | Requests |
|---|---|---|
| review.yml | agent, review | `reviewer`: read-only, on the private reviewer repository |
| agent-fix.yml | skills | `reviewer` |
| agent-fix.yml | publish | `repository`: write on the calling repository, used to push the fix branch and open its pull request |

Tokens are masked in the log, never printed, and revoked once they have been used. Checkmate only issues them to these workflows at the `v1` tag.

## Security

To report a security issue, contact your Checkmate administrator. Do not open a public issue.
