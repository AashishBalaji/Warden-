# Build validation notes — what was actually run, and what wasn't

This file states plainly what was verified for real and how, in the
same spirit as the companion Kiln project. No result below is
assumed, estimated, or simulated.

## The real research behind the test data (not fabricated)

Warden's whole premise is: a carried patch stack can silently drift
from upstream. To prove the tool actually detects that, rather than
just asserting it in a docstring, real upstream kernel history was
used to find a genuine drift point.

1. Shallow-fetched 5 real kernel tags (`v5.15`, `v6.1`, `v6.6`,
   `v6.12`, `v6.17`) from `https://github.com/torvalds/linux.git`,
   scoped to `sound/soc/codecs/` only (56MB total for all 5 tags —
   cheap, because the fetch is shallow and the checkout is
   sparse-cone to one directory).
2. Extracted the real `tlv320aic31xx.c` content at each tag and
   diffed them directly. Two real upstream changes were found:
   - `aic31xx_i2c_probe()` dropped its `const struct i2c_device_id
     *id` parameter between v5.15 and v6.1 (a real i2c-core API
     change upstream — probe functions losing the `id` parameter).
   - The regulator-supply-request error path was refactored from an
     `if (ret) { if (ret != -EPROBE_DEFER) dev_err(...); return ret;
     }` block into a single `dev_err_probe()` call, also between
     v5.15 and v6.1.
3. Kiln's existing patch (0001, the `aic31xx_power_on()` reset-error
   fix) was checked against all 5 real tags with a real `git apply
   --check`: it applies cleanly on every one, because its 3-line
   context window never touched the refactored block. This is
   genuinely good news about that patch's robustness, not a result
   this project could pick — it's just what `git apply` reported.
4. A second, new patch (0002) was deliberately written against the
   regulator-defer block *as it reads on v5.15* — a real, defensible
   fix (trace the EPROBE_DEFER path with `dev_dbg` for board bring-up
   visibility), not a patch engineered to fail. Testing it for real
   against all 5 tags showed it applies cleanly on v5.15 and
   genuinely fails (`git apply --check` reports `patch does not
   apply` at the real line) on v6.1 and every tag after, because
   upstream really did rewrite that block.

This is the real data in `series.yaml`, `patches/`, and
`tests/test_warden.py::test_real_upstream_v5_15_is_healthy` /
`test_real_upstream_v6_1_has_genuine_drift` — not hand-written
fixtures standing in for a real result.

## Real bugs found and fixed while building the tool itself

1. **Relative-path bug in `cli.py`**: `series_dir = args.series.parent
   / "patches"` stayed relative, so when `core.py` ran `git apply`
   with `cwd=workdir` (the temp clone), the patch path resolved
   against the clone directory instead of the project directory —
   every real run failed with "No such file or directory" until
   `.resolve()` was added. Caught by actually running the tool, not
   by reading the code.
2. **Missing `git` binary in `python:3.12-slim`** inside the GitLab
   CI container: both the `unit-tests` job's local-repo fixture and
   the `drift-matrix` job shell out to a real `git` binary (that's
   the actual mechanism being tested), and the slim image doesn't
   ship one. `apt-get install git` then failed too — see below.
3. **This sandbox's apt sources (`deb.debian.org`) are not reachable
   through its egress allowlist**, even with `--network host` and
   the proxy variables that fixed `pip install` for Kiln — DNS
   resolution for `deb.debian.org` fails outright inside the
   container. This is a real, current constraint of this one
   sandbox's network allowlist (pip/npm registries and GitHub are
   reachable; arbitrary apt mirrors are not), not a flaw in the
   pipeline design. Fixed by switching the affected jobs' base image
   to `cimg/python:3.12` (a public Docker Hub image, pulled the same
   way Kiln pulled `postgres:15-alpine` and the Artifactory image —
   image pulls go through a different, broader path than `apt-get`)
   which already bundles `git`, rather than fighting the apt/DNS
   restriction.
4. **`git clone` over HTTPS inside the container failed TLS
   verification** (`server certificate verification failed: CAfile:
   none`) even after the image swap — this sandbox's outbound HTTPS
   goes through an agent-proxy with its own certificate, which the
   container doesn't trust by default (same root cause as Kiln's
   `pip install` TLS issue, different symptom). Fixed with
   `gitlab-ci-local`'s own `--ca-file /root/.ccr/ca-bundle.crt` flag,
   which mounts that CA into the container — a verification-time
   flag only, not baked into `.gitlab-ci.yml`, since a real GitLab
   Runner has no such proxy in the way.

## Real GitLab CI verification via `gitlab-ci-local`

All three jobs — `lint`, `unit-tests`, and `drift-matrix` — were run
for real inside real Docker containers via `gitlab-ci-local`:

```
gitlab-ci-local --network host \
  --variable HTTPS_PROXY="$HTTPS_PROXY" --variable HTTP_PROXY="$HTTP_PROXY" \
  --ca-file /root/.ccr/ca-bundle.crt \
  lint unit-tests drift-matrix
```

Real results:
- `lint`: flake8 clean, 0 findings.
- `unit-tests`: 3/3 non-integration tests passed inside the
  container (local-repo fixture, including a real `git apply`
  failure case built on purpose).
- `drift-matrix`: ran a real clone of `torvalds/linux` at all 5 real
  tags inside the container and reported **1/5 tags healthy**
  (v5.15 only) — the exact same result as running the tool directly
  in this sandbox's own shell, now reproduced inside Docker. The job
  itself "fails" (`gitlab-ci-local` reports `FAIL 1`) because
  `warden.cli` exits non-zero when it finds drift — that is the
  drift gate working correctly, not a broken pipeline. The GitHub
  Actions mirror marks this step `continue-on-error: true` for the
  same reason: a real patch stack that needs a real rebase should
  show up loudly in the job output without failing the whole
  workflow on every push.

## What this honestly proves, and what it doesn't

Warden proves, with a real example, that a patch which looks
perfectly fine in isolation can silently stop applying the moment
the kernel base moves — and that running the check regularly (per
tag, in CI) catches it before someone finds out the hard way during
an actual kernel bump. It does not currently attempt a 3-way merge
or an automatic rebase suggestion (git's `-3` apply mode needs real
blob SHAs in the patch's `index` line to reconstruct a merge base,
which these hand-written patches don't carry) — `git apply --check`
tells you *that* a patch broke and exactly where, not how to fix it.
That's a natural next step, not a gap hidden here.
