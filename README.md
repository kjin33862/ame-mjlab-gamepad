# ame-mjlab-gamepad

Drive a trained [ame_mjlab](https://github.com/psrobotics/ame_mjlab) Unitree G1
velocity policy live with a gamepad, in mjlab's play viewer.

This is a small add-on that runs **on top of** Shuang's
[ame_mjlab](https://github.com/psrobotics/ame_mjlab) (`psrobotics/ame_mjlab`).
It is **not part of ame_mjlab** and is not maintained by its authors. It
imports the `ame` package from an ame_mjlab checkout and copies none of its
code.

`play_gamepad.py` opens the play environment of a task, loads your checkpoint
and, every control step, overwrites the twist command (`vx, vy, wz`) of the
viewed robot with the gamepad sticks. The policy sees that command and the
viewer's command arrows show it. Press B to switch back to the task's own
random commands at any time.

It is a single file with no extra dependencies. It runs inside ame_mjlab's
environment through `uv run --project`.

## Requirements

- **Linux only.** The pad is read from the Linux joystick API
  (`/dev/input/js*`) using only the Python standard library.
- **ame_mjlab** installed as described in its own README:
  <https://github.com/psrobotics/ame_mjlab>. Its requirements apply too
  (NVIDIA GPU, CUDA 12.8 driver).
  - The default mode works with upstream `psrobotics/ame_mjlab`.
  - `--terrain` / `--difficulty` also need the `ame.eval` package, which
    exists only in the `eval` branch of my fork,
    <https://github.com/kjin33862/ame_mjlab/tree/eval>.
- **[uv](https://docs.astral.sh/uv/getting-started/installation/)**, which
  ame_mjlab already uses.
- **An Xbox controller** (USB or wireless) or any other pad the Linux
  joystick driver exposes with standard stick and A/B codes.
- **A trained checkpoint** (`model_<iter>.pt`). ame_mjlab ships none; train
  one with `uv run train <task>` in ame_mjlab.

Tested with mjlab commit
[`898a700`](https://github.com/mujocolab/mjlab/commit/898a700514f8d3c92146ec158190e6f2e4d8c11a),
the version ame_mjlab pins. The script hooks into mjlab's viewer and
velocity-command internals, so a different mjlab version may break it.

## Setup

```bash
# 1. ame_mjlab, per its README (here at ~/ame_mjlab)
git clone https://github.com/psrobotics/ame_mjlab ~/ame_mjlab
cd ~/ame_mjlab && uv sync && cd -
#    For --terrain, use the fork's eval branch instead:
#    git clone -b eval https://github.com/kjin33862/ame_mjlab ~/ame_mjlab

# 2. this repo, anywhere
git clone https://github.com/kjin33862/ame-mjlab-gamepad
cd ame-mjlab-gamepad

# 3. plug in the controller, check it shows up
ls /dev/input/js*

# 4. run
uv run --project ~/ame_mjlab python play_gamepad.py \
  --task Ame-Velocity-Rough-G1 \
  --checkpoint-file ~/ame_mjlab/logs/rsl_rl/ame_g1/<run-dir>/model_<iter>.pt
```

`--project ~/ame_mjlab` runs the script with ame_mjlab's environment
(mjlab, torch and the registered `ame` tasks) without installing anything
into this folder.

## Two modes

**Task play board** (no `--terrain`). The task's own play environment, the
same one `uv run play <task>` opens, with one robot. Pressing B gives back
that task's random commands.

**Single evaluation tile** (`--terrain NAME --difficulty X`). One 1 × 1
evaluation tile of a single terrain family at a fixed difficulty, as
`scripts/view_eval.py` in the fork builds it. The robot spawns at the tile
centre with yaw 0. Map and proprioceptive noise are off, there is no episode
time limit, and B switches to the gap task's training twist commands. The
actor's observation set (with or without `base_lin_vel`) is read from the
checkpoint. This mode needs `ame.eval` from the fork's `eval` branch (see
Requirements).

Valid `--terrain` names:

| name | terrain |
|---|---|
| `flat` | flat ground |
| `pyramid_stairs` | pyramid stairs, climbing up |
| `pyramid_stairs_inv` | inverted pyramid stairs, climbing down |
| `random_stairs` | stairs with random step heights |
| `open_stairs` | mjlab's open stairs (`BoxOpenStairsTerrainCfg`) |
| `random_grid` | random-height grid of blocks |
| `discrete_obstacles` | scattered box obstacles |
| `random_rough` | random rough ground |
| `hf_pyramid_slope` | pyramid slope, up |
| `hf_pyramid_slope_inv` | inverted pyramid slope, down |
| `wave` | wave-shaped ground |
| `stepping_stones` | stepping stones with a centre platform |
| `stone_lattice` | the gap task's stone lattice (gap = 0.45 × difficulty) |

All but `stone_lattice` are the families of the AME training mix
(`ame.eval.EVAL_TERRAINS`).

## Controller mapping

| control | action |
|---|---|
| left stick up / down | `lin_vel_x`: forward / backward |
| left stick left / right | `lin_vel_y`: sideways (left = +y) |
| right stick left / right | `ang_vel_z`: turn (left = counter-clockwise) |
| **A** | reset the robot |
| **B** | toggle gamepad ↔ the task's random commands |

Each axis is scaled to the task's training command range. A fully deflected
stick gives the range's end (up = max `vx`, down = min `vx`), centre gives 0,
and a dead zone (default 10 %) absorbs stick drift. On startup the script
prints the ranges in use, e.g.

```
[gamepad] /dev/input/js2; ranges vx (-0.3, 1.5), vy (-0.5, 0.5), wz (-1.0, 1.0), dead zone 0.1
```

If the pad is unplugged, the commands drop to zero and the script reconnects
when it comes back.

## Command-line options

| option | default | meaning |
|---|---|---|
| `--task TASK` | *(required)* | Task id, e.g. `Ame-Velocity-Rough-G1` (`uv run list-envs` in ame_mjlab lists them). |
| `--checkpoint-file PATH` | *(required)* | Checkpoint to load (`~` is expanded). |
| `--viewer {auto,native,viser}` | `auto` | `native` = MuJoCo window, `viser` = browser viewer. `auto` picks `native` when `DISPLAY` or `WAYLAND_DISPLAY` is set, else `viser`. |
| `--terrain NAME` | none | One evaluation tile of that terrain family instead of the task's play board (see [Two modes](#two-modes) for the names). **Needs the fork's `eval` branch.** |
| `--difficulty X` | `0.0` | Terrain difficulty in [0, 1], used with `--terrain`. |
| `--js-device PATH` | first gamepad found | Joystick device, e.g. `/dev/input/js0`. |
| `--dead-zone X` | `0.1` | Stick dead zone as a fraction of full deflection. |
| `--max-vx V` | training max | `vx` [m/s] at stick fully up (≥ 0). |
| `--min-vx V` | training min | `vx` [m/s] at stick fully down (≤ 0). |
| `--max-vy V` | training | `vy` [m/s] at stick fully left; fully right is `-V` (≥ 0). |
| `--max-wz W` | training | `wz` [rad/s] at right stick fully left; fully right is `-W` (≥ 0). |
| `--device DEV` | `cuda:0` if available, else `cpu` | Torch device. |

A `--max-*` / `--min-vx` value outside the training range is allowed, but
the script prints a warning: the policy never saw those commands and may not
follow them, or may fall.

### Examples

```bash
P="uv run --project ~/ame_mjlab python play_gamepad.py"
CKPT=path/to/model_9999.pt

# Basic: the task's play board, native window if you have a display
$P --task Ame-Velocity-Rough-G1 --checkpoint-file $CKPT

# The gap task's play board
$P --task Ame-Mjlab-Gap-Velocity-Rough-G1 --checkpoint-file $CKPT

# Browser viewer (e.g. over SSH; forward the port it prints)
$P --task Ame-Velocity-Rough-G1 --checkpoint-file $CKPT --viewer viser

# A specific controller and a bigger dead zone for a drifting stick
$P --task Ame-Velocity-Rough-G1 --checkpoint-file $CKPT --js-device /dev/input/js1 --dead-zone 0.2

# Slower, gentler driving: cap forward speed, no backward
$P --task Ame-Velocity-Rough-G1 --checkpoint-file $CKPT --max-vx 0.8 --min-vx 0

# Beyond the training range (prints a warning)
$P --task Ame-Velocity-Rough-G1 --checkpoint-file $CKPT --max-wz 2.0

# One evaluation tile: descending pyramid stairs at difficulty 0.5 (fork's eval branch)
$P --task Ame-Mjlab-Gap-Velocity-Rough-G1 --checkpoint-file $CKPT \
  --terrain pyramid_stairs_inv --difficulty 0.5

# The gap task's stone lattice at difficulty 0.3
$P --task Ame-Mjlab-Gap-Velocity-Rough-G1 --checkpoint-file $CKPT \
  --terrain stone_lattice --difficulty 0.3

# Run on the CPU
$P --task Ame-Velocity-Rough-G1 --checkpoint-file $CKPT --device cpu
```

## Known harmless warnings and output

- **`GLFWError: (65548) b'Wayland: The platform does not provide the window
  position'`**. Printed once when the native window opens on a Wayland
  desktop. GLFW cannot query window positions on Wayland. The viewer works
  normally.
- **Long manager tables and the actor/critic model printout at startup**.
  This is mjlab's and rsl_rl's normal logging, not an error.
- **`[gamepad] connected: /dev/input/jsN` before everything else**. The pad
  is opened before the environment is built, so this comes first.
- **The startup `[gamepad] ...` lines appear only at exit when output is
  redirected to a file**. Python buffers stdout when it is not a terminal.
  Use `python -u` if you are logging to a file.

## Troubleshooting

**`no gamepad found under /dev/input/js*; pass --js-device`**
The script waits up to 30 s for a pad, then gives up. Check that
`ls /dev/input/js*` lists it. If it doesn't:
- Wired Xbox pads use the in-kernel `xpad` driver (`sudo modprobe xpad`).
- Xbox Wireless over Bluetooth works best with
  [xpadneo](https://github.com/atar-axis/xpadneo). The Xbox Wireless USB
  adapter needs [xone](https://github.com/medusalix/xone).
- Keyboards and mice can also register as `js*` devices. The script skips
  anything without two sticks and an A button. If it still picks the wrong
  device, pass `--js-device`.

**`Permission denied` on `/dev/input/js*`**
Check `ls -l /dev/input/js*`. Add yourself to the owning group (usually
`input`), then log out and back in:
`sudo usermod -aG input $USER`.

**The sticks do nothing**
- Check that the mode is `gamepad`, not `random commands` (press B; the
  terminal prints the mode).
- Test the pad on its own with `jstest /dev/input/js0` (package `joystick`).
- A different pad may report other axis codes. The script reads the left
  stick as `ABS_X`/`ABS_Y` and the right stick as `ABS_RX`.

**The robot drifts with the sticks centred**
Raise `--dead-zone`, e.g. `0.15`–`0.2`.

**`ModuleNotFoundError: No module named 'ame.eval'`**
You passed `--terrain` with upstream ame_mjlab. Use the fork's `eval`
branch (see Requirements), or drop `--terrain`.

**`ModuleNotFoundError: No module named 'mjlab'` / `'ame'`**
`--project` must point at the ame_mjlab checkout and it must have been
`uv sync`ed. Run `uv run --project ~/ame_mjlab list-envs` to check.

**`checkpoint not found`, or an error loading the state dict**
Check the path. The checkpoint must come from a run of the same `--task`,
because loading is strict.

**No window appears**
Without a display, `auto` uses viser; open the URL it prints. For the
Wayland `GLFWError` see [Known harmless warnings](#known-harmless-warnings-and-output).

**`AttributeError` inside the viewer or the command term**
Your mjlab is probably not the tested commit `898a700`. Use the version
pinned by ame_mjlab's `uv.lock`.

## Credits

The runner, checkpoint-loading and viewer-selection code follows mjlab's
[`scripts/play.py`](https://github.com/mujocolab/mjlab/blob/898a700514f8d3c92146ec158190e6f2e4d8c11a/src/mjlab/scripts/play.py)
(Apache-2.0).

## License

MIT. See [LICENSE](LICENSE).
