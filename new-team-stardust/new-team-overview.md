# 新团队概览 / New Team Overview

> 本文档帮助理解这个项目的技术架构和全栈技术栈，为成长为全栈工程师打下基础。

---

## 1. 技术栈总览

### 1.1 后端技术栈

| 技术 | 用途 | 类比说明 |
|------|------|----------|
| **Go (Gin + GORM + JWT)** | 主要 API 服务 | 相当于 Node.js + Express + Sequelize |
| **Temporal** | 异步任务/工作流引擎 | 相当于一个超级强大的"任务调度中心"，比 cron 更可靠 |
| **Node.js (Fastify + FFmpeg)** | 视频处理微服务 | 专门处理视频的独立服务 |

**对比理解（Frontend 视角）：**

```
传统前端眼中的后端：
Node.js/Express  → 处理请求 + 数据库操作 + 业务逻辑

本项目后端分工：
Go (Gin)         → API 网关 + 业务逻辑（主要入口）
Temporal         → 异步任务调度（类似于"后台任务队列"）
Node.js (Fastify) → 专门干视频处理这件事
```

### 1.2 前端技术栈

- **Vue 3 SPA** - 单页面应用
- **Axios** - HTTP 请求库，调用 API

### 1.3 数据层

- **PostgreSQL** - 关系型数据库（Go 服务使用 GORM 操作）
- **火山云 TOS** - 媒体文件存储（视频、图片等）

### 1.4 AI 供应商

系统集成了多个 AI 服务商：

| 供应商 | 用途 |
|--------|------|
| **豆包** (默认) | 图像、视频生成 |
| **Seedance** | 视频生成 |
| **海螺** | 视频生成 |
| **可灵** | 图像 + 视频 |
| **腾讯 MPS** | 视频处理 |
| **MiniMax** | 语音克隆 |
| **讯飞 TTS** | 文字转语音 |
| **快绘 AI** | 图像 |

---

## 2. 项目结构解析

### 2.1 目录结构

```
mono (单体仓库)
├── apps/
│   ├── web/              # 前端应用 (Vue 3)
│   ├── admin-web/        # 管理后台 (Vue 3)
│   ├── api-go/           # Go API 服务 (主 API)
│   ├── worker-go/        # Temporal Worker (处理异步任务)
│   └── video-service/    # 视频处理微服务 (Node.js)
├── packages/
│   ├── share-types/      # 共享类型定义 (前后端共用)
│   └── go-shared/        # Go 共享代码 (含数据库)
├── docs/                 # 文档
└── docker/               # Docker 编排文件
```

### 2.2 请求流向（数据流）

```
前端 (Vue)
  ↓ axios
API Server (Go + Gin + JWT)
  ├→ 数据库 (PostgreSQL + GORM)
  ├→ Temporal Client → Temporal Server → Worker (Go)
  │                                         ├→ AI 供应商
  │                                         └→ Video Service
  └→ 火山云 TOS (媒体存储)
```

### 2.3 关键理解点

**Q: 为什么需要 Temporal？**
A: 有些任务很耗时（如视频生成、AI 生成），不能让用户等待。 Temporal 负责：
- 把任务拆解成步骤
- 可靠执行（即使服务器重启也不丢任务）
- 支持任务状态查询
- 支持失败重试

**Q: 为什么视频处理是独立服务？**
A: 视频处理是 CPU 密集型任务，独立出来不会影响主 API 性能。

**Q: JWT 在哪里？**
A: Go API 使用 JWT 做身份验证，前端在请求头中携带 Token。

---

## 3. 产品信息

- **产品地址**: ai-workflow.huamaobook.com
- **定位**: AI 工作流平台（推测是让用户创建和管理 AI 生成任务的可视化平台）

---

## 4. 开发建议

### 4.1 快速上手路径

1. **先了解 Go 基础** - 语法和 Node.js 类似，但类型更严格
2. **理解 GORM** - 就像 Sequelize，是 ORM 库
3. **理解 Gin** - 路由框架，类似 Express
4. **理解 Temporal** - 这是新概念，需要重点学习
5. **了解视频服务** - FFmpeg 用于视频编解码

### 4.2 必读文档

- [Go 快速入门](./go-quick-start.md) - Go 语言速查
- [Temporal 工作流基础](./workflow-basics.md) - 异步任务概念
- [术语表](./tech-stack-glossary.md) - 技术术语解释

---

## 5. 待理解区域（个人笔记）

- [ ] Go 协程和并发模型
- [ ] Temporal Workflow 定义和 Activity
- [ ] GORM 关联查询
- [ ] JWT 中间件实现
- [ ] FFmpeg 命令行参数
- [ ] 火山云 TOS SDK 使用
