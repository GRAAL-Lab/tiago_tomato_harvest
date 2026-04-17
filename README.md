# Tiago Tomato Harvest

A ROS-based system for tomato detection and harvesting, implementing 3D visualization and robot manipulation using the TIAGo robot.

## Table of Contents

- [Getting Started](#getting-started)
- [Tomato Detection Topic](#tomato-detection-topic)
- [GUI](#gui)
  - [GUI Installation](#gui-installation)
- [MoveIt Configuration](#moveit-configuration)
- [RViz Configuration](#rviz-configuration)
- [Real Robot Setup](#real-robot-setup)
- [Octomap in Simulation](#octomap-in-simulation)


## Getting Started

**1. Launch the simulator:**

```bash
roslaunch agri_challenge tiago.launch use_moveit_camera:=true
```

**2. Launch the vision pipeline:**

```bash
roslaunch tomato_detection vision_manager.launch
```

**3. Launch the GUI:**

Download `gui.zip` from the latest release assets (this avoids the need to install the full Qt framework locally).
Extract the archive and run the executable found in the `/bin` subdirectory:

```bash
./tomato_guiApp
```

---

## Tomato Detection Topic

Subscribe to the following topic to receive tomato detections:

```bash
rostopic echo "/tomato_vision_manager/tomato_position"
```

### Message Fields

Each message contains a `Pose`:

| Field | Description |
|---|---|
| `Pose.position` | 3D position of the tomato relative to the `base_footprint` frame |
| `Pose.orientation.x` | Ripeness class (see below) |
| `Pose.orientation.y` | Tomato ID |
| `Pose.orientation.z` | Diameter in metres |

**Ripeness classes:**

| Value | Meaning |
|---|---|
| `0` | Ripe |
| `1` | Half-ripe |
| `2` | Unripe (green) |

---

## GUI

Built with Qt/QML. Navigate between pages by swiping with the mouse.

### Central Page — Colour Segmentation

- **Save** — saves the current parameters to `VisionConfig/config.toml`
- **Restore** — reloads parameters from the same file

> **Note:** The config file already contains good default values for the simulation. Press **Restore** to immediately see the colour mask applied.

The tabs in the top-right corner show different visualisations:
- Raw camera feed (no filters)
- Camera feed with colour mask applied
- Circle overlay highlighting detected tomatoes

### Left Page — Robot Home

Contains a **HOME** button that moves the robot to its default configuration.

> **Note:** The movement may take a moment to begin. Once complete, the result is displayed below the button. Real-time feedback is not currently supported.

### Right Page — YOLO & 3D View

- **Left panel:** YOLO detection output
- **Right panel:** Interactive 3D visualisation of detected tomatoes (supports rotation and zoom)

---

## GUI Installation

If the GUI fails to launch, install the following missing library:

```bash
sudo apt install libxcb-cursor-dev
```

---

## MoveIt Configuration

Before using MoveIt with collision detection, the TIAGo configuration files must be modified.

### 1. Create a PointCloud sensor configuration

The file below contains examples for different sensor types:

```
<TiagoTutorialsInstallDir>/src/tiago_moveit_config/config/sensors_rgbd.yaml
```

Rather than modifying it directly, create a new file in the same directory:

```
<TiagoTutorialsInstallDir>/src/tiago_moveit_config/config/sensors_pointcloud.yaml
```

with the following contents:

```yaml
sensors:
 - sensor_plugin: occupancy_map_monitor/PointCloudOctomapUpdater
   point_cloud_topic: /xtion/depth_registered/points
   max_range: 2.5
   octomap_resolution: 0.02
   padding_offset: 0.03
   padding_scale: 1.0
   point_subsample: 1
   filtered_cloud_topic: output_cloud
```

### 2. Update the sensor manager launch file

Edit:

```
<TiagoTutorialsInstallDir>/src/tiago_moveit_config/launch/tiago_moveit_sensor_manager.launch.xml
```

The final file should look like this:

```xml
<launch>

  <rosparam command="load" file="$(find tiago_moveit_config)/config/sensors_pointcloud.yaml" />

  <param name="octomap_frame" type="string" value="odom" />
  <param name="octomap_resolution" type="double" value="0.02" />
  <param name="max_range" type="double" value="5.0" />

</launch>
```

> **Note:** If you kept the original `sensors_rgbd.yaml` instead of creating the new file, update the `rosparam` line accordingly.

### 3. Launch with collision detection enabled

```bash
roslaunch agri_challenge tiago.launch use_moveit_camera:=true
```

---

## RViz Configuration

A pre-configured RViz setup is provided. To load it automatically, add the following line at line 31 of the `tiago.launch` file in `agri_challenge`:

```xml
<include file="$(find tomato_detection)/launch/rviz.launch"/>
```

---

## Real Robot Setup

### Network Configuration

Add the following two lines to the end of `~/.bashrc`:

```bash
export ROS_MASTER_URI=http://10.68.0.1:11311
export ROS_HOSTNAME=10.68.0.130
```

> **Warning:** These lines will break simulation. Comment them out and restart your terminal (or `source ~/.bashrc` in all open terminals) before switching back to simulation.

> **Note:** `ROS_HOSTNAME` may differ from the value shown above. After connecting to the TIAGo Wi-Fi network, verify your IP address with:
> ```bash
> ip addr
> ```
> Look for the address listed under `wlp...`.

### Things to Know

- Powering off the robot requires two people: one to support its arms and one to press the power button.
- Self-collisions are poorly handled — be careful.
- **Collision detection is disabled by default.** To enable it, open the web interface and activate it under *startup extras*.
- To disable the robot's idle head movement, open the web interface and disable `head_manager`.

### Web Interface

Access the web interface at `$ROS_MASTER_URI` on port 8080:

```
http://10.68.0.1:8080
```

### TIAGo Wi-Fi Password

```
TheEngineRoom
```

---

## Octomap in Simulation

If `use_moveit_camera:=true` was not passed at launch, or when using the real robot without octomap generation configured, use the `tomato_octo` node. Launch it with:

```bash
roslaunch tomato_octo tomato_octo.launch
```

This starts the `octomap_server` and the node responsible for publishing the octomap to the MoveIt `PlanningScene`.

When running in simulation, also run the following throttle command (the real robot already publishes this topic at the correct rate):

```bash
rosrun topic_tools throttle messages /xtion/depth_registered/points 3 /throttle_filtering_points/filtered_points
```
