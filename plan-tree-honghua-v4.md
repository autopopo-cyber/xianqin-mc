# 红婳 Plan-Tree v2 — NAV_DOG 测试数据准备

> 辅助组: Plan-Tree v2 | 最后更新: 2026-04-27

---

## 活跃任务

### [·] NAV-R6-FIX | 测试Fixture准备 | 红婳 | 🔴
- 流入: 萱萱下发
- 预期完成: 今日 18:00
- 产出:
  - [ ] `fixtures/obstacle_pointclouds/` — 3组已知点云 (空旷/单障碍/多障碍)
  - [ ] `fixtures/room_layouts/` — L28 窄门场景的场地描述
  - [ ] 每组数据附 ground truth: 预期障碍物列表 + 预期路径
- 数据来源: MuJoCo 仿真中录制，或手工构造
- 判定: 俊秀和雪莹可以直接 `import fixtures` 使用
