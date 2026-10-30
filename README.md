# Warden

A kernel/driver patch-stack health checker — a companion project to
**Kiln**, both built against HARMAN's **Associate Engineer – Gitlab
CI/CD** job description (R-56518-2026, Bangalore). Kiln proves a
single carried kernel patch applies today; Warden answers the
question that actually matters once a project ships: **if we bump
our kernel base, which of our carried patches still apply, and which
ones silently need a rebase?**

Real out-of-tree patches drift from upstream as the kernel moves
forward. Warden re-applies a real, ordered patch stack against real
upstream kernel tags and reports, per patch, exactly where it breaks
— so that discovery happens in CI, not during an actual kernel bump
three months into a project.

Full real verification story, including every real bug hit and fixed
along the way, is in
[`docs/validation-notes.md`](docs/validation-notes.md).

## What's here

```
series.yaml            the patch-stack manifest (repo, sparse paths,
                        ordered list of patches + their targets)
patches/                the two real patches in the stack
warden/
  core.py               real git clone + git apply --check logic
  cli.py                 `warden check` / `warden matrix` commands
tests/test_warden.py     local-repo unit tests + real-upstream
                          integration tests (see below)
.gitlab-ci.yml           lint / unit-tests / drift-matrix pipeline
.github/workflows/ci.yml GitHub Actions mirror of the same jobs
```

## The real finding this project is built around

Patch `0001` (carried from Kiln: fixes a swallowed
`aic31xx_reset()` failure in `aic31xx_power_on()`) applies cleanly
against every real kernel tag checked, from `v5.15` through `v6.17`.

Patch `0002` (new: traces an `EPROBE_DEFER` path for board bring-up
visibility) applies cleanly against `v5.15` — and **genuinely fails**
from `v6.1` onward, because upstream really refactored that error
path into a single `dev_err_probe()` call between those two tags.
Both outcomes come from real `git apply --check` runs against real
sparse-cloned upstream source, not a hand-written example.

```
$ python3 -m warden.cli matrix --tags v5.15,v6.1,v6.6,v6.12,v6.17

=== v5.15: HEALTHY ===
  [OK  ] 0001-ASoC-tlv320aic31xx-fix-swallowed-reset-failure-on-p.patch
  [OK  ] 0002-ASoC-tlv320aic31xx-trace-regulator-probe-defer.patch

=== v6.1: DRIFTED ===
  [OK  ] 0001-ASoC-tlv320aic31xx-fix-swallowed-reset-failure-on-p.patch
  [FAIL] 0002-ASoC-tlv320aic31xx-trace-regulator-probe-defer.patch
           error: patch failed: sound/soc/codecs/tlv320aic31xx.c:1661
           error: sound/soc/codecs/tlv320aic31xx.c: patch does not apply

1/5 tags healthy
```
(`v6.6`, `v6.12`, `v6.17` all drift the same way as `v6.1` — the
refactor landed once and stuck.)

## What was verified for real, and how

| Piece | How it was verified | Result |
|---|---|---|
| The drift itself | real sparse clones of 5 real kernel tags, real `git apply --check` of both real patches against each | 1/5 healthy — a genuine, reproducible finding |
| `warden.core` / `warden.cli` | a real local-repo pytest fixture (its own tiny real git repo) + the real upstream integration tests above | 5/5 tests pass |
| Code quality | `flake8 --max-line-length=100` | 0 findings |
| `.gitlab-ci.yml` | run for real with `gitlab-ci-local` against real Docker containers, including the network-dependent `drift-matrix` job | all 3 jobs ran; drift-matrix correctly reports the real drift (see validation-notes.md for the TLS/apt findings along the way) |
| `.github/workflows/ci.yml` | mirrors the same 3 jobs for GitHub-hosted CI | — |

## What wasn't attempted, and why

Warden does not attempt an automatic 3-way rebase. `git apply -3`
needs real blob SHAs in a patch's `index` line to reconstruct a merge
base, which hand-written patches like these don't carry — so Warden
tells you precisely *that* a patch broke and *where* (the real `git
apply` error), not how to fix it automatically. That's a reasonable
next step, documented rather than hidden.

## Running it yourself

```bash
pip install pyyaml

# Check the stack against one real kernel tag
python3 -m warden.cli check --tag v6.6

# Check it against several tags at once (what CI gates on)
python3 -m warden.cli matrix --tags v5.15,v6.1,v6.6,v6.12,v6.17 --json

# Run the test suite (unit tests only, no network)
python3 -m pytest -v -m "not integration"

# Include the real-upstream integration tests (needs network)
python3 -m pytest -v
```
