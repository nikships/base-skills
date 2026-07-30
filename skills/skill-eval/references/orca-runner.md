# Orca Runner

How to run a skill eval by spawning a Droid in a visible Orca tab and watching it work.

You are the evaluator. You stay in your own tab, in the conversation with the user. Each subject agent — the Droid that actually uses the skill under test — gets its own fresh tab in the current workspace. The user can see both: you reasoning here, and the subject working over there.

Read this file when you're about to run evals. The commands below are verified against Orca 1.4.162.

## Contents

- [Why a separate Droid, not a subagent](#why-a-separate-droid-not-a-subagent)
- [Resolve the CLI](#resolve-the-cli)
- [Spawning a subject](#spawning-a-subject)
- [The subject prompt](#the-subject-prompt)
- [Watching the run](#watching-the-run)
- [Intervening when a run goes wrong](#intervening-when-a-run-goes-wrong)
- [Detecting completion](#detecting-completion)
- [Timing and token data](#timing-and-token-data)
- [Cleaning up](#cleaning-up)
- [Serial vs parallel](#serial-vs-parallel)
- [Known constraints](#known-constraints)

---

## Why a separate Droid, not a subagent

A subagent shares your process and your model. A Droid in its own tab is a genuinely separate agent session: its own context, its own skill discovery, its own transcript. That matters for two reasons.

First, **skill loading is real.** The subject loads and reads the SKILL.md the way a real user's agent would, rather than inheriting the text from your context. If the body contradicts itself or leaves a step underspecified, the subject actually stumbles — which is the signal you're trying to measure.

Second, **the user can watch.** The whole point of putting this in a tab rather than a subagent is that the human sees the run happening and can form their own judgment, instead of taking your word for it afterward.

## Resolve the CLI

Follow the resolution order in the `orchestration` skill stub: use `$ORCA_CLI_COMMAND` if set, `orca-dev` in a dev checkout exposing `ORCA_DEV_REPO_ROOT`, `orca-ide` on Linux outside an Orca terminal, otherwise `orca`. Below, `ORCA` is a placeholder — substitute the resolved executable.

Never run bare `orca` on Linux outside an Orca-managed terminal; it resolves to the GNOME Orca screen reader and starts speech on the user's machine.

Confirm the app is up before anything else:

```bash
ORCA status --json
```

If `result.app.running` is false, start it with `ORCA open --json`.

## Spawning a subject

One fresh tab per subject run, in the **current** worktree. Do not create a new git worktree — the eval reads and writes files in the workspace the user is already looking at, and a separate checkout would hide the outputs from them.

```bash
ORCA terminal create --worktree active --title "eval-<name>-with-skill" --command "droid" --json
```

Read `result.terminal.handle` from the response — that's your handle for every later call. Then wait for the TUI to accept input:

```bash
ORCA terminal wait --terminal <handle> --for tui-idle --timeout-ms 60000 --json
```

Check `result.wait.satisfied` is `true` before sending. Sending before the TUI is ready loses the prompt into a dead buffer, and you'll spend a while wondering why the subject is idle.

Use `--title` to encode the eval name and configuration (`eval-chart-labels-with-skill`, `eval-chart-labels-baseline`). The user is looking at a tab bar; titles are the only way they can tell which run is which.

### Fresh tab per attempt, always

Never reuse a tab across runs, and never reuse one after an intervention. A Droid that has already read a skill has it in context — edit the file on disk and the running session keeps working from the stale version it already loaded. Reuse silently evaluates the old skill and reports it as the new one, which is worse than no result at all because you'll trust it.

## The subject prompt

Send the task with `terminal send`. The `--enter` flag submits it:

```bash
ORCA terminal send --terminal <handle> --text "<prompt>" --enter --json
```

Confirm `result.send.accepted` is `true`.

### With-skill runs: invoke by slash command

Invoke the skill explicitly with `/<skill-name>` as the first thing in the prompt, followed by the eval prompt:

```
/<skill-name> <the eval prompt, verbatim>
```

So evaluating `/Users/nik/.agents/skills/grok-imagine` looks like:

```
/grok-imagine create an image of a cat
```

The slash command is what guarantees the subject loads the skill. Describing the task and hoping the skill fires would be measuring discoverability instead of behavior, and the two failures are indistinguishable in the results — a bad run would leave you unable to tell whether the skill is wrong or was simply never read.

The skill name is the directory name, which should match the `name` field in its frontmatter. Check they agree before you start; if they don't, the slash command follows the directory.

Real eval prompts usually need more than the task line — input file paths, and where to save outputs. Append those after the task:

```
/<skill-name> <the eval prompt, verbatim>

Input files: <absolute paths, or omit this line entirely>

Save your outputs to: <workspace>/iteration-<N>/eval-<ID>-<name>/with_skill/run-1/outputs/
Save the files the eval cares about — <e.g. the .docx, the final CSV> — plus a short
notes.md covering anything you were unsure about or worked around.
```

Keep the task wording verbatim from `evals.json`. Rephrasing it per run means the with-skill and baseline arms aren't answering the same question.

### Baseline runs: same prompt, no slash command

For a **no-skill baseline**, send the identical prompt with the `/<skill-name>` prefix stripped — just the task, input files, and output path, writing to `.../baseline/run-1/outputs/`.

For a **previous-version baseline**, you can't use a slash command, since two versions of a skill can't both own the same name. Snapshot the old version somewhere outside the skills directory and point the subject at it by path instead:

```
Use the skill at <workspace>/skill-snapshot/SKILL.md for this task.

<the eval prompt, verbatim>
...
```

That's a deliberate asymmetry between the arms — one invoked by slash command, one by path — so note it when you report results. It's usually harmless, but if the two arms diverge in a way that looks like loading behavior rather than skill content, this is the first thing to suspect.

Everything else stays identical across arms: same task wording, same input files, same output instructions. Any other difference becomes a confound you can't separate from the skill's effect.

Quote carefully. The prompt goes through a shell; use a single-quoted heredoc or a file if it contains quotes, backticks, or `$`.

### Confirm the skill actually loaded

Droid prints an explicit confirmation line when a slash command resolves:

```
●  Skill "skill-eval" activated
```

That line is the first thing to look for when you start polling. If you don't see it — the subject echoes the literal `/<skill-name>` text, or just starts improvising — the skill isn't installed where that Droid can see it. Skills are discovered from `~/.agents/skills/` and `~/.factory/skills/`, so a skill that only exists in a repo checkout won't resolve. Copy or symlink it into one of those directories, then respawn.

Treat that as an environment problem, not a skill failure. Grading a run where the skill never loaded produces a number that looks like a verdict on the skill and isn't.

This is also why every eval iteration needs a fresh tab: the skill is read from disk at activation, so a session that activated the skill before your edit keeps using the old version.

## Watching the run

Poll while the subject works. This is the part that makes a tab-based eval worth more than a subagent: you see the run unfold and can catch a wrong turn before it wastes the whole attempt.

```bash
ORCA terminal read --terminal <handle> --limit 40 --json
```

The response shape is `result.terminal` with these fields:

- `tail` — array of output lines
- `status` — `running` or exited
- `oldestCursor`, `nextCursor`, `latestCursor` — paging cursors
- `limited` — true when more output exists than was returned
- `truncated`, `returnedLineCount`

For a first look, read the tail. To follow along without re-reading everything, keep the `nextCursor` from the previous call and pass it back:

```bash
ORCA terminal read --terminal <handle> --cursor <nextCursor> --limit 40 --json
```

Poll every 20-30 seconds. Tighter than that mostly returns the same spinner frame and burns your context for nothing.

What's worth watching for:

- **Skill not actually read.** No Read of SKILL.md early on means the subject is improvising and the run is measuring nothing.
- **Reinventing a bundled script.** If the skill ships `scripts/build_chart.py` and the subject is writing its own chart code, that's a finding about the skill's discoverability, not about the subject.
- **Thrashing.** Three different approaches to the same subproblem usually means the skill's instructions are ambiguous at that step.
- **Permission or auth walls.** Droid may block on a permission prompt or a model rate limit (see Known constraints). Neither is a skill failure — recognize them and don't score them as one.

Note the transcript is a TUI rendering: box-drawing characters, spinners, redrawn frames. Read it for the shape of what happened, and confirm details against the output files on disk rather than trusting a line that may have been partially overwritten.

## Intervening when a run goes wrong

When polling shows the run going badly wrong — not a small detour, but something that makes the attempt worthless, like the subject never loading the skill or going in circles on an instruction that turns out to be self-contradictory — stop it rather than letting it burn to completion.

1. **Close the subject's tab.**
   ```bash
   ORCA terminal close --terminal <handle> --tab --json
   ```

2. **Record why.** Write a short note in the run directory saying what you saw and at which point. An aborted run is evidence about the skill, and it's easy to lose track of once the tab is gone.

3. **Fix the skill.** Make the edit the failure points to, and keep it a real fix rather than a patch aimed at this one prompt. If the subject couldn't find the bundled script, the skill needs to point at it more clearly — not a line reading "make sure to use build_chart.py for the chart in eval 2."

4. **Discard the partial run.** Don't grade it or fold it into the benchmark. Delete or clearly mark the directory. A half-run scored alongside complete runs quietly corrupts the pass rate and the timing stats.

5. **Restart with a brand-new tab and a brand-new Droid.** This is what guarantees the skill is re-read from disk. Same eval, fresh `terminal create`, fresh `tui-idle` wait, fresh prompt.

Tell the user what you saw and what you changed before restarting. From their side a tab just vanished; a one-line explanation ("the subject never loaded the skill, I made the entry point clearer and restarted it") keeps the run legible.

Use a light touch. Every intervention is a judgment call you're making instead of collecting data, and a skill that only succeeds because you kept rescuing it hasn't been shown to work. If the run is merely slower or messier than you'd like, let it finish — that mess is a finding.

## Detecting completion

Droid is idle at its prompt when the task is done. Wait for that:

```bash
ORCA terminal wait --terminal <handle> --for tui-idle --timeout-ms 1800000 --json
```

Always pass a generous `--timeout-ms`; real eval tasks run minutes, not seconds. Note `tui-idle` also goes true when the subject is idle waiting on a permission prompt, so confirm completion by checking that the expected output files exist:

```bash
ls -la <workspace>/iteration-<N>/eval-<ID>-<name>/with_skill/run-1/outputs/
```

Empty outputs plus an idle TUI means the subject is stuck or asked a question, not that it finished. Read the tail to see which.

## Timing and token data

The old subagent flow got `total_tokens` and `duration_ms` from a task-completion notification. There's no equivalent for a terminal run, so capture wall clock yourself: record a timestamp before `terminal send` and another when the run goes idle, and write `timing.json` in the run directory:

```json
{
  "duration_ms": 412000,
  "total_duration_seconds": 412.0
}
```

Leave `total_tokens` out rather than guessing. `aggregate_benchmark.py` falls back to `execution_metrics.output_chars` as a size proxy when tokens are absent, and an invented number is worse than a missing one — it looks authoritative in the benchmark table.

Droid writes session logs under `~/.factory/sessions/`, so exact token counts can sometimes be recovered from there if the user cares enough to dig.

## Cleaning up

Close each subject tab once its outputs are saved and graded:

```bash
ORCA terminal close --terminal <handle> --tab --json
```

Leave your own tab alone. Only close handles you created — verify with `ORCA terminal list --json` if you're unsure which is which, since the user has other work open.

## Serial vs parallel

**Subject-only runs go serial.** One tab at a time, watched properly. The reason to run subject-only is that you want to see what happens, and you can't actually watch four tabs at once.

**With-baseline runs go parallel in pairs.** Spawn the with-skill and baseline tabs for one eval at the same time, so they finish together and share whatever machine conditions are in play. Spawning the baseline later means comparing runs under different load, which shows up as a timing difference that has nothing to do with the skill.

Run one eval's pair at a time rather than all evals at once. Three evals × 2 arms is six agent tabs, which is unwatchable and can hit model rate limits partway through — leaving you with a half-populated benchmark.

## Known constraints

Verified against Orca 1.4.162 and Droid 0.180.0:

- **`droid` is not one of Orca's known agent ids.** `worker-start --agent droid` and `worktree create --agent droid` don't accept it; the documented ids are `claude`, `codex`, `omp`, `pi`, `grok`. Use `terminal create --command "droid"`, which works.
- **`terminal wait --for tui-idle` works correctly for Droid.** Confirmed satisfied on a fresh session.
- **`orchestration` lifecycle is optional here.** `worker_done` reporting, dispatch preambles, and `check --wait` are for workers that report back through Orca's inbox. This eval loop reads terminal output and files on disk directly, so plain `terminal create` / `send` / `read` / `wait` is enough. If you do want task provenance, create a Run and Task and dispatch to the handle with `dispatch --to <handle>` — but note `--inject` targets recognized agent CLIs, and Droid isn't in that list, so send the prompt with `terminal send` instead.
- **`worker-read --source auto` won't give a hook transcript for Droid.** It promises exact transcripts for Codex, Claude, OpenClaude, and Grok; anything else falls back to terminal output. Just use `terminal read`.
- **Interactive `droid` has no `--model` flag.** The subject inherits the user's configured default model. If that model is rate-limited, the subject stalls immediately with a usage-limit message — which looks nothing like a skill failure once you've seen it, but will waste a run if you don't check. `droid exec` does support `-m/--model`, and interactive `droid` supports `--settings <path>`, if you need to pin a model.
- **`terminal stop` takes `--worktree`, not `--terminal`.** It stops every terminal in a worktree. To close one tab, use `terminal close --terminal <handle> --tab`.
