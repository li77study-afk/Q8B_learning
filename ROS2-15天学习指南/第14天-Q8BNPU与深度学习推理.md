# 第 14 天：Q8B NPU、QAIRT、QAI AppBuilder 与 ROS2 推理节点

## 今日目标与时间

总计约 7～9 小时：1 小时理解 Q8B NPU 软件栈，1 小时启用并验证 `fastrpc`，1.5 小时安装 QAI AppBuilder 并跑官方 NPU 示例，1 小时理解 QAIRT 转换边界，1.5 小时把 QNNContext 接入 ROS2 图像话题，1～2 小时与 CPU 基线对比和分析性能。今天 NPU 验证是必做项，CPU 代码只是对照。

## 1. 先画清楚 Q8B 的 NPU 软件栈

Q8B 的实际硬件是 `SC8280XP`，NPU 是 Qualcomm Hexagon，Radxa 文档将它归为 `V68`。同一个芯片在不同工具中的名字不同，必须记住下面这张表：

| 用途 | Q8B 应填的值 | 解释 |
|---|---|---|
| 硬件身份 | `SC8280XP` | Snapdragon 8cx Gen 3 平台 |
| Hexagon 架构 | `V68` / `v68` | fastrpc 和 DSP 库使用 |
| fastrpc 测试 | `fastrpc_test -a v68` | 板端验证 DSP/NPU 运行环境 |
| QAI AppBuilder 下载模型 | `--chipset 6490` | SC8280XP 与 QCS6490 共用 V68 模型目标 |
| QNN EP 环境 | `PRODUCT_SOC=8280 DSP_ARCH=68` | Radxa QNN EP 文档的变量 |
| QAIRT context 配置 | `dsp_arch=v68, soc_id=37` | SC8280XP 的 QAIRT 配置 |

模型运行链是：

```text
PyTorch / ONNX
       -> QAIRT / QAI Hub / AIMET
       -> DLC 或量化模型
       -> Context-Binary (.bin)
       -> QNNContext / qnn-net-run
       -> Hexagon HTP NPU
```

`onnxruntime` 直接使用默认 CPU provider 不算 NPU。只有看到 QNN Execution Provider 或 QAI AppBuilder 的 `Runtime.HTP` / QNNContext，并且板端环境验证通过，才算真正用上 Q8B NPU。

## 2. 板端启用和验证 NPU

先看系统和设备，不要直接复制库文件：

```bash
uname -m
cat /etc/os-release
ls -l /dev/fastrpc-* 2>/dev/null || true
find /usr/lib/dsp -maxdepth 3 -type f \( -name '*Qnn*' -o -name '*qnn*' \) 2>/dev/null | head -n 30
```

Q8B 的 T2 或更高版本 Radxa 系统镜像通常已经预装 NPU 运行环境。如果 `/dev/fastrpc-cdsp` 不存在，先确认你是不是使用了通用 Ubuntu ISO。Radxa 文档还特别说明，`fastrpc` 软件包依赖 Radxa 官方 apt 源；第三方镜像不要盲目安装。

若确认是对应的 Radxa 官方镜像，再执行：

```bash
sudo apt update
sudo apt install -y fastrpc libcdsprpc1
sudo apt install -y fastrpc-test
fastrpc_test -a v68
```

- `fastrpc` 和 `libcdsprpc1` 提供运行库；`fastrpc-test` 提供验证程序。
- `-a v68` 选择 Q8B 的 Hexagon V68 架构。
- 预期最后出现 `RESULT: All applicable tests PASSED`。
- 日志中可能包含 DSP 调试信息；先看最后的 PASS/FAIL，不要只看 WARNING。

把结果保存：

```bash
mkdir -p ~/q8b_ros2_course/notes/day14
fastrpc_test -a v68 2>&1 | tee ~/q8b_ros2_course/notes/day14/fastrpc-v68.txt
```

如果失败，记录：系统镜像、`ls -l /dev/fastrpc-*`、失败行和温度。不要跳过这一步直接安装 Python 包；用户态库缺失时，Python 层一定无法修复硬件运行环境。

## 3. 安装 QAI AppBuilder：先用官方现成模型

Radxa 官方推荐 QAI AppBuilder，它把 QNNContext 封装为 Python API，并提供图像分类、YOLOv8 检测、分割、姿态和超分辨率示例。先创建隔离环境：

```bash
mkdir -p ~/q8b_ros2_course/npu
cd ~/q8b_ros2_course/npu
python3 --version
sudo apt install -y python3-venv git
python3 -m venv --system-site-packages .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -c "import sys; print(sys.executable); print(sys.version)"
```

`--system-site-packages` 让虚拟环境可以看到 apt 安装的 `rclpy`、OpenCV 等系统库，同时把 QAI AppBuilder 隔离在 `.venv`。每个新终端都要重新 `source .venv/bin/activate`。

Radxa 页面当前给出的 Linux aarch64 wheel 示例是 Python 3.12 和 `v2.48.40`：

```bash
python -m pip install \
  https://github.com/qualcomm/qai-appbuilder/releases/download/v2.48.40/qai_appbuilder-2.48.40-cp312-cp312-manylinux_2_39_aarch64.whl
python -c "import qai_appbuilder; print(qai_appbuilder.__file__)"
```

只有当 `python3 --version` 是 3.12 时才直接使用这个 `cp312` wheel。版本不匹配时，打开 Radxa 页面和 GitHub Release，选择与实际 Python ABI 匹配的 aarch64 wheel；不要用 `--force` 绕过。

配置 QAI AppBuilder 的 DSP 库路径：

```bash
export ADSP_LIBRARY_PATH="$(python -c "import os, qai_appbuilder; print(os.path.join(os.path.dirname(qai_appbuilder.__file__), 'libs'))")"
echo "$ADSP_LIBRARY_PATH"
find "$ADSP_LIBRARY_PATH" -maxdepth 1 -type f -name '*Skel.so' -print
```

`ADSP_LIBRARY_PATH` 指向 Hexagon skel 动态库；没有这个变量，模型可能下载成功但无法在 HTP/NPU 初始化。

## 4. 跑通 Radxa 官方 QAI AppBuilder 示例

克隆与 wheel 版本相匹配的仓库：

```bash
cd ~/q8b_ros2_course/npu
git clone --depth 1 --branch v2.48.40 https://github.com/qualcomm/qai-appbuilder.git
cd qai-appbuilder
python -m pip install requests tqdm qai-hub py3-wget Pillow torch torchvision opencv-python-headless
```

这些示例依赖可能较大；如果 `torch` 在 ARM64 上没有匹配 wheel，先保留已经安装的 `qai_appbuilder`，并把依赖错误记下来，再按 Radxa 示例页的当前依赖版本处理。

运行 GoogLeNet 分类示例：

```bash
source ~/q8b_ros2_course/npu/.venv/bin/activate
export ADSP_LIBRARY_PATH="$(python -c "import os, qai_appbuilder; print(os.path.join(os.path.dirname(qai_appbuilder.__file__), 'libs'))")"
cd ~/q8b_ros2_course/npu/qai-appbuilder/samples
python3 ComputerVision/Image_Classification/googlenet/googlenet.py --chipset 6490
```

首次运行会从 AI Hub 下载测试图片、标签和针对目标芯片的 `.bin` 模型。SC8280XP 必须使用 `--chipset 6490`，不是 `8280`。成功时应看到 Top 5 分类结果；出现 HTP/FastRPC WARNING 不一定失败，查看退出码和结果。

再运行 Q8B 最有价值的目标检测示例：

```bash
python3 ComputerVision/Object_Detection/yolov8_det/yolov8_det.py --chipset 6490
ls -lh ComputerVision/Object_Detection/yolov8_det/output.png
```

Radxa 文档指出该示例在不同模型包中可能出现输出顺序差异，QCS6490/SC8280XP 需要按示例页的说明修正输出解析，否则可能出现 NMS 的 `IndexError`。这不是“模型不能上 NPU”，而是后处理代码对输出布局的假设错误。

## 5. QAIRT 模型转换路线（理解主机端和板端的边界）

如果以后要把自己的 PyTorch/ONNX 模型放到 Q8B NPU，不能只把 `.onnx` 拷到板上。Radxa 的完整流程是：

```text
主机 x86_64 Ubuntu 22.04
  ONNX/PyTorch -> qairt-converter -> DLC
  -> qairt-quantizer + calibration raw -> 量化 DLC
  -> qnn-context-binary-generator + V68 配置 -> .bin
Q8B SC8280XP
  .bin + qnn-net-run/QNNContext -> Hexagon V68 HTP NPU
```

QAIRT SDK 页面当前要求 x86_64 Linux、Ubuntu 22.04、Python 3.10。Q8B 是 ARM64，因此完整 SDK 的转换工具应在 x86 Linux PC 或 x86 Ubuntu 22.04 虚拟机/WSL 环境准备；Q8B 负责运行转换后的模型。若当前只有 Windows PC，没有 x86 Ubuntu，先用 QAI Hub 或 Radxa 已编译模型，不要在 Q8B 上硬装不匹配的 SDK。

在 x86 Ubuntu 主机上只做环境认识练习：

```bash
export QAIRT_VERSION=2.42.0.251225
wget "https://softwarecenter.qualcomm.com/api/download/software/sdks/Qualcomm_AI_Runtime_Community/All/${QAIRT_VERSION}/v${QAIRT_VERSION}.zip"
unzip "v${QAIRT_VERSION}.zip" -d qairt
cd "qairt/${QAIRT_VERSION}"
source bin/envsetup.sh
echo "$QNN_SDK_ROOT"
qairt-converter --help | less
qairt-quantizer --help | less
qnn-context-binary-generator --help | less
```

- `QAIRT_VERSION` 是 shell 环境变量，`${...}` 会展开变量值。
- `unzip -d qairt` 把压缩包解到指定目录；`cd` 进入解压后的 SDK。
- `envsetup.sh` 设置工具路径；`qairt-converter`、`qairt-quantizer` 和 `qnn-context-binary-generator` 分别对应转换、量化、生成 Context-Binary。
- 这里只验证工具可见，不要凭空对自己的模型执行转换；必须先知道输入张量名、形状、颜色顺序和校准数据。

SC8280XP 的 Context-Binary 配置关键值是 `dsp_arch: v68`、`soc_id: 37`。这与 QAI AppBuilder 下载模型的 `--chipset 6490` 是不同字段，不能互换。模型转换完成后，按 Radxa 文档把 `.bin`、`qnn-net-run`、`libQnnHtp.so`、`libQnnHtpV68Stub.so` 和 `libQnnHtpV68Skel.so` 放到 Q8B，再用 raw 输入列表运行。今天先掌握边界，实际自定义模型转换作为 15 天后的进阶项目。

## 6. 把 QNNContext 接入 ROS2 图像节点

先复制官方下载好的 GoogLeNet context binary 到课程目录：

```bash
find ~/q8b_ros2_course/npu/qai-appbuilder/samples \
  -path '*/Image_Classification/googlenet/models/googlenet.bin' -print
mkdir -p ~/q8b_ros2_course/models/q8b_npu
cp ~/q8b_ros2_course/npu/qai-appbuilder/samples/ComputerVision/Image_Classification/googlenet/models/googlenet.bin \
  ~/q8b_ros2_course/models/q8b_npu/
ls -lh ~/q8b_ros2_course/models/q8b_npu/googlenet.bin
```

进入 ROS2 视觉包并创建 NPU 节点：

```bash
source ~/q8b_ros2_course/npu/.venv/bin/activate
cd ~/ros2_ws/src/q8b_vision
nano q8b_vision/npu_classifier.py
```

输入：

```python
import cv2
import numpy as np
import qai_appbuilder
import rclpy
from cv_bridge import CvBridge
from qai_appbuilder import LogLevel, ProfilingLevel, QNNConfig, QNNContext, Runtime
from rclpy.node import Node
from rclpy.qos import qos_profile_sensor_data
from sensor_msgs.msg import Image
from std_msgs.msg import String


class GoogleNet(QNNContext):
    def Inference(self, input_data):
        return super().Inference([input_data])[0]


class NpuClassifier(Node):
    def __init__(self):
        super().__init__('npu_classifier')
        self.declare_parameter('input_topic', '/image_raw')
        self.declare_parameter('model', '')
        model_path = self.get_parameter('model').value
        if not model_path:
            raise ValueError('parameter model must point to googlenet.bin')
        QNNConfig.Config(Runtime.HTP, LogLevel.WARN, ProfilingLevel.BASIC)
        self.model = GoogleNet('googlenet', model_path)
        self.bridge = CvBridge()
        self.result_pub = self.create_publisher(String, 'npu_classification', 10)
        topic = self.get_parameter('input_topic').value
        self.subscription = self.create_subscription(
            Image, topic, self.infer, qos_profile_sensor_data)
        self.count = 0

    def infer(self, message):
        image = self.bridge.imgmsg_to_cv2(message, desired_encoding='bgr8')
        rgb = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
        resized = cv2.resize(rgb, (224, 224)).astype(np.float32) / 255.0
        input_tensor = np.expand_dims(resized, axis=0)  # NHWC: 1,224,224,3
        output = self.model.Inference(input_tensor).reshape(-1)
        index = int(np.argmax(output))
        result = String()
        result.data = f'class_index={index}, raw_score={float(output[index]):.4f}'
        self.result_pub.publish(result)
        self.count += 1
        if self.count % 10 == 0:
            self.get_logger().info(f'npu_frames={self.count}, {result.data}')

    def destroy_node(self):
        del self.model
        super().destroy_node()


def main(args=None):
    rclpy.init(args=args)
    node = NpuClassifier()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

这里的关键区别是：`QNNConfig.Config(Runtime.HTP, ...)` 明确选择 HTP，`GoogleNet` 加载的是 QAI Hub 生成的 `googlenet.bin`，不是普通 `.onnx`。官方示例把输入整理为 NHWC；不要把第 9 天 OpenCV DNN 的 NCHW 预处理不加检查地搬过来。

在 `q8b_vision/setup.py` 的 `console_scripts` 增加：

```python
'npu_classifier = q8b_vision.npu_classifier:main',
```

保持 NPU 虚拟环境激活，重新构建：

```bash
source ~/q8b_ros2_course/npu/.venv/bin/activate
cd ~/ros2_ws
python -m colcon build --packages-select q8b_vision --symlink-install
source install/setup.bash
python -c "import rclpy, qai_appbuilder; print('ROS2 and QAI AppBuilder imports passed')"
ros2 run q8b_vision npu_classifier --ros-args \
  -p input_topic:=/image_raw \
  -p model:="$HOME/q8b_ros2_course/models/q8b_npu/googlenet.bin"
```

另一个终端也要激活同一个 venv，然后：

```bash
source ~/q8b_ros2_course/npu/.venv/bin/activate
source ~/ros2_ws/install/setup.bash
ros2 topic echo /npu_classification
```

如果 `ros2 run` 找不到 `qai_appbuilder`，说明构建或运行使用了错误的 Python 解释器。比较：

```bash
which python
which ros2
head -n 1 "$(ros2 pkg prefix q8b_vision)/lib/q8b_vision/npu_classifier"
```

三者应指向同一 venv 体系。不要在回调里反复加载 `.bin`，模型加载只允许发生一次。

### 6.1 可选：用 QNN Execution Provider 保留 ONNX Runtime 接口

QAI AppBuilder 适合直接使用 Qualcomm 预编译的 `.bin`；如果你的项目已经围绕 ONNX Runtime 编写，Radxa 还提供 QNN Execution Provider。它不是普通 CPU provider，而是把量化 ONNX 图交给 QNN/HTP。先用官方 QNN EP 页面提供的 aarch64 wheel 和 AI Hub 的 INT8/w8a8 模型，不要把第 8 节的普通浮点 `mobilenetv2-7.onnx` 直接当成 NPU 模型。

在已经激活的 Q8B NPU venv 中：

先把命令中的 `<path_to_qairt>` 替换成 QAIRT SDK 的实际目录；尖括号是占位符，不能原样输入，否则 Bash 会把它当成重定向符号。

```bash
source <path_to_qairt>/2.42.0.251225/bin/envsetup.sh
python --version
python -m pip install \
  https://github.com/ZIFENG278/onnxruntime/releases/download/v1.23.2/onnxruntime_qnn-1.23.2-cp312-cp312-linux_aarch64.whl
export PRODUCT_SOC=8280
export DSP_ARCH=68
export ADSP_LIBRARY_PATH="$QNN_SDK_ROOT/lib/hexagon-v${DSP_ARCH}/unsigned"
python -c "import onnxruntime; print(onnxruntime.get_available_providers())"
```

只有 Python 3.12 才直接使用 `cp312` wheel；`QNN_SDK_ROOT` 必须来自 QAIRT SDK 的 `source bin/envsetup.sh`。把实际 AI Hub 下载的量化 ONNX 模型路径填入下面脚本：

```bash
nano ~/q8b_ros2_course/npu/run_qnn_ep.py
```

```python
import numpy as np
import onnxruntime as ort


options = ort.SessionOptions()
options.add_session_config_entry('session.disable_cpu_ep_fallback', '1')
session = ort.InferenceSession(
    '/absolute/path/to/quantized_model.onnx',
    sess_options=options,
    providers=['QNNExecutionProvider'],
    provider_options=[{'backend_path': 'libQnnHtp.so'}],
)
input_name = session.get_inputs()[0].name
input_shape = session.get_inputs()[0].shape
print('provider=', session.get_providers())
print('input=', input_name, input_shape)
sample = np.ones((1, 3, 224, 224), dtype=np.uint8)
outputs = session.run(None, {input_name: sample})
print('output_count=', len(outputs), 'first_shape=', outputs[0].shape)
```

`session.disable_cpu_ep_fallback=1` 是故意的：只要有算子不能在 QNN HTP 上执行，就让程序报错，而不是悄悄退回 CPU。运行成功后才把这段 session 初始化移入 ROS2 callback 外部，并在 callback 中复用同一个 session。QAI AppBuilder 和 QNN EP 二选一即可，不要在同一个节点中同时加载两个 NPU runtime。

## 7. CPU 基线：先建立正确的深度学习概念

- 训练：用带标签数据更新模型参数，通常不在 Q8B 上做。
- 推理：固定模型参数，对新图像计算结果，Q8B 主要做这个。
- 预处理：尺寸、颜色通道、归一化、NCHW/NHWC；错一个就可能得到错误结果。
- 后处理：把 logits/概率转换成类别、置信度或框。
- ONNX：跨框架的模型交换格式；OpenCV DNN 可以在 CPU 上读取部分 ONNX 模型。

今天选择轻量图像分类模型，不做复杂目标检测后处理。掌握数据流和性能方法后，再换 YOLO 等检测模型。

## 8. CPU 基线：检查并安装 OpenCV DNN

```bash
python3 -c "import cv2; print(cv2.__version__); print(hasattr(cv2, 'dnn'))"
python3 -c "import numpy; print(numpy.__version__)"
```

如果缺少：

```bash
sudo apt update
sudo apt install -y python3-opencv python3-numpy wget
```

使用系统 apt 的好处是 ARM64 上通常比盲目 `pip install torch` 更稳定、占空间更少。深度学习框架以后可在有明确硬件加速支持时再安装。

## 9. CPU 基线：下载一个轻量 ONNX 分类模型

先创建模型目录并查看空间：

```bash
mkdir -p ~/q8b_ros2_course/models ~/q8b_ros2_course/model_samples
df -h ~
cd ~/q8b_ros2_course/models
wget -O mobilenetv2-7.onnx \
  https://github.com/onnx/models/raw/main/validated/vision/classification/mobilenet/model/mobilenetv2-7.onnx
ls -lh mobilenetv2-7.onnx
```

该 URL 来自 ONNX Model Zoo 的官方模型仓库路径；若仓库改版导致 404，打开 https://github.com/onnx/models 搜索 `mobilenetv2-7.onnx`，不要下载来历不明的模型。

下载 ImageNet 标签：

```bash
wget -O imagenet_classes.txt \
  https://raw.githubusercontent.com/pytorch/hub/master/imagenet_classes.txt
wc -l imagenet_classes.txt
```

`wc -l` 检查标签行数；应该接近 1000。模型输出类别索引与标签文件必须匹配。

## 10. CPU 基线：先脱离 ROS2 测一张图片

把第 8 天拍到的 JPEG 放进 `model_samples`；如果没有，使用任何合法 jpg。编写：

```bash
nano ~/q8b_ros2_course/models/classify_one.py
```

输入：

```python
import sys

import cv2


model_path, image_path, labels_path = sys.argv[1:4]
net = cv2.dnn.readNetFromONNX(model_path)
image = cv2.imread(image_path)
if image is None:
    raise RuntimeError(f'cannot read image: {image_path}')

blob = cv2.dnn.blobFromImage(
    image, scalefactor=1.0 / 255.0, size=(224, 224),
    swapRB=True, crop=False)
net.setInput(blob)
output = net.forward().reshape(-1)
index = int(output.argmax())
with open(labels_path, encoding='utf-8') as file:
    labels = [line.strip() for line in file if line.strip()]
print(f'class_index={index}, raw_score={float(output[index]):.4f}, label={labels[index]}')
```

运行：

```bash
python3 ~/q8b_ros2_course/models/classify_one.py \
  ~/q8b_ros2_course/models/mobilenetv2-7.onnx \
  ~/q8b_ros2_course/model_samples/frame01.jpg \
  ~/q8b_ros2_course/models/imagenet_classes.txt
```

重点不是盲目信任分类结果，而是核对：输入是否读到、blob 的尺寸和通道顺序、模型输出形状、标签索引。不同模型需要的 mean/scale 不同；本例的预处理必须以所下载模型的说明为准，不要把别的模型的预处理参数复制过来。

## 11. CPU 基线：建立 ROS2 推理节点

在第 9 天包中添加文件：

```bash
cd ~/ros2_ws/src/q8b_vision
nano q8b_vision/classifier.py
```

输入：

```python
import cv2
import rclpy
from cv_bridge import CvBridge
from rclpy.node import Node
from rclpy.qos import qos_profile_sensor_data
from sensor_msgs.msg import Image
from std_msgs.msg import String


class Classifier(Node):
    def __init__(self):
        super().__init__('classifier')
        self.declare_parameter('input_topic', '/image_raw')
        self.declare_parameter('model', '')
        self.declare_parameter('labels', '')
        model = self.get_parameter('model').value
        labels_path = self.get_parameter('labels').value
        self.net = cv2.dnn.readNetFromONNX(model)
        with open(labels_path, encoding='utf-8') as file:
            self.labels = [line.strip() for line in file if line.strip()]
        self.bridge = CvBridge()
        self.result_pub = self.create_publisher(String, 'classification', 10)
        topic = self.get_parameter('input_topic').value
        self.sub = self.create_subscription(
            Image, topic, self.infer, qos_profile_sensor_data)
        self.count = 0

    def infer(self, message):
        image = self.bridge.imgmsg_to_cv2(message, desired_encoding='bgr8')
        blob = cv2.dnn.blobFromImage(
            image, 1.0 / 255.0, (224, 224),
            swapRB=True, crop=False)
        self.net.setInput(blob)
        output = self.net.forward().reshape(-1)
        index = int(output.argmax())
        result = String()
        result.data = f'{self.labels[index]} raw_score={float(output[index]):.4f}'
        self.result_pub.publish(result)
        self.count += 1
        if self.count % 10 == 0:
            self.get_logger().info(f'processed={self.count}, {result.data}')


def main(args=None):
    rclpy.init(args=args)
    node = Classifier()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

在 setup.py 增加：

```python
'classifier = q8b_vision.classifier:main',
```

编译并运行：

```bash
cd ~/ros2_ws
colcon build --packages-select q8b_vision --symlink-install
source install/setup.bash
ros2 run q8b_vision classifier --ros-args \
  -p input_topic:=/image_raw \
  -p model:="$HOME/q8b_ros2_course/models/mobilenetv2-7.onnx" \
  -p labels:="$HOME/q8b_ros2_course/models/imagenet_classes.txt"
```

另一个终端：

```bash
ros2 topic echo /classification
```

如果模型文件参数为空或路径错误，节点会在启动时失败；这是有用的快速失败，不要在回调中每帧重新加载模型。

## 12. CPU 基线：用 bag 做推理和性能测量

```bash
ros2 bag play ~/q8b_ros2_course/bags/day08/camera_demo --rate 0.25
```

观察：

```bash
ros2 topic hz /classification
ros2 topic bw /image_raw
btop
```

记录输入 FPS、推理 FPS、CPU、内存、温度和结果稳定性。若输入 30 FPS、推理只能 2 FPS，不能无限积压；第 15 天会加入“只处理最新帧”或降低输入频率的设计。

## 13. 今日验收

- [ ] 能说出 Q8B 的 `SC8280XP`、Hexagon `V68`、`soc_id=37` 和 AppBuilder `--chipset 6490` 的区别。
- [ ] `fastrpc_test -a v68` 通过，并保存了输出日志。
- [ ] QAI AppBuilder GoogLeNet 官方示例在 HTP/NPU 上得到分类结果。
- [ ] 能解释为什么 QAI AppBuilder 使用 `Runtime.HTP`、为什么要设置 `ADSP_LIBRARY_PATH`。
- [ ] 能把 `googlenet.bin` 加载到 ROS2 的 `npu_classifier`，发布 `/npu_classification`。
- [ ] 能区分训练、推理、预处理、后处理和模型格式。
- [ ] 能脱离 ROS2 对一张图片运行 ONNX/OpenCV 推理。
- [ ] ROS2 节点能从 Image 话题读取图像并发布分类 String。
- [ ] 模型只加载一次，且路径通过参数传入。
- [ ] 已测量 NPU/CPU 推理 FPS、CPU、内存和温度，并写出一个性能瓶颈。

## 官方主线

- OpenCV DNN：https://docs.opencv.org/4.x/d2/d58/tutorial_table_of_content_dnn.html
- ONNX Model Zoo：https://github.com/onnx/models
- ROS2 parameters：https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Parameters/
- Q8B NPU 总览：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev
- 板端启用 NPU：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/fastrpc-setup
- NPU 快速验证：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/quick-example
- QAI AppBuilder：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qai-appbuilder
- QAIRT 模型移植：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qairt-usage
- QNN Execution Provider：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qnn-onnxrt-execution-provider
