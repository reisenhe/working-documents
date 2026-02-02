## Open Canvas 项目架构说明

### 1. 项目总体介绍

Open Canvas 是一个围绕“文档 + 智能体协作”的 Web 应用：

- **核心理念**：把传统“聊天对话”升级成围绕一个持续演化的 Artifact（文稿/代码）的协作过程。
- **使用场景**：写作、代码重构、邮件/文档撰写、风格改写、总结与润色等。
- **关键特性**：
  - 内置记忆系统（反射代理维护长期偏好与风格）。
  - 单个会话绑定一个 Artifact，支持版本回溯与多次改写。
  - 文本与代码双模式，并支持 Web 搜索增强上下文。

整体以 **monorepo** 组织：

- `apps/web`：Next.js 前端（UI + API 路由代理）。
- `apps/agents`：基于 LangGraph 的智能体图（后端推理逻辑）。
- `packages/shared`：前后端共享的类型、模型配置与工具函数。
- `packages/evals`：评估与测试相关逻辑。

前端通过 API 路由将请求代理到本地运行的 LangGraph Server（`apps/agents`），共同组成完整的应用。

---

### 2. 前端架构概览（apps/web）

#### 2.1 技术栈

- **框架**：Next.js 14（App Router）。
- **语言**：TypeScript + React 18。
- **状态与数据**：
  - React Context（`UserContext`、`ThreadProvider`、`GraphContext`、`AssistantContext`）。
  - `zustand`（轻量状态管理，用于 UI / 运行时状态）。
- **UI 组件**：
  - Tailwind CSS + Radix UI + 自定义组件（`apps/web/src/components/ui`）。
  - `@assistant-ui/react` 用于对话体验（线程、消息流、Composer 等）。
- **富文本 & 代码编辑**：
  - BlockNote / Markdown 编辑器。
  - CodeMirror 语言高亮支持多种语言（JS/TS/Python/SQL 等）。
- **认证与存储**：
  - Supabase 作为认证和后端数据存储（会话、用户等）。
- **与后端通讯**：
  - Next.js API Route 代理到 LangGraph Server（通过 `LANGGRAPH_API_URL`）。

#### 2.2 页面与路由结构

- `src/app/page.tsx`
  - 主页，包裹了 `UserProvider`、`ThreadProvider`、`AssistantProvider`、`GraphProvider`，渲染核心 Canvas 界面。
  - 实际使用中会在用户未登录时重定向至 `/auth/login`（登录页面）。
- `src/app/auth/*`
  - 登录、注册、确认邮箱、登出等页面，与 Supabase 认证对接。
- `src/app/api/[..._path]/route.ts`
  - 通用代理路由：对 `/api/*` 的请求进行统一处理，并转发到 LangGraph Server。
  - 负责：
    - 从 Supabase 获取当前用户与 Session。
    - 将 Session / User 信息注入到代理请求的 `config.configurable` 中（供后端使用）。

#### 2.3 核心 UI 组件

- `components/canvas/CanvasComponent`
  - 整个“画布 + 聊天”区域的布局与行为核心。
  - 职责：
    - 管理 `chatStarted`（是否已开始对话）。
    - 管理聊天面板折叠状态 `chatCollapsed`（同时与 URL query 同步）。
    - 响应 Quick Start（根据文本/代码模板快速生成初始 Artifact）。
    - 通过 `GraphContext` 与 Artifact 状态联动。
- `components/artifacts/ArtifactRenderer`
  - 负责根据 Artifact 类型（text/code）渲染具体内容：
    - 文本：Markdown 渲染器。
    - 代码：CodeMirror 等代码视图。
  - 使用 `@opencanvas/shared/utils/artifacts` 提供的工具函数（如 `getArtifactContent`）统一解析 Artifact 数据结构。
- `components/chat-interface/*`
  - 封装对话 UI：输入框、消息列表、消息发送逻辑等。
  - 集成 `@assistant-ui/react` 的线程模型（Thread）、Composer、附件等能力。

#### 2.4 Context 设计

- `UserContext` (`contexts/UserContext.tsx`)
  - 负责从 Supabase 获取当前用户信息（`supabase.auth.getUser()`）。
  - 提供 `user`、`loading` 与 `getUser` 方法给其余组件使用。
- `ThreadProvider` (`contexts/ThreadProvider.tsx`)
  - 管理当前对话线程（Thread）的：
    - ID、模型配置（`modelName`, `modelConfig`）。
    - 与 LangGraph Server 的请求交互（创建新线程、搜索历史线程等）。
  - 通过 API `/api/threads/*` 调用代理，该代理最终触发后端 Open Canvas graph。
- `GraphContext` (`contexts/GraphContext.tsx`)
  - 封装与 Artifact 相关的状态：
    - 当前 Artifact（`ArtifactV3`）。
    - 高亮片段（代码/文本高亮，用于局部修改）。
    - 是否已启动聊天（`chatStarted`）。
- `AssistantContext` (`contexts/AssistantContext.tsx`)
  - 管理当前助手配置（代表不同人物/风格的 agent 配置）。
  - 关联“Quick Actions”“预设 Prompt”等。

#### 2.5 前端关键概念

- **Artifact**

  - 定义在 `@opencanvas/shared/types` 中（`ArtifactV3`、`ArtifactCodeV3`、`ArtifactMarkdownV3`）。
  - 一个 Artifact 代表当前会话下的“主作品”：可以是 Markdown 文档，也可以是代码文件。
  - 支持版本化（历史版本索引）与多内容片段（例如多段代码/多段文本）。

- **Thread**

  - 代表一条对话历史（一个会话），包含消息流与该会话所关联的 Artifact。
  - 通过 LangGraph SDK 与 LangGraph Server 对应的线程概念映射。

- **Quick Actions**

  - 一系列预定义或自定义的操作（例如“润色”、“翻译”、“总结”、“改写为某种风格”等）。
  - 通过调用不同的 LangGraph 节点（如 `rewriteArtifactTheme`、`CHANGE_ARTIFACT_LANGUAGE_PROMPT` 等）实现特定变换。

- **Web 搜索集成**
  - `apps/agents/src/web-search/*` 实现了一个子图，用于根据消息自动判断是否需要触发 Web 搜索。
  - 前端在需要时启用 `webSearchEnabled`，后端再将搜索结果以特殊 AI 消息注入 `_messages`。

---

### 3. 后端架构概览（apps/agents）

#### 3.1 技术栈

- **LangGraph**（`@langchain/langgraph`）
  - 声明式状态机/有向图，用于描述多步 Agent 工作流（节点、边、条件路由）。
- **LangChain Core + 各类 LLM Provider SDK**
  - `@langchain/openai`、`@langchain/anthropic` 等。
- **运行方式**
  - 通过 LangGraph CLI 以“图服务器”的形式运行（监听 54367 端口）。
  - `apps/agents` 本身导出 graph 配置，供 LangGraph Server 加载。

#### 3.2 Open Canvas Graph（核心智能体流程）

入口在：`apps/agents/src/open-canvas/index.ts`。

- **状态定义**：`OpenCanvasGraphAnnotation`（`state.ts`）

  - 继承自 `MessagesAnnotation`，并扩展了多种 Annotation 字段：
    - `_messages`：内部消息列表，用来存放原始 + 总结后的消息（不直接暴露给用户）。
    - `artifact`：当前 Artifact（`ArtifactV3`）。
    - `highlightedCode` / `highlightedText`：用户选中的片段（用于局部改写）。
    - `language` / `artifactLength` / `readingLevel` / `regenerateWithEmojis` 等：用户偏好与变换选项。
    - `customQuickActionId`：选中的自定义动作。
    - `webSearchEnabled` / `webSearchResults`：Web 搜索控制与结果。
  - `_messages` 的 reducer 会对“总结消息”进行特殊处理（使用 `OC_SUMMARIZED_MESSAGE_KEY` 标记），以实现“长对话自动摘要 + 截断”的能力。

- **图结构（StateGraph）**：

  - 起始节点：`generatePath`
    - 根据当前状态（是否已有 Artifact、是否有高亮、是否启用 Web 搜索等）决定下一步路由。
  - 主要节点：
    - `generateArtifact`：首次生成 Artifact。
    - `updateArtifact`：整体修改 Artifact。
    - `updateHighlightedText`：基于高亮片段做局部改写。
    - `rewriteArtifact` / `rewriteArtifactTheme` / `rewriteCodeArtifactTheme`：不同维度的改写（主题、风格、代码语义等）。
    - `replyToGeneralInput`：仅回复消息，不改写 Artifact。
    - `customAction`：执行用户配置的自定义 Prompt 动作。
    - `generateFollowup`：生成“跟进”回复（如：提示用户已完成修改，并可继续操作）。
    - `reflect`：反射节点，将对话与 Artifact 进行总结，写入长期记忆（Style / Fact 等）。
    - `summarizer`：当消息长度过大（> 300k 字符）时触发，将历史消息压缩成总结消息。
    - `webSearch`：调用 Web 搜索子图。
    - `routePostWebSearch`：根据搜索结果是否存在决定是走 `rewriteArtifact` 还是 `generateArtifact` 等。
    - `generateTitle`：为会话生成标题（仅在前两轮对话后）。
  - 结束逻辑：
    - `cleanState` 节点负责重置部分状态（如 `DEFAULT_INPUTS`）。
    - `conditionallyGenerateTitle` 判断是否需要生成标题或执行总结。

- **路由逻辑亮点**：
  - `routeNode`：根据 `state.next` 字段将执行权交给指定节点。
  - `simpleTokenCalculator`：根据 `_messages` 中字符总量决定是否进入 `summarizer`。
  - `routePostWebSearch`：同时考虑“是否已存在 Artifact”与“是否有搜索结果”决定下一步是生成还是改写。

#### 3.3 Prompt 设计与复杂操作

在 `prompts.ts` 中定义了大量模板，用于指挥模型完成细粒度编辑：

- **NEW_ARTIFACT_PROMPT**：创建新 Artifact 的总控 Prompt。
  - 强调：
    - 使用完整聊天上下文。
    - 避免三重反引号包裹代码（因为 UI 不做 Markdown 渲染时会出问题）。
    - 使用反射记忆 `{reflections}` 指导风格与用户偏好。
- **UPDATE_HIGHLIGHTED_ARTIFACT_PROMPT**：仅修改高亮片段。
  - 约束：只返回修改后的高亮文本，不返回整个文档、不包裹标签或多余内容。
- **CHANGE_ARTIFACT_LANGUAGE / READING_LEVEL / LENGTH / ADD_EMOJIS 等 PROMPT**
  - 这些 Prompt 对 Artifact 进行特定维度的变换（语言、阅读难度、长度、emoji 装饰等）。
  - 通用模式：给出 `<artifact>` 包裹的原文 + `<reflections>` 记忆 + 严格输出格式约束。
- **路由相关 Prompt**：
  - `ROUTE_QUERY_PROMPT`：根据当前 Artifact 是否存在、最近几条消息和应用上下文，判断用户是想“生成新 Artifact”、“改写 Artifact”还是“仅聊天回复”。

这些 Prompt 是实现“强约束、弱代码”的关键：通过 Prompt 工程约束模型行为，而不是在代码里做复杂后处理。

#### 3.4 与 Supabase 的结合

在 `apps/agents/src/utils.ts` 中：

- `getUserFromConfig(config)`：
  - 从 `config.configurable.supabase_session` 中读取 Supabase Session（由前端代理层注入）。
  - 使用 `SUPABASE_SERVICE_ROLE` + `NEXT_PUBLIC_SUPABASE_URL` 创建服务端 Supabase 客户端。
  - 通过 `supabase.auth.getUser(accessToken)` 恢复当前用户。
  - 让图中的节点能够访问用户级别数据（如自定义 Quick Actions、历史记忆等）。

这一链路依赖前端 API 代理在请求体中注入 `supabase_session` 和 `supabase_user_id`，由 `apps/web/src/app/api/[..._path]/route.ts` 负责。

---

### 4. 共享模块（packages/shared）

#### 4.1 主要内容

- `src/types.ts`
  - 定义了跨前后端共享的核心类型：
    - Artifact 相关类型（`ArtifactV3`、`ArtifactCodeV3`、`ArtifactMarkdownV3`）。
    - 各种选项枚举（语言、阅读等级、长度、编程语言等）。
- `src/models.ts`
  - 定义前端可选模型列表、Provider 映射等（OpenAI、Anthropic、Fireworks 等）。
  - 提供 `getModelConfig` 等方法，在 agents 端根据模型名称选择正确的 LLM 与参数。
- `src/utils/*`
  - `artifacts.ts`：处理 Artifact 的解析、版本切换、内容提取等。
  - `thinking.ts`：与思维链/中间推理相关的工具（例如是否展示“思考过程”）。
  - `urls.ts`：拼接或解析后端/前端 URL 的工具。
- `src/prompts/quick-actions.ts`
  - 预置 Quick Actions 的配置模板（如“翻译”、“改写成更正式”、“总结”等）。

#### 4.2 设计意义

- 保证 **前后端对 Artifact / 模型 / 配置的理解完全一致**。
- 减少类型漂移与魔法字符串，方便未来维护与扩展。

---

### 5. 关键依赖与解决的问题

#### 5.1 LangGraph + LangChain

- **解决的问题**：如何用更结构化的方式编排多步智能体流程，而不是“单次大 Prompt”。
- **在项目中的角色**：
  - 每个节点（Node）代表一项清晰任务（生成、改写、总结、反射、路由、Web 搜索等）。
  - 通过状态机与条件边将复杂交互流程拆分为易维护的子步骤。

#### 5.2 Supabase

- **解决的问题**：统一的用户认证、会话存储、共享状态管理。
- **用途**：
  - 邮箱 + 第三方 OAuth 登录。
  - 存储用户的线程、Artifact 历史、Quick Actions 等。

#### 5.3 Assistant UI / Rich Editor 生态

- `@assistant-ui/react`、BlockNote、CodeMirror 等组合：
  - 提供接近“IDE + 文档编辑器 + ChatGPT”的复合体验。
  - 结合 Artifact 概念，将“对话”与“对象（文档/代码）”强绑定。

#### 5.4 多模型支持

- 通过 shared 模块配置：
  - 同时支持 Anthropic、OpenAI、Fireworks、Gemini 等。
  - 可以按需要切换不同 Provider，以平衡成本、速度与质量。

---

### 6. 典型调用链路（从前端到后端）

1. **用户登录**：

   - 前端使用 Supabase JS SDK 完成 OAuth / 邮箱登录。
   - Session 写入浏览器 Cookie。

2. **用户输入请求（例如“帮我改写这段代码并加注释”）**：

   - Canvas 中的聊天组件捕获输入，通过 `ThreadProvider` 发送请求到 `/api/threads/*`。

3. **Next.js API 代理层**：

   - 路由 `apps/web/src/app/api/[..._path]/route.ts`：
     - 调用 `verifyUserAuthenticated`，利用 Supabase Server 客户端获取用户与 Session。
     - 构造请求体：在 `config.configurable` 内注入 `supabase_session`、`supabase_user_id`。
     - 将请求转发到 `LANGGRAPH_API_URL`（默认 `http://localhost:54367`）对应路径。

4. **LangGraph Server（apps/agents）**：

   - 接收请求并把其映射到 `open_canvas` graph。
   - 依据当前 state 和用户的输入：
     - Route 节点判断走“generate / rewrite / reply / webSearch”等路径。
     - 调用相应 LLM（如 OpenAI / Anthropic）完成文本/代码生成或改写。
     - 更新 Artifact、生成 followup 消息、写入反射记忆。

5. **前端更新 UI**：
   - 接收 LangGraph 返回的新消息与 Artifact 状态。
   - 更新 Canvas 展示：
     - 覆盖/新增 Artifact 内容。
     - 更新聊天记录。
     - 视情况更新 Quick Actions 可用性。

---

### 7. 如何阅读与扩展本项目（如果未来想再继续）

如果未来你或其他人想恢复维护，可以按以下顺序切入：

1. **从前端开始**：

   - 阅读 `apps/web/README.md` 获取运行方式与配置说明。
   - 理解 `CanvasComponent` 与 `ThreadProvider` 的交互方式。

2. **理解 Artifact 数据模型**：

   - 阅读 `packages/shared/src/types.ts` 与 `packages/shared/src/utils/artifacts.ts`。
   - 搞清楚 Artifact 的版本、多片段与类型（text/code）是如何表达的。

3. **深入 LangGraph 流程**：

   - 阅读 `apps/agents/src/open-canvas/index.ts` 和 `state.ts`。
   - 梳理节点之间的边和条件路由，理解每个节点的责任。

4. **修改或新增功能**：
   - 如果只是前端交互/样式调整，集中在 `apps/web/src/components` 与 `contexts` 即可。
   - 如果要新增一个“Quick Action”，通常需要：
     - 在 `shared` 中增加 Prompt 配置。
     - 在 `apps/agents` 的某个节点中接入新的 Prompt 或逻辑。

---

### 8. 小结

- Open Canvas 的本质是：
  - 一个 Next.js 前端 + 一个 LangGraph 智能体服务器。
  - 中间通过 API 代理层和共享类型/工具解耦。
- 关键价值在于：
  - 用 **图式编排（LangGraph）** 管理复杂对话/编辑流程。
  - 用 **Artifact 模型** 把“对话”与“文档/代码”严密关联起来。
  - 通过 **反射记忆** 逐步积累用户的风格与偏好。

即便你决定不再继续维护，这份架构说明可以帮助未来的维护者快速理解该项目的整体设计与关键抽象。
