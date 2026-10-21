# Lab 6: Odometry localization, trajectory "78"

The robot draws a **7**, stops, beeps and pauses for 1 s, then drives over to draw an **8**. While it drives it logs its odometry.

| File | Purpose |
|---|---|
| `trajectory_sim.qrs` | Program for the TRIK Studio simulator. Writes `odometry_data_sim.txt`. |
| `trajectory_real(2).qrs` | Program for the real TRIK robot. Writes `odometry_data_real.txt`. |
| `Lab6_odometry_robotics.ipynb` | Colab notebook: both trajectories, error over time, end-point check. |

## How the program works
1. **Setup:** clear encoders E3/E4, delete the old log, set `r`, `B` and the waypoint arrays `X`, `Y`, `E`.
2. **Loop, every 20 ms:**
   - **Odometry:** `path = (θR+θL)·r/2`, `φ = (θR−θL)·r/B`, `x += Δpath·cos φ`, `y += Δpath·sin φ`.
   - Write `x,y,time` to the log.
   - **Odometry** uses the mid-step heading `(φ+φ_prev)/2`, which matters when the loop is slow.
   - **Go-to-goal**, in physical units: heading error `e = atan2(dy,dx) − φ`,
     `v = min(Vw[i]·cos³(e), Va[i] + 1.2·d)` m/s, `ω = clamp(2.5·e, ±3)` rad/s.
     `Vw` is 0.30 m/s on the straight legs and 0.18 m/s on the loops of the 8.
     `Va` slows the robot down before stops and sharp corners.
     Each wheel's speed `v ∓ ω·B/2` (m/s) becomes motor power `% = speed·180/(π·r·kM)`.
3. **When a waypoint is within 5 cm** (or already passed, within 15 cm and behind the robot), look at its event code `E[i]`:
   - `0` – go to the next waypoint.
   - `3` – start of the 8: put the marker down.
   - `1` – end of the 7: stop, beep, lift the marker, pause 1 s.
   - `2` – end of the 8: stop, beep, red LED, finish.

Frame: start pose = origin, x forward, y to the robot's left. The digits are 0.6 m tall and each is about 0.4 m wide.

## Differences between the two files
Only these differ; the rest of the program is identical.

| | sim | real |
|---|---|---|
| `r` (wheel radius) | 0.028 m (2D-model wheel, 5.6 cm diameter) | 0.0375 m (real robot) |
| `B` (wheel base) | 0.154 m (2D-model wheels) | 0.194 m (real robot) |
| `kM` (motor gain) | 5.5 deg/s per % (2D-model constant) | 9.40 deg/s per % |
| Log file | `odometry_data_sim.txt` | `odometry_data_real.txt` |

