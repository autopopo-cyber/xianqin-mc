# 王翦 Plan-Tree v4 — NAV_DOG 感知+控制

> 实验组: Plan-Tree v4 (含决策点) | 最后更新: 2026-04-27
> 版本: 4.0 | 对比对象: 俊秀/雪莹 (v2)

---

## 活跃任务节点

### NAV-R2 | 感知层 ObstacleDetector | 王翦 | 🔴 P0 [·]

#### 轨迹

- 04-27 流入: 萱萱下发 NAV_DOG 重设计。需要独立于 FAR 的障碍物检测器，纯 Python，可脱离 ROS2 测试。

- 04-27 决策: 用哪种聚类算法？
  分叉:
    A) scipy.spatial.KDTree + 距离聚类 → 快（C实现），scipy已在venv
    B) 纯 numpy 手写 DBSCAN → 无额外依赖，但慢，自己写的容易有bug
  选择: A
  理由: scipy 是项目现有依赖（MuJoCo 用），KDTree 成熟可靠。障碍物检测对延迟敏感（<50ms），纯Python聚类可能超时。
  放弃B的理由: 重复造轮子，验证成本高。DBSCAN 参数调优需要数据，而我们现在没有真机点云数据。

- 04-27 决策: 障碍物表示用包围球还是包围盒？
  分叉:
    A) 包围球: center(xyz) + radius → 存储小，碰撞检测快
    B) 包围盒: min(xyz) + max(xyz) → 更精确但存储大
  选择: A
  理由: 导航场景中障碍物形状不重要，重要的是"离多远"。包围球的距离计算是欧氏距离，O(1)。
  放弃B的理由: FAR/V-Graph 内部也用圆柱体近似，包围盒精度用不上。

- 04-27 产物:
  - `perception/obstacle_detector.py` — ObstacleDetector 实现
  - `interfaces/perception.py` — Obstacle, ObstacleDetector Protocol
  - fixture: L28壁障场景点云数据
- 04-27 判定: [ ] PASS/FAIL
  - [ ] detect(已知点云) → 正确障碍物列表
  - [ ] is_path_clear → L28窄门场景返回正确
  - [ ] 纯 Python，`import rclpy` → ImportError

- 流出: → NAV-R4 (控制层依赖)
- 关联: 被依赖←#NAV-R4(硬依赖) | 被依赖←#NAV-R1(硬依赖: 需要LidarScan类型定义) | 验证反馈→#NAV-R6

--- NOW (2026-04-27) ---

#### 预测

乐观路径: KDTree方案直接可用 → 今日完成 (置信度 0.75)
  条件: L28 的点云数据结构清楚，fixture 数据够用

悲观路径: 点云坐标系不一致 → 加坐标系转换 → 今晚完成 (置信度 0.2)
  风险: MuJoCo lidar 坐标系和 ROS2 不同

降级方案: 只做 Euclidean 距离阈值检测（不聚类），功能弱但够用

---

### NAV-R4 | 控制层 VelocityController | 王翦 | 🟡 P1 [ ]

#### 轨迹

- 04-27 流入: 需要路径执行器，含紧急避障。依赖 R2(感知) + R1(SIM接口)

- 04-27 决策: 执行策略 — 开环还是闭环？
  分叉:
    A) 开环: 路径→逐点→速度指令序列，执行中不检测新障碍物 → 简单
    B) 闭环: 每步执行前调用 is_path_clear，新障碍物→replan → 安全但复杂
  选择: B
  理由: 避障场景的核心价值就是动态重规划。开环做的是"盲走"，无法验证 ObstacleDetector 的价值。
  放弃A的理由: 开环在仿真中也能跑，但验收场景 S4(动态障碍物) 必须闭环才能通过。

- 04-27 产物:
  - `control/velocity_controller.py` — VelocityController 实现
  - 闭环执行循环: step→check→execute→next
  - emergency_stop < 200ms
- 04-27 判定: [ ] PASS/FAIL
  - [ ] 执行路径不碰撞已知障碍物
  - [ ] 动态障碍物触发 replan
  - [ ] 单步延迟 <10ms

- 流出: → NAV-R6 (集成测试)
- 关联: 被依赖←#NAV-R2(硬依赖: ObstacleDetector) | 被依赖←#NAV-R1(硬依赖: SimInterface)

--- NOW (2026-04-27) ---

#### 预测

乐观路径: R2 完成后直接联调 → 今晚完成 (置信度 0.5)
  条件: R2 今日完成

悲观路径: R2 延期 → R4 也延期到明早 (置信度 0.4)
  风险: 依赖链传导

降级方案: 用 Mock Detector 先完成控制逻辑，明早换真实 Detector

---

## 预测区

### 短期 (24h)
NAV-R2: 乐观 今日 17:00 (0.75) | 悲观 今日 21:00 (0.2) | 降级: 简化版
NAV-R4: 乐观 今日 22:00 (0.5) | 悲观 明早 09:00 (0.4) | 降级: Mock模式

### 关键路径
R2 → R4 → R6

### 资源预测
Agent上下文: 当前 2轮 | 衰减拐点≈10轮 | 建议 8轮后刷新
