## Isaacsim 与 IsaacLab
```sh
# 1.启动 isaacsim
kill -9 -f isaacsim
isaacsim

# 2.跑 IsaacLab demo
cd ~/IsaacLab
python scripts/demos/bipeds.py
python scripts/demos/deformables.py
pkill -9 -f bipeds.py
pkill -9 -f deformables.py

# 3.下载模型文件到本地
aria2c -x 16 -s 16 -d /tmp "http://dcb18d6mfegct.cloudfront.net/Assets/Isaac/5.0/Isaac/IsaacLab/Robots/Unitree/G1/g1_minimal.usd"
cp /tmp/g1_minimal.usd ~/IsaacLab/

# 4.Train
python scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Velocity-Flat-G1-v0 --headless
nvitop
pip install nvitop
pkill -9 -f train.py

# 5. tensorboard 查看曲线
ssh -p 31112 root@183.147.142.40 -L 6006:127.0.0.1:6006
tensorboard --logdir ~/IsaacLab/logs

# 6. screen 虚拟终端【Ctrl A + D 退出】
screen
sudo apt install screen
screen -R isaac  # 取名 isaac
python scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Cartpole-v0 --headless
screen -ls

# 7.Play
python scripts/reinforcement_learning/rsl_rl/play.py --task=Isaac-Cartpole-v0
python scripts/reinforcement_learning/rsl_rl/play.py --task=Isaac-Velocity-Flat-G1-v0 
```

## robot_lab
```sh
# 1.单次 git clone 加速
git clone https://gh-proxy.org/https://github.com/isaac-sim/IsaacLab
./isaaclab.sh -i
git clone https://github.com/fan-ziqi/robot_lab.git
python -m pip install -e source/robot_lab

# 2.Train
python scripts/reinforcement_learning/rsl_rl/train.py --task=RobotLab-Isaac-Velocity-Rough-Unitree-Go2W-v0 --headless

# 3.tensorboard
tensorboard --logdir /root/gpufree-data/RobotLab/robot_lab/logs

# 4.Play
python scripts/reinforcement_learning/rsl_rl/play.py --task=RobotLab-Isaac-Velocity-Rough-Unitree-Go2W-v0 --load_run 2026-06-13_14-19-17 --checkpoint /root/gpufree-data/RobotLab/robot_lab/logs/rsl_rl/unitree_go2w_rough/2026-06-13_14-19-17/model_2600.pt
```

