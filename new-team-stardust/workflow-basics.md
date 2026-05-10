# Temporal 工作流基础

> 写给完全没有工作流开发经验的工程师。以外卖订单为类比，帮助理解核心概念。

---

## 1. 什么是 Temporal？

**Temporal** 是一个**可靠的工作流执行引擎**。

### 为什么不只用 setTimeout + 数据库？

普通方案的问题：
- 服务器重启 → 任务丢失
- 任务执行到一半失败 → 没有重试机制
- 任务状态不透明 → 不知道进行到哪一步

Temporal 的解决：
- **持久化**：任务状态存储在数据库（PostgreSQL），重启不丢失
- **重试机制**：自动重试失败的任务
- **状态可见**：随时查询任务执行到哪一步
- **编排能力**：可以把多个步骤串联成一个完整流程

---

## 2. 核心概念

### 2.1 类比：外卖订单

想象你下一个外卖订单：

```
下订单 → 商家接单 → 骑手取餐 → 配送中 → 送达
```

这个流程就是一个 **Workflow**（工作流）。

- **Activity**：每个步骤（接单、取餐、配送）就是一个 Activity
- **Workflow**：把 Activities 按顺序编排的完整流程
- **Worker**：执行 Workflow 和 Activity 的进程

### 2.2 三个核心组件

| 组件 | 职责 | 类比 |
|------|------|------|
| **Workflow** | 定义流程逻辑（不执行具体操作） | 外卖流程规则手册 |
| **Activity** | 具体的业务操作 | 商家接单、骑手取餐等实际动作 |
| **Worker** | 运行 Workflow 和 Activity 的进程 | 商家、骑手等执行者 |

---

## 3. 代码示例

### 3.1 定义 Activity（具体操作）

```go
// Node.js - 假设你在写一个异步函数
async function notifyUser(userID, message) {
  await sendPushNotification(userID, message);
}

// Go Temporal - Activity 定义
func SendNotification(ctx workflow.Context, userID string, message string) error {
  // 这里放实际业务逻辑
  return sendPushNotification(userID, message)
}
```

### 3.2 定义 Workflow（编排流程）

```go
// Go Temporal - Workflow 定义
func OrderWorkflow(ctx workflow.Context, order Order) error {
  // 1. 创建订单
  err := workflow.ExecuteActivity(ctx, CreateOrderActivity, order).Get(ctx, nil)
  if err != nil {
    return err
  }

  // 2. 通知商家
  err = workflow.ExecuteActivity(ctx, NotifyMerchantActivity, order.MerchantID).Get(ctx, nil)
  if err != nil {
    return err
  }

  // 3. 通知用户
  err = workflow.ExecuteActivity(ctx, SendNotificationActivity, order.UserID, "订单已创建").Get(ctx, nil)
  if err != nil {
    return err
  }

  return nil
}
```

### 3.3 Worker（执行者）

```go
// Node.js - 你可能用 Bull 做后台任务
// worker.js
import Queue from 'bull';
const notificationQueue = new Queue('notifications');

notificationQueue.process(async (job) => {
  await sendPushNotification(job.data.userID, job.data.message);
});

// Go Temporal Worker
worker := worker.New(temporalClient, "default-namespace", "my-task-queue", worker.Options{})
worker.RegisterWorkflow(OrderWorkflow)      // 注册工作流
worker.RegisterActivity(CreateOrderActivity) // 注册活动
worker.RegisterActivity(NotifyMerchantActivity)
worker.Run(worker.InterruptCh())
```

---

## 4. Temporal 的优势

### 4.1 对比普通队列（如 Bull/BullMQ）

| 特性 | Bull (Node.js) | Temporal |
|------|---------------|----------|
| 重启丢失 | 会丢（除非用 Redis） | 不丢 |
| 状态持久化 | 基本不支持 | 完整支持 |
| 失败重试 | 支持但需要手动配置 | 自动 |
| 流程编排 | 简单队列，不支持复杂流程 | 支持复杂编排 |
| 状态查询 | 有限 | 完整执行历史 |

### 4.2 为什么这个项目用 Temporal？

看架构图：

```
前端 → Go API → Temporal Client → Temporal Server → Go Worker
                                                      ↓
                                              AI 供应商 / 视频处理
```

视频生成任务：
1. 用户发起请求
2. API 接收请求，创建一个 Workflow Execution
3. Worker 执行：调用 AI 供应商 → 处理视频 → 返回结果
4. 全程可追踪，失败自动重试

---

## 5. 开发建议

### 5.1 你需要理解的部分

作为前端工程师，你可能**不需要写 Temporal 代码**，但需要理解：

1. **请求是如何触发的** - 前端调 API，API 启动 Workflow
2. **如何查询任务状态** - 通过 API 查询 Workflow 执行状态
3. **SSE 如何配合** - Server-Sent Events 用于实时推送任务进度

### 5.2 如果你要深入

- 官方文档：https://docs.temporal.io/
- Go SDK：https://docs.temporal.io/go
- 重点理解：**Activity vs Workflow** 的区别

---

## 6. 快速检查清单

- [ ] 能说清楚 Workflow 和 Activity 的区别
- [ ] 知道项目里 Worker 在 `apps/worker-go`
- [ ] 知道 Workflow 是在 `worker-go` 里定义的
- [ ] 理解为什么需要 Temporal 而不是普通队列
