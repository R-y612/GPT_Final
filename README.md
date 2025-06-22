# From Scratch to Task: Building a SO-ARM100 Robot Arm for Custom Grasping with LeRobot

**Read this in [Chinese](README_ZH.md)**

## Overview
This project implements a custom grasping pipeline for the **SO-ARM100** robotic arm using the **LeRobot framework**. The work includes hardware assembly, calibration, dataset collection, ACT policy training, and task evaluation for scotch tape grasping. Key outcomes are:
- A functional SO-ARM100 system with teleoperation capabilities
- A custom dataset for material grasping
- Validated ACT policy performance under controlled conditions

## Prerequisites
- **Hardware**:
  - SO-ARM100 robotic arm (6x STS3215 servos per arm)
  - At least 2 RGB-D camera 
  - Ubuntu-compatible workstation

- **Software**:
  - Ubuntu 22.04 LTS
  - Python 3.10+
  - ROS2 Humble
  - PyTorch 2.0+ (CUDA enabled recommended)
  - LeRobot framework


## Workflow Implementation
<font color='Crimson'>***The project is based on detailed guidance from [seeed studio](https://wiki.seeedstudio.com/cn/lerobot_so100m/)***</font>
**1.  Set device permissions**
```
ls /dev/ttyACM*  #checking all ports connected.
sudo chmod 666 /dev/ttyACM*
```
**2. Config all servos(repaet for IDs 1-6)**
```
python lerobot/scripts/configure_motor.py \
  --port /dev/ttyACM0 \
  --brand feetech \
  --model sts3215 \
  --baudrate 1000000 \
  --ID 1
```

**3. Calibrate both arms**
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

**4. Adding Cameras**
```
python lerobot/common/robot_devices/cameras/opencv.py \
    --images-dir outputs/images_from_opencv_cameras
```

**5. Teleporation Testing**
```
python lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=teleoperate \
```

**6.Dataset Recording**
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
It's recommended to record at least 10 episode for a single position.
***Recording environment is especially important!!! Make sure this environment can be replicated identically!!!***
Try to reduce micro-adjustments during human demonstrations, which might give rise to policy overﬁtting to suboptimal trajectory segments.

**7. Taining**
```
python lerobot/scripts/train.py \
  --dataset.repo_id=${HF_USER}/{dataset_name} \
  --policy.type=act \
  --output_dir=outputs/train/act_so100_test \
  --job_name=act_so100_test \
  --device=cuda \
  --wandb.enable=true # Register and get API first from Weights and Biases
```
Register [Wandb](https://wandb.ai/) here.

**8. Evaluation**
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
## **Key Technical Findings**
**1. Hardware Requirements:**
- Strict servo ID sequencing (1-6) is critical
- Physical reset needed for unresponsive servos

**2. Policy Performace:**
-  Works best in controlled environments (*stable, well illuminated, clean*)
-  Limited generalization to new object positions
-  Sensitive to camera angle variations
  

## Finding Our Dataset and ACT-Trained Model
**1. Dataset**
- You can find **[our datasets](https://huggingface.co/datasets/aaaRick/so100_test613sec)** on huggingface. 
- Also, it can be directly loaded:
 ```python
from datasets import load_dataset
dataset = loda_dataset("aaaRick/so100_test613sec")
```
**2. Model**
- You can find [**our model**](https://huggingface.co/aaaRick/act_so100_test613/tree/main) on huggingface. 
- The repo id is aaaRick/so100_test613.

## FAQ
**1. This error might occur throughout the whole project.**
```
ConnectionError: Read failed due to comunication eror on port /dev/ttyACM0 forgroup key Present_Position_Shoulder_pan_Shoulder_lift_elbow_flex_wrist_flex_wrist_roll_griper: [TxRxResult] There is no status packet!
```
- First off, check ***cable disconnection***, this error usually is caused by wire disconnection, or power off.
  
**2. This problem occurs at calibration.**

```
lerobot.common.robot_devices.motors.feetech.JointOutOfRangeError: Wrong motor position range detected for gripper. Expected to be in nominal range of [0, 100] % (a full linear translation), with a maximum range of [-10, 110] % to account for some imprecision during calibration, but present value is-inf %
```
- Check whether the leader and follower arm ports are mismatched in the conﬁguration ﬁle. Before recalibration, port assignments should be veriﬁed. 
- Subsequently, using ["FT Servo Debug (a servo diagnostic utility)"](https://github.com/Kotakku/FT_SCServo_Debug_Qt) to diagnose servo connectivity and positional accuracy assists to avoid unnecessary reinitialization.


**3. A servo remains unresponsive during calibration yet functions correctly in teleporation**
- This issue can often be mitigated by manually resetting the joint position of the affected arm segment before initiating teleoperation. 
- Recalibration for more times might resolve this problem.
  
**4. Dataset recorded failed to push_to_hub**
```
requests.exceptions.ConnectionError: (MaxRetryError('HTTPSConnectionPool(host=\'huggingface.co\', port=443): Max retries exceeded with url: /datasets/aaaRick/so100_test610first.git/info/lfs/objects/verify (Caused by NameResolutionError("<urllib3.connection.HTTPSConnection object at 0x78ec38485f00>: Failed to resolve \'huggingface.co\' 

```
- The upload process will automatically resume when recording episodes to the same dataset next time.
- Additionally, you can try to git clone your dataset repository and manually upload.


## License
[MIT License](LICENSE.md) - Free for academic use with attribution

## References
1. Seeed Studio. (2025). SO-ARM100 Integration Guide
