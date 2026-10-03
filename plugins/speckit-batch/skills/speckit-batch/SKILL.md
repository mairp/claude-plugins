---
name: speckit-batch
description: Run a Spec Kit skill (speckit-analyze by default, or any /speckit-* command) over several specs at once on a chosen harness — claude (Claude Code) or dsh (DeepSeek Harness) — with the model and reasoning effort pinned per run, sequentially or N features in parallel, with a shared queue so two harnesses can drain batches side by side, and background tracking that reports per-feature results. Use this whenever the user wants to analyze, clarify, plan, or otherwise process a range, list, or batch of spec folders ("run analyze on 105-107 with fable", "do 108 to 111 on dsh with glm-5.3 max reasoning", "batch the remaining specs 4-5 at a time", "launch the next batch when this one ends"), whenever a spec-kit command must run on dsh with a specific model, and whenever more than one spec needs the same speckit command — even if the word "batch" is never said.
---

# speckit-batch

Run one Spec Kit command across many `specs/<id>-*` folders, on either harness, with the
model and reasoning bound per run. Everything mechanical lives in two scripts under
`scripts/`; your job is to pick the parameters, launch, track, and report.

## What the user usually means

| They say | You run |
|---|---|
| "analyze 105-107 with claude/fable" | `--harness claude --model fable 105-107` |
| "108 to 111 on dsh, glm-5.3, max reasoning" | `--harness dsh --model glm-5.3 --reasoning max 108-111` |
| "run 3 at a time" | add `--jobs 3` |
| "next batch when this one ends", "batches of 4-5" | `chain.sh` with a queue file (below) |
| "clarify / plan / tasks instead of analyze" | `--command speckit-clarify` etc. |
| "and fix what you find" | that is the default `--prompt-tail`; pass your own to change it |

Spec numbering has gaps (no 114, 120, 121, 132, ...), so selectors are flexible and mixable:
`105-107` (range), `122,123,124` (list), `113` (one id), `130-adapter-dsh` (folder name).
Missing ids inside a range are logged as `SKIP`, never an error. Duplicates run once.

## Launch

```bash
S=~/.claude/skills/speckit-batch/scripts
cd <spec-kit-repo>            # or pass --repo; it must hold specs/ and .specify/

# Claude Code — model alias or id, effort low|medium|high|xhigh|max
nohup $S/run-batch.sh --harness claude --model fable --reasoning high \
  --settings ~/.claude/settings-speckit-qmd-only.json 105-107 > /dev/null 2>&1 &

# dsh — model id under a declared provider (glm-* -> zai, qwen* -> local-high, else --provider)
nohup $S/run-batch.sh --harness dsh --model glm-5.3 --reasoning max --jobs 2 108-111 > /dev/null 2>&1 &
```

Always `--dry-run` first: it prints the resolved feature list and the exact prompt, exits
non-zero on a bad selector or an undeclared dsh provider, and writes nothing. Run each
harness from `nohup … &` (or Bash `run_in_background`) — a batch takes 10–60 min per feature.

The default `--prompt-tail` (see `DEFAULT_PROMPT_TAIL` in the script) is the auto-repair
variant, carrying the owner's words "propose and resolve every issue automatically with the
best suggestion". It allows edits anywhere in this feature's folder (not just spec/plan/tasks),
says what must not change (ids, numbering, as-is literals), forbids questions, git and code
edits, and asks for one read + fix + re-verify of the edited items only. **Nothing may be left
open**: a finding owned by another feature or the orchestrator gets a ready-to-apply proposal
(target file/section, exact text, why) under "Proposed upstream fixes" in the feature's
analyze-report.md. (An earlier tail allowed "Deferred with a reason"; on 2026-09-30 that left
33 findings open with no solution, against the owner's instruction — do not bring it back.)
The closing format (findings table with a State column, edited files,
`safe for /speckit-implement: yes|no`) lets the tracker read results uniformly. For a read-only pass: `--prompt-tail "report only, change nothing"`.

Logs land in `<repo>/../speckit-batch-runs/logs/` (override with `--logdir`):
`<harness>.status` (append-only: `RANGE`, `START`, `END rc=N`, `SKIP`, `ALL-DONE`) and
`<harness>-<feature>.log` (the agent's final message plus stderr).

## Keep both harnesses busy: the queue

When the user wants the remaining specs done in successive batches, write a queue and attach
one `chain.sh` per harness. Each chain claims the top line under `flock` (so a batch never
runs twice), runs it with `run-batch.sh`, and repeats until the queue is empty. `--wait-pid`
makes a chain wait for a batch that is already running before it claims anything.

```bash
printf '%s\n' "112-117" "118-124" "125-129" "130-140" "141-145" "146-149" > queue.txt
nohup $S/chain.sh --queue queue.txt --wait-pid <claude-batch-pid> -- \
  --harness claude --model fable --reasoning high --settings ~/.claude/settings-speckit-qmd-only.json &
nohup $S/chain.sh --queue queue.txt --wait-pid <dsh-batch-pid> -- \
  --harness dsh --model glm-5.3 --reasoning max &
```

Size batches by *folder count* (4–5 folders), not by id span — a `130-140` line is five
folders here because of the gaps. A queue line is a list of selectors (`"119 122 123"` is
three ids; write ranges as `118-124`, never `118 124` — the chain tolerates that legacy form
but nothing else does). A chain writes `CHAIN claimed batch …` and, at the end, `CHAIN-DONE`
into the same status file, so one `cat logs/*.status` tells the whole story.

**A pool that refills** ("keep N busy, start the next as soon as one ends"): write the queue
with one feature per line and start N chains with `--jobs 1` on it, a few seconds apart.
Every live session counts against the user's cap, whichever chain started it. A chain stops
(`CHAIN-STOPPED run-batch failed`) and puts its batch back if the runner itself fails.

**Provider quota.** A 5-hour usage cap (z.ai returns `429 … Usage limit reached`; Claude Code
prints `You've hit your session limit · resets 9pm (Asia/Dubai)` and exits 1) is not an
analysis result. For claude, each run has its own session id (`<harness>-<feature>.session`
while unfinished) and the attempt after a quota stop resumes that session instead of paying
for the run twice. The Claude limit is shared with every other Claude session of the user,
the interactive ones included — say so before launching many at once.
`run-batch.sh` recognises it (`END … QUOTA`), stops launching features,
writes the unfinished ones to `<harness>.requeue`, logs `ABORTED` and exits 75; `chain.sh`
puts them back at the front of the queue and, with the default `--on-quota wait`, sleeps
until the reset time in the error (`--reset-tz`, default Asia/Shanghai — z.ai prints Beijing
time) and resumes; `--on-quota stop` ends the chain instead. `--not-before <date>` starts a
chain only after a known reset. Learned the hard way on 2026-09-30: without this, a chain
burned six batches in twenty seconds. Two parallel sessions on one provider reach the cap
about twice as fast — the second harness is the real concurrency here.

## Track and report

Do not poll in the foreground. Spawn one tracking subagent per harness (model `sonnet` — never
fan out on the session's own frontier model) and tell it: the status file, the `ALL-DONE`
marker to wait for (use the Monitor tool), the per-feature logs, and what to report:

- per feature: exit code, wall-clock from `START`/`END`, the findings table and what was
  repaired (from the log; `specs/<feature>/analyze-report.md` if the skill wrote one),
- for dsh: confirm `request/context` in the session JSONL shows the intended provider/model.
  Do not look for a `skill` tool call: dsh pre-injects the skill body into the user message
  when the prompt names it, so the evidence is the `check-prerequisites.sh` call and a real
  findings section in the log,
- anything that says the skill did not run: a log without a findings section, complaints
  from `check-prerequisites.sh`, `rc≠0`, an aborted turn.

Relay the report to the user per batch as it lands; while waiting, say plainly that it is
still running rather than guessing. If `specs/` is git-ignored in the repo (it is in
specstride), `git diff` shows nothing — use file mtimes and the reports themselves.

## Things that bite

- **An implementation loop in the same repo.** If a loop implements a feature there (a
  `.specstride/lock`, a PROPOSER session), do not run a batch that edits `specs/`: the loop's
  static checks may scan every spec, and a red check makes its proposer "repair" files outside
  its scope (specstride, 2026-09-30). Finish the batch before the loop starts, or wait for it to
  end. Where the repo has a static spec check, add "run it after your edits and keep it green" to
  `--prompt-tail`.

- **A feature the Specstride campaign is running is skipped.** The campaign driver lists what
  it implements right now in `<repo>/.mixture-of-loops/campaign/active.json`; `run-batch.sh`
  logs `SKIP <feature> (running in the Specstride campaign, active.json)` and does not run it.
- **dsh ignores `--patch` for the model.** Only `$DSH_HOME/settings.yaml` binds
  `agent-default-model` at run time. `run-batch.sh` therefore builds a throwaway `DSH_HOME`
  (symlinks + rewritten `settings.yaml`) per batch; never edit `~/.dsh/settings.yaml` for this.
  Details and the verification recipe: `references/dsh-model-binding.md`.
- **Concurrent runs in one repo are safe only because each run is pinned** through
  `SPECIFY_FEATURE`/`SPECIFY_FEATURE_DIRECTORY`; `check-prerequisites.sh` would otherwise
  resolve the feature from the shared `.specify/feature.json`. The scripts set both. Keep
  `--jobs` modest (2–3): the ceiling is the provider's rate limit, and every job is a full
  agent session that edits `specs/<feature>/`.
- **Root needs `IS_SANDBOX=1`** for `claude --dangerously-skip-permissions`; the script sets
  it. dsh's equivalent is `DSH_PERMISSION_MODE=danger-full-access`, also set.
- A running bash script must not be edited in place (bash reads it incrementally). Copy first.
