# 探索规划模块

基于四足机器人的危险源自主搜索与识别比赛 —— 探索规划（exploration + planning）模块。

负责根据机器人当前的栅格地图，检测未知区域边界（Frontier），生成探索候选点，并结合机器人当前位置与信息增益选择合适的探索目标，通过 `move_base` 控制机器人自主移动，实现未知环境的持续探索。

核心流程：

1. 根据 `/map` 获取当前环境信息
2. 使用 RRT 方法搜索未知区域边界
3. 对候选 Frontier 进行聚类和筛选
4. 根据距离和信息增益选择探索目标
5. 将目标发送给 `move_base`
6. 机器人移动后地图更新，重新进行 Frontier 检测

---

## 环境

* ROS1 Noetic
* Gazebo Classic
* Unitree Go2
* `move_base`
* 栅格地图：`nav_msgs/OccupancyGrid`

主要依赖：

```bash
sudo apt install ros-noetic-navigation
sudo apt install python3-numpy
sudo apt install python3-sklearn
```

---

## 输入

### `/map`

类型：

```text
nav_msgs/OccupancyGrid
```

探索规划模块的主要输入地图。

地图需要包含：

```text
-1    Unknown（未知区域）
 0    Free（空闲区域）
100   Occupied（障碍物）
```

探索算法主要寻找：

```text
已知区域 ↔ 未知区域
```

之间的 Frontier。

因此输入地图需要能够正常区分**已知区域和未知区域**。

---

### TF

需要能够获取机器人在地图坐标系中的位置。

主要使用：

```text
map → odom → base_footprint
```

默认参数：

```text
global_frame: /map
robot_frame: base_footprint
```

---

## 输出

### `/detected_points`

类型：

```text
geometry_msgs/PointStamped
```

RRT 探索检测得到的 Frontier 候选点。

---

### `/filtered_points`

类型：

```text
go2_rrt_exploration/PointArray
```

经过聚类和筛选后的探索候选点。

这些点最终提供给目标分配模块进行选择。

---

### `/move_base/goal`

探索规划模块最终通过 `move_base` Actionlib 发送导航目标。

机器人接收到目标后自主移动。

---

## 算法流程

整体流程如下：

```text
                         /map
                           │
                           ↓
                    当前环境栅格地图
                           │
                           ↓
                    RRT Frontier检测
                           │
                           ↓
                  /detected_points
                           │
                           ↓
                    Frontier聚类
                    与无效点筛选
                           │
                           ↓
                  /filtered_points
                           │
                           ↓
                  探索目标选择
                           │
                    ┌──────┴──────┐
                    │             │
                  距离          信息增益
                    │             │
                    └──────┬──────┘
                           ↓
                     最优探索目标
                           │
                           ↓
                    move_base
                           │
                           ↓
                        Go2移动
                           │
                           ↓
                       地图更新
                           │
                           └──────────→ 下一轮探索
```

---

## Frontier 检测

探索规划采用 RRT（Rapidly-exploring Random Tree）对当前地图进行搜索。

RRT 在指定探索区域内进行随机采样，根据地图中的：

```text
Free / Occupied / Unknown
```

状态判断采样点。

当 RRT 从已知可通行区域扩展到未知区域附近时，将该位置作为 Frontier 候选点输出。

全局和局部 RRT 检测器共同提供探索候选点。

---

## Frontier 聚类与筛选

原始 RRT 会产生多个 Frontier 候选点。

`filter.py` 对这些点进行聚类，并计算每个区域的代表点，同时清除距离过近或不满足条件的候选点。

最终得到：

```text
/filtered_points
```

作为可供机器人选择的探索目标。

---

## 探索目标选择

`assigner.py` 根据当前机器人位置和 Frontier 候选点计算目标收益。

主要考虑：

* Frontier 的信息增益
* 机器人到 Frontier 的距离
* 机器人当前位置附近的目标优先级
* 已分配目标的影响

综合计算后选择收益最高的 Frontier 作为下一步探索目标。

随后通过 `move_base` Actionlib 将目标发送给机器人。

---

## 使用方法

### 1. 编译

进入 catkin 工作区：

```bash
cd ~/Go2_frontier_based_exploration/workspace_ros1/one_ws
```

编译：

```bash
catkin_make
```

加载环境：

```bash
source devel/setup.bash
```

---

### 2. 启动完整系统

首先启动：

* Gazebo
* 机器人
* 建图
* `move_base`

确保 `/map`、TF 和导航系统正常运行。

然后启动探索规划：

```bash
roslaunch go2_rrt_exploration simple.launch
```

---

### 3. RViz 初始化探索区域

启动 RViz 后选择：

```text
Publish Point
```

在地图上**连续点击 5 次**。

用于初始化 RRT 探索区域。

正常情况下可以看到 RRT 的采样/连接线。

---

### 4. 开始自主探索

完成 5 次 `Publish Point` 后，探索规划模块开始自动工作：

```text
地图
 ↓
Frontier检测
 ↓
Frontier筛选
 ↓
选择探索目标
 ↓
move_base
 ↓
机器人移动
```

机器人到达目标后，随着传感器获取新的环境信息，地图不断更新，系统继续寻找新的 Frontier 并发送下一目标。

整个过程无需手动重复发送导航目标。

---

## 参数

当前单机器人默认配置：

```yaml
n_robots: 1
namespace: ""
global_frame: /map
robot_frame: base_footprint

map_topic: /map
frontiers_topic: /filtered_points

info_radius: 1
info_multiplier: 3.0

hysteresis_radius: 3.0
hysteresis_gain: 2.0

delay_after_assignement: 0.5
rate: 100
```

其中主要参数：

| 参数                  | 说明              |
| ------------------- | --------------- |
| `map_topic`         | 输入栅格地图          |
| `frontiers_topic`   | 输入筛选后的 Frontier |
| `global_frame`      | 全局地图坐标系         |
| `robot_frame`       | 机器人坐标系          |
| `info_radius`       | 信息增益计算范围        |
| `info_multiplier`   | 信息增益权重          |
| `hysteresis_radius` | 目标保持范围          |
| `hysteresis_gain`   | 目标保持增益          |
| `n_robots`          | 机器人数量           |

---

## 模块接口

### 输入

```text
/map
    nav_msgs/OccupancyGrid

/tf
    TF

/clicked_point
    geometry_msgs/PointStamped
```

### 中间结果

```text
/detected_points
    geometry_msgs/PointStamped

/filtered_points
    go2_rrt_exploration/PointArray
```

### 最终输出

```text
/move_base/goal
    move_base_msgs/MoveBaseActionGoal
```

---

## 总体闭环

```text
       建图
        │
        ↓
      /map
        │
        ↓
  Frontier检测
        │
        ↓
 Frontier筛选
        │
        ↓
 探索目标选择
        │
        ↓
    move_base
        │
        ↓
      Go2移动
        │
        ↓
   传感器获取新环境
        │
        ↓
      地图更新
        │
        └────────→ 继续探索
```

> **建图模块负责提供当前环境地图，探索规划模块负责根据地图决定“下一步去哪里”，导航模块负责执行目标。三者共同实现机器人未知环境自主探索。**
