# Introduction to the ROS Navigation Stack using AWS DeepRacer Evo



## Overview

[Nav2](https://navigation.ros.org/), the second generation of the Robot Operating System (ROS) Navigation Stack, is a 2D navigation software framework that you can use to help your robot navigate autonomously from a starting position to a goal position. The updated software is the second generation of the ROS Navigation stack and consists of plugins and packages built on ROS2, a generic software prototyping tool that implements algorithms related to path planning, autonomous navigation, localization, mapping, and more. 

Integrating the ROS2 stack with AWS DeepRacer device software allows you to run the ROS Nav2 stack with a given map in a simulation as well as on the [DeepRacer Evo](https://aws.amazon.com/deepracer/), a 1/18th scale 4-wheel vehicle platform that features a LiDAR sensor and Ackermann steering. To do this, we are open-sourcing additional components for the integration.

**AWS DeepRacer components open-sourced for integration:**

* **[deepracer_bringup](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_bringup):** Launch and configuration parameter files
* **[deepracer_description](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_description):** URDF files for the AWS DeepRacer device
* **[deepracer_gazebo](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_gazebo):** `deepracer_drive `plugin required to move the car in simulation
* **[deepracer_nodes](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_nodes):** Set of nodes that are responsible for making the device compatible with the ROS2 Nav stack


Three main components of the Nav2 stack are planners, controllers, and recoveries. Planners and controllers are at the heart of a navigation task. The task of a planner is to compute a path to complete an objective function. Controllers, also known as local planners in ROS 1, are the way the robot follows the globally computed path or completes a local task. The controller will have access to a local environment representation to attempt to compute feasible control efforts for the robot base to follow. The general task in Nav2 for a controller is to compute a valid control effort to follow the global plan. Recoveries are used to get the robot out of a bad situation or attempt to deal with various forms of issues to make the system fault-tolerant. The goal of recoveries are to deal with unknown or failure conditions of the system and autonomously handle them. To learn more about other components such as behavior trees, navigation servers, and action servers, see the [Nav2 documentation](https://navigation.ros.org/). 

**Nav2 tools at a glance** (from [Nav2 documentation](https://navigation.ros.org/))**:**

* **Nav2 Planner**: Plans a path from A to B around obstacles 
* **Nav2 Controller:** Controls the robot as it follows the path
* **Nav2 Recoveries:** Compute recovery behaviors in case of failure 
* **Nav2 Behavior Trees and BT Navigator:** Builds complicated robot behaviors using behavior trees 
* **Nav2 Costmap 2D**: Converts sensor data into a costmap representation of the world 
* **AMCL:** Localizes the robot on the map 
* **Map Server:** Loads, serves, and store maps 
* **Nav2 Waypoint Follower:** Follows sequential waypoints 
* **Nav2 Lifecycle Manager**: Manage the lifecycle and watchdog for the servers 
* **Nav2 Core:** Plugins to enable your own custom algorithms and behaviors 


The Nav2 stack defines plugin interfaces for users to create their own custom costmap layer, planner, controller, behavior tree or recovery plugins. One of the powerful features of the Nav2 stack of packages is this ability to extend usage by adding more custom plugins that can be configured to be used through parameter files. Each plugin implements a specific behavior and supports a particular set of robot base platforms. These plugins can then be further configured to the platform and sensor specifications of the robot. 

In our case, the AWS [DeepRacer Evo](https://aws.amazon.com/deepracer/) robot is an 1/18th scale 4WD with monster truck chassis and Ackermann drive type platform with independent servo and motors to control the wheels. The AWS DeepRacer hardware consists of two independent forward facing 4 MP RGB cameras, a 360 degree planar LiDAR (restricted to view only 300 degrees with reverse orientation), an integrated accelerometer and gyroscope providing the IMU data. More details about the AWS DeepRacer hardware and simulation artifacts can be found [here](https://github.com/aws-deepracer/aws-deepracer).

There are two flows for using ROS Nav2 on AWS DeepRacer, the Simulation flow and the Device flow. Both flows have different prerequisites to work with Nav2 stack.

## **Coordinate frame standards and state estimation requirements in navigation stack**:

The [REP-105](https://www.ros.org/reps/rep-0105.html) (ROS Enhancement Proposal) specifies naming conventions and the semantic meaning for coordinate frames of mobile platforms used with ROS. The coordinate frame commonly called `base_link` is rigidly attached to the mobile robot base. The coordinate frame called `odom` is a world-fixed frame that has to be continuously updated. This frame helps to get the locally accurate localization. The coordinate frame called `map` is a world fixed frame, with its Z-axis pointing upwards. This can have discrete jumps and helps in determining global pose estimates. Along with these, there are few static transformations with the sensor pose data (`base_link → camera_link` and `base_link → laser`) published to indicate the positions of the sensors.

According to community standards, there are 2 major transformations involving the coordinate frames that need to be provided within the navigation project. The `map` to `odom` transform is provided by a positioning system (localization, mapping, SLAM) and `odom` to `base_link` by an odometry system. This `map → odom → base_link` frames and their corresponding TF transformations are the core requirements for any robot to work with the navigation stack.

<p align="center">
<img src="/media/dr-sim-tf-transforms-1.png" height="350" >
</p>


It is the job of the global positioning system (GPS, SLAM, Motion Capture) to, at minimum, provide the `map` -> `odom` transformation. **The navigation stack provides `amcl` which is an Adaptive Monte-Carlo Localization technique based on a particle filter for localization of a static map.** It also provides the SLAM Toolbox as the default SLAM algorithm for use to position and generate a static map. 

It is the role of the odometry system to provide the `odom` -> `base_link` transformation. Odometry can come from many sources including LiDAR, RADAR, wheel encoders, VIO, and IMUs. The goal of the odometry is to provide a smooth and continuous local frame based on robot motion. The global positioning system will update the transformation relative to the global frame to account for the odometric drift. [Robot Localization](https://github.com/cra-ros-pkg/robot_localization/) is typically used for this fusion. It will take in `N` sensors of various types and provide a continuous and smooth odometry to TF and to a topic. A typical mobile robotics setup may have odometry from wheel encoders, IMUs, and vision fused in this manor. The smooth output can be used then for dead-reckoning for precise motion and updating the position of the robot accurately between global position updates.

In AWS DeepRacer, the [deepracer_gazebo](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_gazebo) package contains the gazebo plugin responsible for providing the ground truth odometry information along with subscription to the Twist messages published on `/cmd_vel` topic in the simulation flow, and the [rf2o_laser_odometry](https://github.com/MAPIRlab/rf2o_laser_odometry) package provides the odometry information based on the consecutive LiDAR laser scan messages in the device flow. The [deepracer_nodes](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_nodes) contains the required nodes to convert the Twist messages published on `/cmd_vel` topic to servo messages.

## **Setup instructions**

### **Using Nav2 stack with AWS DeepRacer**

In this demo we walk through two different flows using the ROS Nav2 stack with AWS DeepRacer. In the simulation flow, we can see how to use the Nav2 *Navigation2Goal*, while using the Smac global planner, and regulated pure pursuit controller in AWS RoboMaker. In the device flow, we will walkthrough the clone, build and installation of the new packages that help in running Nav2 stack on the AWS DeepRacer device.

These two main parts are explained in detail below:

* **Part 1 - Simulation flow**
    * Part 1.1 - Clone and build the demo application in AWS RoboMaker Development Environment
    * Part 1.2 - Navigate the AWS DeepRacer using *Navigation2Goal* in Rviz and Gazebo simulation
* **Part 2 - Device flow**
    * Part 2.1 - Clone and build the robot packages on AWS DeepRacer device
    * Part 2.2 - Launch the AWS DeepRacer Robot related packages
    * Part 2.3 - Launch the ROS2 Navigation Stack
    * Part 2.4 - Visualize the robot in RViz

### **Prerequisites**

The simulation flow uses AWS RoboMaker integrated development environment. In order to complete this tutorial you’ll need an AWS account, with user permissions for AWS RoboMaker and [AWS Cloud9](https://aws.amazon.com/cloud9/).

### **Part 1.1 – Clone and build the demo application in AWS RoboMaker Development Environment.**

This section will guide you through cloning and building the demo application.

**1.1.1 – Create a development environment**

We will be creating an AWS RoboMaker integrated development environment (IDE) to run the AWS DeepRacer simulation. From the AWS RoboMaker menu, select Development Environments then Create environment.

<p align="center">
<img src="/media/dr-sim-robomaker-list-dev-env-1.png">
</p>

Name your environment **deepracer-nav2-env**, choose **Foxy (Latest)** as your ROS distribution, and click **Create**.

<p align="center">
<img src="/media/dr-sim-robomaker-create-dev-env-1.png">
</p>

After a few seconds, you should see that the development environment appears.

<p align="center">
<img src="/media/dr-sim-robomaker-dev-env-1.png" height="450" >
</p>

**1.1.2 – Clone and build the demo application**

From the bash terminal within the IDE, run the following commands to clone the demo application and install the needed dependencies.


```
mkdir -p deepracer_nav2_ws
cd deepracer_nav2_ws
git clone https://github.com/aws-deepracer/aws-deepracer.git

sudo apt update

rosdep install -i -r -y --from-paths .
cd aws-deepracer
rosinstall deepracer_description
```

The dependency install may take a few minutes to run. When completed, build the code.

```
colcon build
```

### **Part 1.2 – Navigate AWS DeepRacer using Navigation2Goal in Rviz and Gazebo simulation**

This section will guide you through viewing the simulation and navigating the AWS DeepRacer car.

**1.2.1 – Open the Virtual Desktop**

Open the Virtual Desktop in your Development Environment.

<p align="center">
<img src="/media/dr-sim-launch-virtual-desktop-1.png" >
</p>

After a few seconds, you should see that the Virtual Desktop opens up in a new tab.

<p align="center">
<img src="/media/dr-sim-virtual-desktop-1.png" height="450" >
</p>

**1.2.2 – Launch the nav_amcl_demo_sim flow**

Open a terminal in the Virtual Desktop

<p align="center">
<img src="/media/dr-sim-application-list-1.png" height="450" >
</p>

From the [deepracer_bringup](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_bringup) package, launch the `nav_amcl_demo_sim.launch.py`.


```
cd environment/deepracer_nav2_ws/aws-deepracer
export GAZEBO_RESOURCE_PATH=/usr/share/gazebo-11:{$GAZEBO_RESOURCE_PATH}
source install/setup.bash

ros2 launch deepracer_bringup nav_amcl_demo_sim.launch.py 
```

<p align="center">
<img src="/media/dr-sim-cli-1.png" height="450" >
</p>

After a few seconds, you should see the GZClient with the AWS DeepRacer device spawned in `aws-robomaker-bookstore-store` world. Enlarge the window, zoom, and scroll until you find the AWS DeepRacer in the center of the bookstore world.

<p align="center">
<img src="/media/dr-sim-gazebo-1.png" height="450" >
</p>

**1.2.3 – Using rviz tool to navigate the AWS DeepRacer car**

Next, open another terminal to launch rviz tool to send navigation goals to the AWS DeepRacer.

```
rviz2
```

<p align="center">
<img src="/media/dr-sim-rviz-1.png" height="450" >
</p>

Open the `nav2_default_view.rviz` file from the [deepracer_description](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_description) package. In the rviz window you should now see the view looking down at the robot location. Set the initial robot location by choosing the *2D Pose Estimate* tool from the toolbar, then placing the arrow pointing up from the robots location. The robot footprint should now appear as a green square, and the map should appear highlighted in color.

This means that the AWS DeepRacer is now ready to navigate! Use the middle mouse scroll to zoom out and get a view of the entire map. Use the *Navigation2 Goal* button on the tool bar to set a goal for the AWS DeepRacer to move forward in the map. The AWS DeepRacer should immediately begin moving to the location you selected.

<p align="center">
<img src="/media/dr-sim-open-rviz-1.gif" height="450" >
</p>
<p align="center">
<img src="/media/dr-sim-rviz-navigation2goal-1.gif" height="450" >
</p>

### **Clean up**

In order to clean up any resources used, open the AWS RoboMaker console.

1. Close your development environment
    1. You can either leave it for use another time 
    2. *OR* you can delete it by choosing **Development environments** from the AWS RoboMaker menu, choosing your environment from the list (ex: **deepracer-nav2-env**), then selecting **Delete**

### **Part 2.1 — Clone and build the robot packages on the AWS DeepRacer device**

This section will guide you through cloning and building the AWS DeepRacer robot packages required for ROS Navigation. As part of this, we shall fetch and install the third party ROS2 packages and the AWS DeepRacer packages required to make the AWS DeepRacer device compatible with ROS Nav2 stack. This provides the required interfaces and transforms for ROS Nav2 stack to initialize.

The third party ROS2 packages required to satisfy ROS Nav2 requirements for a robot include the [`rplidar_ros`](https://github.com/youngday/rplidar_ros2) package, [`rf2o_laser_odometry`](https://github.com/MAPIRlab/rf2o_laser_odometry) package. We also provide the [`aws-deepracer-camera-pkg`](https://github.com/aws-deepracer/aws-deepracer-camera-pkg), [`aws-deepracer-servo-pkg`](https://github.com/aws-deepracer/aws-deepracer-servo-pkg), `cmd_velocity_to_servo-pkg`, `enable_deepracer_nav_pkg`, `aws-deepracer_interfaces_pkg` and the transforms as part of the launcher in [deepracer_bringup](https://github.com/aws-deepracer/aws-deepracer/tree/main/deepracer_bringup) ([`deepracer.launch.py`](https://github.com/aws-deepracer/aws-deepracer/blob/main/deepracer_bringup/launch/deepracer.launch.py)).

**2.1.1 - Create a development environment**

To get started with ROS Nav2 bringup on the AWS DeepRacer, we first need to install ROS Navigation on the device. Follow the steps below to download and install ROS Navigation.

```
sudo apt-get update -y
sudo apt install ros-foxy-navigation2 -y
sudo apt install ros-foxy-nav2-bringup -y
```

**2.1.2 - Stop the deepracer-core.service that is currently running on the device**

```
systemctl stop deepracer-core
```

**2.1.3 - Download the robot related packages from AWS DeepRacer repository**

```
mkdir -p ~/deepracer_nav2_ws
cd ~/deepracer_nav2_ws
git clone https://github.com/aws-deepracer/aws-deepracer.git
cd ~/deepracer_nav2_ws/aws-deepracer/
```

**2.1.4 - Clone the [rf2o_laser_odometry](https://github.com/MAPIRlab/rf2o_laser_odometry) and [rplidar_ros](https://github.com/youngday/rplidar_ros2) dependency packages on the AWS DeepRacer device**

```
cd ~/deepracer_nav2_ws/aws-deepracer/deepracer_nodes && ./install_dependencies.sh
```

**2.1.5 - Fetch unreleased dependencies**

```
source /opt/ros/foxy/setup.bash 
cd ~/deepracer_nav2_ws/aws-deepracer/deepracer_nodes
rosws update
```

**2.1.6 - Build the robot packages**

```
cd ~/deepracer_nav2_ws/aws-deepracer/ && colcon build --packages-select deepracer_interfaces_pkg deepracer_bringup cmdvel_to_servo_pkg enable_deepracer_nav_pkg rf2o_laser_odometry rplidar_ros camera_pkg servo_pkg

```

**2.1.7 - (Optional) Use SLAM toolbox to create a map**

2.1.7.1 - Launch the AWS DeepRacer robot related packages (See Part 2.2):

```
source /opt/ros/foxy/setup.bash 
source ~/deepracer_nav2_ws/aws-deepracer/install/setup.bash
ros2 launch deepracer_bringup deepracer.launch.py
```

2.1.7.2 - Launch the ROS Navigation packages:

```
source /opt/ros/foxy/setup.bash
source ~/deepracer_nav2_ws/aws-deepracer/install/setup.bash
ros2 launch deepracer_bringup deepracer_navigation_dr.launch.py use_sim_time:=False params_file:=/root/deepracer_nav2_ws/aws-deepracer/deepracer_bringup/config/nav2_slam_params.yaml
```

2.1.7.3 - Launch the SLAM toolbox:

```
source /opt/ros/foxy/setup.bash
source ~/deepracer_nav2_ws/aws-deepracer/install/setup.bash
ros2 launch deepracer_bringup slam_toolbox.launch.py use_sim_time:=False params_file:=/root/deepracer_nav2_ws/deepracer/deepracer_bringup/config/slam_toolbox.yaml
```

2.1.7.4 - Create and save the map:
Using your favorite tele-op method (Example: [https://github.com/ros2/teleop_twist_keyboard](https://github.com/ros2/teleop_twist_keyboard)), move the AWS DeepRacer around the required space smoothly and slowly. You can view the map in RViz. To save this map to file:

```
`ros2 run nav2_map_server map_saver_cli -f ~/map`
```


<p align="center">
<img src="/media/dr-slam.gif" height="250" align="left">
<img src="/media/slam-hallway-map.png" height="250" >
</p>

### **Part 2.2 — Launch the AWS DeepRacer Robot related packages**

As part of this section, we launch the nodes required for the AWS DeepRacer to be compatible with ROS Nav2 stack. This launches the sensor packages and their transformations including LiDAR / Camera / Laser to Odom, as well as other bringup and motor related packages including deepracer_interfaces, cmdvel to servo converter, servo package.

**2.2.1 - Launch using the AWS DeepRacer launcher provided**

```
source /opt/ros/foxy/setup.bash 
source ~/deepracer_nav2_ws/aws-deepracer/install/setup.bash
ros2 launch deepracer_bringup deepracer.launch.py
```

### **Part 2.3 — Launch the ROS Navigation Stack**

As part of this section, we shall launch the Navigation stack using the nav_bringup we have installed as a prerequisite in 2.1.1. A well defined map is passed as a parameter for static localization and navigation plugin configurations are passed in the `nav2_params_nav_amcl_dr_demo.yaml` in deepracer_bringup.

```
source /opt/ros/foxy/setup.bash 
ros2 launch nav2_bringup bringup_launch.py use_sim_time:=False autostart:=True map:=<Path to Map> params_file:=<Path to YAML Config>
```

### **Part 2.4 — Visualize the robot in RViz**

To visualize the robot localization and navigation, launch RViz and open the `nav2_default_view.rviz` config file.

```
ros2 run rviz2 rviz2
```


Set the initial robot location by choosing the *2D Pose Estimate* tool from the toolbar.

<p align="center">
<img src="/media/dr-2d-pose-estimate-1.png" height="250"  align="left">
  <img src="/media/dr-2d-pose-estimate-2.png" height="250" >
</p>

Use the *Navigation2Goal* button on the tool bar to set a goal for the AWS DeepRacer car to move.

<p align="center">
<img src="/media/dr-navigate-to-goal-1.png" height="250" align="left">
<img src="/media/dr-navigate-to-goal-2.png" height="250" >
</p>

### **Conclusion**

These new additions to the AWS DeepRacer device software stack help in integrating ROS Navigation stack, and leverage the open source packages to navigate the car. 