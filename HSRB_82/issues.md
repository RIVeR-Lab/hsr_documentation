This augments the documentation found here:
https://git.hsr.io/tmc/hsr-ros2-setup/-/blob/main/README-EN.md

# Steps that we needed help with
## Unplug HDMI
**Description:** instructions say we must disconnect the head display HDMI
**Solution:** remove cover and unplug hdmi cable labeled 1030 on the cpu board on the right side (looking at it from the front)

Long term: need male VGA to female HDMI adapter that connects the VGA port by the E-Stop to the disconnected male HDMI

## Reinstalling OS
**Description:** doing fresh ubuntu installation we got intermitent errors preventing loading into new boot

**Solution:** do a clean installation with the distribution listed

We had lots of trouble getting into the computer after installation \
No fix was found, but it eventually just booted in properly part of the time
- Continuing the installation seems to have resolved the issues

If you come to a blank black screen, press any key and it will continue -- this happened a few times after booting in recovery mode

## Network configuration 
**Description:** post running ansible on host PC network not working

**Solution:** Ran the following command to bring the network back up and running before being able to connect again:

```sudo ip link set wlp3s0 up```

Also make sure wifi is on with:

```sudo nmcli radio wifi on```

Even with that fix, can't use the wifi options in settings, must use the `sudo wpa_gui` mentioned in the docs

Also turning of wifi in settings and turning back on breaks it, but fixed with a reboot

## Rewriting Arduino
**Description:** arduino didn't have access to the serial port
**Solution:** add the user to the `dialout` group in order to have access to the serial port /dev/ttyUSB0:

```sudo usermod -a -G dialout <USER>```

Then run `newgrp dialout` to apply the changes

## Restoring Arduino Software for ROS1
We didn't do it, figuring that we do want the ROS2 not ROS1 arduino
- Might revisit though because of Calibration section specifically

## Creating Docker Image
**Description:** permission denied for ./make.sh
**Solution:** use `sudo ./make.sh`

Additionally This took over an hour, so be patient.

## Running ROS2 Setup with Ansible
**Description:** permission denied when running ansible commands
**Solution:** add `sudo`

## Setting up Cyclone DDS
**Description:** would error out (forget what specifically)
**Solution:** use ROS_DOMAIN_ID != 0 

## Camera calibration
**Description:** cameras cannot launch because of missing calibration files. The camera calibration files already exists but they are not located in the correct directory the bringup launch file is pointing to.

**Solution:** some of the calibration files are located in /hsrb_robot/hsrb_bringup/config, others in the correct directory but some information is wrong.

Replace all of the files in the following directories:
- Files from folder *Files 1* to /etc/opt/tmc/robot/conf.d/calib_results
- Files from folder *Files 2* to  ~/.ros/camera_info

Then, in hsr_base.py (hsrb_bringup package), line 91, replace the camera_name with 'tgb_PS1080_PrimeSense'

## Gripper issue
**Description:** ros2 action set_distance, the gripper keeps opening and closing.

**Solution:** the PID gains are set too high. Changed them in hsrb_controllers/hsrb_gripper_controller/src/hrh_gripper_set_distance_action.cpp package.

## tmc_pgr_camera package missing
**Description:** steps in HSR documentation make us remove the tmc_pgr_camera package. However, this is needed to launch the rgbd head camera.

**Solution:** install flycapture (https://github.com/RhobanDeps/flycapture/tree/master). In head_center_camera.yaml, change video_device to /dev/video2 (video0 is gripper camera).

## PATH_TOLERANCE_VIOLATED
**Description:** base does not move because of the path_tolerance_violated error. This is because the odometry is not updating, the error between current and desired base pose keeps increasing. 

**Solution:** in omni_base_controller.cpp, update the base_odometry to be the same as the wheel_odometry, since that is updating correctly.


# Unsolved
## Python Interface
**Description:** errors when running 

docker exec -it docker.humble.diag.service /bin/bash -c ". /etc/opt/tmc/robot/ros2_setup.sh;ros2 run hsrb_interface_py ihsrb.py"

**Solution:** needs something running on host pc?
- seems to time out waiting for a service
- Errors with ddsi_udp_conn_write to udp/<IP> fixed by ROS_DOMAIN_ID != 0 I believe