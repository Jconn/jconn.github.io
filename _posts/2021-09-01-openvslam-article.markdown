---
layout: post
title:  "Adding sensor fusion to openvslam: does it help?"
date:   2021-09-02
pimage: /assets/img/title_picture_openvslam.png
description: Can off the shelf ros packages give openvslam an accuracy boost?   
---


Visual SLAM, or VSLAM, is a way of providing the pose information of a camera by using the images from that camera. With just a monocular camera, VSLAM algorithms aren't able to gauge the depth scale of the camera's trajectory. But with an RGBD camera or a stereo camera system, that problem goes away, and true pose and odometry info is able to be recovered.

Image data is heavy, and most visual SLAM algorithms only use 20 to 30 images a second, either due to limitations on the camera side or the cpu running the VSLAM algorithm. Sometimes, the camera moves too fast for these 20 to 30hz updates to track correctly, and accuracy is lost, or even worse, tracking is lost completely. Algorithms can mitigate this by integrating other sensor modalities, like IMUs, that provide sensor readings on the order of 100-1000hz.

Recently I've been looking at adding more sensor modalities into [openvslam](https://github.com/OpenVSLAM-Community/openvslam{:target="_blank"}), not just IMUs, but other data as well, like wheel odometry and gps measurements. Since we use ROS2 in this house, I started off by trying to add the output of the robot_localization into openvslam. Openvslam updates its pose at each camera frame update, but before processing a new frame, it estimates what it thinks the camera pose currently is. It does this by using the velocity of the calculated from the last two poses. My goal was to replace this init pose estimation with the output of the [robot_localization](https://github.com/cra-ros-pkg/robot_localization/tree/galactic{:target="_blank"}) kalman filter. I tested out my efforts by driving my robot around in my webots environment. This first test went okay, position _seemed_ to be tracked okay. 

The second test was the analytical one - testing against the EuRoC MAV Dataset. This test didn't go so well. My tracking was terrible! tracking was lost so many times that I didn't even bother calculating root mean square error. The problem was that the robot_localization kalman filter was just too slow. Openvslam is extremely sensitive to these position estimate before each camera frame is processed. The kalman filter is a _filter_; the estimated position lagged the position estimate given by openvslam. This lag was too much for the system to handle. The green arrow is the output of openvslam, and the red arrow is the output of my robot_localization ekf. The lag in the two signals is too much to feedback into openvslam. In this gif, green is the pose from openvslam, and red is the pose from the kalman filter taking openvslam as input.![Alt Text](/assets/gif/lagging_ekf.gif)

With the robot_localization try not giving good results, I moved on to trying to add just imu data to openvslam. I didn't want to have to try any filtering or bias estimation, so I kept things simple by using the [madgwick filter](https://github.com/ccny-ros-pkg/imu_tools/tree/eloquent{:target="_blank"}) package in the ros2 ecosystem. This removed the gravity vector from the imu acceleration and gave me some support for bias estimation. After a few tries, I was able to get performance similar to the base algorithm without imu usage. To save time I ran mostly on MH04, because I saw that other algorithms had performance improvements adding an imu to their run of this data. Surprisingly, I didn't get a performance improvement on this data. My results were:

| EuRoC MH04 30 run     | ros baseline | imu + cam |
| :---                  | :----:       | :---:     |
| rmse mean(meters)     | **0.0671**   | 0.0693    |
| rmse std. dev(meters) | 0.0196       | 0.0178    |

I was curious if there was a point where an imu would strictly improve pose accuracy, so I downsampled the camera data in the EuRoC dataset from 20hz to 5hz. It was here that my imu showed its value. 

| EuRoC MH04 30 run 5 hz camera frames | ros baseline | imu + cam |
| :---                                 | :----:       | :---:     |
| rmse mean(meters)                    | 0.231        | **0.101** |
| rmse std. dev(meters)                | 0.240        | 0.033     |


5hz camera frames and 200hz imu of EuRoC MH04 
![Alt Text](/assets/gif/5hz_movement.gif "5hz camera frames and imu with openvslam")
At 5hz, there are just not enough frames to capture motion accurately, whereas that 200hz imu is able to inform what motion is occuring between frames. That being said, at 5hz both methods start to lose tracking. There are just not enough common keypoints between frames to maintain tracking.


What I learned is that its pretty hard to improve the baseline performance of openvslam. And on the EuRoC dataset, there hardly seems to be a need to improve that performance. Where additional sensor modalities will really shine is where the camera framerate is too low to capture camera motion.


As always, this work can be found on my [my github](https://github.com/Jconn/){:target="_blank"}. 

