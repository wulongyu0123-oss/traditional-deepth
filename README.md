# traditional-deepth

`traditional-deepth` 是一套基于 `ROS 2 + HP60C` 的“传统视觉 + 深度几何”孔检测示例工程。

它分成两个 ROS 2 包：

- `hole_detection_msgs`
  这是自定义消息包，只负责定义检测结果的数据格式。
  它不做图像处理，也不做检测。
  它提供了两个消息：
  - `Hole3D.msg`：单个孔的三维中心、直径、置信度
  - `HoleArray.msg`：一帧图像里所有孔的集合

- `plane_hole_detector`
  这是核心算法包。
  它负责订阅 `HP60C` 的彩色图、深度图、相机内参，然后完成：
  - 2D 图像中的孔轮廓检测
  - 孔周围平面的深度拟合
  - 孔中心三维坐标计算
  - 孔实际直径估计
  - 发布检测结果、标注图、RViz 标记

## 工作原理

这套方案没有使用 YOLO。

流程是：

1. 从彩色图中找圆孔候选轮廓
2. 在孔外侧一圈区域采样深度点
3. 用这些深度点拟合孔所在平面
4. 用孔中心像素射线和该平面求交，得到 3D 圆心
5. 把轮廓点投影到平面里拟合圆，得到实际孔径

这样做的好处是：

- 不需要训练数据集
- 适合规则圆孔
- 结果可以直接接到机械臂坐标链

## 目录说明

```text
traditional-deepth/
  README.md
  hole_detection_msgs/
  plane_hole_detector/
```

如果你想单独使用这个工程，通常需要把这两个包放到某个 ROS 2 工作空间的 `src/` 目录下。

## 依赖条件

你需要已经具备以下环境：

- Ubuntu + ROS 2
- `HP60C` 相机驱动可正常工作
- 能发布这些话题：
  - `/ascamera_hp60c/camera_publisher/rgb0/image`
  - `/ascamera_hp60c/camera_publisher/depth0/image_raw`
  - `/ascamera_hp60c/camera_publisher/rgb0/camera_info`

还需要这些 ROS 2 依赖包：

```bash
sudo apt install \
  ros-$ROS_DISTRO-cv-bridge \
  ros-$ROS_DISTRO-message-filters \
  ros-$ROS_DISTRO-tf2-ros \
  ros-$ROS_DISTRO-geometry-msgs \
  ros-$ROS_DISTRO-visualization-msgs
```

## 使用方法

### 方法 1：直接放进你现有的 `hp60c_ws`

如果你已经有工作空间 `/home/user/hp60c_ws`，最简单的方式是把这两个包复制到：

```text
/home/user/hp60c_ws/src/
```

然后编译：

```bash
source /opt/ros/$ROS_DISTRO/setup.bash
cd /home/user/hp60c_ws
colcon build --symlink-install --packages-select hole_detection_msgs plane_hole_detector
source install/setup.bash
```

### 方法 2：新建一个独立工作空间

```bash
mkdir -p ~/traditional_ws/src
cp -r /home/user/traditional-deepth/hole_detection_msgs ~/traditional_ws/src/
cp -r /home/user/traditional-deepth/plane_hole_detector ~/traditional_ws/src/
```

如果你的 `HP60C` 驱动包不在这个工作空间里，还要把相机驱动包也放进来，或者先 source 已经安装好的相机驱动环境。

然后编译：

```bash
source /opt/ros/$ROS_DISTRO/setup.bash
cd ~/traditional_ws
colcon build --symlink-install
source install/setup.bash
```

## 启动步骤

### 第一步：启动 HP60C 相机驱动

如果你已经在别的工作空间里有 `ascamera` 驱动，例如：

```bash
source /opt/ros/$ROS_DISTRO/setup.bash
source /home/user/ascam_ros2_ws/install/setup.bash
ros2 launch ascamera hp60c.launch.py
```

确认相机话题正常：

```bash
ros2 topic list | grep ascamera_hp60c
```

### 第二步：启动孔检测节点

如果你是在 `hp60c_ws` 中构建的：

```bash
source /opt/ros/$ROS_DISTRO/setup.bash
source /home/user/hp60c_ws/install/setup.bash
ros2 launch plane_hole_detector plane_hole_detector.launch.py
```

如果你是在 `traditional_ws` 中构建的：

```bash
source /opt/ros/$ROS_DISTRO/setup.bash
source ~/traditional_ws/install/setup.bash
ros2 launch plane_hole_detector plane_hole_detector.launch.py
```

## 输出话题

运行后会发布这些话题：

- `/plane_hole_detector/annotated_image`
  标注后的图像，能看到检测到的孔中心和三维信息

- `/plane_hole_detector/holes`
  检测结果消息，类型是 `hole_detection_msgs/msg/HoleArray`

- `/plane_hole_detector/markers`
  RViz 可视化标记

查看结果：

```bash
ros2 topic echo /plane_hole_detector/holes
```

## RViz2 查看方法

打开 RViz2：

```bash
rviz2
```

添加：

- `Image`
  话题选择 `/plane_hole_detector/annotated_image`

- `MarkerArray`
  话题选择 `/plane_hole_detector/markers`

这样你可以同时看到图像检测结果和 3D 标记。

## 参数文件

参数文件在：

```text
/home/user/traditional-deepth/plane_hole_detector/config/plane_hole_detector.yaml
```

如果你是从 `hp60c_ws` 中运行，源码位置对应为：

```text
/home/user/hp60c_ws/src/plane_hole_detector/config/plane_hole_detector.yaml
```

最常需要调的参数有：

- `threshold_inverse`
  黑孔白底常用 `true`

- `adaptive_block_size`
  控制自适应阈值窗口大小

- `adaptive_c`
  控制阈值偏移

- `min_contour_area`
  去掉太小的噪声轮廓

- `min_circularity`
  限制轮廓圆度

- `min_axis_ratio`
  限制椭圆长短轴比，避免非圆孔误检

- `target_frame`
  如果填成 `base_link`，节点会尝试通过 `tf2` 把结果转到机械臂基坐标系

## 三维坐标说明

当前节点输出的是每个孔的：

- 三维圆心坐标
- 实际孔径
- 置信度

默认情况下，坐标在相机坐标系下输出。  
如果 `target_frame` 为空字符串，就使用相机图像的 `frame_id`。  
如果 `target_frame` 设置为 `base_link`，并且 TF 正常，输出就会变成机械臂基坐标系。

## 常见问题

### 1. 图像里能看到孔，但没有检测结果

先调这些参数：

- `threshold_inverse`
- `adaptive_block_size`
- `adaptive_c`
- `min_contour_area`
- `min_circularity`

### 2. 2D 位置看起来对，但 3D 坐标明显偏

先检查：

- 深度图和彩色图是否对齐
- `camera_info` 是否正确
- 深度单位是否正常

### 3. 孔中心坐标有，但直径不稳定

通常是这些原因：

- 孔边缘深度噪声大
- 工件表面反光
- 相机角度过斜
- 轮廓提取不稳定

### 4. 机械臂坐标不对

先检查：

- `target_frame` 是否设置正确
- 相机到 `base_link` 的 TF 是否已经发布
- 手眼标定或外参标定是否正确

## 下一步建议

建议按这个顺序继续推进：

1. 先把当前传统方法跑通
2. 调到能稳定输出孔中心和直径
3. 接入 `base_link` 坐标变换
4. 最后再把 YOLO 融合进来，作为候选区域生成器或复核模块

这样会更稳，也更容易定位问题。
