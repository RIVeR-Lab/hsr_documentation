Put/replace these files in /etc/opt/tmc/robot/conf.d/calib_results

In hsr_base.py (hsrb_bringup package), line 91, replace the camera_name with 'tgb_PS1080_PrimeSense' 
- Was head_rgbd_sensor, presumably refers to rgbd_sensor_rgb_camera_params.yaml in which case it should actually be 'rgb_PS1080_PrimeSense'
