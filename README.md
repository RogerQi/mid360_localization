# mid360_localization

This repo supports three localization mode using livox mid360 lidar.

- FAST-LIO-only (without backend)
- Online SLAM (with loop closure)
- Localization only (requires pre-built map)

## Installation

Prereq: ROS and build-essentials.

Disclaimer: I've only tested it on an AGX Orin with ROS Noetic. This should support ROS2, but not tested.

Follow the README instructions in these two repos to compile the codes. You don't need to clone code again or repeat installation for the third party packages --- I have merged the duplicate third party packages in this repo.

([here](https://github.com/engcang/FAST-LIO-SAM-QN/tree/master) and [here](https://github.com/engcang/FAST-LIO-Localization-QN/)).

I suggest using a standalone ROS workspace with only this repo because it requires some customized compile flags that may affect other packages.

Set up livox mid360 with ethernet cable and power supply however you want.

## Check before start

If both hardware and software are properly setup, you should be able to invoke LIVOX driver by running,

```bash
roslaunch livox_ros_driver2 rviz_MID360.launch
```

## Running localization

### Mode 1: FAST-LIO-only

Since the repo is modularized, and we use FAST-LIO as a third party package, which is also setup in the ROS environment, you can simply follow instructions in [official FAST LIO](https://github.com/hku-mars/FAST_LIO) to run the localization.

### Mode 2: Online SLAM

Set up a terminal session to run lidar

```bash
roslaunch livox_ros_driver2 msg_MID360.launch
```

Then, set up another session to build the map.

```bash
roslaunch fast_lio_sam_qn run.launch lidar:=livox
```

There are three caveats (坑) that I've run into here

1. The config files provided by the SAM_QN and the LOCALIZATION_QN files are obsolete and do not work with livox mid 360.
   I have updated the config in these two commits to customize these two repos to work with livox mid 360.
2. The code is not very specific about where the constructed point cloud is dumped. The path is at TBD. Note that this path gets overwritten every time it runs. So be sure to manually save the map after reconstruction
3. The loop closure mechanism in SAM_QN was designed for outdoor scenes, such as self-driving. Therefore, for indoor scene, the loop closure sometimes actually performs worse than front-end-only mapping. I suggest using two runs to build the map (with/without loop closure). Save both bags. And check which one is better afterwards manually. To turn off loop closure, set `loop_detection_timediff_threshold` in [this file](https://github.com/RogerQi/mid360_localization/blob/main/FAST-LIO-SAM-QN/fast_lio_sam_qn/config/config.yaml#L12) to be 9999999.

### Mode 3: Localization-only with pre-built map

**Note: to use this mode, the user must have saved the built .bag (should be just handful of megabytes with key lidar frames+poses) from mode 2 run.**

Turn on the livox sdk first

```bash
roslaunch livox_ros_driver2 msg_MID360.launch
```

Provide the keyframes BAG file by updating the `saved_map` parameter in [this file](https://github.com/RogerQi/mid360_localization/blob/main/FAST-LIO-Localization-QN/fast_lio_localization_qn/config/config.yaml#L4).

Then, run,

```bash
roslaunch fast_lio_localization_qn run.launch lidar:=livox
```

