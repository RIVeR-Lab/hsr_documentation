# Robot bringup
## Base
ros2 launch hsrb_brinup robot.launch.py

ros2 launch hsrb_common_launch hsrb_common.launch.py

## Movit
ros2 launch hsrb_moveit_config hsrb_demo.launch.py

ros2 launch hsrb_moveit_config hsrb_example.launch.py example_name:=moveit_ik_demo