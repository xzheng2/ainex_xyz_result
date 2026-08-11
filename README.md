# ainex_xyz_result

Experiment results for the AiNex BT stack. Code lives in a **separate** repository
(`ainex_xyz`, working copy `/home/pi`); this repo holds only what runs produce.

## Layout

```
results/<body_id>/<run_id>/
    run_meta.json                    independent variables — written BEFORE the run
    metrics.json                     dependent variables — written AFTER it
    bt_debug_lastrun.jsonl           BT decision log, one line per event
    bt_ros_comm_debug_lastrun.jsonl  ROS traffic log
    bb_current.json                  final blackboard snapshot
index/<body_id>.jsonl                one line per run, appended at publish time
```

The split is not cosmetic: `run_meta.json` has to be stamped before anything can drift,
and `metrics.json` cannot exist until there is something to measure.

### `metrics.json` (`ainex.run_metrics/1`)

Written by `xyz_behavior/tools/close_run.py`. Two halves, and the second is the reason a
human is in the loop at all.

*Reduced from the logs* (`bt_observability/run_metrics.py`):

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
`xyz_behavior/config/bodies/README.md` for why that and not the hostname.

**Publish before runs age out.** A body keeps only the newest 10 run directories in
`xyz_behavior/log/runs/`; the BT node deletes older ones at startup whether or not they
were published. This repository is where a run becomes permanent.

## Why the path is partitioned by body, and why there are no per-body branches

Several robot bodies push into this repo. Every body writes **only** under its own
`results/<body_id>/` and `index/<body_id>.jsonl` prefix, so two bodies can commit and
push concurrently and git will never produce a conflict — different paths cannot
conflict. That is the whole reason for the partition.

Per-body *branches* would give the same isolation and then take something valuable
away: a branch per body never merges, so results can never be compared in one checkout
and every shared change has to be cherry-picked N times. One branch, partitioned paths.

The index is sharded per body for the same reason — a single shared `index.jsonl` would
be the one file every body appends to, and therefore the one file that always conflicts.
Read it by concatenating the shards.

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

## Reading the index

`index/<body_id>.jsonl` carries one flat line per run: identity and provenance from
`run_meta.json`, plus `outcome`, `interventions`, `failure_mode`, `ticks_total`,
`first_success_tick` and `closed` from `metrics.json`. That is deliberately enough to
build an ablation table without opening a single run directory — a 3-variant × 2-body ×
5-trial campaign is 30 directories, and reaching into each one at analysis time is exactly
what this file exists to avoid. Drill into a run's own `metrics.json` for the per-node and
per-dwell detail.

Shards concatenate: `cat index/*.jsonl`.

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
