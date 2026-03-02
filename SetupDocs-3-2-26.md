## About This Document

This document explains the procedure for installing Ubuntu 22.04 + ROS 2 Humble on HSR-B/C.


## Limitations of HSR-B

There are restrictions when using the head display.

* Installation and robot startup are not possible when the head display is connected.
* Therefore, [remove the body cover](https://docs.hsr.io/hsrb_maintenance_manual_en/src/remove_cover.html#id19) and unplug the HDMI cable from the head display at the cpu board.
* Instead, connect an appropriate display.

After installation and startup, you can use the head-mounted display by configuring the [settings](https://git.hsr.io/tmc/hsr-ros2-setup/-/blob/main/README-EN.md?ref_type=heads#hsr-b-only-head-display-setting).

## Preparation

It is assumed that the following two PCs are on the same network:

* Host PC (with Ubuntu installed and capable of running the latest version of Ansible).
* HSR internal PC.

### Backup Files from the HSR

Log into the HSR internal PC with the administrator user and back up necessary files, such as user-created map data, to the host PC.

Additionally, back up the following two directories to the host PC:

- `/opt/tmc/vt`
- `/etc/opt/tmc/robot/conf.d`

```shell
$ sudo tar cjf vt.tar.bz2 /opt/tmc/vt
$ sudo tar cjf conf.d.tar.bz2 /etc/opt/tmc/robot/conf.d
$ scp vt.tar.bz2 conf.d.tar.bz2 <Host PC>
```

### Installing Ansible on the Host PC

Reference:
https://docs.ansible.com/ansible/2.9_ja/installation_guide/intro_installation.html#ubuntu-ansible

Run the following commands on the host PC:

```shell
$ sudo apt update
$ sudo apt install software-properties-common
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible
```


## OS Setup for the HSR Internal PC

### Obtaining Setup Configuration Files

On the host PC, obtain the complete set of setup configuration files from the following repository:

```shell
$ git clone https://git.hsr.io/tmc/hsr-ros2-setup
```

### Reinstalling the OS

On the HSR internal PC, perform a clean installation using the Ubuntu 22.04 ISO image.

- Set the username as administrator/password.
- The installation was tested using ubuntu-22.04.5-desktop-amd64.iso obtained from https://releases.ubuntu.com/jammy/.

After installation, configure the PC to run Ansible.

Add the packages necessary for Ansible:

```shell
$ sudo apt update
$ sudo apt install python3 python3-apt gpg ssh
```

To allow Ansible to use sudo, add the following configuration:

```shell
$ sudo visudo
```

Modify the line starting with %sudo to include NOPASSWD: before ALL, as follows:

```
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) NOPASSWD: ALL
```

### Running the OS Setup with Ansible

Execute the following on the host PC:

```shell
$ cd /path/to/hsr-ros2-setup
```

First, create the repository list for apt:

```shell
$ make
```

This will generate files starting with package_list_ under ansible/roles/update.

Next, prepare for Ansible execution:

```shell
$ cd /path/to/hsr-ros2-setup/ansible
```

Modify the configuration file. Use the `notmc` configuration.

- Change the hostname under `[robot]` to the HSR internal PC’s hostname or IP.
- Modify the variables under `[robot:vars]`:
    - `RELEASE_VERSION=ros2.humble`
    - `ROBOT_VERSION=<HSR version>`
    - `LOCALE=locale to use`

Once this is set, run the following:

```shell
$ ansible-playbook -i notmc setup.yml
```

For HSR-B, the network configuration will fail with a "Connection timed out" error.
Follow the next steps to configure the network.

For HSR-C, restart the HSR internal PC after completion.

### (HSR-B Only) Network Configuration

Work directly on the HSR internal PC by connecting a keyboard.

1. `sudo enable_wireless`
2. Reboot the HSR internal PC.
3. `sudo wpa_gui`
4. Configure the network and save.

### (HSR-B Only) Head display Setting

Disable the sleep setting.

 ```shell
 $ sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
 ```
If you disable the sleep setting, you will get a lot of logs, so rewrite `logind.conf` as follows.

  - `/etc/systemd/logind.conf`

  ```
  HandleSuspendKey=ignore
  HandleHibernateKey=ignore
  HandleLidSwitch=ignore
  HandleLidSwitchExternalPower=ignore
  HandleLidSwitchDocked=ignore
  ```

Next, disable Screen Blank on the Ubuntu login screen.
Replace the timeout value in `greeter.dconf-defaults` with 0.
- `/etc/gdm3/greeter.dconf-defaults`

```
[org/gnome/settings-daemon/plugins/power]
sleep-inactive-ac-timeout=0
sleep-inactive-battery-timeout=0
```
In addition, disable Screen Blank in the GUI power management.

After setting up, the head display can be used by following any of the steps below.

* Reconnect head display
  * Turn on the power with the head display disconnected.
  * Connect the head display after the robot starts up.

* Attach VGA to HDMI adapter
  * Attach the [VGA to HDMI adapter](./docs/images/conversion_vga2hdmi.png) before turning on the power.
    * [This product](https://www.amazon.co.jp/HOUREIHOU-%E9%87%91%E3%83%A1%E3%83%83%E3%82%ADVGA%E2%86%92HDMI-%E3%83%93%E3%83%87%E3%82%AA%E5%A4%89%E6%8F%9B%E3%82%A2%E3%83%80%E3%83%97%E3%82%BF-%E7%B5%A6%E9%9B%BB%E7%94%A8USB%E3%82%B1%E3%83%BC%E3%83%96%E3%83%AB%E4%BB%98%E5%B1%9E-HDMI%E5%A4%89%E6%8F%9B%E3%82%A2%E3%83%80%E3%83%97%E3%82%BF/dp/B0CGMCSY1G/ref=asc_df_B0CGMCSY1G?mcid=3994af7781023a2ca1358cf49df63f9a&tag=jpgo-22&linkCode=df0&hvadid=707458733439&hvpos=&hvnetw=g&hvrand=11587541420171319086&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=1009461&hvtargid=pla-2246683922545&psc=1&gad_source=1) has been tried for VGA to HDMI conversion.
  * Turn on the power


### Updating cti-serial

Work on the HSR internal PC.

First, verify that the kernel has been updated correctly:

```shell
$ uname -a
Linux hsrc 5.15.96-rt61-hsr-tmc #tmc.20241202.0806 SMP PREEMPT Mon Dec 2 08:07:49 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux 
```

If it shows `5.15.96-rt61`, the update was successful.

Next, update cti-serial.
Before starting, use scp to transfer `cti-serial-dkms_1.46.1-0_all.deb` file from the host PC to the HSR internal PC.
The location should be under `hsr-ros2-setup/ansible/roles/update/files/deb`.

On the HSR internal PC, in the directory where you scp'd the file, run:

```shell
$ sudo dpkg -i cti-serial-dkms_1.46.1-0_all.deb
$ sudo modprobe -a cti_serial_core
$ sudo modprobe -a cti_8250_pci
```

### (HSR-C Only)Installing cgos-vmalloc driver

Work on the HSR internal PC.

Install the cgos-vmalloc driver.
Before starting, use scp to transfer `cgv-1-1_cgoslx-x64-1.03.032_vmalloc.tar.xz` file from the host PC to the HSR internal PC.
The location should be under `hsr-ros2-setup/ansible/roles/update/files/deb`.

In the directory where you scp'd the file on the HSR internal PC, please extract the file.

```shell
$ tar -Jxvf cgv-1-1_cgoslx-x64-1.03.032_vmalloc.tar.xz
```

After that, please execute the following.

```shell
$ cd cgoslx-x64-1.03.032_vmalloc
$ make
$ sudo make install
```

### (HSR-B Only) Rewriting Arduino

Work on the HSR internal PC. Use ssh -X or an external display for the process.

First, clone the repository containing the Arduino sketch:

```shell
$ git clone -b humble https://github.com/hsr-project/hsrb_drivers.git
```

The sketch file is `hsrb_drivers/hsrb_digital_io/sketches/hsrb_usbio_3_serial.ino`.

Next, install Arduino: https://docs.arduino.cc/software/ide-v1/tutorials/Linux/

Before starting the Arduino IDE, identify the serial port to use for writing.
For example, if the following is displayed, the serial port is /dev/ttyUSB1:
```shell
$ ll /dev | grep ttyUSB_IO
lrwxrwxrwx   1 root          root           7 Nov 22 05:57 ttyUSB_IO -> ttyUSB1
```

After starting Arduino, follow these steps:

1. Select the appropriate serial port under Tools -> Serial Port (`ttyUSB0` or `ttyUSB1`).
2. Select Arduino UNO under Tools -> Board.
3. Open the sketch from Arduino: `sketches/hsrb_usbio_3_serial.ino`
4. Compile the sketch under Sketch -> Verify/Compile.
5. Write the sketch to the board under Sketch -> Upload.

#### Restoring Arduino Software for ROS1

To restore the Arduino software for ROS1, execute the following commands on the HSR onboard PC.
**The user ID and password are sensitive information. Please do not disclose them.**
```shell
$ wget https://raw.githubusercontent.com/arduino/arduino-flash-tools/refs/heads/master/tools_linux_64/avrdude/etc/avrdude.conf
$ wget https://docs.hsr.io/command/common/data/arduino_firm/arduino_firm.hex --http-user hsr-user --http-password jD3k4G2e --no-check-certificate
$ sudo avrdude -C avrdude.conf -p atmega328p -P /dev/ttyUSB_IO -c arduino -b 115200 -D -U flash:w:arduino_firm.hex:i
```

### Restoring Files

Transfer the backup files from the host PC to the HSR internal PC:

```shell
$ scp vt.tar.bz2 conf.d.tar.bz2 administrator@<HSR internal PC>
```

Then, on the HSR internal PC, restore the files:

```shell
$ sudo tar xf vt.tar.bz2 -C /
$ sudo tar xf conf.d.tar.bz2 -C /
```

### Creating Docker Image

Execute the following on the host PC:

```shell
$ cd /path/to/hsr-ros2-setup
$ cd docker
$ ./make.sh
```

This will clone and unpack the necessary repositories into `docker_ros2/hsr_repos` and use them to generate an image named `docker.hsr.ros2`.
This process takes time.

### Running ROS 2 Setup with Ansible

Execute the following on the host PC:

```shell
$ cd /path/to/hsr-ros2-setup
```

Before execution, modify the configuration file. In `ansible/hosts`, specify the IP or hostname of the HSR internal PC.

```
[local]
<HSR internal PC hostname or IP> ansible_ssh_user=administrator
```

Then, execute the following:

```shell
$ cd ansible
$ . run_ros2.sh
```

### Startup of services for ROS 2

Run the following on the HSR internal PC:
By executing this script, the node will be in a state of automatic startup (the node will start up when the stop SW is released).

```shell
$ bash start_ros2_docker.sh
```

### Configuring Cyclone DDS

This document outlines the basic configuration required for communication outside a Docker container or between external systems and the robot. 
For more detailed configurations, refer to the following resources:

* https://docs.ros.org/en/humble/How-To-Guides/DDS-tuning.html#cyclone-dds-tuning
* https://github.com/eclipse-cyclonedds/cyclonedds?tab=readme-ov-file#run-time-configuration
* https://autowarefoundation.github.io/autoware-documentation/main/installation/additional-settings-for-developers/network-configuration/dds-settings/#dds-settings-for-ros-2-and-autoware

To load the Cyclone DDS configuration during HSR software startup, add the following line to the end of the `/etc/opt/tmc/robot/ros2_setup.sh`:

```
export CYCLONEDDS_URI=/etc/opt/tmc/robot/cyclonedds_profile.xml
```

Next, create the file `/etc/opt/tmc/robot/cyclonedds_profile.xml`. Below is a template for the file.

* For HSR-C, remove the `NetworkInterfaceAddress` line on the 4th line.
* Set the `Peer Address` to the IP address of the remote PC that will communicate with the robot.

```
<CycloneDDS>
  <Domain>
    <General>
      <NetworkInterfaceAddress>wlp3s0</NetworkInterfaceAddress>
      <AllowMulticast>false</AllowMulticast>
      <EnableMulticastLoopback>false</EnableMulticastLoopback>
      <MaxMessageSize>65500B</MaxMessageSize>
    </General>
    <Discovery>
      <ParticipantIndex>auto</ParticipantIndex>
      <MaxAutoParticipantIndex>100</MaxAutoParticipantIndex>
      <Peers>
        <Peer Address="XXX.XXX.XXX.XXX"/>
        <Peer Address="localhost"/>
      </Peers>
    </Discovery>
  </Domain>
</CycloneDDS>
```

Restart the ROS 2 service:

```shell
$ sh stop_ros2_docker.sh
$ sh start_ros2_docker.sh
```

At this point, ROS 2 communication should be possible outside the Docker container on the HSR onboard PC.
After [installing ROS 2](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html) on the HSR onboard PC, install Cyclone DDS and set up the required environment variables.
Ensure the `ROS_DOMAIN_ID` matches the value set during the execution of `start_ros2_docker.sh`:

```shell
$ sudo apt update
$ sudo apt install -y ros-humble-rmw-cyclonedds-cpp
$ export ROS_DOMAIN_ID=XXX
$ export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
$ export CYCLONEDDS_URI=/etc/opt/tmc/robot/cyclonedds_profile.xml
$ source /opt/ros/humble/setup.bash
$ ros2 topic list
```

Create a `/cyclonedds_profile.xml` on the remote PC.
Set the `Peer Address` to the IP address of the HSR:

```
<CycloneDDS>
  <Domain>
    <General>
      <AllowMulticast>false</AllowMulticast>
      <EnableMulticastLoopback>false</EnableMulticastLoopback>
      <MaxMessageSize>65500B</MaxMessageSize>
    </General>
    <Discovery>
      <ParticipantIndex>auto</ParticipantIndex>
      <MaxAutoParticipantIndex>100</MaxAutoParticipantIndex>
      <Peers>
        <Peer Address="YYY.YYY.YYY.YYY"/>
        <Peer Address="localhost"/>
      </Peers>
    </Discovery>
  </Domain>
</CycloneDDS>
```

Install Cyclone DDS on the remote PC and configure the environment variables for ROS 2 communication.
Ensure the `ROS_DOMAIN_ID` matches the value set during the execution of `start_ros2_docker.sh`:

```shell
$ sudo apt update
$ sudo apt install -y ros-humble-rmw-cyclonedds-cpp
$ export ROS_DOMAIN_ID=XXX
$ export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
$ export CYCLONEDDS_URI=/cyclonedds_profile.xml
$ source /opt/ros/humble/setup.bash
$ ros2 topic list
```

### Python Interface

You can use it with the following command:

```shell
$ docker exec -it docker.humble.diag.service /bin/bash -c ". /etc/opt/tmc/robot/ros2_setup.sh;ros2 run hsrb_interface_py ihsrb.py"
```

### Calibration
Calibration can be performed in the docker environment inside the robot.

First, prepare the docker image. Login to the robot with administrator privileges and pull the docker image.

```shell
$ docker pull docker.hsr.io/hsrb/robot:noetic-calib
```

Stop the services for ros2 and start the service for calibration.
Execute the following command.

```shell
$ sh start_ros1_docker.sh
```

Check “docker environment in the robot” in the calibration chapter of the [HSR User Manual](https://docs.hsr.io/hsrc_user_manual_en/howto/calibration/calibration.html).

Please pay attention to the following two points when performing calibration.

The following expressions in the user manual should be read as follows
  - `docker.hsrb` → `docker.ros1.hsrb`
  - `docker.hsrb.user` → `docker.ros1.hsrb.user`

If the robot has been rebooted, start the services for the calibration by executing the following command.
  ```shell
  $ sh start_ros1_docker.sh
  ```

After calibration, stop the service with the following command.

```shell
$ sudo systemctl stop docker.hsrb.roscore.service
```

The service for ROS2 can then be restart.

```shell
$ sh start_ros2_docker.sh
```

### Using ROS2 Docker Image in a ROS1 Environment

In a clean installation of ubuntu20.04/noetic, you can use a ROS2 Docker to operate the HSR while keeping ROS1 intact.

**For HSRB, when switching between ROS1 and ROS2, it is necessary to rewrite the Arduino. Please refer to [this link](https://git.hsr.io/tmc/hsr-ros2-setup/-/blob/main/README-EN.md?ref_type=heads#hsr-b-only-rewriting-arduino) for switching instructions.**

First, create a ROS2 Docker image.

Execute the following on the host PC:

```shell
$ cd /path/to/hsr-ros2-setup/docker
$ ./make.sh
```

Once the Docker image is created, compress it and copy it to the HSR.

```shell
$ docker save docker.hsr.ros2:latest | gzip > docker.tgz
$ scp docker.tgz administrator@<HSR internal PC hostname or IP>:~/
```

Next, load the Docker image on the HSR.
Execute the following on the HSR internal PC:

```shell
$ gunzip -c docker.tgz | docker load
```

Next, copy the ROS2 service startup scripts.
Execute the following on the host PC:

```shell
$ scp -r scripts/ administrator@<HSR internal PC hostname or IP>:~/
```

Next, configure Cyclone DDS for ROS2.
Refer to [this link](https://git.hsr.io/tmc/hsr-ros2-setup/-/blob/main/README-EN.md?ref_type=heads#configuring-cyclone-dds) and create `/etc/opt/tmc/robot/cyclonedds_profile.xml`.

Set the `ROS_DOMAIN_ID` environment variable to the DOMAIN_ID.
Execute the following on the HSR internal PC:

```shell
$ export ROS_DOMAIN_ID=XXX
```

In `scripts/cyclonedds_profile.xml`, set the IP address of the PC that will communicate with the robot in the Peer Address.

Run the script to start ROS2:

```shell
$ cd ~/scripts/
$ ./BRINGUP_HSR_ROS_HUMBLE.sh
```

You can use the Python Interface with the following command:

```shell
$ docker exec -it docker.humble.robot.service /ros_entrypoint.sh ros2 run hsrb_interface_py ihsrb.py
```

If you want to switch back to ROS1, execute the following script:

```shell
$ cd ~/scripts/
$ ./BRINGUP_HSR_ROS_NOETIC.sh
```
