# 技术术语表

> 按类别整理项目涉及的技术术语，方便速查。

---

## 1. 编程语言 & 框架

| 术语 | 全称 | 简要解释 |
|------|------|----------|
| **Go / Golang** | - | Google 开发的编译型语言，以高性能和并发支持著称 |
| **Gin** | - | Go 的 Web 框架，类似 Express |
| **GORM** | Go ORM | Go 的对象关系映射库，类似于 Sequelize/Prisma |
| **JWT** | JSON Web Token | 一种身份验证机制，Token 包含用户信息 |
| **Node.js** | - | JavaScript 运行时 |
| **Fastify** | - | Node.js 的高性能 Web 框架 |
| **FFmpeg** | - | 视频/音频处理工具命令行工具 |
| **Vue 3** | - | 前端框架，当前项目使用 |
| **Axios** | - | HTTP 客户端库 |

---

## 2. 数据库 & 存储

| 术语 | 全称 | 简要解释 |
|------|------|----------|
| **PostgreSQL** | - | 关系型数据库，功能丰富 |
| **GORM** | Go ORM | Go 的数据库 ORM（见上） |
| **TOS** | Tencent Object Storage | 火山云对象存储，存放视频/图片等媒体文件 |

---

## 3. 异步任务 & 工作流

| 术语 | 全称 | 简要解释 |
|------|------|----------|
| **Temporal** | - | 可靠的工作流执行引擎，支持任务持久化和重试 |
| **Workflow** | - | Temporal 中的流程定义，编排 Activities |
| **Activity** | - | Temporal 中的具体业务操作步骤 |
| **Worker** | - | 执行 Workflow 和 Activity 的进程 |

---

## 4. AI 供应商

| 术语 | 厂商 | 用途 |
|------|------|------|
| **豆包** | 字节跳动 | 图像、视频生成（默认供应商） |
| **Seedance** | - | 视频生成 |
| **海螺** | - | 视频生成 |
| **可灵** | 快手 | 图像 + 视频生成 |
| **腾讯 MPS** | 腾讯云 | 视频处理 |
| **MiniMax** | MiniMax | 语音克隆 |
| **讯飞 TTS** | 科大讯飞 | 文字转语音 |
| **快绘 AI** | - | 图像生成 |

---

## 5. 架构相关

| 术语 | 简要解释 |
|------|----------|
| **Mono Repo** | 单体仓库，多个项目放在一个代码库中 |
| **SPA** | Single Page Application，单页面应用 |
| **SSE** | Server-Sent Events，服务器向浏览器推送数据的技术 |
| **微服务** | 将大型应用拆分为独立部署的小服务（项目中的 video-service） |

---

## 6. Docker & 部署

| 术语 | 全称 | 简要解释 |
|------|------|----------|
| **Docker** | - | 容器化平台，应用打包工具 |
| **Docker Compose** | - | Docker 编排工具，管理多个容器 |

---

## 7. 其他

| 术语 | 全称 | 简要解释 |
|------|------|----------|
| **API Server** | Application Programming Interface Server | 提供 API 接口的后端服务 |
| **SSE** | Server-Sent Events | HTTP 持久连接，服务器可向客户端推送数据 |
| **JWT Middleware** | JWT Middleware | JWT 身份验证中间件 |

---

## 8. 常见缩写

| 缩写 | 含义 |
|------|------|
| **Gin** | Go 语言的 Web 框架（不是 Python Flask） |
| **GORM** | Go ORM，Go 语言的数据库 ORM |
| **TOS** | 火山云对象存储 |
| **MPS** | Media Processing Service，媒体处理服务 |
| **TTS** | Text To Speech，文字转语音 |
| **SSE** | Server-Sent Events，服务器推送事件 |
| **JWT** | JSON Web Token，身份验证令牌 |
