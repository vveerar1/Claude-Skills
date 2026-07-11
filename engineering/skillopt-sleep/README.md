# SkillOpt-Sleep (vendored plugin)

This folder started as a **verbatim copy** of the `skillopt_sleep` engine and
the Claude Code plugin surface from
[microsoft/SkillOpt](https://github.com/microsoft/SkillOpt)
(`skillopt_sleep/`, `plugins/claude-code/`, `plugins/run-sleep.sh`), only
relocated so path resolution (`CLAUDE_PLUGIN_ROOT`-relative lookups in
`scripts/sleep.sh` / `scripts/run-sleep.sh`) resolves correctly at this
folder's location. Licensed under the [MIT License](LICENSE) © Microsoft
Corporation. A small number of targeted patches were made afterward to close
gaps between this plugin's own safety claims and what the code actually did
— see **Deviations from upstream** below. Re-apply all of these on re-vendor;
they are not upstream yet.

## Deviations from upstream

1. **Cosmetic — `check_paths.py` wording.** `skills/skillopt-sleep/SKILL.md`'s
   frontmatter `description` said "...consolidate validated CLAUDE.md/SKILL.md
   behind a held-out gate" — the `CLAUDE.md/SKILL.md` substring reads as a
   broken relative path to this repo's `[A-Za-z0-9_\-./]+/SKILL\.md` linter
   regex. Reworded to "CLAUDE.md and SKILL.md"; no behavior or meaning
   changed.
2. **Safety — secrets weren't redacted in the files that actually go live.**
   `staging.py`'s `redact_secrets()` was applied to `diagnostics.json` and CLI
   error logging, but **not** to `proposed_SKILL.md` / `proposed_CLAUDE.md` —
   the exact files `adopt()` copies over your live `CLAUDE.md` / managed
   `SKILL.md` (with `--auto-adopt`, with no human in the loop). Since
   `reflect()`'s prompt is built from real harvested session text, a secret
   pasted into a real debugging session could have landed in your live memory
   file unredacted, despite the "secrets are redacted from prompts" claim
   below. Fixed: `write_staging()` now runs both through `redact_secrets()`
   before writing.
3. **Safety — the crontab line was built via unescaped f-string
   interpolation.** `scheduler.py`'s `_runner_cmd()` wrapped `project` (an
   arbitrary filesystem path) in manual `"..."` quoting, then wrote the result
   straight into your real crontab — which cron runs through `sh -c` on every
   fire. A path containing `"`, `` ` ``, `$( )`, or `;` could break out of the
   quoting and inject an arbitrary command into your crontab. Fixed: `project`,
   `logdir`, `log`, and the repo root are now `shlex.quote()`-d before
   interpolation.
4. **Safety — `max_tokens_per_night` was a dead config key.** `config.py`
   declared it in `DEFAULTS`, and `budget.py` already had a `Budget` /
   `plan_depth` heuristic built for exactly this purpose, but nothing in the
   production `run_sleep_cycle()` path ever read it — a `--backend
   claude`/`--backend codex` night had no real ceiling on API spend. Fixed:
   `cycle.py` now starts a `Budget` right after backend construction (so
   harvest/mine spend counts too), sizes `dream_rollouts` down via
   `plan_depth()` when the remaining budget is tight, and appends a `report`
   note whenever it caps rollouts or the budget is exhausted at night's end —
   no silent truncation. This caps *rollout depth per task*, not a hard
   mid-call abort inside a single `dream_consolidate()` call; a night can
   still overshoot the cap somewhat if an individual rollout is unusually
   token-heavy. That residual gap is real and not yet closed.
5. **Cosmetic — dead hardcoded path.** `backend.py`'s `resolve_codex_path()`
   listed `~/.nvm/versions/node/v22.22.3/bin/codex` as a candidate ahead of
   the generic "any nvm node version" scan a few lines later, which already
   covers it. Removed; no behavior change for anyone not on that exact nvm
   version, one less leftover-looking line for everyone else.
6. **Safety — `redact_secrets` was a dead config key.** `config.py` declared
   `"redact_secrets": True` in `DEFAULTS`, but `write_staging()` called
   `redact_secrets()` unconditionally — the safe direction, but the knob had
   no effect either way. Fixed: `cycle.py` now reads
   `cfg.get("redact_secrets", True)` and threads it through `write_staging()`
   and the `diagnostics.json` fields; disabling it is honored (it's the
   user's config) but never silently — a `report` note fires whenever it's
   off.
7. **Hardening — `adopt()` now re-redacts as defense-in-depth.** Previously
   `adopt()` copied the staged `proposed_SKILL.md`/`proposed_CLAUDE.md`
   straight to the live path with `shutil.copy2`. Since `write_staging()`
   already redacts, this was redundant for the common case, but didn't cover
   a human hand-editing the staged proposal between `stage` and `adopt` (the
   exact workflow staging exists to allow). `adopt()` now reads, re-runs
   `redact_secrets()`, and writes each file rather than a raw byte copy;
   `_backup()` of the *prior* live file is unaffected (that's a backup of
   what already existed, not the incoming content).
8. **Safety — `scheduler.py`'s `extra` param wasn't shell-quoted.**
   `project`/`logdir`/`log`/repo-root were `shlex.quote()`-d in fix #3 above,
   but `extra` (today only ever `""` or the literal `"--auto-adopt"` from
   `__main__.py`) was appended raw. Not exploitable today since it's a
   hardcoded flag literal, but reopens the same class of bug if a future
   change lets `extra` carry anything else. Fixed: `extra` is now
   `shlex.split()` into tokens and each token `shlex.quote()`-d before
   joining, so a multi-token or attacker-influenced `extra` can't break out
   of the command the way `project` used to.

## What this plugin is

SkillOpt-Sleep gives a local Claude Code agent a nightly **sleep cycle**: it
reviews real past sessions in this repo, replays recurring tasks offline on
your own API budget, and consolidates what it learns into this repo's
`CLAUDE.md` memory and `SKILL.md` skills — but **only** through a held-out
validation gate, and **only** after you explicitly adopt the staged proposal.

It is the deployment-time companion to the (not vendored) `skillopt` training
package: SkillOpt trains a skill offline against a labeled benchmark;
SkillOpt-Sleep applies the same bounded-edit + held-out-gate discipline to
*actual usage of this repo* instead, so it needs no benchmark dataset.

```
harvest ~/.claude transcripts (read-only)
  → mine recurring tasks
  → replay offline
  → consolidate (reflect → bounded edit → GATE)
  → stage proposal (nothing live changes)
  → you review and run "adopt" (backs up first)
```

## Why this is a fit for a skills library with no test harness

This repo's [CLAUDE.md](../../CLAUDE.md) intentionally has no build system or
test framework, and skill `scripts/` are stdlib-only with no LLM calls, so the
full `microsoft/SkillOpt` training package (benchmark-driven, requires
labeled train/val/test data per task, needs `numpy`/`openai`/`azure-*`) was
**not** vendored — there is no natural ground-truth benchmark for something
like `finance/dcf-valuation` or `c-level-advisor/vpe-advisor`.

`skillopt_sleep`, by contrast, has **zero third-party dependencies** (stdlib
only), its default `mock` backend spends no API budget, and it mines its
"benchmark" from how the skills in *this* repo actually get used in real
sessions rather than a pre-labeled dataset. That matches the repo's
deterministic-first, portable-first philosophy far better than the training
package does.

## Use in this repo

```bash
# from the repo root:
engineering/skillopt-sleep/scripts/sleep.sh status                          # what's happened (read-only)
engineering/skillopt-sleep/scripts/sleep.sh dry-run  --project "$(pwd)"     # safe preview, stages nothing
engineering/skillopt-sleep/scripts/sleep.sh run      --project "$(pwd)"     # full cycle, stages a proposal
engineering/skillopt-sleep/scripts/sleep.sh adopt    --project "$(pwd)"     # apply staged proposal (backs up first)
```

Or, once the plugin is installed via Claude Code's plugin marketplace, use
the bundled `/skillopt-sleep [run|dry-run|status|adopt|harvest|schedule|unschedule]`
slash command (see `commands/skillopt-sleep.md` and `skills/skillopt-sleep/SKILL.md`).

Default backend is `mock` (deterministic, **no API spend** — safe to try
immediately). Add `--backend claude` to spend real budget replaying this
repo's own recurring tasks and get genuine lift on `CLAUDE.md` / a target
`SKILL.md`.

## Safety model

- Harvest is **read-only** over `~/.claude` session transcripts.
- Edits are proposed, gated against a held-out replay slice, and **staged**
  under `.skillopt-sleep/staging/<date>/` — nothing live is touched.
- `adopt` is explicit and backs up the prior file first (unless you opt into
  `--auto-adopt`).
- `max_tasks_per_night` is a hard cap (mining stops there). `max_tokens_per_night`
  sizes `dream_rollouts` down via `plan_depth()` and is reported when hit, but
  is not a hard mid-call abort — see deviation #4 above for the exact scope.
- Secrets (API keys, bearer tokens, private-key blocks) are redacted before
  anything is written to the staging dir, including `proposed_SKILL.md` /
  `proposed_CLAUDE.md` (deviation #2 above) — not just diagnostics — and
  re-redacted again at `adopt()` time as defense-in-depth against a
  hand-edited staged proposal (deviation #7). Disabling this via
  `redact_secrets: false` is honored but never silent — it logs a report
  note (deviation #6).
- The generated crontab line, including the `extra` flags parameter, is
  fully `shlex.quote()`-d, not just the path arguments (deviations #3, #8).

## What was and wasn't vendored

| Vendored | Not vendored |
|---|---|
| `skillopt_sleep/` engine (stdlib-only) | `skillopt/` training package (needs `numpy`/`openai`/`azure-*` + labeled benchmarks) |
| `plugins/claude-code/skills\|hooks\|commands\|scripts/` | `plugins/codex/`, `plugins/copilot/`, `plugins/devin/`, `plugins/openclaw/` (other-agent plugin variants) |
| `plugins/run-sleep.sh` shared launcher | `skillopt_webui/` (optional Gradio dashboard) |
| `LICENSE` | `docs/`, `ckpt/`, `data/`, `index.html` (training-package docs/site/checkpoints) |

## Updating

Re-vendor from upstream when the plugin changes:

```bash
git clone --depth 1 https://github.com/microsoft/SkillOpt.git /tmp/skillopt-upstream
cp -r /tmp/skillopt-upstream/skillopt_sleep engineering/skillopt-sleep/skillopt_sleep
cp -r /tmp/skillopt-upstream/plugins/claude-code/skills/skillopt-sleep engineering/skillopt-sleep/skills/skillopt-sleep
cp -r /tmp/skillopt-upstream/plugins/claude-code/hooks engineering/skillopt-sleep/hooks
cp -r /tmp/skillopt-upstream/plugins/claude-code/commands engineering/skillopt-sleep/commands
cp /tmp/skillopt-upstream/plugins/claude-code/scripts/sleep.sh /tmp/skillopt-upstream/plugins/claude-code/scripts/install-cron.sh engineering/skillopt-sleep/scripts/
cp /tmp/skillopt-upstream/plugins/run-sleep.sh engineering/skillopt-sleep/scripts/run-sleep.sh
cp /tmp/skillopt-upstream/LICENSE engineering/skillopt-sleep/LICENSE
```
