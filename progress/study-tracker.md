# Raft 学习进度追踪器

 **最后更新日期**：2025-01-16
 **学习者**：7年 Java 开发经验
 **学习目标**：深入理解 Raft 算法，使用 Java 实现 Raft 分布式系统
 ---

 ## 快速统计

 - **已掌握主题**：24 个
 - **高严重程度知识缺口**：1 个
 - **中严重程度知识缺口**：0 个
 - **低严重程度知识缺口**：3 个
 - **整体进度**：55%（基础阶段）
 ---

 ## 领域/主题进度汇总表

  | 领域 | 总主题数 | 已掌握 | 进行中 | 待学习 | 完成度 |
  |------|---------|--------|--------|--------|--------|
  | Raft 基础 | 10 | 9 | 0 | 1 | 90% |
  | 日志复制机制 | 10 | 10 | 0 | 0 | 100% |
  | 安全性机制 | 6 | 5 | 0 | 1 | 83% |
  | Raft 实现细节 | 10 | 0 | 0 | 10 | 0% |
  | Java 实现设计 | 8 | 0 | 0 | 8 | 0% |
  | 高级特性 | 5 | 0 | 0 | 5 | 0% |
  ---

 ## 已掌握主题

 ### Raft 基础

 #### 1. 领导人选举机制
 - **掌握日期**：2025-01-14
 - **自信程度**：高
 - **理解的关键点**：
   - 三个服务器状态（Follower、Candidate、Leader）
   - 选举流程：Follower 超时 → Candidate → 请求投票 → 成为 Leader
   - 随机超时机制：150-300ms 随机，避免选票瓜分
   - 心跳机制：Leader 定期发送心跳维持权威
   - 候选人在选举超时后会重新发起选举（term++）

 #### 2. 日志新旧比较规则
 - **掌握日期**：2025-01-14
 - **自信程度**：高
 - **理解的关键点**：
   - 比较两个日志的最后一条记录
   - 如果 Term 不同，Term 大的更新
   - 如果 Term 相同，日志长的更新
   - 示例：[term1, term1, term2] 比 [term1, term1, term1, term1] 更新

 #### 3. 选举限制规则（投票限制）
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - RequestVote RPC 携带候选人的日志信息（lastLogIndex, lastLogTerm）
   - 投票规则：
     1. term 必须 >= currentTerm
     2. 没投过票，或者投给了这个候选人
     3. 候选人的日志必须至少和我一样新
   - 交集原理：大多数已提交日志节点和大多数投票节点的交集确保当选者有所有已提交日志
   - 作用：确保新 Leader 拥有所有已提交的日志

 #### 4. Raft 的三个核心问题
 - **掌握日期**：2025-01-14
 - **自信程度**：中高
 - **理解的关键点**：
   - 领导人选举：Leader 崩溃后如何选出新 Leader
   - 日志复制：Leader 如何将数据同步给 Follower
   - 安全性：如何保证选出的 Leader 一定有所有已提交的数据

 #### 5. 随机超时机制
 - **掌握日期**：2025-01-14
 - **自信程度**：中高
 - **理解的关键点**：
   - 每个节点的选举超时：150-300ms 随机
   - 避免多个节点同时超时和选票瓜分
   - 选票瓜分时，所有 Candidate 重新发起选举（term++）
   - 随机超时确保最终会有一个节点先超时
   - Java 实现：ScheduledExecutorService + Random

 #### 6. 提交规则（完整理解）
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - Leader 只能提交当前任期内的日志条目
   - 不能单独提交旧任期的日志
   - 必须通过提交当前任期的日志来间接提交旧任期日志
   - 日志的 term 永远不变，提交只是改变 committed 标志
   - 避免旧任期日志被提交后被新 Leader 覆盖
   - 提交规则间接阻止包含旧日志的节点当选（通过日志匹配特性）

 ### 日志复制机制

 #### 1. AppendEntries RPC 的参数和返回值
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - 参数：term（Leader 的任期）、leaderId（Leader 的 ID）、prevLogIndex（前一条日志的索引）、prevLogTerm（前一条日志的任期）、entries（要追加的日志条目）、leaderCommit（Leader 的 commitIndex）
   - 返回值：term（Follower 的 term）、success（是否成功）
   - prevLogIndex 和 prevLogTerm 是"前一条日志"，用于一致性检查，不是"最后提交的日志"

 #### 2. AppendEntries RPC 的两个用途
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - 心跳（Heartbeat）：entries 为空列表，定期发送（通常 50-100ms），维护 Leader 权威
   - 日志复制（Log Replication）：entries 包含要追加的日志条目，发送到所有 Follower
   - prevLogIndex 和 prevLogTerm 用于一致性检查

 #### 3. Follower 的一致性检查
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - 规则 1：Leader 的 term 必须 >= Follower 的 term
   - 如果 Leader term 更大，Follower 更新自己的 term 并转换为 Follower
   - 规则 2：检查前一条日志是否匹配（prevLogIndex 处的日志 term 必须等于 prevLogTerm）
   - 两个条件都满足才接受日志

 #### 4. Follower 接收日志后的处理
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - 删除 prevLogIndex 之后的所有日志
   - 追加新日志
   - 更新 commitIndex（如果 Leader 的 commitIndex 更大，但不能超过自己的日志长度）

 #### 5. nextIndex 的初始化
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - Leader 成为 Leader 时，为每个 Follower 初始化 nextIndex = log.size() + 1
   - nextIndex 是"下次要发送的日志索引"
   - nextIndex = Leader 最后一条日志 index + 1

 #### 6. Leader 的递减重试机制
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - Leader 发送 AppendEntries 给 Follower（prevLogIndex = nextIndex - 1）
   - 如果 Follower 拒绝（success=false），Leader 递减 nextIndex--
   - Leader 重新发送，直到 Follower 接受
   - 最终找到匹配点，成功同步
   - Leader 不会删除自己的日志（只修改 nextIndex）

 #### 7. commitIndex 的作用
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - commitIndex 是已提交的最高日志索引
   - Leader 复制到大多数节点后更新 commitIndex
   - Follower 通过 AppendEntries 接收 leaderCommit，更新自己的 commitIndex
   - 标记哪些日志可以应用到状态机（<= commitIndex 的日志）
   - 是日志安全的分界线（<= commitIndex 不会被覆盖，> commitIndex 可能被覆盖）

 #### 8. Leader 的日志组成
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - 包含所有已提交的日志（通过选举限制保证）
   - 可能包含未提交的日志（自己产生的，还没复制到大多数）
   - 是日志的源头，不应该被 Follower 影响
   - Leader 不会删除自己的日志

 #### 9. commitIndex 的更新机制
 - **掌握日期**：2025-01-16
 - **自信程度**：高
 - **理解的关键点**：
   - Leader 发送 AppendEntries RPC 收到大多数成功响应后，更新 commitIndex
   - Leader 通过 `matchIndex` 数组知道 Follower 的 lastIndex
   - `matchIndex[serverId]` 记录该服务器已确认的最新日志索引
   - Follower 返回 AppendEntriesResponse 时，Leader 更新 `matchIndex`
   - 检查是否有新的日志可以提交（遍历 commitIndex 到 getLastLogIndex()）

  #### 10. 提交 vs 应用
  - **掌握日期**：2025-01-16
  - **自信程度**：高
  - **理解的关键点**：
   - 提交（Raft 层）：日志已经复制到大多数节点，标记为 committed
   - 应用（状态机层）：执行日志中的命令，修改状态机
   - 不可能存在"提交的日志在大多数节点上不一样"（提交的定义就是"大多数节点一致"）

  #### 11. commitIndex 和 lastApplied 的关系
  - **掌握日期**：2025-01-16
  - **自信程度**：高
  - **理解的关键点**：
   - commitIndex：已提交的最高日志索引（在大多数节点上存在，不可变）
   - lastApplied：已应用到状态机的最高日志索引
   - commitIndex >= lastApplied
   - 为什么需要两个变量：提交和应用是两个独立的过程，可能存在延迟
   - Leader 和 Follower 都会：当 commitIndex > lastApplied 时，应用日志到状态机
   - Leader 更新 commitIndex：收到大多数节点的 AppendEntries 成功响应后
   - Follower 更新 commitIndex：通过 AppendEntries RPC 接收 leaderCommit

  #### 12. lastApplied 的持久化方案
  - **掌握日期**：2025-01-16
  - **自信程度**：高
  - **理解的关键点**：
   - Raft 论文中定义 lastApplied 为非持久化状态
   - 实际工程上不会持久化 lastApplied（避免状态机和 lastApplied 的一致性问题）
   - 通过快照解决：每隔一段时间打个快照，记录快照那一刻的 applyId
   - 快照包含：状态机状态 + lastApplied（即 lastIncludedIndex）
   - 重启后读取快照，恢复状态机状态和 lastApplied

  #### 13. nextIndex 和 matchIndex 的维护流程
  - **掌握日期**：2025-01-16
  - **自信程度**：高
  - **理解的关键点**：
   - 初始化：nextIndex = log.size + 1, matchIndex = 0
   - 发送：preLogIndex = nextIndex - 1, entries = nextIndex 到 log.size
   - 成功：matchIndex = preLogIndex + entries.size, nextIndex = matchIndex + 1
   - 失败：nextIndex--, 重新发送

  #### 14. nextIndex 和 matchIndex 的优化策略
  - **掌握日期**：2025-01-16
  - **自信程度**：高
  - **理解的关键点**：
   - nextIndex 是冗余的，可以从 matchIndex 计算出来
   - 但为了逻辑清晰，论文选择单独维护 nextIndex
   - 指数退避方案：记录失败次数 + 指数退避
   - 快速回退优化：如果返回 conflictIndex，可以直接设置 nextIndex = conflictIndex + 1

 ### 安全性机制

 #### 1. 选举安全性
 - **掌握日期**：2025-01-14
 - **自信程度**：高
 - **理解的关键点**：
   - 一个任期内最多只会有一个 Leader 被选举出来
   - 通过大多数选票规则保证
   - 通过随机超时避免选票瓜分

 #### 2. 日志匹配特性（完整理解）
 - **掌握日期**：2025-01-15
 - **自信程度**：高
 - **理解的关键点**：
   - 如果两个日志在相同索引位置的日志条目的任期号相同，那么它们存储了相同的指令
   - 如果两个日志在相同索引位置的日志条目的任期号相同，那么它们之前的所有日志条目也全部相同
   - 数学递归证明：前一条完全一致才能接收这一条
   - 通过 AppendEntries RPC 的一致性检查保证
   - Follower 只在日志匹配的基础上追加新日志
   - 如果最新日志在多个节点上一致，则之前的日志也一定一致

 #### 3. 提交规则（深入理解 - 危险场景）
 - **掌握日期**：2025-01-16
 - **自信程度**：高
 - **理解的关键点**：
   - 目标：确保核心条件 → 已提交的日志不能被覆盖
   - 强制 Leader 只提交当前任期的日志（旧任期日志通过当前任期日志间接提交）
   - 通过构造危险场景理解：如果违反提交规则，会导致已提交的日志被覆盖
   - 提交规则的作用：阻止旧任期日志被直接提交，确保提交时机正确

 #### 4. 四个机制的配合逻辑
 - **掌握日期**：2025-01-16
 - **自信程度**：高
 - **理解的关键点**：
   - 目标：确保核心条件 → 已提交的日志不能被覆盖
   - 选举限制：保证新当选的 Leader 一定包含所有已提交的日志
   - 提交规则：强制 Leader 只提交当前任期的日志，阻止旧任期日志被直接提交
   - 日志匹配特性：提交当前任期日志后，旧日志自动提交（利用递归特性）
   - AppendEntries：实现日志同步和一致性检查
   - 四个机制配合，确保已提交的日志不会被覆盖

 ---

  ## 知识缺口

  ### 高严重程度

  #### 1. 客户端与 Raft 集群的交互
  - **状态**：待学习
  - **影响范围**：系统设计、客户端实现
  - **关联主题**：客户端交互
  - **学习计划**：基础机制学习后（2025-01-17）

  ### 中严重程度
  - （无）

  ### 低严重程度

  #### 1. 选举超时时间的具体配置建议
  - **状态**：待学习
  - **影响范围**：性能调优、选举稳定性
  - **关联主题**：领导人选举、性能优化
  - **学习计划**：实现阶段（2025-01-19）

  #### 2. 心跳频率的设置
  - **状态**：待学习
  - **影响范围**：网络开销、选举稳定性
  - **关联主题**：领导人选举、性能优化
  - **学习计划**：实现阶段（2025-01-19）

  #### 3. 性能优化技巧
  - **状态**：待学习
  - **影响范围**：系统性能、吞吐量
  - **关联主题**：系统优化
  - **学习计划**：实现阶段（2025-01-19）

 ---

 ## 学习计划

 ### 第 1 周：基础机制（2025-01-14 ~ 2025-01-21）

 #### 2025-01-14（已完成）
 - [x] Raft 基本概念映射
 - [x] 领导人选举机制
 - [x] 随机超时机制
 - [x] 日志新旧比较规则
 - [x] 选举限制规则
 - [x] 提交规则

 #### 2025-01-16（已完成）
 - [x] 复习之前学习的内容
 - [x] 提交规则的深入理解
   - [x] 构造危险场景说明提交规则的作用
   - [x] 理解为什么不能提交旧任期日志
 - [x] commitIndex 的更新机制
   - [x] Leader 如何知道 Follower 的 lastIndex（通过 matchIndex）
   - [x] matchIndex 数组的维护
 - [x] 提交 vs 应用
   - [x] 提交（Raft 层）vs 应用（状态机层）
   - [x] 为什么不可能存在"提交的日志在大多数节点上不一样"
 - [x] 四个机制的配合逻辑
   - [x] 目标：确保"已提交的日志不能被覆盖"
   - [x] 选举限制 + 提交规则 + 日志匹配特性 + AppendEntries 如何配合
 - [x] commitIndex 和 lastApplied 的关系
   - [x] commitIndex 如何更新（Leader 和 Follower）
   - [x] lastApplied 如何更新
   - [x] 日志应用到状态机的时机
   - [x] 为什么需要两个变量
 - [x] nextIndex 和 matchIndex 的维护优化
   - [x] 批量同步优化
   - [x] 快速回退优化

 #### 2025-01-17（计划）
 - [ ] 客户端与 Raft 集群的交互
   - [ ] 客户端如何选择 Leader
   - [ ] 客户端如何处理重试
   - [ ] 客户端如何实现幂等
 - [ ] 提交的完整机制
   - [ ] 什么时候可以提交
   - [ ] 如何通知 Follower 提交
   - [ ] 状态机执行时机

 #### 2025-01-18（计划）
 - [ ] 状态机安全性的完整证明
   - [ ] 领导人完整特性的证明
   - [ ] 如何推导到状态机安全性
 - [ ] 所有服务器需遵守的规则

 #### 2025-01-19（计划）
 - [ ] 实践场景分析
   - [ ] 网络分区场景
   - [ ] 节点崩溃场景
   - [ ] 分裂投票场景
 - [ ] 性能考虑
   - [ ] 选举超时配置
   - [ ] 心跳频率设置

 #### 2025-01-20（计划）
 - [ ] 基础机制综合复习
 - [ ] 答疑和深入讨论
   - [ ] 准备进入实现设计阶段

 ---

 ## 学习会话摘要

 ### 最近会话（详细内容见 sessions 目录）

 #### 2025-01-16（下午）
 **主要主题**：commitIndex 和 lastApplied 的关系 + nextIndex 和 matchIndex 的优化策略
 **核心收获**：
 - 理解为什么需要两个变量（提交和应用独立）
 - 理解 lastApplied 持久化方案（通过快照）
 - 掌握 nextIndex 和 matchIndex 的维护流程
 - 理解指数退避优化策略
 **详细记录**：[sessions/2025-01-16/session-notes.md](./sessions/2025-01-16/session-notes.md)

 #### 2025-01-16（上午）
 **主要主题**：提交规则的深入理解 + 四个机制的配合逻辑
 **核心收获**：
 - 构造了正确的危险场景
 - 理解 Leader 如何通过 matchIndex 知道 Follower 的 lastIndex
 - 构建了完整的知识体系
 **详细记录**：[sessions/2025-01-16/session-notes.md](./sessions/2025-01-16/session-notes.md)

 #### 2025-01-15
 **主要主题**：AppendEntries RPC 的完整机制
 **核心收获**：
 - 理解 prevLogIndex 和 prevLogTerm 的作用
 - 理解 Leader 如何修复 Follower 的日志不一致
 - 理解 Raft 层和状态机层的职责分离
 **详细记录**：[sessions/2025-01-15/session-notes.md](./sessions/2025-01-15/session-notes.md)

 #### 2025-01-14
 **主要主题**：Raft 基础概念和选举机制
 **核心收获**：
 - 理解领导人选举机制
 - 理解选举限制和日志匹配特性
 - 认识到客户端幂等的重要性
 **详细记录**：[sessions/2025-01-14/session-notes.md](./sessions/2025-01-14/session-notes.md)

 ---

 ## 学习笔记摘要

 ### 核心概念理解

 1. **Raft 的设计理念**：
   - 可理解性优先
   - 分解问题：领导人选举、日志复制、安全性
   - 强领导人模式：所有操作通过 Leader

 2. **关键机制**：
   - 随机超时：避免选票瓜分
   - 选举限制：确保新 Leader 有所有已提交日志
   - 提交规则：只能提交当前任期日志，间接提交旧日志

 3. **AppendEntries RPC 的完整机制**：
   - 参数：term, leaderId, prevLogIndex, prevLogTerm, entries, leaderCommit
   - 返回值：term, success
   - 两个用途：心跳（entries 为空）和日志复制（entries 非空）

 4. **commitIndex 的作用**：
   - 已提交的最高日志索引
   - Leader 复制到大多数后更新
   - Follower 通过 AppendEntries 接收 leaderCommit
   - 标记哪些日志可以应用到状态机

 5. **commitIndex 和 lastApplied 的关系**：
   - commitIndex：已提交的最高日志索引（在大多数节点上存在，不可变）
   - lastApplied：已应用到状态机的最高日志索引
   - commitIndex >= lastApplied

 6. **nextIndex 和 matchIndex 的维护流程**：
   - 初始化：nextIndex = log.size + 1, matchIndex = 0
   - 发送：preLogIndex = nextIndex - 1, entries = nextIndex 到 log.size
   - 成功：matchIndex = preLogIndex + entries.size, nextIndex = matchIndex + 1
   - 失败：nextIndex--, 重新发送

 7. **Raft 层和状态机层的职责分离**：
   - Raft 层：保证日志顺序一致，不知道命令的内容
   - 状态机层：执行命令、去重
   - 幂等是状态机层的问题

 ### 技术类比

 | 实际系统 | Raft 对应概念 |
 |---------|--------------|
 | MySQL 主节点 | Leader |
 | MySQL 从节点 | Follower |
 | binlog | 日志（Log） |
 | binlog position | 日志索引（Log Index） |
 | GTID | 任期号（Term） |
 | 半数同步 | 大多数（N/2 + 1） |

 ---

 ## 学习里程碑

 ### 已完成的里程碑

 - [x] 2025-01-14：掌握 Raft 基础概念和选举机制
 - [x] 2025-01-15：完全理解 Raft 的两个关键安全机制（选举限制和提交规则）以及日志匹配特性
 - [x] 2025-01-15：完全理解 AppendEntries RPC 的完整机制和 Leader 如何修复 Follower 的日志不一致
 - [x] 2025-01-16（上午）：深入理解提交规则的作用，构造正确的危险场景；理解 commitIndex 的更新机制；构建完整的知识体系（四个机制的配合逻辑）
 - [x] 2025-01-16（下午）：完全理解 commitIndex 和 lastApplied 的关系；理解 lastApplied 的持久化方案（通过快照）；掌握 nextIndex 和 matchIndex 的维护流程和优化策略

 ### 未来的里程碑

 - [ ] 2025-01-17：学习客户端与 Raft 集群的交互
 - [ ] 2025-01-18：完全理解状态机安全性
 - [ ] 2025-01-20：完成基础机制学习
 - [ ] 2025-01-27：完成实现设计
 - [ ] 2025-02-03：完成高级特性学习
 - [ ] 2025-02-10：完成实践和深化

 ---

 ## 核心答疑记录

 ### Raft 基础相关

 1. **Q: Candidate 如何证明自己的日志"最新"？**
    - A: 先比较 epoch（term），再比较 commit log

 2. **Q: 如果两个节点的日志长度一样，但最后一条记录的 epoch 不同，谁更新？**
    - A: epoch 大的更新

 3. **Q: 随机超时如何处理极端情况下的选票瓜分？**
    - A: 通过随机超时机制，每个候选者会重新发起选举（term++），总会有一个先超时

 ### 日志复制相关

 4. **Q: commitIndex 有啥作用？**
    - A: commitIndex 是已提交的最高日志索引，用于标记哪些日志可以应用到状态机。是日志安全的分界线（<= commitIndex 不会被覆盖，> commitIndex 可能被覆盖）。

 5. **Q: prevLogIndex 和 prevLogTerm 是最后提交的日志吗？**
    - A: 不是！是"前一条日志"（previous log），用于一致性检查。Leader 发送日志时，告诉 Follower："你的日志在 (prevLogIndex, prevLogTerm) 位置应该和我的日志匹配"。

 6. **Q: Leader 如何知道 Follower 的 lastIndex？**
    - A: Leader 通过 `matchIndex` 数组。`matchIndex[serverId]` 记录该服务器已确认的最新日志索引。

 7. **Q: 提交 vs 应用有什么区别？**
    - A: 提交（Raft 层）= 日志已经复制到大多数节点，标记为 committed；应用（状态机层）= 执行日志中的命令，修改状态机。

 8. **Q: 提交规则到底起什么作用？**
    - A: 阻止旧任期日志被直接提交，确保提交时机正确。通过提交当前任期日志，间接提交旧日志，这样可以更好地与日志匹配特性配合。

 9. **Q: Leader 先更新 commitIndex 还是先更新 lastApplied？**
    - A: 先更新 commitIndex，优先保证 Raft 自身的正确性。

 10. **Q: Follower 如何知道需要更新 commitIndex？**
    - A: RPC 请求中有 leader 节点的 commitIndex，和自己的 lastLogIndex 比较就知道是否需要更新了。

 11. **Q: nextIndex 还有优化策略嘛？不就是 log.size + 1 吗？**
    - A: nextIndex 的初始化是 log.size + 1，但真正的问题是 Follower 拒绝时如何快速找到匹配点。优化策略包括：指数退避（记录失败次数 + 指数退避）、快速回退（如果返回 conflictIndex）。

 ### 安全性相关

 12. **Q: 提交成功但响应失败是可能的吗？**
    - A: 是的。客户端需要确保幂等（命令携带唯一 ID，状态机检查去重）。

 13. **Q: 日志的 term 提交时会变成当前任期吗？**
    - A: 不会。term 是日志创建时的任期，永远不会改变。提交只是改变 committed 标志。

 ---

 ## 学习建议

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

 ### 需要持续关注的问题

 1. **日志一致性的完整机制**
 2. **状态机安全性的完整证明**
 3. **实际实现中的边界情况处理**
 4. **性能优化和可扩展性**

 ---

 ## 资源和参考

 ### 核心资源
 - [Raft 论文中文版](../paper/raft-zh_cn.md)
 - [Raft 论文英文版](../paper/raft.pdf)

 ### 推荐阅读
 - Raft 官网：https://raft.github.io/
 - Raft 可视化：https://raft.github.io/raftscope/

 ### 实现参考
 - etcd/raft (Go)
 - TiKV (Rust)
 - raft-rs (Rust)
 - jgroups (Java)

 ---
