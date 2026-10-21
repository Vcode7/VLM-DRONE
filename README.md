# Aero-Deck: Remote Gazebo + PX4 SITL Drone Control Dashboard

A high-performance aerospace mission control dashboard for Windows connecting to a remote **Gazebo SITL + PX4 Autopilot** simulation (e.g. running on a Mac mini over Tailscale).

---

## 🛰️ Architecture & Network Separation

The system clearly differentiates between **Gazebo high-bandwidth sensor streams** and **PX4 flight control telemetry**:

```
+-----------------------------------------------------------------------------------+
|                           Mac mini (Gazebo + PX4 SITL)                            |
|                                                                                   |
|  [Gazebo Simulation]                     [PX4 SITL Autopilot]                     |
|         │ (Frames / Scans)                         │ (MAVLink Heartbeats)         |
|         ▼                                          ▼                              |
|  Gazebo Sensor Bridge WebSocket            MAVLink UDP Broadcast                  |
|  Port 8000 (ws://100.x.x.x:8000)           Port 14540 (udp://100.x.x.x:14540)     |
+-----------------------------------------------------------------------------------+
                                   │  Tailscale Tunnel
                                   ▼
+-----------------------------------------------------------------------------------+
|                        Windows Machine (Aero-Deck Client)                         |
|                                                                                   |
|  FastAPI Backend (Port 8080)                                                      |
|  ├─ Camera WebSocket Client (ws://100.x:8000/ws/camera)                           |
|  ├─ LiDAR WebSocket Client  (ws://100.x:8000/ws/lidar)                            |
|  ├─ MAVSDK-Python Autopilot Manager (udpin://0.0.0.0:14540)                       |
|  │   └─ Continuous Offboard 20Hz Setpoint Streaming + Watchdog                    |
|  └─ Real-time WebSockets to UI (/ws/telemetry, /ws/camera, /ws/lidar)             |
|                                                                                   |
|  Vite + React + TypeScript Frontend (Port 5173)                                   |
|  ├─ Live FPV Camera Viewport with Aerospace HUD & Attitude Ladder                 |
|  ├─ LiDAR 3D Point Cloud (Three.js WebGL) + 2D Polar Radar + Split View           |
|  ├─ Cockpit Telemetry Cards (Alt, Speed, NED Vectors, GPS, Battery, Health)       |
|  ├─ Flight Deck (Dual Virtual D-Pads, Keyboard W/A/S/D, Hold-to-Move)             |
|  ├─ Independent Connect/Disconnect & Diagnostics Panel                            |
|  └─ Activity & Audit Event Log with Severity Filtering                            |
+-----------------------------------------------------------------------------------+
```

---

## 🚀 Quickstart Guide

### Prerequisites
- **Windows 10/11**
- **Python 3.10+** (Python 3.12 verified)
- **Node.js 18+** (Node v22.12 verified)

---

### Step 1: Set Up and Run the Backend

Open a PowerShell terminal in the project directory:

```powershell
cd "e:\vlm drone\backend"

# 1. Create Python virtual environment (if not already created)
python -m venv venv

# 2. Activate virtual environment
.\venv\Scripts\Activate.ps1

# 3. Install dependencies
pip install -r requirements.txt

# 4. Start the backend:
# Option A: Offline Simulation Test Harness mode (Tailscale not connected yet)
python run_backend.py --port 8080 --sim

# Option B: Live SITL mode (connects to Mac mini Tailscale IP)
python run_backend.py --port 8080
```

The backend server runs on `http://127.0.0.1:8080`.

> [!TIP]
> **Unified Single-Port Access:** The backend automatically serves the full production dashboard UI at **`http://127.0.0.1:8080/`** directly alongside the API and WebSockets! You can run the entire system with just Step 1.

---

### Step 2: (Optional) Run Frontend in Live Dev Mode (HMR)

If you are modifying frontend UI code and want live Hot Module Reloading:

```powershell
cd "e:\vlm drone\frontend"

# 1. Install frontend packages (already installed)
npm install

# 2. Launch Vite development server
npm run dev
```

Open your browser to:
👉 **`http://127.0.0.1:8080`** (Unified Dashboard) or **`http://127.0.0.1:5173`** (Vite Dev Server)

---

### Step 3: Running Tests

To verify all backend unit and integration tests (models, MAVSDK offboard clamping, adaptive sensor decoders, and API routes):

```powershell
cd "e:\vlm drone"
.\backend\venv\Scripts\python.exe -m pytest backend/tests -v
```

All 19 tests will execute and pass:
```
backend/tests/test_api_and_sim.py::test_api_read_config PASSED
backend/tests/test_api_and_sim.py::test_api_sim_mode_toggle_and_flight_controls PASSED
backend/tests/test_api_and_sim.py::test_synthetic_sensor_generation PASSED
backend/tests/test_models.py::test_connection_config_defaults PASSED
backend/tests/test_px4_manager.py::test_px4_manager_velocity_clamping PASSED
backend/tests/test_sensor_bridge.py::test_lidar_decode_2d_laserscan PASSED
...
19 passed in 1.37s
```

---

## 🛠️ Mac Mini Side Setup (Tailscale Bridge)

When you are ready to test with your friend's Mac mini:

### 1. Tailscale Connection
- Ensure both your Windows PC and the Mac mini are signed into the same Tailscale tailnet.
- Find your friend's Mac mini Tailscale IP (e.g., `100.64.0.1`).

### 2. Running the Mac Gazebo Bridge
If your friend is running a Gazebo bridge, verify the WebSocket endpoints for camera and LiDAR.
We have also included a ready-to-run reference bridge script in `backend/mock_mac_gazebo_bridge.py`:

```bash
# On Mac mini:
python3 -m pip install fastapi uvicorn websockets pillow pydantic
python3 mock_mac_gazebo_bridge.py --host 0.0.0.0 --port 8000
```
This serves:
- `ws://<MAC_IP>:8000/ws/camera` (continuous JPEG frames)
- `ws://<MAC_IP>:8000/ws/lidar` (2D LaserScan / 3D point cloud)

### 3. Running PX4 SITL on the Mac mini
When running PX4 SITL (e.g. `make px4_sitl gz_x500` or ROS 2 launch), PX4 broadcasts MAVLink over UDP.
Ensure PX4 targets the default UDP port (`14540`) or forwards to your Windows Tailscale IP:
- Default listen string on Windows: `udpin://0.0.0.0:14540`

---

## 🎮 Flight Controls & Keyboard Cheatsheet

| Function | Key | D-Pad Button | Description |
| :--- | :---: | :---: | :--- |
| **Fly Forward** | `W` | Right Pad (Top) | Moves body forward (+X) at configured speed |
| **Fly Backward** | `S` | Right Pad (Bottom) | Moves body backward (-X) |
| **Strafe Left** | `A` | Right Pad (Left) | Moves body left (-Y) |
| **Strafe Right** | `D` | Right Pad (Right) | Moves body right (+Y) |
| **Climb Up** | `Space` | Left Pad (Top) | Climbs vertically (+Z) |
| **Descend Down** | `Shift` | Left Pad (Bottom) | Descends vertically (-Z) |
| **Yaw Left** | `Q` | Left Pad (Left) | Rotates counter-clockwise |
| **Yaw Right** | `E` | Left Pad (Right) | Rotates clockwise |
| **Emergency Hold** | `H` or `Esc` | Red HOLD Button | **Zeroes velocity setpoints immediately** |

### Hold-to-Move Safety Feature
Movement buttons and keyboard keys only command velocity **while actively held**. Releasing the button or key immediately halts velocity streaming and holds position.

---

## 🛡️ Built-in Safety & Reliability Rules

1. **Simulation Controller Banner**: High-visibility banner prevents any confusion with real hardware.
2. **Read-Only on Connect**: The application **NEVER arms or moves automatically** when connecting.
3. **Safety Arm Confirmation Dialog**: Arming requires explicit confirmation with pre-flight checklist verification.
4. **PX4 Offboard Pre-Streaming**: PX4 strictly requires setpoints to stream at >2Hz *before* switching to Offboard mode. Aero-Deck automatically streams zero setpoints for 0.6s before engaging offboard to prevent `COMMAND_DENIED` errors.
5. **Command Watchdog (500ms)**: If no new movement packet is received within 500ms, the backend automatically snaps velocity setpoints to `(0, 0, 0, 0)`.
6. **Browser Focus Loss Safeguard**: If the browser window loses focus or the user switches tabs, an automatic blur listener sends an instant zero-velocity command to prevent flyaways.
7. **Thread & Task Isolation**: Visual sensor decoding (Camera frames & LiDAR scans) runs in separate asyncio background tasks, ensuring that video processing latency **never interferes with the 20 Hz MAVSDK flight control loop**.
8. **Configurable Speed Clamping**: Enforces horizontal speed (0.5–10 m/s), vertical climb (0.2–5 m/s), and yaw rate limits in both frontend and backend.

---

## ⚙️ Configuration & Settings Panel

Click the **Gear icon** in the top navigation bar to open the Settings modal:

- **Gazebo Bridge Base URL**: e.g., `ws://100.64.0.1:8000`
- **Camera Path**: e.g., `/ws/camera` (supports raw JPEG bytes, base64 JSON, and Rosbridge)
- **LiDAR Path**: e.g., `/ws/lidar` (supports 2D LaserScan and 3D PointClouds)
- **PX4 MAVLink UDP Address**: default `udpin://0.0.0.0:14540`
- **Max Horizontal Speed**: default `3.0 m/s`
- **Max Vertical Speed**: default `1.5 m/s`
- **Max Yaw Rate**: default `45.0 °/s`
- **Command Watchdog Timeout**: default `500 ms`
- **Offboard Rate**: default `20 Hz`

Settings are persisted in `backend/settings.json` across server restarts.

---

## 🔍 Troubleshooting

| Issue | Cause | Resolution |
| :--- | :--- | :--- |
| **Heartbeat timeout on udpin://0.0.0.0:14540** | PX4 SITL is not running or broadcasting MAVLink packets to this port | Verify PX4 SITL is active on Mac. Test with `ping <MAC_IP>` over Tailscale. Check Windows Firewall allows UDP on port 14540. |
| **Camera shows "No Camera Feed"** | WebSocket connection refused or bridge offline | Check that the Mac Gazebo bridge is running on port 8000. Verify the URL in Settings (`ws://<MAC_IP>:8000`). |
| **LiDAR points not rendering** | Topic format mismatch | Aero-Deck accepts ROS `sensor_msgs/LaserScan` (`ranges`, `angle_min`, `angle_increment`) or `points` arrays. Verify the bridge output format. |
| **Offboard mode denied by PX4** | Vehicle not armed or setpoints not received | Take off first or ensure vehicle is armed. Aero-Deck streams 20 Hz setpoints prior to offboard initiation. |
| **Tailscale offline** | Friend's Mac is not connected yet | Toggle **"Simulation Test Harness"** in the top banner. This enables the built-in offline simulator so you can test all flight deck features and instruments right now! |

---

## 🧠 VLM Drone Intelligence layer (new — `backend/vlm/`)

Autonomy, perception, memory and safety built **on top of** the Aero-Deck integration above (which is unchanged).

| Read | |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | what existed, what was missing, new modules, data flow, Mac-bridge extensions |
| [docs/SETUP.md](docs/SETUP.md) | Windows/GPU setup, running tests, scenarios, benchmarks, connecting the Mac |
| [docs/BENCHMARKS.md](docs/BENCHMARKS.md) | measured accuracy / latency / VRAM / scenario reliability |
| [docs/MODELS_AND_LICENSES.md](docs/MODELS_AND_LICENSES.md) | model installation and licensing (YOLO = AGPL, InsightFace weights = non-commercial) |
| [docs/LIMITATIONS.md](docs/LIMITATIONS.md) | what is not validated / not implemented yet |

Quick demo (no Mac needed): `cd backend && .\venv\Scripts\python.exe -m vlm.cli.run_scenario --variant loops --seeds 0 --out ..\runs\demo`
→ the drone starts inside a maze, explores with DFS, remembers the map, solves it with BFS and replays the route from memory.
Tests: `python -m pytest backend/tests backend/tests_vlm -q`.
#   V L M - D R O N E  
 