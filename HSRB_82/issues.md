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


# Unsolved
## Python Interface
**Description:** errors when running 

docker exec -it docker.humble.diag.service /bin/bash -c ". /etc/opt/tmc/robot/ros2_setup.sh;ros2 run hsrb_interface_py ihsrb.py"

**Solution:** needs something running on host pc?
- seems to time out waiting for a service
- Errors with ddsi_udp_conn_write to udp/<IP> fixed by ROS_DOMAIN_ID != 0 I believe

## Can't access hand camera
**Description:** hand camera not accessible through rviz

Also can't access through cheese or kamoso
- though also aren't able to access head center cam which we can access through rviz so maybe just prohibited by some hsr thing --- *Fixed itself later*

#### Tried:
	v4l2-ctl --device=/dev/video2 --stream-mmap --stream-count=3 --verbose

Throws:
```
        VIDIOC_REQBUFS returned 0 (Success)
		VIDIOC_QUERYBUF returned 0 (Success)
		VIDIOC_QUERYBUF returned 0 (Success)
		VIDIOC_QUERYBUF returned 0 (Success)
		VIDIOC_QUERYBUF returned 0 (Success)
		VIDIOC_QBUF returned 0 (Success)
		VIDIOC_QBUF returned 0 (Success)
		VIDIOC_QBUF returned 0 (Success)
		VIDIOC_QBUF returned 0 (Success)
		VIDIOC_STREAMON returned -1 (Input/output error)
```

Interesting:\
This gives same error for video2 (center cam that works)
But it works on 41...

#### Tried:
`gst-launch-1.0 pipewiresrc path=34 ! videoconvert ! autovideosink`

** path=34 is from path= listed from `gst-device-monitor-1.0 Video/Source`

Get: 
black screen, not recieving data...

Even nothing when doing `gst-launch-1.0 pipewiresrc path=34 ! videoconvert ! video/x-raw,format=RGB ! fakesink dump=true` which should cat the raw bits.

#### Update following day:

These commands now work for video2 at least, but cheese still isn't working

#### Next steps:   
- check these commands on 41
- open up case and check wiring (hsrb.io has some guidance here)

#### Current status:
Cheese  itself now works on both hsrs

BUT switching feed to (82) hand camera freezes last head camera frame (presumably no data is being sent to overwrite)

Therefore likely disconnected cable (1011 or 1008)
- 1006 seems fine found on the bottom of the cpu (bottom)

#### Partial solution:

This seems to fix camera for command line streaming, **still NOT working in rviz**. Make sure that the device is the hand cam should be video0, **but might be swapped to video2.**

```
v4l2-ctl --device=/dev/video0 --set-ctrl=exposure_auto=1
v4l2-ctl --device=/dev/video0 --set-ctrl=exposure_absolute=300
v4l2-ctl --device=/dev/video0 --stream-mmap --stream-count=3 --verbose
```

Potentially needed to be executed prior to the above commands (resets usbs, but also **might switch video0 and video2**):
```
echo 0 | sudo tee /sys/bus/usb/devices/1-1.4/authorized
sleep 5
echo 1 | sudo tee /sys/bus/usb/devices/1-1.4/authorized
```

`v4l2-ctl --list-devices` should result in the following if the video channels are right:
```
NCM13-J-02 (usb-0000:00:1a.0-1.4):
	/dev/video0
	/dev/video1
	/dev/media0

NCM13-J-02 (usb-0000:02:00.0-1.3.1):
	/dev/video2
	/dev/video3
	/dev/media1
```

Likely explanation:

We were getting an initial frame some of the time, but only the first frame or two. Claude's explanation of this phenomenon is below.

When a UVC camera powers up, the image sensor goes through an initialization sequence — it charges up, sets exposure, and produces a burst of frames before any control negotiation happens. Those first 2-3 frames are essentially "startup frames" from the sensor's cold start. After that, the camera firmware tries to apply the UVC control settings (including control 8, exposure), fails with the -32 error, and the sensor either shuts down or enters a fault state. This is why you see data briefly then nothing.

#### Final Solution:

The usb-b mini cable in the hand was bent, swapping that and plugging into usb port on the robot allows the camera to function. (USB bar on top didn't have enough bandwidth, but the silver ones on the bottom right did)

**TODO:** Full solution will likely be stripping the cable and splicing on the new head as rerouting cable seems extremely difficult. Awaiting instruction from Taskin. 

## Can't access stereo head cameras
**Description:** Can't access cameras from rviz

**Attempts:**
The command `flycap` allows us to access the individual cameras... so not hardware **(can be used to identify left and right)**

KEY ERROR: 
`Not found key 'cameras' or 'cameras' node type is not sequence.`

This is from the `src/tmc_drivers/tmc_pgr_camera/src/tmc_pgr_camera/yaml_point_grey_camera_system_setting.cpp` file

Error is while parsing camera ids in line 192.

This led me to looking at the stereo camera yaml file in `/etc/opt/tmc/robot/conf.d/calib_results/stereo_camera_params.yaml` (specified in line 47 of `src/hsrb_robot/hsrb_bringup/hsrb_bringup/launch/sensors/stereo_camera.py`). The file is in the wrong format according to the comments in `yaml_point_grey_camera_system_setting.cpp`.

**Solution:**
Replace with the proper [file](Files/Files%201//stereo_camera_params.yaml) that only includes the camera id's (found in flycap).

The LEFT camera (the one on the right if you are looking at the face) should be listed as master. Make sure to check that the id's are correct, probably not the ones listed.

*Note: The point grey camera system mentions additional parameters, but that was deprecated and relocated to `src/tmc_drivers/tmc_pgr_camera/launch/capture.launch.py`*