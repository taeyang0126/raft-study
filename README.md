# Raft 学习项目

> 使用 AI 辅导和苏格拉底教学法深入学习 Raft 分布式一致性算法，最终用 Java 实现一个完整的 Raft 集群

## 📚 项目简介

这是一个**Raft 分布式一致性算法学习项目**。通过 AI 辅导的苏格拉底教学法，系统化地学习 Raft 算法的核心原理，并最终用 Java 实现一个完整的 Raft 分布式系统。

**当前进度**：55%（基础阶段 - 2025-01-16）

**学习背景**：7年 Java 开发经验，从实际应用场景出发（MongoDB 读一致性、MySQL 主从复制、分布式任务调度），系统学习分布式一致性算法。

## 🎯 学习目标

1. **深度理解 Raft 算法**：
   - 领导人选举机制
   - 日志复制机制
   - 安全性保证
   - 故障恢复处理

2. **Java 实现设计**：
   - 数据结构设计
   - RPC 通信层
   - 持久化方案
   - 并发控制

3. **工程实践**：
   - 性能优化
   - 测试验证
   - 边界情况处理

## 📖 学习资源

### 核心资料
- [Raft 论文中文版](./paper/raft-zh_cn.md) - 核心学习资料
- [Raft 论文英文版](./paper/raft.pdf) - 原始论文
- [Raft 官网](https://raft.github.io/)
- [Raft 可视化](https://raft.github.io/raftscope/)

### 推荐阅读
- [The Log: What every software engineer should know about real-time data's unifying abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying-abstraction)
- Raft 实现参考：etcd/raft (Go), TiKV (Rust), jgroups (Java)

## 📊 学习进度

| 阶段 | 状态 | 进度 | 完成时间 |
|------|------|------|---------|
| 基础机制 | 🟢 进行中 | 55% | 2025-01-14 ~ |
| 实现设计 | ⚪ 待开始 | 0% | - |
| 高级特性 | ⚪ 待开始 | 0% | - |
| 实践深化 | ⚪ 待开始 | 0% | - |

### 已掌握主题（24个）

**Raft 基础**（6个）
- ✅ 领导人选举机制
- ✅ 日志新旧比较规则
- ✅ 选举限制规则（投票限制）
- ✅ Raft 的三个核心问题
- ✅ 随机超时机制
- ✅ 提交规则（完整理解）

**日志复制机制**（10个）
- ✅ AppendEntries RPC 的参数和返回值
- ✅ AppendEntries RPC 的两个用途
- ✅ Follower 的一致性检查
- ✅ Follower 接收日志后的处理
- ✅ nextIndex 的初始化
- ✅ Leader 的递减重试机制
- ✅ commitIndex 的作用
- ✅ Leader 的日志组成
- ✅ commitIndex 的更新机制
- ✅ 提交 vs 应用
- ✅ commitIndex 和 lastApplied 的关系
- ✅ lastApplied 的持久化方案
- ✅ nextIndex 和 matchIndex 的维护流程
- ✅ nextIndex 和 matchIndex 的优化策略

**安全性机制**（5个）
- ✅ 选举安全性
- ✅ 日志匹配特性（完整理解）
- ✅ Leader 的日志组成
- ✅ 提交规则（深入理解 - 危险场景）
- ✅ 四个机制的配合逻辑

详细进度追踪：[查看进度追踪器](./progress/study-tracker.md)

## 📁 项目结构

```
/
├── paper/                        # Raft 论文
│   ├── raft.pdf                 # 英文原版
│   ├── raft-zh_cn.md            # 中文翻译
│   └── images/                  # 论文图片
 ├── sessions/                    # 学习会话记录
 │   ├── 2025-01-14/             # 2025-01-14 的会话
 │   │   └── session-notes.md    # 详细学习记录
 │   ├── 2025-01-15/             # 2025-01-15 的会话
 │   │   └── session-notes.md    # 详细学习记录
 │   └── 2025-01-16/             # 2025-01-16 的会话
 │       └── session-notes.md    # 详细学习记录
├── progress/                     # 学习进度追踪
│   └── study-tracker.md        # 综合进度追踪器
├── README.md                    # 本文件
└── AGENTS.md                    # AI 助手工作指南
```

## 🎓 学习方法

### 苏格拉底教学法

1. **初步探索**：AI 询问"对这个主题已经了解了什么？"
2. **精炼解释**：基于现有知识提供约 200 字的简洁解释
3. **理解验证**：提出后续问题检查理解程度
4. **适应性跟进**：根据反馈调整教学方法

### 学习记录

每次学习会话都会记录：
- 学生提出的问题和初步理解
- 解释的概念和教学方法
- 学生对理解检查的回应
- 识别的知识缺口
- 掌握的主题及自信程度
- 完成的练习和代码示例

### 进度追踪

`progress/study-tracker.md` 追踪：
- 整体学习进度和统计
- 按领域分类的主题掌握情况
- 知识缺口（高/中/低优先级）
- 学习计划和目标
- 资源清单和复习计划

## 🔧 核心概念理解

### Raft 的设计理念
- **可理解性优先**：相比 Paxos 更容易理解和实现
- **问题分解**：领导人选举、日志复制、安全性
- **强领导人模式**：所有操作通过 Leader，简化管理

### 关键机制
- **随机超时**：150-300ms，避免选票瓜分
- **选举限制**：确保新 Leader 有所有已提交日志
- **提交规则**：只能提交当前任期日志，间接提交旧日志
- **日志匹配特性**：通过递归证明保证一致性

### 技术类比

| 实际系统 | Raft 对应概念 |
|---------|--------------|
| MySQL 主节点 | Leader |
| MySQL 从节点 | Follower |
| binlog | 日志（Log） |
| binlog position | 日志索引（Log Index） |
| GTID | 任期号（Term） |
| 半数同步 | 大多数（N/2 + 1） |

## 🚀 快速开始

### 查看学习进度
```bash
cat progress/study-tracker.md
```

### 查看学习历史
```bash
cat sessions/2025-01-15/session-notes.md
```

### 开始学习
直接向 AI 助手提问你想学习的 Raft 主题，例如：
- "解释 Raft 的领导人选举机制"
- "AppendEntries RPC 是如何工作的？"
- "为什么需要随机超时机制？"

### 学习流程
1. 向 AI 提问主题
2. AI 通过苏格拉底教学法引导学习
3. 系统记录学习会话
4. 自动更新进度追踪器
5. 基于知识缺口调整学习计划

## 💡 学习建议

### 对学习者的建议

1. **保持思考**：
   - 每个机制问"为什么"
   - 思考如果没有这个机制会怎样
   - 尝试用自己的话解释

2. **多画图**：
   - 画出状态转换图
   - 画出日志复制过程
   - 画出时间线

3. **实践验证**：
   - 后期可以写简单的测试用例
   - 模拟各种故障场景
   - 观察系统行为

4. **联系实际**：
   - 思考 Raft 在实际系统中的应用
   - 考虑 Java 实现的细节
   - 考虑性能和可靠性权衡

### 复习策略
- 基于知识缺口优先级安排复习
- 通过实践项目验证理解程度
- 将新旧知识建立联系

## 📝 许可证

本项目仅用于学习目的。

## 🙏 致谢

- Raft 算法由 Diego Ongaro 和 John Ousterhout 设计
- 论文中文翻译由社区贡献
- 感谢所有开源实现和教程的作者

---

**学习日志**：系统追踪每次学习会话，可随时回顾

有问题或建议？欢迎反馈。
