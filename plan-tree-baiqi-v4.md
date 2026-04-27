# 白起 Plan-Tree v4 — NAV_DOG SIM+避障

> 实验组: Plan-Tree v4 (含决策点) | 最后更新: 2026-04-27
> 版本: 4.0 | 对比对象: 俊秀/雪莹 (v2)

---

## 活跃任务节点

### NAV-R1 | SIM层接口标准化 | 白起 | 🔴 P0 [·]

#### 轨迹

- 04-27 流入: 萱萱下发 NAV_DOG 重设计，SIM 层需要从 FAR 解耦

- 04-27 决策: 用什么方案抽出 SimInterface？
  分叉:
    A) 完全新建 SimInterface 抽象基类 + 重写 MuJoCoGo2 适配 → 干净，工作量大(~4h)
    B) Wrapper 模式：先包一层薄适配器暴露标准接口 → 快(~1h)，后续重构
  选择: B
  理由: 先验证接口定义足够覆盖感知层/控制层需求，再决定是否重写
  放弃A的理由: 接口定义可能还需迭代，现在就重写太早

- 04-27 产物: 
  - `interfaces/sim.py` — LidarScan, RobotPose, VelocityCmd dataclass + SimInterface Protocol
  - `adapters/mujoco_adapter.py` — MuJoCoAdapter(SimInterface) wrapper
- 04-27 判定: [ ] PASS/FAIL

- 流出: → NAV-R2 (感知层依赖 SIM 接口) | → NAV-R3 (Nav2 需要 pose)
- 关联: 被依赖←#NAV-R2(硬依赖) | 被依赖←#NAV-R3(硬依赖) | 触发→#NAV-R4(因果触发)

--- NOW (2026-04-27) ---

#### 预测

乐观路径: 接口定义 1h + Wrapper 1h → 今日完成 (置信度 0.7)
  条件: MuJoCoGo2 现有方法够用，不需要新增

悲观路径: 发现接口漏字段 → 回跳重定义 → 今日完成 (置信度 0.3)
  风险: MuJoCoGo2 某些内部状态不好暴露

降级方案: 只定义 Protocol，不实现 Wrapper — 下游可用 Mock 开发

---

### NAV-R3 | Nav2 路径规划适配 | 白起 | 🔴 P0 [ ]

#### 轨迹

- 04-27 流入: L28 发现 searchRadius=0.45 导致壁障间隙仅 0.085m

- 04-27 决策: 如何把壁障参数可配置化？
  分叉:
    A) 只改 localPlanner.cpp → 生效快，但每次改都要重编译 C++
    B) bridge 层 YAML 覆盖 → 灵活，但可能被内部默认值覆盖
    C) A+B 组合 → C++ 改合理默认值 + bridge 读 YAML 覆盖
  选择: C
  理由: 两层的理由不同 — C++ 默认值要安全，YAML 提供场景调优
  放弃A的理由: 单独A每次调参都要重编译，迭代太慢
  放弃B的理由: 单独B如果bridge覆盖失效，退回到危险默认值

- 04-27 产物:
  - `config/nav_params.yaml` — searchRadius, vehicleWidth, minClearance
  - `localPlanner.cpp` patch — 从 YAML 读默认值
  - 验证: wall clearance ≥ 0.15m 通过 L28 窄门场景
- 04-27 判定: [ ] PASS/FAIL

- 流出: → NAV-R4 (路径给控制层执行)
- 关联: 被依赖←#NAV-R1(硬依赖: 需要pose接口) | 信息复用→#NAV-R6(测试场景)

--- NOW (2026-04-27) ---

#### 预测

乐观路径: YAML化 + 验证 PASS → 今日完成 (置信度 0.6)
  条件: Nav2 client 现有实现够用

悲观路径: 发现 Nav2 需要的配置项比预期多 → 延期到明早 (置信度 0.3)
  风险: C++ 编译环境问题

降级方案: 只改 YAML 不改 C++，用 bridge 层覆盖

---

## 预测区

### 短期 (24h)
NAV-R1: 乐观 今日 18:00 (0.7) | 悲观 今日 22:00 (0.3) | 降级: 仅Protocol
NAV-R3: 乐观 今日 20:00 (0.6) | 悲观 明早 09:00 (0.3) | 降级: YAML only

### 关键路径
NAV-R1 → (R2 || R3) → R4 → R6

### 资源预测
Agent上下文: 当前 2轮 | 衰减拐点≈10轮 | 建议 8轮后刷新
