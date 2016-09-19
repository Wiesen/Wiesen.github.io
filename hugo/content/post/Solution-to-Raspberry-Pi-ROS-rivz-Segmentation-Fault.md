+++
date = "2016-09-18T17:32:46+08:00"
draft = false
title = "Solution to Raspberry Pi ROS rivz Core Dumped"

topics = ["Robotics"]
+++

Recently I installed ROS on Raspberry Pi2 (both Jessie and Ubuntu 14.04) in order to implement SLAM algorithm on it. However when runs the rviz (rosrun rviz rviz) I get core dumped message.

After searching I found the solution [here](https://github.com/ros/robot_model/issues/110). What I do is upgrade libpcre3 to 3_8.35 (upgrade collada-dom to 2.4.4 does not help). [Here](http://ports.ubuntu.com/pool/main/p/pcre3/libpcre3_8.35-7.1ubuntu1_armhf.deb) is the download link.

I confirm that rviz now works properly on my Raspberry Pi2 running the official ubuntu 14.04 image.

