```sh
# 终端 1 - 启动 VINS 估计器核心
source ~/gpufree-data/VINS-Mono/devel/setup.bash
roslaunch vins_estimator euroc.launch

# 终端 2 - 启动 RViz 可视化界面
source ~/gpufree-data/VINS-Mono/devel/setup.bash
rviz -d ~/gpufree-data/VINS-Mono/src/config/vins_rviz_config.rviz

# 终端 3 - 播放 bag 文件
cd ~/gpufree-data
rosbag play MH_01_easy.bag --clock
rosbag play MH_02_easy.bag --clock
rosbag play MH_03_medium.bag --clock
rosbag play MH_04_difficult.bag --clock
rosbag play MH_05_difficult.bag --clock

rosbag play V1_01_easy.bag --clock
rosbag play V1_02_medium.bag --clock
rosbag play V1_03_difficult.bag --clock

rosbag play V2_01_easy.bag --clock
rosbag play V2_02_medium.bag --clock
rosbag play V2_03_difficult.bag --clock