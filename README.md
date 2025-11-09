## Notice
Every reboot, you need enable host for docker to use docker display.

`xhost +local:docker`

`mkdir packages`


## Build & run docker (This line would be only run once. If you run this line again the container will be reconstructed)
`docker compose up -d`

## Exec 

`docker container exec -it ros2_foxy_cuda /bin/bash`

## When first into the docker
`ubuntu-drivers autoinstall`

`apt-get install -y $(nvidia-detector)`



## Updated 20251109 for installing docker and use rviz2 for quest visualization

Quest2 visualization refer to another repo "oculus_reader"

# Foxy_Display_Docker — ROS 2 Humble + Meta Quest Visualization (Docker)

This repository provides a Docker-based ROS 2 Humble environment to visualize data from a Meta Quest headset. It includes instructions to set up Docker, enable X11 GUI display from containers, prepare the workspace, install dependencies inside the container, and run the visualization.

## Prerequisites

- Ubuntu 22.04 (jammy) recommended
- Docker Engine + Docker Compose V2
- X11 desktop (for GUI visualization)
- Meta Quest device with Developer Mode enabled and USB debugging on
- USB cable

## 1) Install Docker

```bash
# Update and install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key and repository
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Optional: enable docker and allow non-root usage (log out/in after this)
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

References:
- Ubuntu archive mirror is available at: http://archive.ubuntu.com/ubuntu/ (jammy updates used by apt in the logs above).

## 2) Prepare X11 Access (run every login/reboot as needed)

To allow GUI apps from the container to display on your host X server:

```bash
# Allow local docker containers to connect to X server
xhost +local:docker

# Or use the helper if provided
bash set_display.bash
```

To revoke later:
```bash
xhost -local:docker
```

## 3) Prepare Repository Workspace

This repo’s docker-compose mounts your host `./packages` into the container at `/root/ws/src/`. Keep your ROS 2 packages there.

```bash
# from the repository root
mkdir -p packages
```

You should have your sources something like:
```
Foxy_Display_Docker/
├─ docker-compose.yml
├─ set_display.bash         (optional helper)
└─ packages/
   └─ oculus_reader/        (Meta Quest reader package)
```

## 4) Start/Restart the Container

```bash
# Start container in background
docker compose up -d
# Note: docker-compose.yml 'version' key is obsolete; safe to remove the field if warned.

# Check containers
docker container ls -a

# Exec into the ROS 2 Humble container
docker container exec -it ros2_humble_cuda /bin/bash
```

If you need to stop/remove older containers:

```bash
docker container stop <CONTAINER_ID_OR_NAME>
docker container rm <CONTAINER_ID_OR_NAME>
```

Example:
```bash
docker container stop ros2_humble_cuda
docker container rm ros2_humble_cuda
docker compose up -d
```

## 5) Install Python and ROS Dependencies (inside the container)

Enter the container first:
```bash
docker container exec -it ros2_humble_cuda /bin/bash
```

Then, install pip (if missing), Python deps, and ROS packages as needed. Example flow (from your session):

```bash
# If pip is missing
apt update
apt install -y python3-pip

# ROS dependency (example used by oculus_reader)
apt install -y ros-humble-tf-transformations
```

The `ros-humble-tf-transformations` package is hosted on the ROS 2 repository mirror:
- ROS 2 Ubuntu repo index: http://packages.ros.org/ros2/ubuntu/

Install the Python requirements from your package:
```bash
cd /root/ws/src/oculus_reader
pip install -r requirements.txt
pip install -e .
```

Note on venv: pip warns against installing as root into system Python. For isolation, you can use a virtual environment:

From Python 3.14 docs (Virtual Environments and Packages):
- Create a venv: `python -m venv .venv`
- Activate: `source .venv/bin/activate`
- Install: `python -m pip install -r requirements.txt`
- Deactivate: `deactivate`

See: https://docs.python.org/3/tutorial/venv.html

## 6) Install ADB inside the Container

The `oculus_reader` uses ADB. Install it in the container:

```bash
apt update
apt install -y android-tools-adb  # package 'adb' on Ubuntu 22.04
```

This pulls `adb` and its dependencies from the Ubuntu archive:
- Example packages: android-libadb, android-libbase, android-libboringssl, adb, etc.
- Served by http://archive.ubuntu.com/ubuntu/ (jammy).

Verify ADB:
```bash
adb version
adb devices
```

You should see your Meta Quest listed (enable Developer Mode and USB debugging on the headset, accept the RSA fingerprint prompt).

If you don’t see the device:
- Check cable/USB mode.
- Run `adb kill-server && adb start-server`.
- Ensure permissions; you may need `sudo` or proper udev rules on the host. If udev rules are on the host, also pass the device into the container (see “USB device access” below).

## 7) USB Device Access (if needed)

If the container can’t see the Quest over USB, add device mappings to your Compose file or run with `--device`. Example snippet to add to your docker-compose service:

```yaml
devices:
  - /dev/bus/usb:/dev/bus/usb
group_add:
  - "video"
  - "plugdev"
```

Alternatively, start with:
```bash
docker run --rm -it \
  --device /dev/bus/usb:/dev/bus/usb \
  ...
```

You may also need udev rules on the host for ADB devices.

## 8) Run the Visualization

Inside the container:

```bash
# Assuming DISPLAY is set and xhost permission has been granted on host
cd /root/ws/src/oculus_reader/oculus_reader
python3 visualize_oculus_transforms_ros2.py
```

If you see:
- `RuntimeError: adb not found` → install `adb` in the container (see step 6).
- ROS shutdown error after Ctrl-C (`rcl_shutdown already called`) → benign on double shutdown; just rerun the script.

## 9) Common Tips

- X11:
  - Ensure the container has `DISPLAY` and possibly `XAUTHORITY` set. The helper `set_display.bash` can export required variables and mount X11 sockets.
  - Typical volumes: `-v /tmp/.X11-unix:/tmp/.X11-unix:rw`
- GPU (if CUDA is used):
  - Start with `--gpus all` or proper Compose `deploy.resources.reservations.devices`.
- Workspace:
  - Source code goes in `./packages` on the host; it appears in `/root/ws/src` inside the container.
- ROS 2 environment:
  - If needed, source ROS 2 inside the container: `source /opt/ros/humble/setup.bash`
- Virtual environments (optional but recommended by Python docs):
  - Use `python -m venv .venv` and activate before pip installs to avoid mixing with system Python.

## 10) Troubleshooting

- Container can’t open GUI windows:
  - Run on host: `xhost +local:docker`
  - Ensure `DISPLAY` is forwarded and `/tmp/.X11-unix` is mounted.
- Device not visible via `adb devices`:
  - Check USB cable/port.
  - Confirm Quest developer mode and USB debugging, accept host key on the headset.
  - Map `/dev/bus/usb` into the container.
  - Restart server: `adb kill-server && adb start-server`.
- Pip as root warning:
  - Prefer a Python virtual environment in the container (see Python 3.14 venv guide).

## 11) References to Mirrors and Docs

- Ubuntu APT mirror index: http://archive.ubuntu.com/ubuntu/
  - Example paths used by apt in logs: `jammy`, `jammy-updates`
- ROS 2 Ubuntu repository index: http://packages.ros.org/ros2/ubuntu/
- Python 3.14 Virtual Environments and Packages: https://docs.python.org/3/tutorial/venv.html

---

With this setup, you can reliably run ROS 2 Humble in Docker, install Meta Quest dependencies (including ADB), and visualize transforms from the headset on your Ubuntu desktop.

