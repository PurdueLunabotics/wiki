---
layout: article
date: 2020-06-08
title: Software Documentation 
author: Raghava 
---

# Docs

## How to use this if you're new

From the diagram and subsection descriptions, you can hopefully get a preliminary high-level understanding of each of the parts and how they fit together. 

From this, you can then:

1. Choose which subsection interests you most 
2. Follow links in the specified subsection to get more contextual understanding (Optionally ask questions in the Discord, hop in a voice chat, etc for more fun convo about the area)
3. Ensure you understand the related topics for each subsections with practical understanding/experience by completing a step-by-step tutorial using those prerecs (**Can be done concurrently with #2**)
4. At this point, you should have enough knowledge/experience in your subsection to contribute to the current approach and possibly even change what the current approach is if you find better ways to do things (if so, update these docs!) 

## Architecture Diagram 

```mermaid
graph
  c4[Communications] --> d2
  e1[Human Operator] --> c4
  subgraph Low Level
    direction LR
    a1[Realsense T265 Camera ROS driver]
    a2[Camera driver]
    a3[RPLiDAR driver]
    a4[Embedded control]
  end
  subgraph Mid Level 
    direction LR
    c1[Navigation] --> a3
    c2[Localization] --> a1
    c3[Task-level control] --> a4 
  end
  subgraph High Level 
    d1[Planner]-->d2
    d2[Execution controller]-->c1
  end
  %% execution controller 
  d2-->c2
  d2-->c3
  d2-->c4

  %% Camera and comms.  
  c4-->e1
  c1-->a2
  c2-->a2

  %% Links
  click e1 href "http://wiki.purduelunabotics.org/wiki/software/docs#plannerhuman" "Sensors"
  click c4 href "http://wiki.purduelunabotics.org/wiki/software/docs#communications" "Sensors"

  click a1 href "http://wiki.purduelunabotics.org/wiki/software/docs#sensor-drivers" "Sensors"
  click a2 href "http://wiki.purduelunabotics.org/wiki/software/docs#sensor-drivers" "Sensors"
  click a3 href "http://wiki.purduelunabotics.org/wiki/software/docs#sensor-drivers" "Sensors"
  click a4 href "http://wiki.purduelunabotics.org/wiki/software/docs#embedded-control" "Embedded Control"

  click c1 href "http://wiki.purduelunabotics.org/wiki/software/docs#navigation" "Sensors"
  click c2 href "http://wiki.purduelunabotics.org/wiki/software/docs#localization" "Sensors"
  click c3 href "http://wiki.purduelunabotics.org/wiki/software/docs#task-level-control" "Sensors"

  click d1 href "http://wiki.purduelunabotics.org/wiki/software/docs#plannerhuman" "Sensors"
  click d2 href "http://wiki.purduelunabotics.org/wiki/software/docs#execution-controller" "Sensors"

  %% Colors 
  classDef not_started fill:#ff8181
  classDef in_progress fill:#ffba82
  classDef done fill:#81ff9b
  class c3,c4,d1,d2 not_started
  class a4,c1,c2 in_progress
  class a1,a2,a3,e1 done
```
```mermaid
graph
  l1[Not Started]
  l2[In Progress]
  l3[Done]
classDef not_started fill:#ff8181
classDef in_progress fill:#ffba82
classDef done fill:#81ff9b
class l1 not_started
class l2 in_progress
class l3 done
```

## High-level

### Planner/Human 

**Problem**

At any given time, how does the robot know what to do next? This is the job of the planner. 

**Current solution**

To do this, it utilizes an [FSM](https://en.wikipedia.org/wiki/Finite-state_machine), which models all the states the robot can be in and how they are connected and then deterministically chooses which action to complete next, then feeds it to the execution controller.

Also, if necessary, the human operator can also determine which actions to complete and can provide that information to the execution controller.

**Related topics: to implement**
- [ROS](http://wiki.purduelunabotics.org/wiki/software/docs#ros)
- [Python](http://wiki.purduelunabotics.org/wiki/software/docs#python)
- [FSM](http://wiki.purduelunabotics.org/wiki/software/docs#fsm)

### Execution Controller

Regardless of who is sending the commands, there needs a controller that make a request to complete a certain task and then respond if the task was successful, allowing for the planner or manual controller to execute the next command. This is the job of the execution controller. Also, it can provide helpful logging information. 

**Related topics: to implement**
- [ROS](http://wiki.purduelunabotics.org/wiki/software/docs#ros)
- [Python](http://wiki.purduelunabotics.org/wiki/software/docs#python)
- Understanding of how the different mid-level nodes are implemented

## Mid level

### Navigation

**Problem**

Given a current location and target location, how do we move between current and target location without running into obstacles efficiently?

**Current Solution**

It is feasible to use the A star path planning algorithm and populate obstacles using LiDAR + camera fusion. 

**Related topics:**

For pure navigation:
- [A star algorithm](path-planning-algorithms)

For obstacle detection:
- [ROS](http://wiki.purduelunabotics.org/wiki/software/docs#ros)
- [Python](http://wiki.purduelunabotics.org/wiki/software/docs#python)
- [LiDAR](http://wiki.purduelunabotics.org/wiki/software/docs#lidar)
- [Camera](http://wiki.purduelunabotics.org/wiki/software/docs#normal-camera)
- [Sensor fusion](http://wiki.purduelunabotics.org/wiki/software/docs#sensor-fusion)

### Localization

**Problem**

Where is the robot at any given time? Knowing this with high accuracy is really critical to other subsections such as navigation. But, a problem quickly emerges that each sensor that we use can output trashy data (i.e camera gets dust on it, wheels slip so encoders are off, liDAR can [behave weirdly](https://www.nasa.gov/sites/default/files/atoms/files/13_rmc2017_glossary_rev01_2c_10042016.pdf)). So, how do we use this?

**Current Solution**

We can create a sensor-redundant robust localization system, including both intrinsic and extrinsic locaization.

Intrinsic localization is the robot's location from its starting point and can be determined in any environment, as it does not rely on external references. 

This is currently done with the Intel Realsense T265 localization camera. Using wheel encoders and calculating odometery is not dependable as the terrain results in lots of slippage, therefore adding lots of error. 

Extrinsic localization is the robot's location from the collector bin, which is the ground truth to the rest of the robot's environment, since it we know the environment in reference to the bin with decent accuracy prior to the actual competition. 

This is currently planned to be done using [sensor fusion](https://www.sciencedirect.com/topics/engineering/sensor-fusion) between distance measurements from the BLE signals emitted by a BLE beacon between the robot and a tracker attached to the bin, and triangulated distance measurements of Apriltags. It could be possible to use LiDAR as well.

**Related topics: to implement**

- [ROS](http://wiki.purduelunabotics.org/wiki/software/docs#ros)
- [Python](http://wiki.purduelunabotics.org/wiki/software/docs#python)
- [Kalman Filters](http://wiki.purduelunabotics.org/wiki/software/docs#sensor-fusion)
- [Using Apriltags](http://wiki.purduelunabotics.org/wiki/software/docs#apritags)
- [T265 camera](http://wiki.purduelunabotics.org/wiki/software/docs#t265)

### Communications

The robot we're building is designed to operate on the Moon. Naturally, there is bandwidth and latency limitations that we need to take into consideration. 

Therefore, having most of the compute and decision making done on the robot itself and using ROS's [Multi Machine] approach, seems the most reliable and bandwidth saving system.

One tradeoff of limiting bandwidth usage and latency is less diagnostics and realtime feedback from the robot (meaning that we would need more confidence and testing to ensure system performance).

**Related topics:**
- [Python](http://wiki.purduelunabotics.org/wiki/software/docs#python)
- [Communication/Networking](http://wiki.purduelunabotics.org/wiki/software/docs#communicationnetworking)

### Task-level Control 

In past competition, the goal of the lunar robot was to excavate gravel. From this, the tasks such as excavation of gravel and then deposition of the gravel into the bin would be converted to higher-level instructions that interact with the embedded controllers of each subsystem to follow through the execution of the task. 

Similar to excavation or deposition, any other new task closely follows a similar pattern. 

**Related topics:**
- [Python](http://wiki.purduelunabotics.org/wiki/software/docs#python)
- [ROS](http://wiki.purduelunabotics.org/wiki/software/docs#ros)
- [PID Control](http://wiki.purduelunabotics.org/wiki/software/docs#pid-control)

## Low level

### Sensor Drivers 
The camera, liDAR, BLE beacon all output raw data in bytes. This information needs to be processed and packaged up to make it easier for other components to use it.

This is done mainly through using existing ROS packages that are linked below 

**Related topics:**
- [BLE Beacons](http://wiki.purduelunabotics.org/wiki/software/docs#ble-beacons)
- [Camera](http://wiki.purduelunabotics.org/wiki/software/docs#normal-camera)
- [LiDAR](http://wiki.purduelunabotics.org/wiki/software/docs#lidar)
- [T265](http://wiki.purduelunabotics.org/wiki/software/docs#t265)

### Embedded control 

Each hardware-related system (i.e drivetrain, excavation, deposition) has a controller that ensures the execution of a given command using a simple control scheme with switches and encoders (i.e PD loop to ensure that the given set speed is reached by each wheel). 

**Related topics:**
- [Arduino](http://wiki.purduelunabotics.org/wiki/software/docs#arduino) and [C++](http://wiki.purduelunabotics.org/wiki/software/docs#c++)
- [ROS](http://wiki.purduelunabotics.org/wiki/software/docs#ros)
- [PID Control](http://wiki.purduelunabotics.org/wiki/software/docs#pid-control)

# Resources

**Competition Guidelines**: Not released yet 

## Tools

### ROS  

- [ARC ROS tutorials (Recommended)](https://wiki.purduearc.com/wiki/tutorials/snake-tutorial)
- [Official ROS tutorials](http://wiki.ros.org/ROS/Tutorials)

### GitHub
- [Learn Git under 10 min](https://www.freecodecamp.org/news/learn-the-basics-of-git-in-under-10-minutes-da548267cc91/)

### C++
- [In depth tutorial playlist for C++](https://www.youtube.com/watch?v=18c3MTX0PK0&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb) 
  - Recommended topics:
    - if/else, loops, functions, classes
    - Pointers/References
    - Smart pointers
    - Dynamic Arrays (std::vector) 
- Very useful numeric libraries
  - [Eigen](https://eigen.tuxfamily.org/index.php?title=Main_Page): Extremely efficient matrix math library

### Python/Numpy

**Python**

- [Tutorial](https://www.youtube.com/watch?v=rfscVS0vtbw)
- Important topics to understand: 
  - Basics are good - variables + logic, functions, classes

**Numpy** 
> Must use when working with large arrays (i.e images)

- [Tutorial](https://www.youtube.com/watch?v=8Y0qQEh7dJg) 
- Important topics to understand: 
  - Creating arrays
  - slicing + indexing
  - reshaping
  - linear algebra

**OpenCV**: 
> Use for computer vision and image transformations like color detection, tracking, etc

- [Tutorial](https://www.youtube.com/watch?v=oXlwWbU8l2o) 
- Important topics to understand: 
  - image transformation
  - thresholding
  - read/write images from file
  - resizing
  - tracking

### Arduino
- Top hit for "arduino tutorial" on google should work

### Good coding practices 
- [Optimized C++ code](https://cpp-optimizations.netlify.app/)

TODO

## Algorithms, Theory, Math 

### Sensor Fusion
Sensor fusion uses **Kalman Filters** (or a particle filter) heavily, so understanding those should be good.

- [tutorial walkthrough with code in python](https://datascopeanalytics.com/blog/sensor-fusion-tutorial/)
- [solid conceptual understanding](https://www.kalmanfilter.net/default.aspx)
- [explanation of a particle filter](https://www.youtube.com/watch?v=aUkBa1zMKv4)

### Path Planning Algorithms 

- [A star pathfinding algorithm](https://medium.com/@nicholas.w.swift/easy-a-star-pathfinding-7e6689c7f7b2)
- [navigation ros repo](http://wiki.ros.org/navigation)
  - [ros package outputs velocity commands given a map and costmap](http://wiki.ros.org/base_local_planner?distro=noetic)

### FSM 
- [tutorial slideshow](https://www.clear.rice.edu/engi128/Handouts/Lec17-Robotics.pdf)

### Control Algorithms 

#### PID Controller 
- [tutorial](https://ctms.engin.umich.edu/CTMS/index.php?example=Introduction&section=ControlPID)

## Sensor-related 

### BLE Beacon 
> not complete, can google the following phrases as well

- Purchasing a BLE beacon
- [Adding WiFi/BLE receiving functionality to a Jetson](https://www.jetsonhacks.com/2019/04/08/jetson-nano-intel-wifi-and-bluetooth/)
- Recieving/processing BLE data using ROS

### LiDAR
- [RPLidar ROS Library](https://github.com/slamtec/rplidar_ros)
  - Processes LiDAR data using ROS

### Camera
#### T265
- [quick start instructions](https://github.com/IntelRealSense/realsense-ros#using-t265)
  - Processes t265 camera data using ROS

#### Normal Camera 
- [Camera driver ROS package](http://wiki.ros.org/cv_camera)
  - Processes camera data using ROS
- [Camera calibration ROS package](http://wiki.ros.org/camera_calibration)

## Misc

### Communication/Networking 
> not complete, can google the following phrases or ask on discord

- Monitoring bandwith/latency usage from a router
- Understanding how ROS messages (manual control messages sent to the robot and diagnostic data sent back to team computer)
