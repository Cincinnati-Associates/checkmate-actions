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

Only one of `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` is needed, or `OPENROUTER_API_KEY` with `claude-credential: openrouter` (below). Every other secret is optional, and each is described in the workflow's `secrets:` block.

### A subscription that people also use

With `claude-credential: subscription`, reviews and people draw on the same 5-hour and weekly windows. `usage-reserve-percent` keeps a share of both free:

```yaml
    with:
      claude-credential: subscription
      usage-reserve-percent: 20
```

With 20, a review does not start once either window is 80% used or more. The required check goes red and says which window it was and when it resets, and the review itself spends nothing. The default is 0: no reserve. The check runs before a review starts, so a review that starts just under the line can still cross it.

### Another model through OpenRouter (Preview)

`claude-credential: openrouter` runs the same CLI, skills and sub-agents on a non-Claude model, an OpenAI one for example, through OpenRouter's Anthropic-compatible endpoint. Add an `OPENROUTER_API_KEY` secret, with a credit limit on the key, and pass it with the other secrets:

```yaml
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
    with:
      claude-credential: openrouter
      model: openai/gpt-6.2-sol
```

- `model` must be an OpenRouter id (`vendor/model`). Every model slot of the CLI is pinned to it, so sub-agents and background calls do not fall back to Claude.
- To bill your own OpenAI key, attach it to the OpenRouter account as a BYOK key and set its shared-capacity fallback to "never". Otherwise OpenRouter bills its own credits.
- The diff goes to OpenRouter and to the model's provider. Both must be acceptable processors of the repository's code.
- The CLI's cost figure is not a real cost for these models. The run summary says "billed by OpenRouter": read the usage of the key there.
- Review quality on non-Claude models has not been measured to the bar Claude is held to. Read the verdict as advisory until it has been.

Only the review uses this credential. Agent fixes still need a Claude credential.

### OpenAI's Codex CLI as the engine (Preview)

`engine: codex` runs the same review (stacks, candidate inventory, rule skills, one sub-agent per stack, findings, verdict and gate) on OpenAI's Codex CLI instead of Claude Code. With an API key:

```yaml
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    with:
      engine: codex
```

With the ChatGPT login uploaded to the Checkmate console (AI providers → ChatGPT), lent to one review job at a time: `codex-credential: chatgpt`, and no key. Private and internal repositories only.

- Codex runs in a read-only sandbox, with no network for its commands and no GitHub token in the step. The job proves the sandbox on the runner first and does not review if it does not hold.
- Codex reports tokens, not dollars. The cost in the run summary is computed from a pinned price table in the reviewer's `config/codex.json`, and says so. On a ChatGPT login it is what the API would have charged.
- `codex-model` overrides the orchestrator's model; the two sub-agent models are pinned in the reviewer's config.
- The diff goes to OpenAI, which must be an acceptable processor of the repository's code.
- Review quality on Codex has not been measured to the bar Claude is held to. Read the verdict as advisory until it has been.

#### Waiting in line for the ChatGPT login: `codex-lease-wait`

One login is lent to one job at a time, so on a busy repository another job often holds it. A review that finds it lent waits in line: Checkmate lends it to the review that has waited longest, with one place per pull request, and a newer run of a pull request keeps the place of the run it replaces. A run cancelled by a push gives its place up, and the pull request's place is kept for a few minutes for the new run. `codex-lease-wait` (default 40, at most 60) is how many minutes a review waits in line before it falls back or fails. The job summary says the place in line at the start, how long the review waited, and whether Codex reviewed; the console's ChatGPT page shows the line.

The wait runs on the runner, so it counts as Actions minutes. So that it fits, the agent job's time limit on this path is 120 minutes (240 with a `timeout-minutes` over 60, 360 over 225). The lease is asked for that whole limit less the time waited, so it outlives the job. Only a run started by a pull request event holds its pull request's place; a hand-started retry waits in a place of its own. `timeout-minutes` still limits the review step itself, and the wait is cut short when it would not leave the review its `timeout-minutes`. A Checkmate from before the queue still lends the login: the review asks every 30 seconds, as before.

#### When the ChatGPT login cannot be used: `codex-fallback`

With `codex-credential: chatgpt`, `codex-fallback: claude` (the default) reviews on Claude instead of failing the check, exactly as `engine: claude` would, with `claude-credential` and its secret (pass `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` as for a Claude review). `codex-fallback: none` fails the check, as before.

| What happened | `codex-fallback: claude` with a Claude credential | `none`, or no Claude credential |
|---|---|---|
| Lease granted | Codex reviews | Codex reviews |
| Still waiting in line after `codex-lease-wait` minutes (default 40) | Claude reviews | fails, after waiting at least 10 minutes |
| A newer run of the same pull request took its place in line (409 `queue_superseded`) | Claude reviews | fails |
| 404 `login_not_set`, 409 `login_needs_reupload`, 502 `secret_unreadable` | Claude reviews | fails |
| Checkmate does not answer (no answer, 503 three times, another 5xx) | Claude reviews | fails |
| OpenAI refuses the login during the Codex review | lease released as `auth_failed`, then Claude reviews | fails, lease released as `auth_failed` |
| 403 (`public_repository`, `visibility_unknown`, `workflow_not_allowed`, `owner_not_allowed`), 401, 400 | fails: a configuration error | fails |

A fallback is said in a notice, in the job summary (for example "Codex login lent to owner/repo run N until T; this run was 3 of 5 in line after waiting 40 min; reviewed on Claude."), in one line at the end of the review, and in the cost record (`engine: claude`, `requestedEngine: codex`, `fallback: <reason>`). Cost and usage limits are then Claude's. A Claude review after OpenAI refused the login does not run the `usage-reserve-percent` check. An `api-key` credential is not lent, so it never falls back.

Agent fixes run on Claude only; `engine: codex` is refused there.

### Runner

Every job runs on `ubuntu-latest`, which uses the calling organisation's GitHub Actions minutes. To use other runners, set the `LINDAMAR_RUNNER` Actions variable on the calling repository or organisation:

```sh
gh variable set LINDAMAR_RUNNER --org <org> --visibility all --body blacksmith-2vcpu-ubuntu-2404-arm
```

The jobs wait on the Claude API and do no builds, so the smallest runner is enough. The caller file does not change. Delete the variable to go back to `ubuntu-latest`.

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
