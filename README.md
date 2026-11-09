# ame-mjlab-gamepad

Drive a trained [ame_mjlab](https://github.com/psrobotics/ame_mjlab) Unitree G1
velocity policy live with a gamepad, in mjlab's play viewer.

A single-file add-on that runs on top of Shuang's ame_mjlab
(`psrobotics/ame_mjlab`). It is not part of ame_mjlab. Each control step it
sets the robot's velocity command (`vx, vy, wz`) from the sticks.

## Setup

```bash
# ame_mjlab, per its README. For --terrain use my fork's eval branch:
git clone -b eval https://github.com/kjin33862/ame_mjlab ~/ame_mjlab
cd ~/ame_mjlab && uv sync && cd -

git clone https://github.com/kjin33862/ame-mjlab-gamepad
cd ame-mjlab-gamepad
ls /dev/input/js*   # the gamepad should show up here (Linux only)
```

Upstream `psrobotics/ame_mjlab` is enough if you don't use `--terrain`.

## Controls

| control | action |
|---|---|
| left stick up / down | forward / backward (`vx`) |
| left stick left / right | sideways (`vy`) |
| right stick left / right | turn (`wz`) |
| A | reset the robot |
| B | toggle gamepad ↔ the task's random commands |

Full stick deflection is the end of the task's training command range.

## Examples

```bash
P="uv run --project ~/ame_mjlab python play_gamepad.py"
CKPT=path/to/model_9999.pt

# The task's play board
$P --task Ame-Mjlab-Gap-Velocity-Rough-G1 --checkpoint-file $CKPT

# One evaluation tile: descending stairs at difficulty 0.5
$P --task Ame-Mjlab-Gap-Velocity-Rough-G1 --checkpoint-file $CKPT \
  --terrain pyramid_stairs_inv --difficulty 0.5

# Cap the speed, browser viewer
$P --task Ame-Mjlab-Gap-Velocity-Rough-G1 --checkpoint-file $CKPT --max-vx 0.8 --viewer viser
```

`--terrain` names: `flat`, `pyramid_stairs`, `pyramid_stairs_inv`,
`random_stairs`, `open_stairs`, `random_grid`, `discrete_obstacles`,
`random_rough`, `hf_pyramid_slope`, `hf_pyramid_slope_inv`, `wave`,
`stepping_stones`, `stone_lattice`.

Other options: `--max-vx`, `--min-vx`, `--max-vy`, `--max-wz` (stick ranges;
beyond training prints a warning), `--dead-zone` (default 0.1),
`--js-device`, `--viewer {auto,native,viser}`, `--device`. See `--help`.

On Wayland, `GLFWError: ... does not provide the window position` is harmless.

## License

MIT. See [LICENSE](LICENSE).
