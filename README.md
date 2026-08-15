# ainex_xyz_result

Experiment results for the AiNex BT stack. Code lives in a **separate** repository
(`ainex_xyz`, working copy `/home/pi`); this repo holds only what runs produce.

## Layout

```
results/<body_id>/<study>/<lane>/<run_id>/
    run_meta.json                    independent variables — written BEFORE the run
    metrics.json                     dependent variables — written AFTER it
    bt_debug_lastrun.jsonl           BT decision log, one line per event
    bt_ros_comm_debug_lastrun.jsonl  ROS traffic log
    bb_current.json                  final blackboard snapshot
index/<body_id>__<study>__<lane>.jsonl   one line per run, appended at publish time

sessions/<body_id>/<study>/<lane>/<session_id>/
    session_meta.json                which card, which CLI, which subagents
    transcript.jsonl                 distilled coding-session transcript
    session_metrics.json             tokens, tool calls, rework, clarifications
    guard_events.jsonl               this session's slice of the guard log
index/sessions/<body_id>__<study>__<lane>.jsonl   one line per session
```

The split is not cosmetic: `run_meta.json` has to be stamped before anything can drift,
and `metrics.json` cannot exist until there is something to measure.

`results/` is what the ROBOT did. `sessions/` is what the coding AGENT did getting there
— they do not correspond one to one, which is why they are published by two different
tools. The session index lives in its own subdirectory so `cat index/*.jsonl` keeps
meaning "every run" and never mixes two record kinds.

### `study` and `lane` — which experiment, which arm

`study` (`exp1`, `exp2`, …) says which experiment a row belongs to. `lane` (`a`/`b`/`c`/`d`)
says which SD card, and therefore which asset mix, produced it within that experiment.

Both are part of the partition key rather than mere fields, because **all four arms run on
the same robot**: `body_id` is identical across them, so partitioning by body alone would
put four concurrent writers on one index file — the exact conflict the sharding exists to
prevent — and leave the arms indistinguishable in the table.

`study` sits *above* `lane` because a card outlives a study. The same lane-`a` card runs
exp1 and later exp2; without the separating level those two would pool under one prefix
and every table built from it would silently average across two experiments that were
never meant to be compared.

Resolution mirrors `body_id`: `AINEX_STUDY` → `xyz_behavior/log/.study` and `AINEX_LANE` →
`log/.lane`. Set the lane once when a card is imaged and the study whenever a card starts
a new experiment:

```bash
python3 .../new_run.py --study exp1 --lane a     # then just --variant/--trial from here
python3 .../new_run.py --study exp2              # when this card moves to the next study
```

A run that cannot name both is refused, not guessed. `study` is validated as a slug rather
than against a closed list — a fixed set would mean editing the instrumentation package to
start a third experiment, and that package must stay byte-identical across all four lanes.
`new_run.py --print-study` shows what a card is currently set to.

### `metrics.json` (`ainex.run_metrics/1`)

Written by `xyz_behavior/tools/close_run.py`. Two halves, and the second is the reason a
human is in the loop at all.

*Reduced from the logs* (`expt_run_lab/run_lab/run_metrics.py`):

| field | meaning |
|---|---|
| `ticks_total`, `tick_id_first/last` | how long the run was, in ticks |
| `first_success_tick` | first tick the root returned SUCCESS |
| `root_status_counts` | how many ticks the tree spent in each status |
| `node_ticks`, `node_status_counts` | per-node tick counts — which subtree consumed the run |
| `dwells.<state_key>` | per LatchedDwell: `latch_count`, `reset_count`, `ticks_to_first_latch`, `max_stable_ticks` |
| `duration_s_approx` | file-mtime span; the logs carry no wall clock |
| `log_present` | false when a run was opened but the robot never drove it |

`reset_count` is the direct observable for a dwell-N ablation: the claim is that a larger
N suppresses false detections, and a false detection appears in the log as the dwell being
reset.

*Recorded by the operator*: `outcome` (success/failure/aborted), `interventions`,
`failure_mode`, `note`. No log holds these — a behaviour tree only knows what it itself
judged. Whether the ball went in, whether the robot fell in a pose no L1 node recognised,
how many times someone repositioned it: that is ground truth, and it is the ablation's
primary outcome.

`run_id` is `<UTC timestamp>_<variant>_t<trial>`, e.g. `20260811T201400Z_ablA_t1`.
`body_id` is the robot's WiFi access point SSID, e.g. `HW-ROBOPARKS676EF55C` — see
`expt_run_lab/config/bodies/README.md` for why that and not the hostname.

### `session_metrics.json` (`ainex.session_metrics/1`)

Written by `xyz_behavior/tools/publish_session.py`. The process half of the experiment:

| field | meaning |
|---|---|
| `tokens`, `tokens_subagent` | input / output / cache_creation / cache_read, kept apart so delegation cost is visible |
| `api_responses` | distinct API responses, not transcript lines |
| `tool_calls_total`, `tool_calls_by_name` | every `tool_use` block |
| `files_touched`, `file_path_calls` | distinct paths, and calls carrying one |
| `rework_events`, `rework_files` | repeated Edit/Write of one file *within one human turn* |
| `clarification_calls`, `clarification_questions` | `AskUserQuestion`; one call can bundle several questions |
| `human_turns`, `subagent_count`, `tool_errors`, `duration_s` | shape of the session |
| `guard` | counts by action and rule, plus `fix_on_first_retry` |

Two things that are easy to get wrong if you recompute these yourself. The CLI writes
**one transcript line per content block**, and every line of one API response repeats
that response's `usage` object verbatim — summing per line inflated output tokens by
2.56× on a measured session, so totals are deduplicated by `message.id`. And subagent
work is **not** inlined: it lives in separate `subagents/agent-*.jsonl` files, and the
spawning call returns only `{"status":"async_launched"}`, so a main-file-only total
silently undercounts every session that delegated.

`transcript.jsonl` is **distilled, not copied**: role, timestamps, ids, usage, and for
each content block only its type, tool name, `file_path` and question count. All message
text, reasoning, tool inputs and tool outputs are dropped. That is a disclosure decision
before it is a size one — this repository is public, and a raw transcript carries the
full text of every file read and the complete output of every command run. It is also a
10× size reduction. Do not "improve" it by keeping message text.

`guard_events.jsonl` records what the write-time contract guards did: `blocked`,
`warned`, `pass` (pattern matched, content clean) and `reminder`. `pass` is what makes
`fix_on_first_retry` computable — without an event for a clean write, "the agent fixed
it" and "the agent moved on to something else" are the same absence. An **empty** guard
log is a measurement, not a failure: the lanes without the skills+hooks engine have no
guards to fire, and that zero is the control.

**Publish before runs age out.** A body keeps only the newest 10 run directories in
`xyz_behavior/log/runs/`; the BT node deletes older ones at startup whether or not they
were published. This repository is where a run becomes permanent.

## Why the path is partitioned by writer, and why there are no per-writer branches

Several writers push into this repo. Every writer writes **only** under its own
`results/<body_id>/<study>/<lane>/` and `index/<body_id>__<study>__<lane>.jsonl` prefix, so two can commit
and push concurrently and git will never produce a conflict — different paths cannot
conflict. That is the whole reason for the partition.

**A writer is a body, a study *and* a lane — not a body.** The partition originally keyed
on body alone, back when one body meant one SD card. The ablation broke that: four cards,
four asset mixes, one robot. All four report the same `body_id`, so keying on body alone
would have put four concurrent writers back on one index file — the precise failure the
sharding exists to prevent. `study` joined the key for a different reason: not conflict,
but conflation. A card runs exp1 and later exp2, and those must never share a prefix.

Per-writer *branches* would give the same isolation and then take something valuable
away: a branch per writer never merges, so results can never be compared in one checkout
and every shared change has to be cherry-picked N times. One branch, partitioned paths.

The index is sharded for the same reason — a single shared `index.jsonl` would be the one
file everybody appends to, and therefore the one file that always conflicts. Read it by
concatenating the shards.

## Why results are not in the code repo

The code repo is cloned onto every body and must stay small and buildable. Results grow
without bound, are append-only, and are worthless to a `catkin build`. Keeping them
apart also means a result can name the exact code commit that produced it
(`run_meta.json.git.sha`) without the circularity of storing results inside the tree
being described.

## Reproducibility contract

`run_meta.json` records `git.dirty` — whether the code that the robot actually runs had
uncommitted changes when the run started. **A run with `dirty: true` is not reproducible**:
the code that produced it does not exist under any commit. Such runs are fine for
exploration and must not be used as evidence in an ablation table.

`dirty` is **scoped**, and the scope is recorded next to it in `git.dirty_scope`
(currently `docker/ros_ws_src` — the ROS workspace). The code repository's root is a home
directory that also holds development tooling no running node ever loads, so a change
there does not make a run irreproducible. Counting it would leave the flag permanently on,
and a warning that is always on is a warning nobody reads. `git.repo_dirty` keeps the
whole-repository state as information only, without a path list.

**A run without `metrics.json` is not evidence either.** It has no dependent variables —
nothing says what happened. Such runs are published (an exploratory run is still worth
keeping) and are marked `"closed": false` in the index; exclude them from any table, the
same way `dirty: true` runs are excluded.

### `constants_sha256` — the instrument, not the subject

`run_meta.json` also records `constants_sha256`: a digest per tree that is supposed to be
**identical across all four lanes** — `expt_run_lab` (the measuring apparatus itself),
`xyz_perception`, and the agent's `.claude/hooks` and `.claude/skills`. Two lanes whose
runs disagree on any of these were not running the same instrument, and comparing them
measures the instrument as much as the engine. Check it before building any cross-lane
table; it is the only record that survives once the working tree has moved on.

`ActionGroups` is deliberately **not** in the digest. The motion assets are hand-tuned and
are expected to change during a campaign, so hashing them would flag ordinary work as
drift — and a constant that is not actually constant trains everyone to ignore the field.

Two entries come back `null` when a run is started inside the `ainex` container: its home
is `/home/ubuntu`, so the host's `.claude/` is genuinely not visible. That is expected and
is not the same as a mismatch.

**Why a digest and not just a lock.** The lane cards carry a `permissions.deny` block that
stops the `Edit` tool from touching `expt_run_lab` and `xyz_perception`. That lock is real
but narrow: it names one tool, so `Write` reaches those paths untouched, and so does
anything going through `Bash` — `sed -i`, `python3 -c`, a redirect — as does a human at a
shell. Adding `Write` would close one of those gaps but not the `Bash` one, and no rule
can: a coding agent legitimately needs `Bash`. So the deny block is a speed bump that
keeps honest edits from happening by accident, and **`constants_sha256` is the actual
backstop** — it does not prevent a change, it makes one undeniable after the fact, in the
run's own metadata, which is the only place still trustworthy once the data is here and
the card has moved on.

## Reading the index

`index/<body_id>__<study>__<lane>.jsonl` carries one flat line per run: identity and
provenance from `run_meta.json` (including `study` and `lane`), plus `outcome`, `interventions`, `failure_mode`,
`ticks_total`, `first_success_tick` and `closed` from `metrics.json`. That is deliberately
enough to build an ablation table without opening a single run directory — a 4-lane ×
3-variant × 5-trial campaign is 60 directories, and reaching into each one at analysis
time is exactly what this file exists to avoid. Drill into a run's own `metrics.json` for
the per-node and per-dwell detail.

Always filter by `study` before aggregating. Concatenating every shard mixes experiments;
the shard filename carries the study precisely so `cat index/*__exp1__*.jsonl` is the
natural way to read one.

`index/sessions/<body_id>__<study>__<lane>.jsonl` does the same for coding sessions: one
flat line carrying tokens, tool calls, rework, clarifications and the guard counts. Filter
by `study`, group by `lane`, and you have the process half of the ablation without opening
a session directory either.

Shards concatenate: `cat index/*.jsonl` for runs, `cat index/sessions/*.jsonl` for
sessions. The two are deliberately not in the same glob — they are different record
kinds and a single `cat` over both would produce a table with half its columns null.

## Publishing a run

Runs are produced on the robot under `xyz_behavior/log/runs/<run_id>/` (inside the ROS
bind mount, so the containerised BT node can write there). Close the run, then publish:

```bash
python3 .../xyz_behavior/tools/close_run.py --latest --outcome success
python3 .../xyz_behavior/tools/publish_runs.py --commit
```

Close first — publishing an unclosed run copies it without an outcome, and re-publishing
later needs `--force`.

The robot side never needs to know this repo exists — that keeps ROS code free of git.

## Publishing a coding session

Sessions are published separately, at the end of a working session rather than the end of
a run — one session can open several runs, a run can span sessions, and most sessions open
none:

```bash
python3 .../xyz_behavior/tools/publish_session.py --commit          # the live session
python3 .../xyz_behavior/tools/publish_session.py --all             # every session
```

Run it on the **host**: the transcripts are the host's, and nothing is mounted into a
container. Sessions are pruned by the CLI over time, so export before that happens — an
un-exported transcript dies with the SD card and none of it can be reconstructed.
