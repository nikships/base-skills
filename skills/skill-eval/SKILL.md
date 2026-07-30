---
name: skill-eval
description: Build and run evals on an existing skill, benchmark it against a baseline, and iterate on the skill based on the results. Use this whenever someone wants to test a skill, write eval cases or assertions for a skill, check whether a skill actually helps, compare two versions of a skill, benchmark skill performance with variance analysis, or debug why a skill underperforms — even if they only say something vague like "is this skill any good?" or "make this skill better."
---

# Skill Eval

A skill for evaluating skills and improving them based on what the evals reveal.

The loop looks like this:

- Read the skill under evaluation and figure out what "working well" means for it
- Build an eval set: realistic test prompts, plus assertions that are objectively checkable
- Run the skill on those prompts alongside a baseline (no skill, or the previous version)
- Help the user evaluate the results both qualitatively and quantitatively
  - Use `eval-viewer/generate_review.py` to put the actual outputs in front of the user
  - Aggregate the assertion grades into a benchmark so there's a number to move
- Revise the skill based on the user's feedback and whatever the benchmark exposes
- Rerun and compare against the previous iteration

Your job is to figure out where the user is in this loop and jump in there. Usually they arrive with a skill that already exists and no evals — start by building the eval set. Sometimes evals already exist, in which case go straight to running them. Sometimes they've already run evals and want help interpreting them or fixing the skill.

Be flexible about rigor. If the user says "I don't need a whole benchmark, just try it on a couple prompts and tell me what you think," do that instead.

## Communicating with the user

This skill gets used by people across a wide range of familiarity with coding jargon. Pay attention to context cues to calibrate your phrasing. In the default case:

- "evaluation" and "benchmark" are borderline, but OK
- for "JSON" and "assertion" you want to see serious cues from the user that they know what those things are before using them without explaining them

It's fine to briefly define a term if you're in doubt.

---

## Building the eval set

### Understand what you're evaluating

Read the skill's SKILL.md and its bundled resources before writing a single test case. You need to know what it claims to do, what shape the output should take, and where it's likely to be fragile. If there's a `scripts/` directory, skim it — a skill that bundles a script has an implicit assertion ("the run should actually use the script rather than reinventing it").

Then ask the user the things you can't infer:

1. What does success look like for this skill? What would make you say "yes, that's right"?
2. Where do you suspect it's weak, or what made you want to test it?
3. Are there input files or realistic examples we should feed it?
4. Are the outputs objectively verifiable (file transforms, data extraction, code generation, fixed workflow steps), or subjective (writing style, visual design)?

That last one shapes everything downstream. Objectively verifiable skills get real assertions and a meaningful pass rate. Subjective skills are better judged by the human looking at outputs in the viewer — don't force assertions onto things that need human judgment, because a pass rate built from mushy assertions is worse than no pass rate at all. It creates false confidence and it'll push you to optimize the wrong thing.

### Write the test prompts

Come up with 2-3 test prompts to start — the kind of thing a real user would actually say, not sanitized one-liners. Concrete details matter: file paths, column names, company names, a bit of backstory, the occasional typo or lowercase sentence. Prompts that are too clean produce runs that are too easy, and then the eval can't tell a good skill from a mediocre one.

Aim for coverage rather than repetition: the skill's central happy path, plus at least one case that pokes at the part the user suspects is weak or that the skill's instructions handle ambiguously.

Share them with the user before running: "Here are a few test cases I'd like to try. Do these look right, or do you want to add more?"

Save them to `evals/evals.json` inside the skill directory. Don't write assertions yet — just the prompts. You'll draft assertions in the next step while the runs are in flight, which is a better use of that waiting time.

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User's task prompt",
      "expected_output": "Description of expected result",
      "files": []
    }
  ]
}
```

See `references/schemas.md` for the full schema (including the `assertions` field you'll add shortly).

Later, once the skill is holding up on the small set, expand the test set and run again at larger scale. Three prompts tell you whether the skill is broken; a dozen tell you whether it generalizes.

---

## Running the evals

This section is one continuous sequence — don't stop partway through. Do NOT use `/skill-test` or any other testing skill.

**You are the evaluator.** You stay here, in the conversation with the user. The agent that actually uses the skill is a separate Droid, spawned into its own visible tab in the current Orca workspace. The user can watch both: you reasoning here, the subject working over there.

Read `references/orca-runner.md` for the verified command recipes — spawning, prompting, polling, intervening, and cleanup. The rest of this section is the decision-making around those commands.

Put results in `<skill-name>-workspace/` as a sibling to the skill directory, in exactly this layout:

```
<skill-name>-workspace/iteration-1/
└── eval-0-chart-axis-labels/
    ├── eval_metadata.json
    ├── with_skill/run-1/{outputs/, timing.json, grading.json}
    └── baseline/run-1/{outputs/, timing.json, grading.json}
```

Two naming rules are load-bearing, and getting either wrong fails silently rather than erroring:

- **Eval directories must start with `eval-<N>-`.** `aggregate_benchmark.py` globs `eval-*`; a directory named `chart-axis-labels` is invisible to it and you'll get an empty benchmark with no warning. Put the descriptive name after the number — `eval-0-chart-axis-labels` — so it's both discoverable and readable in the viewer.
- **Each configuration needs a `run-N` subdirectory.** The aggregator skips any configuration directory with no `run-*` child. Even for a single run, write `run-1/`.

Configuration names are discovered dynamically, so `with_skill`, `baseline`, and `old_skill` all work. Outputs go in `run-1/outputs/`; `grading.json` and `timing.json` sit beside it in `run-1/`. Don't create all of this upfront — make directories as you go.

### Step 1: Decide the baseline with the user

Ask before spawning anything, because it changes both the run count and what the numbers mean:

- **Subject only.** One Droid per test case, using the skill. Fastest, and you get to watch each run properly. Good when the user wants to see how the skill behaves, or is iterating quickly on obvious problems. No pass-rate comparison — the grades are absolute, not relative.
- **With a baseline.** Two Droids per test case. Pick the baseline that answers the user's actual question:
  - *"Does this skill earn its place?"* → no skill at all. Same prompt, no skill reference, outputs to `baseline/run-1/outputs/`.
  - *"Is my revision better than what I had?"* → the previous version. Snapshot the skill before editing (`cp -r <skill-path> <workspace>/skill-snapshot/`), point the baseline subject at the snapshot, outputs to `old_skill/run-1/outputs/`.

Early on, the no-skill baseline is the most revealing thing you can run — if the skill isn't beating no-skill, no amount of wordsmithing will save it. Suggest it when the user hasn't formed a view yet.

**Concurrency follows from this choice.** Subject-only runs go serial, one tab at a time, because the reason to run subject-only is to watch. With-baseline runs spawn as a pair in parallel so both arms hit the same machine conditions — but still one eval's pair at a time, not all evals at once.

Write an `eval_metadata.json` per test case, at the eval directory root so both configurations share it (assertions can be empty for now). Name each eval for what it tests — that name is the section header in the viewer. If this iteration changed any eval prompts, write these files fresh; don't assume they carry over.

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### Step 2: Spawn the subject and watch it work

Per `references/orca-runner.md`: create a tab running `droid`, wait for `tui-idle`, send the prompt, then poll `terminal read` every 20-30 seconds while it works.

The with-skill prompt invokes the skill by slash command, with the eval prompt following it verbatim:

```
/<skill-name> <the eval prompt>
```

Confirm the subject prints `● Skill "<skill-name>" activated` before you judge anything else. Without that line the skill never loaded, and whatever follows says nothing about the skill. The baseline arm sends the same prompt without the slash command.

Watching is the point. A subagent hands you a summary of its own performance; a visible tab lets you and the user see what actually happened — which is where the interesting findings live. Look for the subject rewriting a script the skill already bundles, ignoring a step the skill spells out, or thrashing between approaches where the instructions are ambiguous.

**If a run is going badly wrong**, don't let it burn to completion. Close the tab, note what you saw, fix the skill, discard the partial run, and restart with a brand-new tab and a brand-new Droid — that's what forces the edited skill to be re-read from disk. A reused session keeps the stale version in context and will quietly report results for the old skill. The full procedure is in the runner reference.

Keep a light touch, though. Every intervention is you making a judgment instead of collecting data, and a skill that only worked because you kept rescuing it hasn't been shown to work. A run that's merely slower or messier than you'd like should finish — that mess is a finding. Intervene when the attempt has become worthless, not when it's imperfect.

Tell the user when you intervene and why. From their side a tab just disappeared.

### Step 3: While runs are in progress, draft assertions

Polling leaves real gaps — use them for assertion work rather than idling. Draft quantitative assertions for each test case and explain them to the user. If assertions already exist in `evals/evals.json`, review them and explain what they check.

Good assertions are objectively verifiable and have descriptive names — they should read clearly in the benchmark viewer so someone glancing at the results immediately understands what each one checks. Two failure modes to watch for:

- **Trivially satisfiable.** "Output is a PDF file" passes whether or not the skill did anything useful. If an assertion would pass in the no-skill baseline too, it isn't measuring the skill.
- **Overfit to one prompt.** An assertion that hardcodes an exact string from one specific expected output will make the skill get tuned toward that string rather than toward doing the task well.

Watching the run often suggests assertions you wouldn't have thought of from the prompt alone — if you see the subject skip a bundled script, "the run invoked `scripts/build_chart.py`" becomes an obvious thing to check.

Update the `eval_metadata.json` files and `evals/evals.json` with the assertions once drafted. Also tell the user what they'll see in the viewer — both the qualitative outputs and the quantitative benchmark.

### Step 4: Capture timing as each run finishes

There's no completion notification for a terminal run, so record wall clock yourself: timestamp before you send the prompt, timestamp when the run goes idle, and write `timing.json` in the run directory:

```json
{
  "duration_ms": 412000,
  "total_duration_seconds": 412.0
}
```

Omit `total_tokens` rather than guessing — the aggregator falls back to an output-size proxy, and a fabricated number looks authoritative in the benchmark table. Confirm the run actually finished by checking the expected output files exist, since an idle TUI can also mean the subject is stuck on a permission prompt.

Then close the subject's tab, and only the tabs you created.

### Step 5: Grade, aggregate, and launch the viewer

Once all runs are done:

1. **Grade each run** — spawn a grader subagent (or grade inline) that reads `agents/grader.md` and evaluates each assertion against the outputs. Save results to `grading.json` in each run directory, alongside `outputs/`. Grading is fine to do with an ordinary subagent — the reason the subject needed its own Droid was to get honest skill loading, which doesn't apply to a grader reading files after the fact. The grading.json expectations array must use the fields `text`, `passed`, and `evidence` (not `name`/`met`/`details` or other variants) — the viewer depends on these exact field names. For assertions that can be checked programmatically, write and run a script rather than eyeballing it — scripts are faster, more reliable, and reusable across iterations.

2. **Aggregate into benchmark** — run the aggregation script from the skill-eval directory:
   ```bash
   python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
   ```
   This produces `benchmark.json` and `benchmark.md` with pass_rate, time, and tokens for each configuration, with mean ± stddev and the delta. It discovers configuration directories by name, so `with_skill` / `baseline` / `old_skill` all work — but each one needs its `run-*` subdirectory or it gets skipped silently and you'll get an empty benchmark. If generating benchmark.json manually, see `references/schemas.md` for the exact schema the viewer expects. Put each with_skill version before its baseline counterpart.

   For subject-only runs there's no comparison to make, so the delta is meaningless — report the absolute pass rate and say plainly that it's ungrounded without a baseline.

3. **Do an analyst pass** — read the benchmark data and surface patterns the aggregate stats hide. See `agents/analyzer.md` (the "Analyzing Benchmark Results" section) for what to look for — assertions that pass regardless of skill (non-discriminating), high-variance evals (possibly flaky), and time/token tradeoffs.

4. **Launch the viewer** with both qualitative outputs and quantitative data:
   ```bash
   nohup python <skill-eval-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
   ```
   For iteration 2+, also pass `--previous-workspace <workspace>/iteration-<N-1>`.

   If the browser doesn't open, pass `--static <output_path>` to write a standalone HTML file and give the user the path. In that mode "Submit All Reviews" downloads `feedback.json` rather than posting to the server, so copy it into the workspace directory yourself before the next iteration reads it.

Note: use generate_review.py to create the viewer; there's no need to write custom HTML.

5. **Tell the user** something like: "I've opened the results in your browser. There are two tabs — 'Outputs' lets you click through each test case and leave feedback, 'Benchmark' shows the quantitative comparison. When you're done, come back here and let me know."

### What the user sees in the viewer

The "Outputs" tab shows one test case at a time:
- **Prompt**: the task that was given
- **Output**: the files the skill produced, rendered inline where possible
- **Previous Output** (iteration 2+): collapsed section showing last iteration's output
- **Formal Grades** (if grading was run): collapsed section showing assertion pass/fail
- **Feedback**: a textbox that auto-saves as they type
- **Previous Feedback** (iteration 2+): their comments from last time, shown below the textbox

The "Benchmark" tab shows the stats summary: pass rates, timing, and token usage for each configuration, with per-eval breakdowns and analyst observations.

Navigation is via prev/next buttons or arrow keys. When done, they click "Submit All Reviews" which saves all feedback to `feedback.json`.

### Step 6: Read the feedback

When the user tells you they're done, read `feedback.json`:

```json
{
  "reviews": [
    {"run_id": "eval-0-with_skill", "feedback": "the chart is missing axis labels", "timestamp": "..."},
    {"run_id": "eval-1-with_skill", "feedback": "", "timestamp": "..."},
    {"run_id": "eval-2-with_skill", "feedback": "perfect, love this", "timestamp": "..."}
  ],
  "status": "complete"
}
```

Empty feedback means the user thought it was fine. Focus your improvements on the test cases where they had specific complaints.

Kill the viewer server when you're done with it:

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## Iterating on the skill

This is the point of the whole exercise. The evals ran, the user reviewed the results, and now you make the skill better based on what you learned.

### How to think about improvements

1. **Generalize from the feedback.** The skill you're improving will be used many, many times across prompts you'll never see. You and the user are iterating on a handful of examples because it's fast — the user knows those examples cold and can assess new outputs quickly. But a skill that works only on those examples is useless. Resist fiddly overfitty changes and oppressively constrictive MUSTs. If some issue is stubborn, try branching out: a different metaphor, a different recommended workflow. It's cheap to try and you might land on something great.

2. **Keep the prompt lean.** Remove things that aren't pulling their weight. Read the transcripts, not just the final outputs — if the skill is making the model waste time on unproductive detours, try deleting the parts causing that and see what happens. Deletions are experiments too, and the benchmark will tell you if you cut something load-bearing.

3. **Explain the why.** Try hard to explain the **why** behind everything the skill asks the model to do. Today's LLMs are *smart*. They have good theory of mind and with a good harness they go beyond rote instructions. Even if the user's feedback is terse or frustrated, work out what they actually want and why, then transmit that understanding into the instructions. If you find yourself writing ALWAYS or NEVER in all caps, or reaching for rigid structure, that's a yellow flag — reframe and explain the reasoning instead. It's more humane and it works better.

4. **Look for repeated work across test cases.** You watched these runs — use that. If every subject independently wrote a `create_docx.py` or a `build_chart.py`, or took the same undocumented multi-step path, that's a strong signal the skill should bundle it. Write it once, put it in `scripts/`, and point the skill at it. Every future invocation stops reinventing the wheel.

5. **Let the benchmark inform, not dictate.** A pass rate that goes up while the user's qualitative reaction goes down means the assertions are wrong, not the skill. When those two signals disagree, trust the human and fix the assertions.

This work is important (we are trying to create billions a year in economic value here!) and your thinking time is not the blocker; take your time and really mull it over. Write a draft revision, then look at it with fresh eyes and improve it. Get into the head of the user and understand what they want and need.

### The iteration loop

After improving the skill:

1. Apply your improvements to the skill
2. Rerun all test cases into a new `iteration-<N+1>/` directory, with freshly spawned Droids in fresh tabs — never a tab left over from the previous iteration, which would still hold the old skill in context. If the question is "does this skill help at all," the no-skill baseline stays constant across iterations. If the question is "is this revision better," use your judgment: the original version the user came in with, or the previous iteration.
3. Launch the reviewer with `--previous-workspace` pointing at the previous iteration
4. Wait for the user to review and tell you they're done
5. Read the new feedback, improve again, repeat

Keep going until:
- The user says they're happy
- The feedback is all empty (everything looks good)
- You're not making meaningful progress

Then consider expanding the test set and running once more at larger scale, to check that the gains weren't specific to the handful of prompts you've been staring at.

---

## Advanced: Blind comparison

When you want a more rigorous verdict between two versions (e.g., the user asks "is the new version actually better?"), there's a blind comparison system. Read `agents/comparator.md` and `agents/analyzer.md` for details. The idea: give two outputs to an independent agent without telling it which is which, let it judge quality, then analyze why the winner won. The blindness matters — an agent that knows which output came from "the new version" will find reasons to prefer it.

This is optional, requires subagents, and most users won't need it. The human review loop is usually sufficient.

---

## Orca is required

Check Orca before you build an eval set, not after:

```bash
ORCA status --json
```

If `result.app.running` is false, start it with `ORCA open --json` and continue.

If Orca can't be started, or isn't installed, **stop and tell the user.** Don't fall back to subagents, and don't run the test prompts yourself.

The reason is that the fallbacks aren't cheaper versions of this eval — they measure something else and report it in the same format. Running the prompt yourself is the worst case: you've read the skill, you may have written the change you're testing, so ambiguities that would derail a real user get silently smoothed over by context the subject wouldn't have. That produces a pass rate that looks like evidence and isn't. A user who trusts it ships a skill that doesn't work.

So say plainly that the eval needs Orca, that it isn't available, and what you'd need to proceed. Offer to help install or start it. Everything up to the run — reading the skill, agreeing on what good looks like, drafting prompts and assertions — is still useful work you can do in the meantime, and it means the eval is ready to go the moment Orca is.

**Editing a read-only skill**: the installed skill path may not be writeable. Copy it into a writeable skills directory (`~/.agents/skills/<skill-name>/`), edit there, and evaluate that copy. Keep the original directory name and `name` frontmatter field — the slash command follows the directory name, so renaming it to `skill-name-v2` changes how you invoke it and stops matching what the user actually ships.

---

## Reference files

The agents/ directory contains instructions for specialized subagents. Read them when you need to spawn the relevant subagent.

- `agents/grader.md` — How to evaluate assertions against outputs
- `agents/comparator.md` — How to do blind A/B comparison between two outputs
- `agents/analyzer.md` — How to analyze why one version beat another

The references/ directory has additional documentation:
- `references/orca-runner.md` — Verified Orca commands for spawning, watching, and intervening on Droid subject agents. Read this before running evals.
- `references/schemas.md` — JSON structures for evals.json, grading.json, benchmark.json, etc.

---

Repeating the core loop one more time for emphasis:

- Read the skill under evaluation and agree with the user on what good looks like
- Build the eval set (prompts, then assertions)
- Agree on the baseline: subject-only, or with a no-skill/previous-version comparison
- Spawn a Droid per run in its own Orca tab, invoke the skill with `/<skill-name> <prompt>`, and watch it work; intervene and restart with a fresh tab if a run goes badly wrong
- With the user, evaluate the outputs:
  - Create benchmark.json and run `eval-viewer/generate_review.py` so the user can review them
  - Walk through the quantitative results
- Improve the skill, rerun with freshly spawned Droids, compare against the previous iteration
- Repeat until you and the user are satisfied, then expand the test set and confirm the gains hold

Please add steps to your TodoList, if you have such a thing, so you don't forget. Specifically include "Create evals JSON and run `eval-viewer/generate_review.py` so human can review test cases" — that's the step most likely to get skipped.

Good luck!
