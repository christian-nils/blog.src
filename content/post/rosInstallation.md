+++
date        = "2017-04-26T08:38:10+02:00"
draft       = true
title       = "Getting started with MM7150 and Raspberry Pi (software side)"
description = "Step-by-step tutorial to make the MM7150 imu works with the Raspberry Pi."
tags        = [ "MM7150", "Raspberry Pi", "Linux" ]
topics      = [ "Development"]
author      = "Christian-Nils Boda"
+++

# Install ROS
Firt, follow the instructions [here][1] for installing ROS Kinetic on Raspbian. If you do not plan to use GUIs you should follow the instructions for ROS-Comm.
You may encounter a problem when compiling, see [this page][2].

To install ROS released packages, see [this page][3].

To compile and load the catkin workspace, see [this page][4]

To install the ros-tutorial, see [this][5];
Note: to resume on: root@raspberrypi:~/ros_catkin_ws# ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release --install-space /opt/ros/kinetic

[1]: http://wiki.ros.org/ROSberryPi/Installing%20ROS%20Kinetic%20on%20the%20Raspberry%20Pi
[2]: http://answers.ros.org/question/219575/raspberry-pi2-roscpp-failed-during-indigo-installation/
[3]: http://wiki.ros.org/ROSberryPi/Installing%20ROS%20Kinetic%20on%20the%20Raspberry%20Pi#Adding_Released_Packages
[4]: http://wiki.ros.org/ROS/Tutorials/InstallingandConfiguringROSEnvironment#Create_a_ROS_Workspace
[5]: http://answers.ros.org/question/250738/cannot-download-ros-kinetic-ros-tutorials/