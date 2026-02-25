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