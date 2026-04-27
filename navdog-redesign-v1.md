# NAV_DOG SIM+避障重设计 — v1 接口化解耦架构

> 设计: 萱萱 | 日期: 2026-04-27 | 验收: 2026-04-28
> 哲学: 高信噪比 — 每个子任务独立OODA闭环, 接口即契约, 判定数学化
> 实验: 白起/王翦用 Plan-Tree v4 (含决策点), 俊秀/雪莹用 v2 — 对比效果

---

## 一、当前状态诊断 (信噪比过滤)

| 信号 (已知可工作) | 噪声 (待验证/阻塞) |
|---|---|
| MuJoCo Go2 仿真可用 (sinusoidal + MPC) | V-Graph contour_detector → 全黑图 → 0节点 |
| Nav2 action server 可用 | FAR 内部投票机制 (connect_votes_size=5) |
| TARE 探索可用 | 壁障间隙不足 (searchRadius=0.45, 实隙仅0.085m) |
| 3267 tests, harness 1349 passed | Phase B BLOCKED 自 04-16 |

**核心问题**: SIM和避障**耦合太紧**。V-Graph 的障碍物检测直接依赖 FAR C++ 内部数据结构，一个 contour_detector 的 bug 阻塞了整个避障管线。

---

## 二、新架构：四层解耦 + 接口契约

```
┌─────────────────────────────────────────────────┐
│                  SIM 层                          │
│  职责: 提供仿真环境，暴露传感器数据                 │
│  接口: get_lidar_scan()→PointCloud,              │
│        get_pose()→Pose, get_camera()→Image       │
│  实现: MuJoCo Go2 (主), Gazebo (备)              │
│  负责人: 白起                                     │
├─────────────────────────────────────────────────┤
│                 感知层 (ObstacleDetector)         │
│  职责: 点云→障碍物检测，不关心数据来源               │
│  接口: detect(lidar_scan)→List[Obstacle]          │
│        is_path_clear(obstacles, path)→bool        │
│  实现: 纯Python，可脱离FAR独立测试                  │
│  负责人: 王翦                                     │
├─────────────────────────────────────────────────┤
│                 规划层 (PathPlanner)              │
│  职责: 给定障碍物+目标→路径，不关心传感器            │
│  接口: plan(obstacles, start, goal)→Path          │
│        replan(obstacles, current_path)→Path       │
│  实现: Nav2 (主), FAR-VGraph (修复后接入)          │
│  负责人: 白起 (Nav2), 丞相 (V-Graph调研)          │
├─────────────────────────────────────────────────┤
│                 控制层 (VelocityController)       │
│  职责: 路径→速度指令，不关心路径怎么来的             │
│  接口: execute(path)→Iterator[VelocityCmd]        │
│        emergency_stop()→void                      │
│  实现: 纯Python，壁障间隙可配置                     │
│  负责人: 王翦                                     │
└─────────────────────────────────────────────────┘
```

### 关键解耦原则

1. **每层只依赖下层接口，不依赖实现**
2. **接口用 Python Protocol/dataclass 定义，不依赖 ROS2/FAR**
3. **每层可独立测试** — 用 Mock 下层接口
4. **判定数学化** — PASS/FAIL 用数值，不用「看起来OK」

---

## 三、子任务分解 (6个独立OODA闭环)

### #NAV-R1 | SIM层接口标准化 | 白起 | 🔴 P0

**流入**: SIM目前直接耦合到FAR，需要抽出标准接口
**接口契约**:
```python
@dataclass
class LidarScan:
    points: np.ndarray      # (N,3) xyz
    ranges: np.ndarray      # (N,) 距离
    timestamp: float

@dataclass  
class RobotPose:
    x: float; y: float; z: float
    roll: float; pitch: float; yaw: float

class SimInterface(Protocol):
    def get_lidar_scan(self) -> LidarScan: ...
    def get_pose(self) -> RobotPose: ...
    def step(self, cmd: VelocityCmd) -> None: ...
```

**判定**: 
- `SimInterface` 在 MuJoCo Go2 上返回真实数据 ✓
- 延迟 <50ms (1次step+get) ✓
- 可 Mock 用于单元测试 ✓

**决策点** (白起记录):
- 分叉A: 完全新建 SimInterface 抽象层 → 干净但工作量大
- 分叉B: Wrapper 模式包装现有 MuJoCoGo2 → 快但可能漏接口
- **选择: B优先，通过后重构为A** — 先验证接口够用再抽象

---

### #NAV-R2 | 感知层 ObstacleDetector | 王翦 | 🔴 P0

**流入**: 需要一个独立于FAR的障碍物检测器，可脱离ROS2测试
**接口契约**:
```python
@dataclass  
class Obstacle:
    center: tuple[float,float,float]  # xyz
    radius: float                     # 包围球半径
    closest_point: tuple[float,float]  # 最近点xy

class ObstacleDetector(Protocol):
    def detect(self, scan: LidarScan) -> list[Obstacle]: ...
    def is_path_clear(self, obstacles: list[Obstacle], 
                       path: list[tuple[float,float]], 
                       clearance: float=0.15) -> bool: ...
```

**判定**:
- 给定已知点云 → 输出正确障碍物列表 ✓ (用 fixture 数据)
- `is_path_clear` 对 L28 壁障场景返回正确 ✓
- 纯 Python，不 import rclpy ✓

**决策点** (王翦记录):
- 分叉A: 用 scipy.spatial.KDTree 做聚类 → 快但需依赖
- 分叉B: 纯 numpy 手写 DBSCAN → 无依赖但慢
- **选择: A** — scipy 已在 venv，KDTree 可靠且快

---

### #NAV-R3 | Nav2 路径规划适配 | 白起 | 🔴 P0

**流入**: Nav2 已有 client 可用，但壁障间隙配置硬编码
**目标**: 
1. 将 L28 发现的 `searchRadius=0.45` 提为可配置参数
2. 验证 Nav2 在已知障碍物下的路径规划质量
3. 暴露 `plan_path` 接口供上层调用

**判定**:
- `searchRadius` 和 `vehicle_width` 从 YAML 读取 ✓
- 给定障碍物列表 → Nav2 规划的路径避开所有障碍物 ✓
- wall clearance ≥ 0.15m 通过所有门 ✓

**决策点** (白起记录):
- 分叉A: 直接改 localPlanner.cpp → 生效快但改 C++
- 分叉B: 在 bridge 层覆盖参数 → 不改 C++ 但可能被覆盖
- **选择: A+B组合** — C++ 改默认值，bridge 读 YAML 覆盖

---

### #NAV-R4 | 控制层 VelocityController | 王翦 | 🟡 P1

**流入**: 需要一个路径执行器，含紧急避障
**接口契约**:
```python
@dataclass
class VelocityCmd:
    vx: float; vy: float; vyaw: float
    duration: float

class VelocityController(Protocol):
    def execute_path(self, path: Path, 
                      detector: ObstacleDetector,
                      sim: SimInterface) -> Iterator[VelocityCmd]: ...
    def emergency_stop(self) -> None: ...
```

**判定**:
- 沿路径执行，不碰撞已知障碍物 ✓
- 执行中检测到新障碍物 → 触发 `replan` 回调 ✓
- 单步延迟 <10ms ✓

---

### #NAV-R5 | SIM层 Gazebo 备选验证 | 丞相 | 🟡 P1

**流入**: MuJoCo 是目前唯一可用 SIM，需要备选方案
**目标**: 验证 Gazebo Harmonic 环境下 Go2 + lidar 是否可用
**不要求**: 完整集成，只做环境验证

**判定**:
- Gazebo 能启动 Go2 模型 ✓
- lidar 插件产生有效点云 ✓
- 输出报告: Gazebo vs MuJoCo 差异对比 ✓

---

### #NAV-R6 | 集成测试 + 壁障场景 | 俊秀+雪莹 | 🟡 P1

**流入**: 以上模块独立就位后需要集成验证
**场景矩阵**:

| 场景ID | 描述 | 判定标准 |
|--------|------|---------|
| S1 | 直行无障碍 | 到达目标，无停顿 |
| S2 | 单障碍物绕行 | 路径偏离 < 障碍物半径+clearance |
| S3 | 窄门通过 (L28) | wall clearance ≥ 0.15m |
| S4 | 动态障碍物 | emergency_stop 触发 < 200ms |
| S5 | 三障碍物迷宫 | 成功到达，不碰撞 |

**分配**:
- 俊秀: 编写 S1-S3 测试 (v2 Plan-Tree 或不记录决策点)
- 雪莹: 编写 S4-S5 测试 (v2 Plan-Tree 或不记录决策点)
- 红婳: 准备测试 fixture 数据 (障碍物点云、场地布局)

**判定**: 5/5 场景通过 ✓

---

## 四、依赖图 (关键路径)

```
NAV-R1 (SIM接口) ────┐
                     ├──→ NAV-R2 (感知层) ──→ NAV-R4 (控制层)
NAV-R3 (Nav2规划) ──┘                            │
                                                  ↓
NAV-R5 (Gazebo备选) ───────────── (独立)      NAV-R6 (集成测试)
```

**关键路径**: R1 → (R2 || R3) → R4 → R6  
**期望**: R1+R2+R3 今日完成，R4 今晚，R6 明早集成

---

## 五、对比实验设计

| 组 | Agent | Plan-Tree | 任务 | 观察指标 |
|----|-------|-----------|------|---------|
| 实验组 | 白起 | v4 (含决策点) | R1, R3 | 决策点数量, 决策正确率 |
| 实验组 | 王翦 | v4 (含决策点) | R2, R4 | 决策点数量, 接口设计选择 |
| 对照组 | 俊秀 | v2 (无决策点) | R6-S1~S3 | 是否需要返工, 卡住次数 |
| 对照组 | 雪莹 | v2 (无决策点) | R6-S4~S5 | 是否需要返工, 卡住次数 |
| 辅助 | 丞相 | v2 | R5 | Gazebo 环境报告 |
| 辅助 | 红婳 | v2 | R6-fixture | 测试数据准备 |

**核心假设**: v4 组的决策点能让 Agent 事后回溯"为什么当时选A不选B"，减少重复踩坑。

---

## 六、验收标准 (明天 09:00)

- [ ] R1+R2+R3 至少 2/3 完成 (判定 PASS)
- [ ] R6 至少 S1+S2+S3 通过
- [ ] 白起+王翦的 v4 Plan-Tree 含 ≥3 个决策点/人
- [ ] 各层接口定义文件提交到 `vector-os-nano/interfaces/`
- [ ] 无一例「改了A层导致B层崩溃」的耦合事故

---

*此文档为任务设计规范。具体实施由各 Agent 在其 Plan-Tree 中展开。*
*明天 09:00 萱萱验收。*
