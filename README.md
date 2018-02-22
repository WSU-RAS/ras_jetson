Object Detector on NVidia Jetson with ROS
=========================================

## Jetson Setup

We bought the Nvidia Jetson TX2, [Orbitty carrier
board](http://www.wdlsystems.com/Computer-on-Module/Carrier-Boards/CTI-Orbitty-Carrier-for-NVIDIA-Jetson-TX1.html),
and [Connect Tech active heat
sink](http://www.wdlsystems.com/Computer-on-Module/Cooling-Accessories/CTI-NVIDIA-TX1-Module-Active-Thermal-Heatsink.html).
For the AC power adapter, we used a 12 VDC power supply we had sitting around.

Installing JetPack:

* Install Ubuntu 16.04 in a VM and give it enough ram, maybe 4-6 GiB
  ([src](https://devtalk.nvidia.com/default/topic/988616/jetpack-2-3-1-flash-problems-/?offset=7))
* Enable USB3 in VirtualBox (even though the Jetson is only USB2, if you
  only have USB2 you get a "BootRom is not running" error,
  [src](https://devtalk.nvidia.com/default/topic/1002827/jetson-tx2/problem-flashing-tx-2/2))
* Run JetPack installer and install everything on the host.
* When it gets to the point to install to the Jetson, you'll have to put it
  into the Force Recovery mode (unplug power, plug power, hold in on
  RECOVERY, press Reset, let up on Recovery after 2 seconds). Then NVIDIA
  CORP device shows up in "lsusb." In VirtualBox, under USB (bottom right
  corner) check that for the VM. Press Enter in the VM to say to install.
  Then it'll say it doesn't find the USB device. Check the NVIDIA CORP
  again. Then it'll install.
* After copying over the filesystem, it'll error that it can't find the IP.
  Then you can quit the installer.
* Install the Orbitty modifications to support USB and the PWM fan via
  following [their instructions](http://connecttech.com/resource-center/cti-l4t-nvidia-jetson-board-support-package-release-notes/#bkb-h5).
  You'll end up running the command `sudo ./flash.sh orbitty mmcblk0p1`.
  Note, do this *before* installing anything else since this step will
  overwrite the whole filesystem.
* Plug the Jetson into your host computer ethernet (so you can get IP from
  Wireshark) and share your connection or into a router you can get the Jetson
  IP from. Then run the installer again but make sure to select "no action" to
  install the OS ("Flash OS Image to Target") when re-running. Then when it
  gets to installing other software, e.g. CUDA, it'll ask for the IP, user, and
  pass. Specify the IP and then "nvidia" for both username and password.
  ([src](https://devtalk.nvidia.com/default/topic/1002081/jetson-tx2/jetpack-3-0-install-with-a-vm/))
* Note: after it's all done, to shut down VirtualBox, you probably have to
  uncheck the USB device.

Allow LLMNR so we can resolve "tegra-ubuntu" hostname to SSH into it:

    sudo systemctl enable systemd-resolved
    sudo systemctl start systemd-resolved

Disable the display manager (if desired):

    sudo rm /etc/X11/default-display-manager
    sudo touch /etc/X11/default-display-manager

Disable snapd:

    sudo systemctl disable snapd snapd.socket
    sudo systemctl stop snapd snapd.socket

Set CPUs and GPU to max performance on boot (if desired): add this right before
the `exit 0` in */etc/rc.local* script:

    ( sleep 60 && nvpmodel -m 0 && /home/ubuntu/jetson_clocks.sh )&

Enable universe and multiverse repositories (e.g. to install htop):

    sudo add-apt-repository universe
    sudo add-apt-repository multiverse

Update:

    sudo apt update
    sudo apt upgrade

Install TensorFlow

    sudo apt install python{,3}-pip python{,3}-numpy python{,3}-matplotlib htop \
        jnettop protobuf-compiler python{,3}-pil python{,3}-lxml libxml2-dev \
        libxslt1-dev python{,3}-yaml python{,3}-docutils \
        redis-server python{,3}-redis \
        ros-lunar-rosbridge-server ros-lunar-rosbridge-suite \
        ros-lunar-move-base-msgs ros-lunar-tf2-bullet \
        ros-lunar-rosserial ros-lunar-rosserial-arduino

    git clone https://github.com/peterlee0127/tensorflow-tx2.git
    cd tensorflow-tx2
    pip2 install --user tensorflow-1.4.1-cp27-cp27mu-linux_aarch64.whl

    # Object detection with TensorFlow
    pip2 install --user pillow

    # Human detection
    pip2 install --user imutils

    # For some reason catkin won't build without installing via pip
    pip2 install --user docutils rospkg

In your *.ssh/config* for ease of SSHing:

    Host jetson
    HostName tegra-ubuntu
    User nvidia
    ForwardX11 yes
    ForwardX11Trusted yes
    Compression yes

Copy your model files over to the Jetson *~/networks*:

    scp -r /path/to/ras-object-detection/datasets/SmartHome2/*.pb jetson:networks/
    scp /path/to/ras-object-detection/datasets/SmartHome2/tf_label_map.pbtxt jetson:networks/
    scp -r /path/to/ras-object-detection/datasets/SmartHome/test_images jetson:networks/

Install ROS ([src](http://wiki.ros.org/lunar/Installation/Ubuntu)):

    sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
    sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-key 421C365BD9FF1F717815A3895523BAEEB01FA116
    sudo apt-get update
    sudo apt-get install ros-lunar-desktop-full libroscpp-dev librospack-dev libtf2-ros-dev libtf-dev libnodeletlib-dev
    sudo apt-get install python-rosinstall python-rosinstall-generator python-wstool build-essential
    sudo rosdep init
    rosdep update

    source /opt/ros/lunar/setup.bash
    echo "source /opt/ros/lunar/setup.bash" >> ~/.bashrc

## Catkin Workspace

Create our Catkin workspace:

    git clone --recursive https://github.com/WSU-RAS/ras_jetson.git

Setting WSU-RAS repos to use SSH (if desired):

    cd ~/ras_jetson/src
    cd object_detection; git remote set-url origin git@github.com:WSU-RAS/object_detection.git; cd ..
    cd object_detection_msgs; git remote set-url origin git@github.com:WSU-RAS/object_detection_msgs.git; cd ..
    cd cob_perception_msgs; git remote set-url origin git@github.com:WSU-RAS/cob_perception_msgs.git; cd ..
    cd darknet_ros; git remote set-url origin git@github.com:WSU-RAS/darknet_ros.git; cd ..
    cd human-detection; git remote set-url origin git@github.com:WSU-RAS/human-detection.git; cd ..
    cd ras_msgs; git remote set-url origin git@github.com:WSU-RAS/ras_msgs.git; cd ..

    # Verify they're correct:
    git submodule foreach git remote get-url --all origin

Then, generate the protobuf files:

    cd ~/ras_jetson/src/object_detection/models/research/
    protoc object_detection/protos/*.proto --python_out=.

Add to your ~/.bashrc file:

    echo 'export PYTHONPATH=$PYTHONPATH:/home/nvidia/ras_jetson/src/object_detection/models/research/:/home/nvidia/ras_jetson/src/object_detection/models/research/slim/' >> ~/.bashrc

Build everything:

    source /opt/ros/lunar/setup.bash
    cd ~/ras_jetson
    catkin_make --pkg darknet_ros_msgs # Needs to be built before darknet_ros
    catkin_make -DFILTER=OFF -DCMAKE_BUILD_TYPE=Release
    catkin_make install

Source this new workspace:

    source ~/ras_jetson/devel/setup.bash
    echo "source ~/ras_jetson/devel/setup.bash" >> ~/.bashrc

Setup udev rules for camera, then make sure to unplug then plug back in the
camera:

    cd ~/ras_jetson/src/astra_camera
    ./scripts/create_udev_rules

Print [checkerboard](http://wiki.ros.org/camera_calibration/Tutorials/MonocularCalibration?action=AttachFile&do=view&target=check-108.pdf)
and measure square in meters. Mine are 0.025 m. Follow the [tutorial](http://wiki.ros.org/camera_calibration/Tutorials/MonocularCalibration).

    rosrun camera_calibration cameracalibrator.py --size 8x6 --square 0.025 image:=/camera/rgb/image_raw camera:=/camera/rgb

## Arduino Setup for Camera Pan/Tilt
Install Arduino IDE 1.0.6 on some computer (on Arch Linux try the *arduino10*
package in the AUR). Follow [ArbotiX-M instructions](http://learn.trossenrobotics.com/arbotix/7-arbotix-quick-start-guide).
Download the [ArbotiX-M](https://github.com/trossenrobotics/arbotix/archive/master.zip)
files and extract into your *~/sketchbook* folder.

If you can't get permissions to work despite adding yourself to uucp and lock
groups, then make sure that "/run/lock" is in the lock group:

    sudo chgrp lock /run/lock

Then, upload the File -> Sketchbook -> ArbotiX Sketches -> ros.

Setting up on the Jetson so you can control the servos from ROS:

    roslaunch object_detection pantilt.launch
    arbotix_gui

## Connecting to NUC
Since we'll run some on the Jetson and some on the NUC, we'll need to set one
up as the ROS master.  We'll use the NUC.

Jetson (since we enabled systemd-resolved earlier), so we can resolve the NUC
hostname with LLMNR:

    sudo apt install libnss-resolve

In *~/.bashrc* on the Jetson:

    export ROS_MASTER_URI=http://wsu-ras:11311
    machine_ip=($(hostname -I))
    export ROS_IP=${machine_ip[0]}

or, if you have issues resolving it now and then, maybe try:

    export ROS_MASTER_URI=http://$(systemd-resolve -4 wsu-ras | head -n 1 | grep -Eo "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"):11311

Then, replace 127.0.1.1 with 127.0.0.1 in */etc/hosts* on the Jetson.
Otherwise, often it can't connect to the ROS master on the NUC.

In *~/.bashrc* on the NUC:

    export ROS_MASTER_URI=http://wsu-ras:11311
    machine_ip=($(hostname -I))
    export ROS_IP=${machine_ip[0]}
    export TURTLEBOT_3D_SENSOR=astra
    export TURTLEBOT3_MODEL=waffle

Then, run on the NUC:

    git clone --recursive https://github.com/WSU-RAS/ras.git
    cd ras
    catkin_make

    source ~/ras/devel/setup.bash
    roscore
    roslaunch turtlebot3_bringup turtlebot3_robot.launch
    roslaunch turtlebot3_bringup turtlebot3_remote.launch
    rosrun rviz rviz -d $(rospack find turtlebot3_description)/rviz/model.rviz

    rosrun object_detection_msgs go_to.py

Then, run on Jetson:

    roslaunch object_detection everything.launch

    # Optionally either of these, to control the camera
    rosrun arbotix_python arbotix_gui
    rosrun object_detector_ros demo_pan_tilt.py

## YOLO Setup (optional)
Copy the final weights over for YOLO into the *darknet_ros* directory:

    scp path/to/ras-object-detection/datasets/SmartHome/backup_100/SmartHome_final.weights \
        jetson:ras_jetson/src/darknet_ros/darknet_ros/yolo_network_config/weights/SmartHome.weights
    scp path/to/ras-object-detection/datasets/SmartHome/config.cfg \
        jetson:ras_jetson/src/darknet_ros/darknet_ros/yolo_network_config/cfg/SmartHome.cfg
    scp path/to/ras-object-detection/dataset_100.data \
        jetson:ras_jetson/src/darknet_ros/darknet_ros/config/

Create *~/ras_jetson/src/darknet_ros/darknet_ros/config/SmartHome.yaml*,
changing the classes accordingly. Make sure the spaces/tabs are correct or else
it'll error parsing the file. Then change "yolo\_voc.yaml" to "SmartHome.yaml"
in *~/ras_jetson/src/darknet_ros/darknet_ros/launch/darknet_ros.launch*.

    yolo_model:
        config_file:
            name: SmartHome.cfg
        weight_file:
            name: SmartHome.weights
        threshold:
            value: 0.3
        detection_classes:
            names:
              -  food
              -  glass
              -  keys
              -  pillbottle
              -  plant
              -  umbrella
              -  watercan

If you don't want it showing the window with predictions, then set
*enable_opencv* and *use_darknet* to false in
*~/ras_jetson/src/darknet_ros/darknet_ros/config/ros.yaml*.

Edit the *everything.launch* file to use YOLO rather than TensorFlow.
