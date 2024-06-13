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

I suggest using a standalone ROS workspace with only this repo because it requires some customized compile flags that may affect other packages. Let's use this for an example:

```bash
cd ~
# Name for workspace for ROS. Change to yours
mkdir mid360_ws && cd mid360_ws
mkdir src && cd src
git clone --recursive https://github.com/RogerQi/mid360_localization
```

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

After the mapping is complete. Do `Ctrl+C` once. If both `save_map_pcd` and `save_map_bag` flags are true (see tips below), the built map and keyframes will be saved to the project directory of `FAST_LIO_SAM_QN`, which in my case is `~/mid360_ws/src/mid360_localization/FAST-LIO-SAM-QN/fast_lio_sam_qn`.

There are some caveats (坑) that I've run into here. Some tips as well

1. The config files provided by the SAM_QN and the LOCALIZATION_QN files are obsolete and do not work with livox mid 360.
   I have updated the config in these two commits to customize these two repos to work with livox mid 360.
2. This runs SLAM and saves built keyframes+map into directory given above, which can be used for localization. Note that this path gets overwritten every time it runs. So be sure to manually save the map after reconstruction
3. The loop closure mechanism in SAM_QN was designed for outdoor scenes, such as self-driving. Therefore, for indoor scene, the loop closure sometimes actually performs worse than front-end-only mapping. I suggest using two runs to build the map (with/without loop closure). Save both bags. And check which one is better afterwards manually. To turn off loop closure, set `loop_detection_timediff_threshold` in [this file](https://github.com/RogerQi/mid360_localization/blob/main/FAST-LIO-SAM-QN/fast_lio_sam_qn/config/config.yaml#L12) to be 9999999.
4. If you get too many noises, you can change the `blind` parameter in the front end config [here](). This parameter will cause the front end to ignore points that are too close (smaller than eps) to the lidar. In my experience you don't need to tune this and it will still be pretty robust.
5. The `save_map_pcd` in the [config.yaml](https://github.com/RogerQi/mid360_localization/blob/main/FAST-LIO-SAM-QN/fast_lio_sam_qn/config/config.yaml) file generates a built merged point cloud in the world. The `save_map_bag` generates rosbag with keyframes that can be used to build this point cloud.
6. Though lidar-imu calibration would conceptually improve the quality, in my experience it is not required.

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

Caveats (坑) for mode 3:

- The localization may not work in the beginning. It requires some form of initialization (in my case, I have to move the robot arund for ~1m). From the RVIZ, before the localization is properly initialized, the RVIZ is dark. After initialization, you should be able to see the whole pre-built map.
- 

