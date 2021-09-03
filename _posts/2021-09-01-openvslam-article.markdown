---
layout: post
title:  "Adding sensor fusion to openvslam: does it help?"
date:   2021-09-02
pimage: /assets/gif/delta_movement.gif
description: Can off the shelf ros packages give openvslam an accuracy boost?   
---


Visual SLAM, or VSLAM, is a way of providing the pose information of a camera by using the images from that camera. With just a monocular camera, VSLAM algorithms aren't able to gauge the depth scale of the camera's trajectory. But with an RGBD camera or a stereo camera system, that problem goes away, and true pose and odometry info is able to be recovered.

Image data is quite heavy, and so most algorithms only use 20 to 30 images a second, either due to limitations on the camera side or the cpu running the VSLAM algorithm. Sometimes, the camera moves too fast for these 20 to 30hz updates to track correctly, and accuracy is lost, or even worse, tracking is lost completely. Algorithms can mitigate this by integrating other sensor modalities, like IMUs, that provide more updates per second.

Recently I've been looking at adding more sensor modalities into [openvslam](https://github.com/OpenVSLAM-Community/openvslam{:target="_blank"}), not just IMUs, but other data as well, like wheel odometry and gps measurements. Since we use ROS2 in this house, I started off by trying to add the output of the robot_localization into openvslam. Openvslam updates its pose at each camera frame update, but before processing a new frame, it estimates what it thinks the camera pose currently is. It does this by using the velocity of the calculated from the last two poses. My goal was to replace this init pose estimation with the output of the robot_localization kalman filter. I tested out my efforts by driving my robot around in my webots environment. This first test went okay, position _seemed_ to be tracked okay. 

The second test was the analytical one - testing against the EuRoC MAV Dataset. This test didn't go so well. My tracking was terrible! tracking was lost so many times that I didn't even bother calculating root mean square error. The problem was that the robot_localization kalman filter was just too slow. Openvslam is extremely sensitive to these position estimate before each camera frame is processed. The kalman filter is a _filter_; the estimated position lagged the position estimate given by openvslam. This lag was too much for the system to handle(INSERT MEDIA).

With the robot_localization try not giving good results, I moved on to trying to add just imu data to openvslam. I didn't want to have to try any filtering or bias estimation, so I kept things simple by using the [madgwick filter](https://github.com/ccny-ros-pkg/imu_tools/tree/eloquent{:target="_blank"}) package in the ros2 ecosystem. This removed the gravity vector from the imu acceleration and gave me some support for bias estimation. After a few tries, I was able to get performance similar to the base algorithm without imu usage. To save time I ran mostly on MH04, because I saw that other algorithms had performance improvements adding an imu to their run of this data. Surprisingly, I didn't get a performance improvement on this data. My results were:
writing {'rmse_std': 0.01957928370635225, 'rmse_mean': 0.06707345125516433, 'rmse_max': 0.13002343292963184, 'rmse_min': 0.04533321717083381} to euroc_ros_baseline_MH_04/comparisons/results.yaml
writing {'rmse_std': 0.01782430149874279, 'rmse_mean': 0.06925266196752958, 'rmse_max': 0.1011074003127152, 'rmse_min': 0.04901822139589855} to euroc_ros_imu_MH_04/comparisons/results.yaml


I was curious if there was a point where an imu would strictly improve pose accuracy, so I downsampled the camera data in the EuRoC dataset from 20hz to 5hz. It was here that the imu showed its value. 
writing {'rmse_std': 0.24033438885910896, 'rmse_mean': 0.2306021925554946, 'rmse_max': 1.0298466197445322, 'rmse_min': 0.054483297685324894} to euroc_ros_baseline_MH_04/comparisons/results.yaml
writing {'rmse_std': 0.03267079819227422, 'rmse_mean': 0.10124739930754059, 'rmse_max': 0.18463775292126264, 'rmse_min': 0.048000731864542526} to euroc_ros_imu_MH_04/comparisons/results.yaml

What I learned is that its pretty hard to improve the baseline performance of openvslam. And on the EuRoC dataset, there hardly seems to be a need to improve that performance. Where additional sensor modalities will really shine is when tracking is lost, or when the ratio of modality measurements to camera frames is large and frame rate is low.

As always, this work can be found on my [my github](https://github.com/Jconn/){:target="_blank"}. 
