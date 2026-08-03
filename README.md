# cortex-deploy-action

Deploys this repository's merged control logic to the Cortex AICs enrolled on
it in the [Neuron Device Manager](https://app.neuronindustries.com).

## What it does

1. Mints a GitHub Actions **OIDC id-token** (no stored secrets — the workflow
   grants `id-token: write`) and calls `POST /v1/cd/deploy`. The worker
   verifies the token against GitHub's JWKS and takes the repository, branch,
   and commit sha from **verified claims**, never from inputs.
2. The worker validates the snapshot at that sha (hard-refusing anything that
   declares physical outputs), pins the exact bundle bytes, and fans out
   per-controller directives to the enrolled fleet.
3. The action prints the extreme-caution banner naming every target, then
   polls until every controller reaches a terminal state — per-controller
   results land in the job summary. The job fails if any controller fails to
   apply (or the poll times out).

## Usage

Enrollment happens first, in the Device Manager's **Continuous deployment**
tab (org admin, behind the risk acknowledgment). Then commit this workflow —
committing it is a deliberate repo-admin action, and the `acknowledge`
sentence below is required **verbatim**; it is your per-deploy warning, living
where your reviewers see it:

```yaml
# .github/workflows/synapse-deploy.yml
name: Deploy to Cortex controllers
on:
  push:
    branches: ["main"]
permissions:
  id-token: write   # GitHub OIDC — no stored secrets
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: neuron-industries/cortex-deploy-action@v1
        with:
          acknowledge: "I understand this deploys code to physical controllers without on-device review"
```

Run your tests first (the Synapse IDE's exported `synapse-tests.yml` runs the
project's unit tests on every control-logic PR) and protect the branch —
require pull-request review. **Every merge to this branch is a deployment.**

| Input | Default | Notes |
|---|---|---|
| `acknowledge` | — (required) | Must match the worker's phrase verbatim. |
| `audience` | `https://app.neuronindustries.com/cd` | OIDC audience. |
| `api-base` | `https://app.neuronindustries.com` | On-prem MDM installs point this at their MDM. |
| `poll-timeout-s` | `900` | How long to wait for terminal per-controller states. |

## Scope

Data-orchestration controllers **only**. The platform hard-refuses bundles
that declare physical outputs, and the controller re-verifies before applying
— but neither is a substitute for making sure the enrolled controllers cannot
drive actuators, motion, or any physical output.
