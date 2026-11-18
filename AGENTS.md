# Repository Guide

## Environment

- Run commands from the repository root. Generated Gin configs and CLI defaults use repository-relative paths.
- Use Python 3.12 exactly: `uv sync --python 3.12 --extra dev`. Although `pyproject.toml` currently allows newer Python, Numba 0.60/llvmlite 0.43 fails to install on Python 3.14.
- Keep the runtime pins in `pyproject.toml` aligned with Kaggle. In particular, do not upgrade NumPy independently of Numba; the working pair is NumPy 2.0.2 and Numba 0.60.0.

## Architecture

- `smart_buildings/evaluate.py` is the runtime entrypoint: it clears/parses one scenario's Gin config, constructs a fresh environment and policy, derives building metadata, and runs the rollout.
- `smart_buildings/baselines/transfer_policy.py` is the participant-owned policy template and its module docstring is the interface contract. Add or rename runnable policies in `POLICY_MAP` in `evaluate.py`.
- `smart_buildings/smart_control/` contains the simulator. `buildings/` and `weather/` are generator inputs; `training_set.yaml` defines the 30 shipped pairs; `training_scenarios/` contains their generated artifacts.
- Preserve the apparently unused registration imports in `evaluate.py`: Gin needs their import side effects before parsing generated configs.

## Policy Constraints

- Action and observation widths vary by building. For `K` AHUs and `V` zones, actions are discrete `(K+1,)` and continuous `(3K+2+2V,)`; return `int32` and `float32` arrays with exact shapes.
- Never derive continuous offsets. The layout is interleaved; scatter through metadata keys such as `temp_indices`, `pressure_indices`, `recirculation_indices`, `water_indices`, and `zone_action_indices`.
- `ahu_zone_assignments` uses `room_N`, while `zone_action_indices` uses `VAV_N`. Continuous zero is the midpoint of each actuator range, not an off/closed command.
- AHUs cool but do not heat; heating comes from VAV reheat coils and the hot-water loop. Simulator-enforced ventilation floors can make observed positions differ from commands.
- `initialize_policy` catches any constructor `TypeError` and retries a simpler signature. A `TypeError` inside a valid two-argument constructor can therefore appear as a misleading constructor fallback.

## Verification

- Full tests: `uv run pytest -q`. They take about two minutes and some require the checked-in 30-scenario tree.
- Focus one test: `uv run pytest path/to/file_test.py::TestClass::test_name -q`.
- Type check: `uvx ty check smart_buildings`. This is not currently a clean gate (87 existing diagnostics, including optional `mediapy`/`IPython` imports); distinguish touched-file regressions from the baseline.
- There is no configured CI or canonical lint/format command. Ruff, Pyink, isort, pylint, and mdformat are dependencies, but do not claim guessed invocations are repository gates.
- After scenario/reward-scaling changes, run `uv run python check_scales.py`; set `SCEN_DIR` to check a generated alternate scenario tree.

## Evaluation And Scoring

- Fast policy smoke test: `uv run python -m smart_buildings.evaluate --policies transfer --buildings building_train_1 --weathers fairbanks_1 --steps 288`.
- A scoreable episode is 8,640 steps. Never compare a short smoke-test return with shipped baselines; a short run is systematically less negative.
- Score a full results tree with `uv run python score_leaderboard.py <results-dir> --policy transfer --expect 30`. Check the printed scenario count: `--expect` currently prints `INCOMPLETE` but still exits successfully.
- Evaluation outputs under `smart_buildings/results/<building>/<weather>/<policy>/` are large, reproducible, and ignored. Do not add them to source control.

## Generated Files

- Regenerate scenario artifacts rather than hand-editing `training_scenarios/**/{config.gin,config.yaml,floor_plan.npy,zone_map.npy}`. Running `python -m smart_buildings.run_scenario_generator` with no filters rebuilds manifest pairs; explicit buildings/weathers generate their cross product.
- Do not edit or format `smart_buildings/smart_control/proto/*_pb2.py` or `*_pb2.pyi`. When a `.proto` changes, regenerate the stubs with the command documented in `pyproject.toml`.
