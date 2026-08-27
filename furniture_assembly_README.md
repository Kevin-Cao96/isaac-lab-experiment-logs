# Furniture Assembly Benchmark (Minimal)

## Task Definition

- Initial state:
  - Tabletop fixed at (0.6, 0.0, 0.53)
  - 4 table legs scattered on the ground
- Goal state:
  - All 4 legs placed under the tabletop at target positions
- Success condition:
  - Every leg is within 0.05 m of its target position
- Episode length: 10 s timeout

## Environment

- Environment name: `furniture_assembly`
- File: `isaaclab_arena_environments/furniture_assembly_environment.py`
- Assets: `isaaclab_arena_environments/furniture/__init__.py`
  - `tabletop` (CuboidCfg)
  - `table_leg` (CylinderCfg)
  - `ground` (GroundPlaneCfg)
- Task: `isaaclab_arena/tasks/furniture_assembly_task.py`

## Evaluation

```bash
python isaaclab_arena/evaluation/policy_runner.py \
  --policy_type <policy> \
  --num_episodes 20 \
  furniture_assembly
```

## Baseline Results

| Policy | num_episodes | success_rate |
|---|---|---|
| zero_action | 20 | 0.0 |
| scripted_action | 20 | 0.0 |

## Next Steps

- Add grasping / manipulation policy
- Add teleoperation data collection
- Add Mimic data generation
- Train BC / OpenPI and evaluate
