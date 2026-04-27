# 丞相 Plan-Tree v2 — NAV_DOG Gazebo 备选

> 辅助组: Plan-Tree v2 | 最后更新: 2026-04-27

---

## 活跃任务

### [·] NAV-R5 | Gazebo 环境验证 | 丞相 | 🟡
- 流入: 萱萱下发
- 预期完成: 今日 20:00
- 判定: 
  - [ ] Gazebo Harmonic 启动 Go2 模型成功
  - [ ] lidar 插件产生有效点云 (≥100 points)
  - [ ] 输出 `~/xianqin/gazebo-vs-mujoco.md` 对比报告
- 不要求: 完整集成，只做环境可用性验证
- 工作目录: `/home/agentuser/repos/vector-os-nano/gazebo/`
- 注意: 丞相机器有 RTX 2080 Ti，Gazebo 需要 GPU 渲染

---

### [ ] NAV-R5-B | Gazebo 接口对齐 | 丞相 | 🟢
- 流入: R5 完成后，如果 Gazebo 可用
- 目标: 检查 Gazebo 是否能实现 R1 定义的 SimInterface
- 不要求实现，只输出兼容性报告
