# Lab 6: Odometry localization, trajectory "78"
The robot draws the last two digits of my student ID, **7** and **8**, in the TRIK Studio simulator and on the real TRIK robot. While it drives it logs its odometry. The two logs are then compared in Colab.

The robot is driven **only with Motors Forward, Stop Motors and Timer blocks**: no waypoints, no feedback, no steering. A second thread logs odometry, which Lab 6 requires.

## Files

| File | What it is |
|---|---|
| `timed_sim.qrs` | Program for the TRIK Studio 2D simulator. Writes `odometry_data_sim.txt`. |
| `timed_real.qrs` | Program for the real TRIK robot (same sequence of moves). Writes `odometry_data_real.txt`. |
| `odometry_data_sim.txt`, `odometry_data_real.txt` | Raw logs: `x,y,time` (m, m, ms), no header. |
| `Lab6_odometry_robotics.ipynb` | Colab notebook: both trajectories, error over time, end-point check. |

## The trajectory

The start pose is the origin. The robot faces +x, and +y is to its left. Coordinates below are what the simulator program draws, calculated from its powers and timers.

```
 (0,0) ──────────── (0.41,0)        ╭──╮   upper loop, centre (0.71,-0.15)
                      ╱             │    │
                    ╱                ╰──╯   ← (0.71,-0.29) start / end of the 8
                  ╱                 ╭──╮
                ╱                   │    │  lower loop, centre (0.71,-0.44)
   (0.08,-0.57)                      ╰──╯
```

1. **The 7:**
   - top bar, 0.41 m;
   - turn right 120°;
   - diagonal, 0.66 m.
   - Then stop, beep, orange LED and a 1 s pause. The simulator also lifts the marker.
2. **Move to the 8:**
   - turn left 144°;
   - drive 0.69 m to the middle of the 8;
   - turn right 24°, to face +x again.
   - The simulator puts the marker down.
3. **The 8:**
   - upper loop, radius ≈ 0.15 m, counter-clockwise;
   - lower loop, radius ≈ 0.15 m, clockwise.
   - Stop, beep, red LED, end.

## How the program is built

Both files have the same structure.

**Setup**
1. **Start.**
2. **Clear Encoder E3, E4.** The odometry starts from 0.
3. **Remove File.** Deletes the old log, so a new run isn't appended to it.
4. **Expression:** robot size `r`, `B` and the odometry start values (`x`, `y`, `phi`, `path`, `prev_path`, `prev_phi`, `t0`).
5. **Fork:** splits the program into two threads, `main` and `log`.

**Thread `main`: driving**

Each move is three blocks:

| Block | Purpose |
|---|---|
| Motors Forward **M3** | power of the left wheel |
| Motors Forward **M4** | power of the right wheel |
| **Timer** | how long the move lasts; then the next move starts directly |

The simulator file starts with Marker Down and an LED (green). The real file starts with the LED only.

After the 7: Stop Motors (brake) → Play Tone (1000 Hz, 300 ms) → Marker Up (simulator only) → LED orange → Timer 1000 ms → LED green.

Before the loops of the 8: Marker Down (simulator only).

At the end: Stop Motors (brake) → Play Tone → Timer 200 ms (so the last pose gets logged) → **Kill Thread `log`** → LED red → End.

**Thread `log`: odometry every 20 ms**

**Expression → Write To File → Timer 20 ms**, repeated:

```
theta_l = encoder3 * pi / 180;        theta_r = encoder4 * pi / 180;     -- wheel angles, rad
path = (theta_r + theta_l) * r / 2;   phi = (theta_r - theta_l) * r / B; -- distance, heading
x = x + Δpath * cos((phi + prev_phi) / 2);                               -- mean heading of the step
y = y + Δpath * sin((phi + prev_phi) / 2);
s = x .. "," .. y .. "," .. (time() - t0) .. "\n";                       -- one log line
```

Left wheel = M3/E3, right wheel = M4/E4.

## Robot parameters

| | Simulator (`timed_sim.qrs`) | Real robot (`timed_real.qrs`) |
|---|---|---|
| Wheel radius `r` | 0.028 m (2D-model wheel, 5.6 cm diameter) | **0.0375 m (measured)** |
| Wheel base `B` | 0.154 m (2D-model wheels 44 px apart) | **0.194 m (measured)** |
| Log file | `C:\Users\user\Desktop\odometry_data_sim.txt` | `odometry_data_real.txt` (on the robot) |

## Motor powers and timers

| # | Move | Sim: M3 / M4, timer | Real: M3 / M4, timer |
|---|---|---|---|
| 1 | 7: top bar | 56 / 56 %, 2700 ms | 35 / 35 %, 2700 ms |
| 2 | 7: turn right | 40 / −40 %, 1500 ms | 30 / −30 %, 1600 ms |
| 3 | 7: diagonal | 56 / 56 %, 4400 ms | 35 / 35 %, 5000 ms |
| | stop, beep, pause 1 s | | |
| 4 | turn left | −40 / 40 %, 1800 ms | −35 / 35 %, 2000 ms |
| 5 | drive to the middle of the 8 | 56 / 56 %, 4600 ms | 20 / 20 %, 4400 ms |
| 6 | turn right | 40 / −40 %, 300 ms | 20 / −20 %, 320 ms |
| 7 | 8: upper loop (counter-clockwise) | 30 / 96 %, 5600 ms | 8 / 50 %, 8000 ms |
| 8 | 8: lower loop (clockwise) | 96 / 30 %, 5550 ms | 50 / 8 %, 8000 ms |
| | stop, beep, end | | |

Total driving time, without the 1 s pause: 27.5 s in the simulator, 32.0 s on the real robot.

### How the simulator values translate into the shape

In the 2D model, one percent of power turns a wheel at **5.5 deg/s** (a TRIK Studio constant). With `r = 0.028 m`, that is a wheel speed of `p · 5.5 · π/180 · r` m/s.

**Formulas:**
- **Line:** `distance = wheel speed × time`. At 56 %: 0.151 m/s, so 2.7 s → 0.41 m, 4.4 s → 0.66 m, 4.6 s → 0.69 m.
- **Turn on the spot:** `angle = 2 · wheel speed / B × time`. At 40 %: 1.40 rad/s = 80 deg/s, so 1.5 s → 120°, 1.8 s → 144°, 0.3 s → 24°.
- **Loop:** the robot turns at `(v_out − v_in) / B`, and the radius is `(v_out + v_in) / 2 ÷ turn rate`. With 30 % / 96 %: radius 0.147 m and 66 deg/s, so 5.6 s → 370° and 5.55 s → 366°.

**Rounding:** the timers are rounded to round numbers, so the angles differ slightly from an exact 7:
- the turns are 120° / 144° / 24°;
- the loops turn slightly more than one full circle.

### Real robot values

The real robot uses different powers and timers, because it is not the simulated robot:
- its wheels are bigger (r = 3.75 cm vs 2.8 cm);
- its wheel base is wider (19.4 cm vs 15.4 cm);
- its motors do not run at the simulator's 5.5 deg/s per %.

The sequence of moves, and the direction of every move, are the same as in the simulator.

## How to run

1. **Simulator:**
   - Open `timed_sim.qrs` in TRIK Studio (kit TRIK v6.2) and press Run in the 2D model.
   - The robot starts at the origin facing right and draws the "78".
   - The log is written to `C:\Users\user\Desktop\odometry_data_sim.txt`.
2. **Real robot:**
   - Leave about 1 m × 0.8 m of free floor.
   - Upload and run `timed_real.qrs`, then download `odometry_data_real.txt` from the robot.
3. **Analysis:**
   - Open `Lab6_odometry_robotics.ipynb` in Colab and upload both logs.
   - It plots both trajectories and the position error over time.
   - Enter the tape-measured end point in the last cell.


## Limitations

- **Open loop:** nothing corrects errors during the run. Motor speed changes with the battery level, so the same timers can give a slightly different shape on another day.
- **Low inner-wheel power on the loops:** the real robot's loops use 8 % on the inner wheel. At such a low power, motor friction has a large effect, so the loop size is the least repeatable part of the run.
- **Moves follow each other directly:** there is no stop between moves, so the motors' reaction time lets each move run slightly into the next one.
