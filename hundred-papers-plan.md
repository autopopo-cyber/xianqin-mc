# 仙秦舰队 · 百篇文章外看计划

> 启动: 2026-04-27 | 执行: 萱萱
> 背景: DeepSeek 百万 token $0.3（降价10倍），大规模阅读成本降至几毛钱
> 目标: 100篇 agent 社区活跃文章 → 总结提炼 → 反哺舰队设计

## 三个核心方向

| 方向 | 关键问题 | 关联舰队组件 |
|------|---------|------------|
| **多 Agent 协作** | 编排模式、通信协议、冲突解决、角色分工 | MC 总线、忙锁、频率分离 |
| **记忆系统** | 长期记忆架构、RAG、上下文管理、衰减对抗 | MemOS、wiki 分级、Plan-Tree 轨迹 |
| **任务计划** | 分解策略、优先级、动态重规划、预测 | Plan-Tree v3、预测区、关联图 |

## 信息源

| 源 | 频率 | 搜索关键词 |
|----|------|----------|
| ArXiv | 每批5篇 | "multi-agent collaboration" "LLM agent memory" "task planning agent" |
| Reddit r/artificial r/programming | 每批3篇 | multi-agent, agent orchestration, agent memory |
| Hacker News | 每批2篇 | "Show HN" agent, multi-agent system |
| V2EX / 知乎 / GitHub | 补充 | Agent 协作, LLM Agent, 任务编排 |

## 输出规范

每批10篇 → 一个总结文件 `~/llm-wiki/agent-analysis/batch-XXX.md`

每篇提炼：
- 一句话核心观点
- 与我们舰队设计的关联
- 可借鉴的具体机制
- 可反驳的观点（如果有）

## 节奏

- 每天一批（10篇），10天完成
- 每批耗时: 搜索5min + 阅读提炼15min + 写作5min ≈ 25min
- Token 成本: ~50K input/batch × $0.3/M = $0.015/batch，总计 $0.15
