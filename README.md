# tencent-spot-runner

Run a GitHub Actions job on a big Tencent Cloud spot machine that is bought for the run and destroyed after it. Three composite actions:

- `start` buys one spot CVM, registers it as an ephemeral self-hosted runner for the calling repository, and waits until it is online.
- `stop` destroys the machine and removes the runner registration.
- `sweep` destroys every CI machine older than 75 minutes. Run it on a schedule from one repository; it covers every repository that uses this tool.

The API calls are signed with `node:crypto` and sent with `fetch`, so no SDK or other third-party package runs next to the keys. The machine boots with all inbound traffic closed, installs Docker, optionally pulls your job's container image, and serves exactly one job.

## When it is worth it

Buying a machine and getting its runner online takes about 3.5 minutes. That pays off when a job takes more than about 8 to 10 minutes on GitHub's 2-core runners, or for heavy one-off runs (a full end-to-end suite, flake hunting with `--repeat-each`). A suite that finishes in a few minutes on GitHub's runners gets slower, not faster.

Use it only from **private** repositories. A self-hosted runner on a public repository can run code from strangers' pull requests.

## Use the machine fully

Spot machines are billed by the second, so size the job to the machine. Measure, do not guess: run your suite at a few worker counts on the machine, record the time and the CPU load, and pick the smallest count after which the time stops falling. Put the result in `sizes`, which hands it back to your job.

Measured on `SA9.8XLARGE64` (32 vCPU, 64 GB) with a 2,230-test Playwright suite:

| Workers | Time | CPU busy |
| --- | --- | --- |
| 12 | 241 s | 81% |
| 16 | 218 s | 90% |
| 20 | 208 s | 94% |
| 24 | 208 s | 96% |

So 20. A suite that renders WebGL in software kept the same machine 79% busy with only 4 workers and failed at 6, so measure each suite separately.

## Setup

### 1. Tencent Cloud

Create a CAM sub-user with programmatic access only and attach this policy. It can create and pay for CVMs, destroy Hong Kong CVMs, and read instances, images, offers and prices; nothing else.

```json
{
  "version": "2.0",
  "statement": [
    { "effect": "allow", "action": ["finance:trade"], "resource": "*" },
    { "effect": "allow", "action": ["name/cvm:RunInstances"], "resource": "*" },
    {
      "effect": "allow",
      "action": ["name/cvm:TerminateInstances"],
      "resource": "*",
      "condition": { "string_equal": { "cvm:region": "ap-hongkong" } }
    },
    {
      "effect": "allow",
      "action": [
        "name/cvm:DescribeInstances",
        "name/cvm:DescribeImages",
        "name/cvm:DescribeZoneInstanceConfigInfos",
        "name/cvm:InquiryPriceRunInstances"
      ],
      "resource": "*"
    }
  ]
}
```

Notes from setting it up:

- A `cvm:region` condition on `RunInstances` is refused, because it does not apply to the VPC, subnet and system-disk resources the call also needs.
- Without `finance:trade` the call fails with "no payment rights".
- Policy changes take about a minute to apply.
- Set a budget alert in the billing console. A policy cannot cap how many machines are bought.
- The account needs a balance, because spot instances freeze some funds while they run.

### 2. GitHub token

Create a fine-grained token with **Administration: Read and write** on the repositories that will use the runner. It is used only to get a runner registration token and to remove the runner afterwards. A fine-grained token belongs to one user or one organization, so repositories under different owners need one token each.

### 3. Repository secrets

Each repository needs `TENCENT_SECRET_ID`, `TENCENT_SECRET_KEY` and `RUNNER_REGISTRATION_PAT`. To set one secret in several repositories with a single paste (nothing is echoed):

```bash
read -rs VALUE && for repo in owner/repo-a owner/repo-b; do printf %s "$VALUE" | gh secret set TENCENT_SECRET_ID --repo "$repo"; done; unset VALUE
```

### 4. Workflow

```yaml
jobs:
  spot-start:
    runs-on: ubuntu-24.04
    timeout-minutes: 12
    outputs:
      ready: ${{ steps.spot.outputs.ready }}
      label: ${{ steps.spot.outputs.label }}
      instance-id: ${{ steps.spot.outputs.instance-id }}
      workers: ${{ steps.spot.outputs.size && fromJSON(steps.spot.outputs.size).workers }}
    steps:
      - id: spot
        # Fall back to GitHub's runners when no machine can be bought.
        continue-on-error: true
        uses: reidgao/tencent-spot-runner/start@v1
        with:
          secret-id: ${{ secrets.TENCENT_SECRET_ID }}
          secret-key: ${{ secrets.TENCENT_SECRET_KEY }}
          github-token: ${{ secrets.RUNNER_REGISTRATION_PAT }}
          sizes: '[{"cpu":32,"workers":20},{"cpu":16,"workers":10}]'
          prepull-images: mcr.microsoft.com/playwright:v1.59.1-noble

  test:
    needs: spot-start
    if: ${{ needs.spot-start.outputs.ready == 'true' }}
    runs-on: [self-hosted, '${{ needs.spot-start.outputs.label }}']
    container:
      image: mcr.microsoft.com/playwright:v1.59.1-noble
      options: --ipc=host
    steps:
      - uses: actions/checkout@v5
      - run: npm ci && npx playwright test --workers=${{ needs.spot-start.outputs.workers }}

  spot-stop:
    needs: [spot-start, test]
    if: ${{ always() && needs.spot-start.outputs.instance-id != '' }}
    runs-on: ubuntu-24.04
    steps:
      - uses: reidgao/tencent-spot-runner/stop@v1
        with:
          secret-id: ${{ secrets.TENCENT_SECRET_ID }}
          secret-key: ${{ secrets.TENCENT_SECRET_KEY }}
          github-token: ${{ secrets.RUNNER_REGISTRATION_PAT }}
          instance-id: ${{ needs.spot-start.outputs.instance-id }}
          label: ${{ needs.spot-start.outputs.label }}

  hosted-test:
    needs: spot-start
    if: ${{ !cancelled() && needs.spot-start.outputs.ready != 'true' }}
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - run: npm ci && npx playwright test
```

### 5. Sweep on every run, and on a schedule

Tencent Cloud refuses a scheduled destroy (`ActionTimer`) on spot instances, so a sweep is the backstop for a stop job that never ran. Add it to the stop job, after `stop`, so every run sweeps:

```yaml
      - if: ${{ always() }}
        uses: reidgao/tencent-spot-runner/sweep@v1
        with:
          secret-id: ${{ secrets.TENCENT_SECRET_ID }}
          secret-key: ${{ secrets.TENCENT_SECRET_KEY }}
```

A schedule alone is not enough: GitHub can start scheduled workflows hours late. Keep one schedule in one repository as a second layer, off the hour and half hour, where GitHub is busiest:

```yaml
on:
  schedule:
    - cron: '17,47 * * * *'
  workflow_dispatch:

jobs:
  sweep:
    runs-on: ubuntu-24.04
    steps:
      - uses: reidgao/tencent-spot-runner/sweep@v1
        with:
          secret-id: ${{ secrets.TENCENT_SECRET_ID }}
          secret-key: ${{ secrets.TENCENT_SECRET_KEY }}
```

## Inputs worth knowing

| Action | Input | Default | Meaning |
| --- | --- | --- | --- |
| start | `sizes` | `[{"cpu":32},{"cpu":16}]` | Sizes to try, best first. Extra fields come back in the `size` output. |
| start | `max-cny-per-hour` | `5` | Highest spot price the run may pay. |
| start | `prepull-images` | | Docker images to pull at boot. |
| start | `region` | `ap-hongkong` | The policy above only lets the sub-user destroy machines in Hong Kong. |
| sweep | `prefixes` | `ci-` | Machine names to sweep. Machines are named `ci-<repo>-<run>-<attempt>`. |
| sweep | `max-age-minutes` | `75` | Destroy machines older than this. |

## Limits

- Spot machines can be reclaimed with two minutes' notice. A reclaimed machine fails its job; rerun it.
- Only x86 families with at least 2 GB of memory per vCPU are considered (SA, S, C, CA, M, MA).
- Up to eight offers are tried, best size first and cheapest first within a size; an authorization error stops at the first.
