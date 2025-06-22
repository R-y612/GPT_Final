# 使用LeRobot构建SO-ARM100机械臂实现自定义抓取

**切换为[英文](README.md)**

## 概述  
本项目使用**LeRobot框架**为**SO-ARM100**机械臂实现了一个自定义抓取流程。工作内容包括硬件组装、校准、数据集收集、ACT策略训练以及胶带抓取任务评估。主要成果包括：  
- 具备远程操作功能的SO-ARM100系统  
- 用于材料抓取的自定义数据集  
- 在受控条件下验证的ACT策略性能  

## 准备  
- **硬件**：  
  - SO-ARM100机械臂（每臂6个STS3215舵机）  
  - 至少2个RGB-D摄像头  
  - 兼容Ubuntu的工作站  

- **软件**：  
  - Ubuntu 22.04 LTS  
  - Python 3.10+  
  - ROS2 Humble  
  - PyTorch 2.0+（推荐启用CUDA）  
  - LeRobot框架  

## 工作流程  
<font color='Crimson'>***本项目基于[Seeed Studio指南](https://wiki.seeedstudio.com/cn/lerobot_so100m/)***</font>  
**1.  Set device permissions**
```
ls /dev/ttyACM*  #checking all ports connected.
sudo chmod 666 /dev/ttyACM*
```
**2. 配置所有舵机 (ID 1-6重复操作)**
```
python lerobot/scripts/configure_motor.py \
  --port /dev/ttyACM0 \
  --brand feetech \
  --model sts3215 \
  --baudrate 1000000 \
  --ID 1
```

**3. 校准双臂**
```
python lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --robot.cameras='{}' \
  --control.type=calibrate \
  --control.arms='["main_follower"]'
python lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --robot.cameras='{}' \
  --control.type=calibrate \
  --control.arms='["main_leader"]'
  ```

**4. 添加摄像头**
```
python lerobot/common/robot_devices/cameras/opencv.py \
    --images-dir outputs/images_from_opencv_cameras
```

**5. 遥操作测试**
```
python lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=teleoperate \
```

**6.数据集录制**
```
huggingface-cli login --token {your_token} --add-to-git-credential
HF_USER=$(huggingface-cli whoami | head -n 1)
echo $HF_USER
python lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=record \
  --control.fps=30 \
  --control.single_task="put a scotch tape in the circle." \
  --control.repo_id=${HF_USER}/{dataset_name} \
  --control.tags='["so100","tutorial"]' \
  --control.warmup_time_s=5 \
  --control.episode_time_s=30 \
  --control.reset_time_s=30 \
  --control.num_episodes=10 \
  --control.push_to_hub=true \
  #--control.resume=true  #use when restart recording to an established dataset
```
建议每个位置至少录制10组数据。
***录制环境至关重要！确保环境可完全复现！***
尽量减少人工演示时的微调，以避免策略过拟合到次优轨迹片段。segments.

**7. 训练**
```
python lerobot/scripts/train.py \
  --dataset.repo_id=${HF_USER}/{dataset_name} \
  --policy.type=act \
  --output_dir=outputs/train/act_so100_test \
  --job_name=act_so100_test \
  --device=cuda \
  --wandb.enable=true # Register and get API first from Weights and Biases
```
在此注册[Wandb](https://wandb.ai/)

**8. 评估**
```
python lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=record \
  --control.fps=30 \
  --control.single_task="Grasp a tape and put it in the drawing circle." \
  --control.repo_id=${HF_USER}/eval_act_so100_test \
  --control.tags='["tutorial"]' \
  --control.warmup_time_s=5 \
  --control.episode_time_s=30 \
  --control.reset_time_s=30 \
  --control.num_episodes=10 \
  --control.push_to_hub=true \
  --control.policy.path=outputs/train/act_so100_test/checkpoints/last/pretrained_model
```

## **关键结果**  
**1. 硬件要求**：  
- 严格的舵机ID顺序（1-6）至关重要  
- 无响应舵机需物理复位  

**2. 策略性能**：  
- 在受控环境（*稳定、光照充足、整洁*）中表现最佳  
- 对新物体位置的泛化能力有限  
- 对摄像头角度变化敏感  
  
## 查看我们的数据集和模型
**1. 数据集**
你可以在huggingface找到我们的[**数据集**](https://huggingface.co/datasets/aaaRick/so100_test613sec)。
同时，它也可以直接在python中下载:
```python
from datasets import load_dataset
dataset = loda_dataset("aaaRick/so100_test613sec")
```
**2. 模型**
你可以在huggingface找到我们的[**模型**](https://huggingface.co/aaaRick/act_so100_test613/tree/main)。
也可以通过repo id：aaaRick/so100_test613找到。

## 常见问题  
**1. 项目全程可能出现的错误** 
```
ConnectionError: Read failed due to comunication eror on port /dev/ttyACM0 forgroup key Present_Position_Shoulder_pan_Shoulder_lift_elbow_flex_wrist_flex_wrist_roll_griper: [TxRxResult] There is no status packet!
```
- 首先检查***线缆是否断开***，此错误通常由断线或断电引起。  

**2. 校准时出现的问题**  

```
lerobot.common.robot_devices.motors.feetech.JointOutOfRangeError: Wrong motor position range detected for gripper. Expected to be in nominal range of [0, 100] % (a full linear translation), with a maximum range of [-10, 110] % to account for some imprecision during calibration, but present value is-inf %
```
- 检查配置文件中主从臂端口是否匹配。校准前需验证端口分配。  
- 使用["FT Servo Debug"](https://github.com/Kotakku/FT_SCServo_Debug_Qt)诊断舵机连接和位置精度，避免不必要的重新初始化。  

**3. 遥操控中有单独舵机不响应但程序正常运行**  
- 在遥操控开始前手动将该舵机对应的臂提起。  

**4. 数据集上传失败**  
```
requests.exceptions.ConnectionError: (MaxRetryError('HTTPSConnectionPool(host=\'huggingface.co\', port=443): Max retries exceeded with url: /datasets/aaaRick/so100_test610first.git/info/lfs/objects/verify (Caused by NameResolutionError("<urllib3.connection.HTTPSConnection object at 0x78ec38485f00>: Failed to resolve \'huggingface.co\' 

```
- 下次录制时会自动恢复上传。  
- 也可尝试手动克隆仓库并上传数据。  

## 许可证  
[MIT许可证](LICENSE.md) - 学术使用需署名。  

## 参考文献  
1. Seeed Studio. (2025). SO-ARM100 Integration Guide