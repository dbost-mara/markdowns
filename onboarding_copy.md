# Spike Sim Onboarding — spike_ws / spike_ros / spike_sim

A from-zero guide to getting the Spike simulation running on your dev machine.
Everything here targets the **main** branches of `spike_ws` and `spike_ros`.

---

## 1. The big picture (the "what" and "why")

Spike is a counter-drone system: ground stations ("spotters") detect and track
intruder drones, and interceptor drones ("seekers") are launched to take them
down. Real deployments run on Jetson hardware with real cameras and real
drones. You will obviously not be crashing real drones on day one — so we
simulate the whole thing in **NVIDIA Isaac Sim**, a GPU-based 3D simulator that
publishes the same ROS 2 topics the real sensors would.

The cast of robots you'll see in sim:

| Robot | What it is |
|---|---|
| **spotter** | Ground camera platform — 5 cameras, 16-mic array, IMU. The "eyes". |
| **ghost** / **banshee** | Seeker (interceptor) quadcopters, flown by ArduPilot. |
| **aggressor** | The target drone the system is trying to detect and intercept. |

And the software pipeline that consumes them: **perception** (detect drones in
camera frames) → **scout** (fuse detections into 3D tracks) → **captain**
(decide what to do) → **hangar/seeker** (launch and guide interceptors).

## 2. Three names that confuse everyone (the "where")

You'll hear `spike_ws`, `spike_ros`, and `spike_sim` used constantly. They nest
inside each other:

```
spike_ws/                        ← WORKSPACE (repo: marainc/spike_ws)
├── scripts/                     ← setup & container scripts (run_spike_dev.sh …)
├── dev/                         ← deployment tooling for real Jetson hardware
├── docker/                      ← dev container Dockerfiles
├── spike_ros_dependencies.repos ← list of repos that get cloned into src/
└── src/                         ← all source code lives here
    ├── spike_ros/               ← MAIN REPO (repo: marainc/spike_ros)
    │   ├── spike_msgs/          ← custom ROS 2 message definitions
    │   ├── spike_bringup/       ← launch files for the real system
    │   ├── spike_drivers/       ← camera/sensor driver nodes
    │   ├── spike_pipeline/      ← perception pipeline
    │   ├── scout/  captain_sm/  hangar/  seeker/  aggressor/
    │   └── spike_sim/           ← THE SIM PACKAGE ← you are here
    ├── spike_description/       ← robot URDFs and meshes
    ├── isaac_ros2_utils/        ← Isaac Sim ↔ ROS 2 glue (REST API, spawning)
    ├── falcon/                  ← flight controller integration
    └── (audio_common, gscam2, …) ← third-party deps
```

- **`spike_ws`** is a ROS 2 *workspace*: a thin shell repo containing setup
  scripts, deployment tooling, and a `.repos` manifest. It does not contain
  the actual robot code — `vcstool` clones that into `src/` during setup.
- **`spike_ros`** is the main *repository*: a collection of ROS 2 packages
  implementing the system (messages, drivers, perception, decision-making).
- **`spike_sim`** is one *package* inside `spike_ros`: everything needed to run
  the system against Isaac Sim instead of real hardware — launch files that
  spawn robots, ArduPilot SITL wiring, and scripts that start Isaac Sim itself.

## 3. How the sim actually works (architecture)

Two processes, talking over localhost:

```
┌─────────────────────────────────────────────────────────┐
│  Your dev machine (x86_64 + NVIDIA GPU)                  │
│                                                          │
│  ┌─────────────────────┐      ┌───────────────────────┐ │
│  │ Isaac Sim (host)    │      │ spike-dev container   │ │
│  │ - physics + render  │◄────►│ (ROS 2)               │ │
│  │ - ROS 2 bridge      │      │ - spawn launch files  │ │
│  │ - REST API :8080    │      │ - ArduPilot SITL      │ │
│  │ - publishes /clock, │      │ - perception, scout,  │ │
│  │   cameras, IMU      │      │   captain, hangar     │ │
│  └─────────────────────┘      └───────────────────────┘ │
│        both share --network host + same ROS_DOMAIN_ID   │
└─────────────────────────────────────────────────────────┘
```

- **Isaac Sim** runs either natively on the host or inside the **Afterlife
  container** (recommended — see §4.2). Either way it runs a ROS 2 bridge and
  exposes a small **REST API on port 8080** that our tooling uses to
  spawn/delete robots, play/pause the sim, etc.
- **The spike-dev container** runs every ROS 2 node. It uses `--network host`,
  so `localhost:8080` reaches Isaac Sim and DDS traffic flows freely.
- **Seeker drones are flown by real ArduPilot firmware** running in SITL
  (Software In The Loop) mode: Isaac Sim does the physics, SITL does the
  flight control, and they exchange state over UDP. This is the same autopilot
  code that runs on the real drones — only the physics is fake.

## 4. One-time setup (the "how", part 1)

This section assumes a **completely fresh Ubuntu machine**. Steps are in
dependency order — do them top to bottom.

### 4.0 Bare-machine prerequisites

1. **Ubuntu 22.04 or 24.04** on an x86_64 machine with an NVIDIA GPU
   (RTX-class; Isaac Sim is GPU-rendered).
2. **NVIDIA driver** — verify with `nvidia-smi`. If missing:
   ```bash
   sudo ubuntu-drivers install
   sudo reboot
   ```
   On Blackwell GPUs (RTX 50xx) the Afterlife setup recommends driver
   **575.57.08** specifically — see `afterlife/REQUIREMENTS.md` if you hit
   multi-GPU/render issues.
3. **Git + GitHub SSH key** with access to the `marainc` org
   ([add an SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)).
   Test: `ssh -T git@github.com`.
4. **Docker + NVIDIA Container Toolkit** — needed for both the spike dev
   container and Afterlife. If you take the Afterlife path below, its
   `setup.sh` installs Docker for you if it's missing (you'll need to log out
   and back in for the `docker` group to apply). For the container toolkit:
   ```bash
   # https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
   sudo apt install -y nvidia-container-toolkit
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

### 4.1 Clone and populate the workspace (Ensure it's in the home directory)

```bash
git clone git@github.com:marainc/spike_ws.git
cd spike_ws
./scripts/setup_dev_host.sh
```

`setup_dev_host.sh` creates the workspace structure and uses `vcstool` to clone
every repo listed in `spike_ros_dependencies.repos` into `src/` — including
`spike_ros` (main branch), `spike_description`, `isaac_ros2_utils`, `falcon`,
and `ardupilot`. **Do not** `git clone spike_ros` by hand; let the script place
everything.

### 4.2 Get Isaac Sim — two options

You need Isaac Sim from exactly one of these. **Option A is recommended**: no
NVIDIA developer account, no manual download, and it's what the team runs
day-to-day.

#### Option A — Afterlife container (recommended)

[Afterlife](https://github.com/marainc/afterlife) is our simulation platform
repo. Its Docker compose setup runs **Isaac Sim 5.1.0 inside a container**
(image `nvcr.io/nvidia/isaac-sim:5.1.0`), with our custom extensions (spatial
audio, ArduPilot bridge, REST API) mounted in. You view the sim through a
small native WebRTC streaming-client app instead of a local GUI.

```bash
# Must live at ~/afterlife (ensure cloned in home directory) (or set AFTERLIFE_REPO to override)
git clone --recursive git@github.com:marainc/afterlife.git ~/afterlife
cd ~/afterlife
./setup.sh        # checks driver, installs Docker if missing, pulls deps
```

The first launch (next section) will also auto-install the streaming-client
viewer and add a `run_isaac_sim_local` alias to your `~/.bashrc`. Expect the
first run to be slow: it pulls the multi-GB Isaac Sim image and warms shader
caches.

#### Option B — native Isaac Sim on the host

Download Isaac Sim from NVIDIA (https://developer.nvidia.com/isaac-sim —
requires an NVIDIA account) and install to the default location:

```bash
export ISAAC_SIM_PATH=$HOME/isaac-sim   # add to ~/.bashrc
```

Optionally install the WebRTC streaming client for headless use:

```bash
sudo apt install -y libfuse2
cd ~/spike_ws/src/spike_ros/spike_sim
./util/setup_streaming_client_desktop.sh
```

### 4.3 Build ArduPilot SITL

Needed to fly seeker drones:

```bash
git clone https://github.com/ArduPilot/ardupilot.git ~/ardupilot
cd ~/ardupilot
Tools/environment_install/install-prereqs-ubuntu.sh -y
./waf configure --board sitl && ./waf copter
```

### 4.4 Start the dev container and build

```bash
cd ~/spike_ws
./scripts/setup_dev_host.sh
chmod +x ./scripts/run_spike_dev.sh
# I needed this to create the container
# sudo systemctl start nvidia-persistenced
./scripts/run_spike_dev.sh
```

This builds (first run) and starts a persistent ROS 2 dev container with the
workspace mounted at `/workspaces/isaac_ros-dev`. It auto-detects your Ubuntu
version (22.04 → Humble, 24.04 → Jazzy) and adds a `run_spike_dev` alias to
your `~/.bashrc` so next time you can just type `run_spike_dev`.

Inside the container, build:

```bash
cd /workspaces/isaac_ros-dev
colcon build --packages-select spike_sim spike_msgs spike_description spike_common spike_pipeline
source install/setup.bash
```

Rule of thumb: always build with `--packages-select <pkg>...` — a bare
`colcon build` builds the entire workspace including heavy perception packages
you may not need yet.

## 5. Daily sim workflow (the "how", part 2 — and the "when")

You'll do this every time you sit down to work on sim.

### Step 1 — Launch Isaac Sim (host terminal, NOT in the container)

**If you set up Afterlife (Option A):**

```bash
cd ~/spike_ws/src/spike_ros/spike_sim
./util/launch_afterlife_webrtc.sh
# after the first run, the alias works too:  run_isaac_sim_local
```

This restarts the Afterlife container via docker compose and opens the Isaac
streaming-client viewer window. `LOCAL_DEV=1` (which the alias sets) mounts
your local `spike_ws/src` into the container so extension/URDF edits take
effect without rebuilding.

**If you installed Isaac Sim natively (Option B):**

```bash
cd ~/spike_ws/src/spike_ros/spike_sim
./util/launch_isaac_sim_jazzy.sh      # use launch_isaac_sim_humble.sh on 22.04
```

Useful flags: `--usd <scene.usd>` for a custom world, `--headless` for no GUI.

**Either way**, you end up with Isaac Sim running a ROS 2 bridge, a default
scene, and the REST API. Sanity check from another terminal:

```bash
curl localhost:8080/health            # should answer
# Swagger docs for all endpoints: http://localhost:8080/docs
```

### Step 2 — Enter the dev container (second terminal)

```bash
run_spike_dev
```

`ROS_DOMAIN_ID` must match between Isaac Sim and the container (default `0`
on both — only a problem if you've overridden it).

### Step 3 — Spawn robots and run the system

**Option A — full system in one shot** (spotter + aggressor + perception +
scout + captain + hangar with one seeker SITL):

```bash
ros2 launch spike_sim sim_spike.launch.py
```

Foxglove visualization becomes available at `ws://localhost:8765`.

**Option B — piecemeal**, when you only care about one robot:

```bash
# Spotter (cameras + mics + IMU)
ros2 launch spike_sim spawn_spotter.launch.py

# Seeker drone + ArduPilot SITL + MAVProxy console
ros2 launch spike_sim spawn_seeker.launch.py robot:=ghost

# FIRST EVER seeker run: initialize SITL's virtual EEPROM
ros2 launch spike_sim spawn_seeker.launch.py robot:=ghost wipe_eeprom:=true

# A target to detect
ros2 launch spike_sim spawn_robot.launch.py robot:=aggressor namespace:=aggressor_0 z:=5.0
```

### Step 4 — Press PLAY in Isaac Sim

Nothing moves until the simulation is playing. Spawning a robot does **not**
start physics — press **Play** in the Isaac Sim window (or
`curl -X POST localhost:8080/simulation/play`). Once playing, Isaac Sim
publishes `/clock` and sensor topics, and SITL starts receiving physics state.

### Step 5 — Fly a seeker (optional)

In the MAVProxy console that `spawn_seeker` opened:

```
mode guided
arm throttle
takeoff 10
```

### Step 6 — Verify

```bash
ros2 topic list | grep spotter
ros2 topic echo /clock --once                       # sim time is ticking
ros2 run rqt_image_view rqt_image_view /spotter/camera_0/image_raw
```

Spotter topics you should see: `/spotter/camera_0..4/image_raw`,
`/spotter/imu`, `/spotter/odom`.

## 6. When things go wrong

| Symptom | Cause / fix |
|---|---|
| SITL prints "Waiting for JSON data" forever | Sim isn't playing, or robot isn't spawned. Press Play. |
| "Arm: 3D Accel calibration needed" | Re-launch seeker with `wipe_eeprom:=true`. |
| "Compass 1 not healthy" | `sitl_overrides.param` should zero hardware compass IDs; restart SITL with `-w`. |
| Container sees no Isaac Sim topics | `ROS_DOMAIN_ID` mismatch host↔container, or container not on `--network host`. |
| REST API returns `{"detail":"Not Found"}` | Isaac Sim was started without our custom REST server — use the `launch_isaac_sim_*.sh` scripts. |
| Streaming client viewport empty | A stale Kit process is holding port 49100: `pkill -f 'kit/kit'` and relaunch. |
| Drone spins after takeoff | Rotor direction mismatch in the URDF — flip `<rot_dir>` signs and respawn. |
| Docker error about "unresolvable CDI devices nvidia.com/gpu=all" | `sudo nvidia-ctk cdi generate --mode=csv --output=/etc/cdi/nvidia.yaml` |

## 7. Where to read more

| Doc | What it covers |
|---|---|
| `spike_ws/README.md` | Workspace overview, remote-AGX deployment workflows |
| `src/spike_ros/spike_sim/README.md` | Full spike_sim reference (launch args, SITL port map, REST API) |
| `src/spike_ros/spike_sim/ISAAC_SIM_SETUP.md` | Isaac Sim installation details |
| `src/spike_ros/spike_sim/ISAAC_SIM_WORKFLOW.md` | Workflow and ROS service reference |
| `src/spike_ros/spike_sim/DOCKER_USAGE.md` | Container configuration specifics |
| `~/afterlife/QUICKSTART.md` | Afterlife setup details and demos |
| `~/afterlife/REQUIREMENTS.md` | GPU driver requirements (Blackwell notes) |
| `~/afterlife/docs/` | Afterlife env vars, audio sim, extension docs |

Key source locations once you start hacking:

- Robot URDFs/meshes: `src/spike_description/urdf/`
- ArduPilot SITL params: `src/spike_ros/spike_sim/ardu_params/` (per-robot
  `.param` + `sitl_overrides.param` applied on top)
- Isaac Sim REST API + spawn logic: `src/isaac_ros2_utils/isaac_ros2_scripts/`
- Sim launch files: `src/spike_ros/spike_sim/launch/`