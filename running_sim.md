# Running sim (This is assuming installation complete on mara-1)

## Startup and Running all containers and nodes
### Opening WebRTC
This will start the afterlife container, `afterlife-ros2-dev-bridge`, and open up the WebRTC viewer for IsaacSim. Open up terminal 1 and from the home directory run the following.
```sh
# To run visualization in isaac sim
./start_afterlife.sh

# To open Isaac Viewer WebRTC
open_webrtc

# If first time running
alias open_webrtc="$HOME/.local/bin/isaac-streaming-client.AppImage --no-sandbox"
source ~/.bashrc
open_webrtc

# Ensure IP from terminal and WebRTC viewer match

# TODO: Make ./util/launch_afterlife_webrtc.sh command work, as of right now it is currently broken on this current install
```

### Running Spike ROS
Here we will start the container,`spike-dev-container`, that puts Aggressor, Spotter, and Banshee in the scene and starts spinning all their ROS nodes and topics. This includes:
- `/alpha/SmCaptain`
- `/alpha/captain_sm`
- `/banshee_0`
- `/canonical`
- `/field_hangar_1`
- `/panel`
- `/scout_3d_transform`
- `/spotter_0`
In order to see all their nodes spinning, you need to press play in the WebRTC window. In terminal 2, run the following.
```sh
# From installation, we've created an alias to run the bash script
# This command will open the container 
run_spike_dev -f
```
Now that the container is running, we can launch all of our nodes. In terminal 2, run the following.
```sh
# This launches all assets and spawns them into the scene
ros2 launch spike_sim sim_spike.launch.py
```

### Visualizing bounding boxes
Open up foxglove application, open connection.

## Shutdown and Stopping all containers and nodes
### Closing WebRTC
In terminal 1, where the visualizer is open, either press ctrl+C or press "X" on the viewer window. This will close the viewer window.
```sh
# To powerdown visualization container in IsaacSim
./start_afterlife.sh down
# or
docker stop afterlife-ros2-dev-bridge
```

### Shutting down Spike ROS
```sh
# To shutdown the container, in one of the terminals where the container is open run
logout # This brings you to the home directory of your computer
docker stop spike-dev-container

# To verify everything is shutdown run
docker ps
```

## Misc for now
### Audio/GNSS commands
```sh
ros2 run audio_common audio_capturer_node --ros-args -p device:=4 -p format:=1 -p rate:=48000 -p channels:=16
ros2 run ublox_dgnss_node ublox_dgnss_node --ros-args -p CFG_USBOUTPROT_NMEA:=False
```

