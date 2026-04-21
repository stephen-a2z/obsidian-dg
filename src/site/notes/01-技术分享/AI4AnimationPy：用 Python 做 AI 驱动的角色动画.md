---
{"dg-publish":true,"permalink":"/01-技术分享/AI4AnimationPy：用 Python 做 AI 驱动的角色动画/","tags":["python","ai","动画","animation","3d"],"noteIcon":"","created":"2026-04-21T12:05:08.388+08:00","updated":"2026-04-21T13:48:40.001+08:00"}
---


# AI4AnimationPy：用 Python 做 AI 驱动的角色动画

> Meta 开源了一个 Python 框架，把 AI 角色动画的训练、推理、可视化全部统一到一个环境里。不需要 Unity 了。

## 一、它是什么

[AI4AnimationPy](https://github.com/facebookresearch/ai4animationpy) 是 Meta（Facebook Research）开源的 Python 框架，用于**基于神经网络的角色动画**。它是 [AI4Animation](https://github.com/sebastianstarke/AI4Animation)（一个知名的 Unity 项目）的 Python 重写版，由原作者 Sebastian Starke 和 Paul Starke 开发。

核心思路：把动捕数据处理、神经网络训练、实时推理、动画可视化全部放在一个 Python 环境里完成，不再需要 Unity 作为中间环节。

项目地址：https://github.com/facebookresearch/ai4animationpy
文档：https://facebookresearch.github.io/ai4animationpy/
在线 Demo：https://paulstarke-ai4animationpy.hf.space/

## 二、为什么需要它

AI 角色动画的研究一直有一个痛点：**工具链断裂**。

传统流程是这样的：

```
动捕数据 → Unity 处理特征提取 → 导出数据 → Python 训练模型 → 导出 ONNX → 回到 Unity 推理和可视化
```

这个来回切换的过程非常低效。原版 AI4Animation 在 Unity 中处理 20 小时的动捕数据需要 4 小时以上，搭建一个新实验需要 4 小时以上。

AI4AnimationPy 把这些全部统一了：

| 环节 | AI4AnimationPy | AI4Animation (Unity) |
|------|---------------|---------------------|
| 处理 20h 动捕数据 | < 5 分钟 | > 4 小时 |
| 搭建新实验 | ~10 分钟 | > 4 小时 |
| 训练时可视化输入/输出 | 内置支持 | 需要额外的数据流通信 |
| 反向传播穿过推理 | ✅ 支持 | ❌ 不可能 |
| 量化 | 完整 PyTorch 支持 | 仅 ONNX |

一句话：**研究迭代速度提升了一个数量级**。

## 三、核心特性

### 游戏引擎式架构（ECS）

虽然是 Python 框架，但它的架构设计借鉴了游戏引擎的 Entity-Component-System 模式：

- **Entity**：场景中的一个节点，有位置、旋转、缩放，支持父子层级
- **Component**：挂载在 Entity 上的行为组件（Actor、MotionEditor 等）
- **Scene**：管理所有 Entity 的世界

生命周期回调跟 Unity 几乎一样：`Start()` → `Update()` → `Draw()` → `GUI()`

### 动捕数据导入

支持三种主流格式，开箱即用：

- **GLB** — 无额外依赖
- **BVH** — 无额外依赖
- **FBX** — 需要安装 Autodesk FBX SDK

内部统一存储为 `.npz` 格式（每帧每关节 3D 位置 + 4D 四元数）。

### 内置渲染器

基于 Raylib 的实时渲染管线，支持：
- 延迟着色（Deferred Shading）
- 阴影映射（Shadow Mapping）
- SSAO、Bloom、FXAA
- GPU 加速的骨骼蒙皮
- 四种相机模式（自由、固定、第三人称、轨道）

### 神经网络

内置多种网络架构：
- MLP（多层感知机）
- Autoencoder（自编码器）
- Flow / ConditionalFlow（流匹配）
- CodebookMatching（码本匹配）
- PAE（周期自编码器）

### 三种运行模式

- **Standalone**：带窗口渲染，适合交互式开发和演示
- **Headless**：无窗口，适合服务器端训练
- **Manual**：手动控制更新循环，适合集成到其他系统

## 四、安装

### 环境准备

需要 Python 3.12+ 和 Conda。

macOS：

```bash
conda create -y -n AI4AnimationPY python=3.12 pip
conda activate AI4AnimationPY
pip install torch torchvision torchaudio onnx raylib numpy scipy matplotlib scikit-learn einops pygltflib pyscreenrec tqdm pyyaml ipython
pip install onnxruntime
```

Linux：

```bash
conda create -y -n AI4AnimationPY python=3.12 pip
conda activate AI4AnimationPY
pip install torch torchvision torchaudio onnx raylib numpy scipy matplotlib scikit-learn einops pygltflib pyscreenrec tqdm pyyaml ipython
pip install onnxruntime-gpu
```

Windows（带 CUDA）：

```bash
conda create -n AI4AnimationPY python=3.12
conda activate AI4AnimationPY
pip install msvc-runtime==14.40.33807
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
pip install nvidia-cudnn-cu12==9.3.0.75 nvidia-cuda-runtime-cu12==12.5.82 nvidia-cufft-cu12==11.2.3.61
pip install onnxruntime-gpu==1.19.0
```

### 安装框架

```bash
git clone https://github.com/facebookresearch/ai4animationpy.git
cd ai4animationpy
pip install -e .
```

### 验证安装

```bash
# 无窗口模式验证
python Demos/Empty/Program.py

# 有窗口模式验证（会弹出一个渲染窗口）
python Demos/Actor/Program.py
```

## 五、快速上手

### Hello World：最简程序

```python
from ai4animation import AI4Animation


class Program:
    def Start(self):
        print("Hello World")

    def Update(self):
        pass

    def Draw(self):
        pass

    def GUI(self):
        pass


if __name__ == "__main__":
    AI4Animation(Program(), mode=AI4Animation.Mode.HEADLESS)
```

引擎会调用一次 `Start()`，然后无限循环 `Update()`。把 `HEADLESS` 换成 `STANDALONE` 就会打开一个渲染窗口。

### 加载角色模型

```python
from ai4animation import Actor, AI4Animation, Rotation, Time, Vector3


class Program:
    def Start(self):
        entity = AI4Animation.Scene.AddEntity("Actor")
        self.Actor = entity.AddComponent(Actor, "path/to/character.glb")
        self.Actor.Entity.SetPosition(Vector3.Create(0, 0, 0))

    def Update(self):
        self.Actor.Entity.SetRotation(Rotation.Euler(0, 120 * Time.TotalTime, 0))
        self.Actor.SyncFromScene()


if __name__ == "__main__":
    AI4Animation(Program(), mode=AI4Animation.Mode.STANDALONE)
```

这会在窗口中显示一个持续旋转的 3D 角色。

### 导入动捕数据

```python
from ai4animation import Motion

# 从不同格式加载
motion = Motion.LoadFromGLB("character.glb")
motion = Motion.LoadFromBVH("character.bvh", scale=0.01)  # BVH 通常是厘米，需要缩放

# 保存为内部格式
motion.SaveToNPZ("character")

# 从内部格式加载
motion = Motion.LoadFromNPZ("character.npz")
```

批量转换整个目录：

```bash
convert --input_dir path/to/motions --output_dir path/to/output --skeleton Cranberry
```

### 播放动画

```python
from ai4animation import (
    AI4Animation, ContactModule, Dataset, MotionEditor,
    MotionModule, GuidanceModule, RootModule,
)


class Program:
    def Start(self):
        editor = AI4Animation.Scene.AddEntity("MotionEditor")
        editor.AddComponent(
            MotionEditor,
            Dataset(
                "path/to/npz/folder",
                [
                    lambda x: RootModule(x, "Hips", "LeftUpLeg", "RightUpLeg", "LeftArm", "RightArm"),
                    lambda x: MotionModule(x),
                    lambda x: ContactModule(x, [
                        ("LeftFoot", 0.1, 0.25),
                        ("LeftToeBase", 0.05, 0.25),
                        ("RightFoot", 0.1, 0.25),
                        ("RightToeBase", 0.05, 0.25),
                    ]),
                    lambda x: GuidanceModule(x),
                ],
            ),
            "path/to/character.glb",
            bone_names,
        )
        AI4Animation.Standalone.Camera.SetTarget(editor)

    def Update(self):
        pass


if __name__ == "__main__":
    AI4Animation(Program())
```

MotionEditor 提供了一个 GUI 时间轴，可以拖动浏览动画片段，同时可视化各个 Module 提取的特征（根轨迹、脚部接触、引导信号等）。

## 六、训练神经网络

这是框架最核心的价值——训练和可视化在同一个环境里。

### 完整训练流程

```
Dataset（加载 .npz）→ DataSampler（采样批次）→ FeedTensor（组装输入）
    → Network（前向传播）→ ReadTensor（读取输出）→ Loss → Optimizer
```

### 示例：训练一个 MLP

```python
import numpy as np
import torch
from ai4animation import AdamW, AI4Animation, CyclicScheduler, MLP, Plotting, Tensor


class Program:
    def Start(self):
        self.Network = Tensor.ToDevice(
            MLP.Model(input_dim=1, output_dim=100, hidden_dim=128, dropout=0.1)
        )
        self.Optimizer = AdamW(self.Network.parameters(), lr=1e-4, weight_decay=1e-4)
        self.Scheduler = CyclicScheduler(
            optimizer=self.Optimizer,
            batch_size=32, epoch_size=320,
            restart_period=10, t_mult=2, policy="cosine", verbose=True,
        )
        self.LossHistory = Plotting.LossHistory("Loss", drawInterval=500, yScale="log")
        self.Trainer = self.Training()

    def Update(self):
        try:
            next(self.Trainer)
        except StopIteration:
            pass

    def Training(self):
        for epoch in range(1, 151):
            for _ in range(10):
                x = np.random.uniform(0, 1, (32, 1)).astype(np.float32)
                y = (np.linspace(-1, 1, 100).reshape(1, -1) ** 2) * x

                xBatch = Tensor.ToDevice(torch.tensor(x))
                yBatch = Tensor.ToDevice(torch.tensor(y))

                _, losses = self.Network.learn(xBatch, yBatch, epoch == 1)
                self.Optimizer.zero_grad()
                sum(losses.values()).backward()
                self.Optimizer.step()
                self.Scheduler.batch_step()

                for k, v in losses.items():
                    self.LossHistory.Add((Plotting.ToNumpy(v), k))
                yield

            self.Scheduler.step()
            self.LossHistory.Print()


if __name__ == "__main__":
    AI4Animation(Program(), mode=AI4Animation.Mode.STANDALONE)
```

关键设计：`Training()` 是一个 **generator**，每个 batch 后 `yield` 一次，让引擎有机会更新渲染。这样你可以在 Standalone 模式下实时看到训练过程中的 loss 曲线和动画效果。

### 导出模型

训练完成后导出为 ONNX 用于部署：

```python
from ai4animation import Utility, ONNXNetwork

# 导出
Utility.SaveONNX(network, input_dim, "model.onnx")

# 推理
model = ONNXNetwork("model.onnx")
output = model.Run(input_tensor)
```

## 七、可用的公开数据集

框架兼容多个公开动捕数据集：

| 数据集 | 角色 | 格式 | 说明 |
|--------|------|------|------|
| [Cranberry](https://github.com/sebastianstarke/AI4Animation) | Cranberry | FBX & GLB | SIGGRAPH 2024 论文配套 |
| [100Style retargeted](https://github.com/orangeduck/100style-retarget) | Geno | BVH / FBX | 100 种风格化运动 |
| [LaFan](https://github.com/ubisoft/ubisoft-laforge-animation-dataset) | Ubisoft LaFan | BVH | Ubisoft 开源动捕数据 |
| [ZeroEggs retargeted](https://github.com/orangeduck/zeroeggs-retarget) | Geno | BVH / FBX | 语音驱动手势 |
| [Motorica retargeted](https://github.com/orangeduck/motorica-retarget) | Geno | BVH / FBX | 多样化运动 |
| [NSM](https://github.com/sebastianstarke/AI4Animation/tree/master/AI4Animation/SIGGRAPH_Asia_2019) | Anubis | BVH | 场景交互运动 |
| [MANN](https://github.com/sebastianstarke/AI4Animation/tree/master/AI4Animation/SIGGRAPH_2018) | Dog | BVH | 四足动物运动 |

## 八、内置 Demo

项目自带多个交互式 Demo，都在 `Demos/` 目录下：

| Demo | 说明 |
|------|------|
| Biped Locomotion | 基于 100Style 数据集训练的风格化双足运动控制器 |
| Quadruped | 四足动物（狗）运动控制，支持步态切换和动作姿势 |
| Training | 未来运动预测的交互式训练可视化 |
| ECS | Entity 层级和组件系统演示 |
| IK | 实时逆运动学（FABRIK）求解 |
| MocapImport | GLB/FBX/BVH 动捕数据导入 |
| MotionEditor | 动画浏览和特征可视化编辑器 |

运行方式：

```bash
python Demos/Locomotion/Program.py
python Demos/Quadruped/Program.py
python Demos/IK/Program.py
```

也可以直接在浏览器里体验 Web Demo：https://paulstarke-ai4animationpy.hf.space/

## 九、架构速览

```
ai4animation/
├── AI4Animation.py          # 引擎入口
├── Scene.py                 # 场景管理
├── Entity.py                # 场景图节点
├── Components/
│   ├── Actor.py             # 骨骼角色
│   ├── MotionEditor.py      # 动画播放 UI
│   └── MeshRenderer.py      # 静态网格渲染
├── Animation/
│   ├── Motion.py            # 帧数据 + 骨骼层级
│   ├── Dataset.py           # NPZ 文件加载
│   ├── RootModule.py        # 根轨迹
│   ├── ContactModule.py     # 脚部接触检测
│   ├── MotionModule.py      # 全身轨迹
│   └── GuidanceModule.py    # 姿态引导
├── Math/
│   ├── Tensor.py            # NumPy/PyTorch 双后端
│   ├── Quaternion.py        # 四元数运算
│   └── Vector3.py           # 3D 向量运算
├── AI/
│   ├── DataSampler.py       # 多线程批次生成
│   ├── Networks/            # MLP、AE、Flow 等
│   ├── FeedTensor.py        # 输入缓冲
│   ├── ReadTensor.py        # 输出缓冲
│   └── ONNXNetwork.py       # ONNX 推理
├── Import/
│   ├── GLBImporter.py       # GLB 解析
│   ├── BVHImporter.py       # BVH 解析
│   └── BatchConverter.py    # CLI 批量转换
├── IK/
│   └── FABRIK.py            # 逆运动学求解器
└── Standalone/
    ├── RenderPipeline.py    # 延迟渲染管线
    ├── Camera.py            # 相机系统
    └── SkinnedMesh.py       # GPU 骨骼蒙皮
```

八层架构：Engine → ECS → Animation → Math → AI → Import → Rendering → IK，每层职责清晰，可以独立使用。

## 十、适合谁

- **动画/游戏 AI 研究者** — 不用再在 Unity 和 Python 之间来回切换，训练和可视化在同一个环境
- **动捕数据工程师** — 快速导入、转换、浏览动捕数据，支持主流格式
- **技术美术** — 理解 AI 动画管线的工作原理，用 Python 快速原型验证
- **学生** — 学习角色动画和运动合成的好起点，代码结构清晰，文档完善

不适合的场景：如果你需要的是生产级游戏引擎集成，目前还是用原版 AI4Animation（Unity）更合适。AI4AnimationPy 的定位是研究和快速迭代。

## 十一、相关资源

- 项目主页：https://github.com/facebookresearch/ai4animationpy
- 完整文档：https://facebookresearch.github.io/ai4animationpy/
- 在线 Demo：https://paulstarke-ai4animationpy.hf.space/
- 原版 Unity 项目：https://github.com/sebastianstarke/AI4Animation
- Demo 视频：https://youtu.be/LKl7MzFENUs
- 许可证：CC BY-NC 4.0（非商业用途）

---

*最后更新：2026-04-21*
