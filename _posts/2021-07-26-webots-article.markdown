---
layout: post
title:  "Building a Delta Robot in Webots"
date:   2021-07-27
pimage: /assets/gif/delta_movement.gif
description: A Gazebo developers perspective 
---



I've been using Gazebo with ROS2 for the past two years. Recently, a coworker wanted to evaluate webots. Their interest got me interested, and so I thought I'd give it a try.

Webots is an open source robotics simulator developed by Cyberbotics. Cyberbotics open-sourced the project in 2019 after 20 years of proprietary usage. The 2021a release included support for ROS2 Foxy, the most recent lts release of ROS2. I use Foxy, so I thought now was a good time to get into it and see how Webot feels with ROS2. 

In order to get to know Webots, I started with trying to create a delta robot. I started by adding solids and connecting children to those solids through the Webots GUI. I loved being able to construct the robot in this way, with visual feedback that my components were correct. A nice quality of life feature is the ability to copy-paste components and all of their children. This made it really easy to add and remove components in the middle of a long chain. 

In Gazebo I was used to creating a lot of intermediate links to construct a robot, and so I started with doing that in Webots with solids. I also gave each of these intermediate solids a physics node. This ended up being a mistake, with all of these intermediate nodes slowing down my simulation and ultimately causing the joints connected to the lower platform to not work.

After asking some quesions on the Webots discord I simplified my design, and only added solids that had an actual physical piece of the delta associated with it. At this point, everything started working as expected, and my delta looked like it was ready to start being moved. 


![The finished product](/assets/img/delta_static.png){:target="_blank"}


The next step was creating a controller to move my delta's motors. Webots has a python interface for ROS2 (which they might be [deprecating](https://github.com/cyberbotics/webots_ros2/issues/227#issuecomment-853150841){:target="_blank"}), that made it really easy for me to set up a motor test and verify that my delta was operating correctly.


![Alt Text](/assets/gif/delta_movement.gif)



Overall, Webots felt pretty nice to use. It took me a little while to get used to some of the differences in Webots and Gazebo when constructing my robot, but once I learned those things robot construction and control went pretty smoothly. 

The code for the delta is available on [my github](https://github.com/Jconn/delta_webots){:target="_blank"}. 


