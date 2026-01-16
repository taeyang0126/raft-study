# Raft 学习会话 - 2026-01-14

## 会话概述

**日期**：2026-01-14
**持续时间**：正在进行
**主要主题**：Raft 一致性算法基础
**学习目标**：深入理解 Raft 算法，为使用 Java 实现 Raft 分布式系统做准备

---

## 学生背景

- **工作经验**：7年 Java 开发经验
- **Raft 了解程度**：听说过但不了解细节
- **分布式系统经验**：
  - 了解 CAP 定理
  - 熟悉消息队列
  - 了解分布式事务（事务最终一致性）
  - 有 MongoDB 读一致性问题的实际经验
  - 了解 MySQL 主从复制
  - 有分布式任务调度经验（xxl-job）
- **Java 并发经验**：
  - 处理过 HashMap 并发问题
  - 使用过锁机制（synchronized、Lock）
  - 熟悉并发集合、原子类

---

## 学生初步理解评估

### 分布式系统相关问题
1. **MongoDB 读一致性问题**：
   - 问题：insert 后立即 query 查不到数据
   - 解决方案：休眠一段时间（业务不要求非常及时），重要业务考虑只读主节点

2. **MySQL 主从复制**：
   - 理解主从复制的基本流程
   - 知道主节点崩溃时从节点会从自身进度重新拉取 binlog
   - 理解 3 节点集群：挂 1 个可用，挂 2 个不可用

3. **主从失联的脑裂问题**：
   - 知道需要过半票数才能成为主节点
   - 理解这可以避免脑裂

4. **日志复制**：
   - 需要主节点接收请求并转发
   - 主节点负责同步数据到其他节点
   - 主节点挂了，其他节点申请成为主节点
   - 用 epoch 和最新 commit log 作为参选信息
   - 收到半数以上票数成为主节点
   - 网络问题因为无法收到半数票所以没问题

---

## 已学习内容

### 1. Raft 基本概念映射

| 学生原有概念 | Raft 术语 |
|------------|----------|
| 主节点 | Leader（领导人） |
| 子节点 | Follower（跟随者） |
| 选举时的 epoch | Term（任期） |
| 最新 commit log | 日志的新旧程度 |
| 半数以上票数 | 大多数（N/2 + 1） |
| 脑裂问题 | 选举安全性保证 |

### 2. Raft 的三个核心问题
1. **领导人选举**：Leader 崩溃后，如何选出新 Leader
2. **日志复制**：Leader 如何将数据同步给 Follower
3. **安全性**：如何保证选出的 Leader 一定有所有已提交的数据

### 3. 服务器状态
- **Follower（跟随者）**：被动，只响应请求
- **Candidate（候选人）**：主动竞选领导人
- **Leader（领导人）**：处理所有请求

### 4. 领导人选举机制

#### 选举流程
1. 所有节点启动时都是 Follower
2. Follower 在一段时间内没收到 Leader 的心跳 → 转成 Candidate
3. Candidate：
   - term++（epoch 自增）
   - 给自己投票
   - 发送 RequestVote RPC 给其他节点
4. 获得大多数选票 → 成为 Leader

#### 随机超时机制
- 每个节点的选举超时：150ms ~ 300ms 随机
- 避免多个节点同时超时和选票瓜分
- 如果选票瓜分，所有 Candidate 会重新发起选举（term++）
- 随机超时确保最终会有一个节点先超时

#### Java 伪代码实现
```java
class RaftNode {
    Random random = new Random();
    ScheduledFuture electionTimeout;

    void resetElectionTimeout() {
        int timeout = 150 + random.nextInt(151);
        if (electionTimeout != null) {
            electionTimeout.cancel();
        }
        electionTimeout = scheduler.schedule(
            () -> startElection(),
            timeout,
            TimeUnit.MILLISECONDS
        );
    }

    void startElection() {
        currentTerm++;
        voteForSelf();
        sendRequestVoteRPCs();
        resetElectionTimeout();
    }

    void onHeartbeat(Heartbeat hb) {
        resetElectionTimeout();
    }
}
```

### 5. 日志新旧比较规则

**比较两个日志的最后一条记录：**
1. 如果最后一条记录的 Term（epoch）不同 → Term 大的更新
2. 如果 Term 相同 → 日志更长的更新

**示例**：
```
节点 A: [term1, term1, term2]       (最后一条是 term2)
节点 B: [term1, term1, term1, term1] (最后一条是 term1)

结论：节点 A 更新（因为 term2 > term1）
```

### 6. 日志一致性问题

#### 典型的不一致场景
5 节点集群（S1, S2, S3, S4, S5）：

**Term 2（Leader S1）**：
- S1 写了日志 [1,2,3,4,5]
- S1 把 4,5 复制到 S2（成功）
- S1 崩溃，没来得及复制到 S3, S4, S5
- 结果：S1, S2 有 [1,2,3,4,5]，S3, S4, S5 只有 [1,2,3]

**Term 3（Leader S5）**：
- S5 当选 Leader（S5 只有 [1,2,3]）
- S5 写新日志到位置 4，得到 [1,2,3,6,7]
- 复制到 S3, S4, S5（成功）
- 结果：S3, S4, S5 有 [1,2,3,6,7]，S1, S2 有 [1,2,3,4,5]

**关键点**：这不是"失败"，而是因为 Leader 崩溃导致部分节点落后了。

### 7. 安全性问题的危险场景（论文图 8）

假设有 5 个节点：S1, S2, S3, S4, S5

**(a) Term 2，Leader S1**：
- S1 写入日志到位置 2（term=2）
- 复制到 S2（成功）
- S1 崩溃前没来得及提交

**(b) Term 3，Leader S5 当选**：
- S5 写入新日志到位置 2（term=3）
- 复制到 S3, S4, S5（成功）
- 现在 S3,S4,S5 有 term=3 的日志，S1,S2 有 term=2 的日志

**(c) S1 重启，再次当选 Leader（Term 4）**：
- S1 开始复制它的 term=2 的日志
- 现在大多数节点（S1,S2,S3,S4）都有 term=2 的日志
- **如果 S1 此时决定提交 term=2 的日志...**

**(d) S1 崩溃，S5 再次当选 Leader（Term 5）**：
- S5 发现大多数节点（S2,S3,S4,S5）有 term=3 的日志
- S5 用 term=3 的日志覆盖 term=2 的日志
- **问题**：term=2 的日志可能已经被 S1 应用到状态机了！
- **结果**：不同节点的状态机执行了不同的命令 → 违反一致性

### 8. Raft 的两个关键安全机制

#### 机制 1：选举限制（投票规则）

**RequestVote RPC 携带候选人的日志信息**：
```java
class RequestVoteRequest {
    int term;
    String candidateId;
    int lastLogIndex;  // 最后一条日志的索引
    int lastLogTerm;   // 最后一条日志的任期
}
```

**投票规则**：
```java
boolean shouldVote(RequestVoteRequest req) {
    // 规则 1：term 必须 >= currentTerm
    if (req.term < currentTerm) return false;

    // 规则 2：没投过票，或者投给了这个候选人
    if (votedFor != null && !votedFor.equals(req.candidateId)) {
        return false;
    }

    // 规则 3：候选人的日志必须至少和我一样新
    if (!isLogAtLeastAsNew(req.lastLogTerm, req.lastLogIndex)) {
        return false;
    }

    return true;
}

boolean isLogAtLeastAsNew(int lastLogTerm, int lastLogIndex) {
    // 如果最后一条日志的 term 不同，term 大的更新
    if (lastLogTerm > getLastLogTerm()) return true;
    if (lastLogTerm < getLastLogTerm()) return false;

    // 如果 term 相同，日志长的更新
    return lastLogIndex >= getLastLogIndex();
}
```

**意义**：
- 只有拥有大多数已提交日志的节点才能当选 Leader
- 新 Leader 一定包含了所有已提交的日志

#### 机制 2：提交规则

**关键规则**：
> Leader 只能提交当前任期内的日志条目！

**示例**：
```
Term 4，Leader S1：
- S1 有 term=2 的老日志
- S1 可以复制 term=2 的日志到大多数节点
- 但是 S1 不能提交 term=2 的日志！
- S1 必须写一条新的 term=4 的日志
- 当 term=4 的日志被复制到大多数 → 可以提交 term=4 的日志
- 因为"日志匹配特性"，此时 term=2 的日志也自动被提交了
```

**意义**：
- 避免了旧任期日志被提交后被新 Leader 覆盖的问题
- 确保只有包含最新日志的节点才能当选

---

## 教学方法使用

### 1. 问题驱动的探索
- 通过了解学生的实际经验切入（MongoDB、MySQL 主从）
- 用学生熟悉的概念类比 Raft 的概念

### 2. 苏格拉底式提问
- 引导学生思考核心问题
- 通过思考题验证理解
- 鼓励学生用自己的话解释

### 3. 纠正误解
- 纠正学生对"日志不一致原因"的误解（不是失败，是崩溃）
- 纠正对"提交规则"的理解（不能单独提交旧任期日志）

### 4. 代码示例
- 用 Java 伪代码展示核心逻辑
- 结合学生熟悉的 Java 并发经验

---

## 理解检查

### 正确理解的内容
1. ✅ 领导人选举的基本流程
2. ✅ 随机超时机制的作用
3. ✅ 日志新旧比较规则
4. ✅ 选举限制的重要性
5. ✅ 提交规则的必要性

### 需要加深理解的内容
1. ⚠️ 日志不一致的根本原因（不是失败，是崩溃）
2. ⚠️ 提交规则如何具体防止数据不一致（需要更多示例）
3. ⚠️ 日志匹配特性（Log Matching Property）

---

## 待学习内容

1. **日志复制的完整机制**：
   - AppendEntries RPC 的详细流程
   - 一致性检查机制
   - Leader 如何处理 Follower 的日志不一致

2. **日志匹配特性（Log Matching Property）**：
   - 特性的定义
   - 如何保证
   - 与提交规则的关系

3. **状态机安全性的证明**：
   - 领导人完整特性的完整证明
   - 如何推导到状态机安全性

4. **实际实现问题**：
   - AppendEntries RPC 的完整协议
   - nextIndex 和 matchIndex 的维护
   - 日志压缩（Snapshot）

5. **Java 实现的具体细节**：
   - 数据结构设计
   - 并发控制
   - 持久化

---

## 识别的知识缺口

### 高严重程度
- [ ] AppendEntries RPC 的完整流程和实现
- [ ] Leader 如何发现并修复 Follower 的日志不一致
- [ ] 日志匹配特性的完整理解

### 中严重程度
- [ ] commitIndex 和 lastApplied 的关系
- [ ] nextIndex 和 matchIndex 的具体维护逻辑
- [ ] 客户端与 Raft 集群的交互

### 低严重程度
- [ ] 选举超时时间的具体配置建议
- [ ] 心跳频率的设置
- [ ] 性能优化技巧

---

## 掌握的主题

### 高信心度
- [ ] 领导人选举机制 (2026-01-14) - 理解核心流程和随机超时机制
- [ ] 日志新旧比较规则 (2026-01-14) - 能准确比较两个日志的新旧程度
- [ ] 选举限制规则 (2026-01-14) - 理解为什么需要限制投票条件

### 中高信心度
- [ ] Raft 的三个核心问题 (2026-01-14) - 能描述领导人选举、日志复制、安全性
- [ ] 随机超时机制 (2026-01-14) - 理解如何避免选票瓜分
- [ ] 提交规则的重要性 (2026-01-14) - 知道只能提交当前任期的日志

---

## 后续学习计划

1. **学习 AppendEntries RPC 的完整机制**（下一次会话）
   - RPC 的参数和返回值
   - 一致性检查的实现
   - 日志复制的完整流程

2. **理解 Leader 如何修复 Follower 的日志不一致**
   - nextIndex 的维护
   - 递减重试机制
   - 最终达成一致

3. **学习日志匹配特性**
   - 特性的定义
   - 归纳法的证明思路
   - 与一致性的关系

4. **状态机安全性的完整证明**
   - 领导人完整特性的证明
   - 如何推导到状态机安全性

5. **开始讨论 Java 实现的设计**
   - 数据结构设计
   - 线程模型
   - 持久化方案

---

## 学生提出的有趣问题和观点

1. **MongoDB 读一致性问题**：
   - 学生提出了实际工程中的问题
   - 理解从节点延迟的根本原因
   - 知道不同的解决方案（休眠、只读主节点）

2. **MySQL 主从复制的类比**：
   - 学生理解主从复制的基本概念
   - 对 binlog 的理解有助于理解 Raft 日志
   - 知道复制位置的重要性

3. **脑裂问题的思考**：
   - 学生知道需要过半票数避免脑裂
   - 理解 Raft 的选举安全性

4. **对日志不一致原因的猜测**：
   - 最初认为是"写入失败"
   - 实际上是"Leader 崩溃"
   - 通过具体场景纠正理解

5. **对提交问题的直觉**：
   - 学生理解"提交后不能覆盖"的重要性
   - 通过危险场景意识到问题的严重性

---

## 教学反思

### 有效的教学方法
1. ✅ 从学生的实际经验切入（MongoDB、MySQL）
2. ✅ 用 Java 伪代码展示核心逻辑
3. ✅ 通过具体场景（5 节点集群）演示问题
4. ✅ 苏格拉底式提问，引导学生思考

### 需要改进的地方
1. ⚠️ 对"日志不一致"的描述需要更清晰
2. ⚠️ 需要更多图示来辅助理解
3. ⚠️ 可以提供更多实际代码示例

### 下次会话的改进
1. 准备更多的图示来展示日志复制流程
2. 提供更完整的 Java 实现示例
3. 设计更多的实践场景让学生思考
