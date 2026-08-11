# ainex_xyz_results

Experiment results for the AiNex BT stack. Code lives in a **separate** repository
(`ainex_xyz`, working copy `/home/pi`); this repo holds only what runs produce.

## Layout

```
results/<body_id>/<run_id>/
    run_meta.json      what code and what body produced this run
    bt_debug.jsonl     BT tick log (round 2: written here by the logger)
    metrics.json       whatever the run measured
index/<body_id>.jsonl  one line per run, appended at publish time
```

`run_id` is `<UTC timestamp>_<variant>_t<trial>`, e.g. `20260811T2014Z_ablA_t1`.

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

`run_meta.json` records `git.dirty` — whether the code working tree had uncommitted
changes when the run started. **A run with `dirty: true` is not reproducible**: the code
that produced it does not exist under any commit. Such runs are fine for exploration and
must not be used as evidence in an ablation table.

## Publishing a run

Runs are produced on the robot under `xyz_behavior/log/runs/<run_id>/` (inside the ROS
bind mount, so the containerised BT node can write there). A host-side step copies
finished runs into this repo and appends the index:

```bash
python3 /home/pi/docker/ros_ws_src/xyz_behavior/tools/publish_runs.py
```

The robot side never needs to know this repo exists — that keeps ROS code free of git.
