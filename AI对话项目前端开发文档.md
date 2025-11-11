# 前言
AI对话项目与传统前端开发的主要差异在于SSE（Server-Sent Events）流式数据的接收处理及响应渲染。为便于实践理解，我们已搭建集成基于 Nestjs框架 + 公司本地AI服务的演示后端项目，并在前端采用@microsoft/fetch-event-source实现流式通信。本文将基于该项目解析AI交互的数据格式规范与请求模式，通过示例展示不同响应类型的数据结构特征，为后续业务功能扩展提供技术实现参考。
演示项目地址：
在这个文档中，你可以了解到：
- SSE流式通信核心技术
- @microsoft/fetch-event-source工程化实践
- 流式接口生命周期管理
- 消息界面布局基础框架与设计方式
- Markdown富文本渲染方案

---
# SSE流式通信基础
技术原理：SSE（Server-Sent Events）是浏览器与服务器建立的单向通信通道，通过消息流形式接收服务器实时推送的数据
核心价值：相比传统请求模式，SSE流式输出具备以下核心优势：
1. 实时性：在模型运算过程中逐步返回中间结果，实现边生成边展示；
2. 低延迟：用户可在AI思考阶段即开始阅读已生成内容；
3. 稳定性：长文本输出时有效避免单次响应超时问题
<img width="631" height="343" alt="image" src="https://github.com/user-attachments/assets/db59253d-f788-42c6-806f-e76f4004f891" />

使用方法：
1. 使用浏览器提供的 EventSource 实例
2. 使用服务器发送事件 - Web API | MDN
3. 使用工具库 @microsoft/fetch-event-source
相比基础API，其支持自定义请求参数、请求头配置及错误重连机制，更适合复杂业务场景的工程化需求。具体实现方案将在后续模块结合代码示例展开说明
# fetchEventSource 请求参数规范
工具库 @microsoft/fetch-event-source的使用示例，这部分会对请求参数进行讲解
```ts
import { fetchEventSource } from '@microsoft/fetch-event-source';

/** 创建消息流 */
function createEventStream<T = any>(url: string, params: T) {
  const controller = new AbortController();
  fetchEventSource(url, {
    signal: controller.signal,                       // 支持中断对话流程
    method: 'POST',                                  // 支持结构化数据传输
    headers: { 'Content-Type': 'application/json' }, // 必需设置JSON格式
    openWhenHidden: true,                            // 保持后台标签页连接
    body: JSON.stringify(params),                    // 参数序列化
    // 流式通信生命周期钩子
    async onopen(response) {},
    onmessage(msg) {},
    onclose() {},
    onerror(err) {}
  })
  return controller;
}
```
参数说明 ：
1. signal：通过 AbortController.abort() 支持用户主动中断对话
2. method：POST请求支持结构化参数（如权限令牌），适用于复杂业务场景
3. headers：必须设置 Content-Type: application/json 保证数据正确解析
4. openWhenHidden：保持后台连接持续接收数据，避免切换标签页导致中断
5. body：需将参数转换为 JSON 字符串格式
# fetchEventSource 接口回调函数
这个部分将会对工具库 @microsoft/fetch-event-source的回调函数进行解析
## onopen(response: Response) => Promise<void>
连接建立后的回调函数
response 是简化后的响应类型，结构使用了部分 mdn 定义的 Response类型，包含以下关键属性：
1. Response.ok - 检查握手状态（true 表示成功）
2. Response.headers - 验证流式通道有效性：
```ts
response.headers.get('Content-Type')?.includes(EventStreamContentType) // 需从库导入
```
1. Response.body - 服务器返回的可读流
2. Response.status - HTTP状态码（如401/500错误提示）
## onmessage(ev: EventSourceMessage) => void;
接受消息的回调函数，流式消息将会将服务器的传输结果传入到该函数里
EventSourceMessage 是库封装好的固定结构，每条消息的内容将会从 data 属性中获取
```ts
interface EventSourceMessage {
  id: string;      // 消息唯一标识
  event: string;   // 事件类型（默认为'message'）
  data: string;    // 实际传输内容
  retry?: number;  // 重连间隔毫秒数
}
```
## onclose?: () => void
服务器发生了意外的链接关闭时的回调函数，可以在这个函数内编写重新请求提示的展示逻辑
## onerror?: (err: any) => Error
流式请求出现错误时的回调函数，可以在这个函数内编写对应错误的提示展示逻辑
# 消息界面布局
<img width="613" height="906" alt="image" src="https://github.com/user-attachments/assets/288d85c4-45b3-41f2-baaa-ef9507804900" />

通常一个ai对话会包含两个区域：
1. 消息列表区
2. 输入发送区
## 消息列表区
而消息列表区也会分为用户提问消息 与 ai回复消息。有些项目可能会展示一些系统提示消息，但总的来说可以分为 用户消息 和 非用户消息 两种样式。在这个示例中，两种样式表现为令消息气泡靠左或靠右两个对齐方向。
如此一来，前端开发者可以先准备两种角色枚举值，以便样式和特定条件上进行判断，并提供一个对话消息钩子封装这个模块的逻辑
```ts
import { MessageRoleEnum } from "@/enums/message.enum";
import { createEventStream } from "@/controllers/sse.controller";
import { ref } from "vue";

/** 服务端提供ai对话的接口地址 */
const SSE_URL = '/api/sse/stream';

/** 聊天消息hooks */
export function useChat() {
  /** 消息列表 */
  const messages = ref<ChatMessage[]>([]);
  /** 最新一条ai消息 */
  const AiMsg = ref<ChatMessage | null>(null);
  /** 发送消息 */
  function sendMsg(message: string) {
    // 添加用户消息记录
    messages.value.push({
      id: Date.now().toString(),
      content: message,
      role: MessageRoleEnum.USER,
    });
    // 清空最新一条ai消息
    AiMsg.value = null;
    // 创建sse连接
    createEventStream(
      SSE_URL,
      {
        message,
        delay: 0, // 模拟接口延迟
        isThink: false, // 开启思考模式
      },
      {
        /** 接收消息 */
        onmessage: (data) => {
          if (!AiMsg.value) {
            AiMsg.value = {
              id: Date.now().toString(),
              content: "",
              role: MessageRoleEnum.AI,
            };
            messages.value.push(AiMsg.value);
          }
          // 更新最新的ai消息连接
          AiMsg.value.content += data.content;
        },
      }
    );
  }

  return { AiMsg, messages, sendMsg };
}
```
这个钩子会对外返回消息列表(messages)、发送消息事件、以及最新一条ai消息3个变量供模板使用。模板只需要循环渲染消息列表(messages)即可
## 输入发送区
接收用户输入的消息，并提供发送按钮。
而在对话项目在用户发送消息后，ai回复中间，许多对话类项目只会提供 中止消息接收 的操作，并且禁止用户继续输入的交互修改。这将会对消息的生命周期状态有所要求
在此提供一些可供参考的消息状态，在实际的项目应用中可结合产品需求进一步调整
```ts
/** 消息状态枚举 */
export enum MessageStatusEnum {
  FAIL = -1, // 失败 onerror 回调函数或主动使用中止时可添加的状态
  INITIAL = 0, // 初始 onopen 回调函数可添加的状态
  SUCCESS = 1, // 成功 onmessage 回调函数可添加的状态
  DONE = 2 // 完成
}
```

# 消息样式-处理 markdown 格式
当前的各种 ai 对话格式经过了 openai 的规范化，返回内容都会符合 markdown 的格式，我们只需要使用工具库 markdown-it 即可
同时也可以使用一些基础的 markdown 样式库，例如 github-markdown-css，并确保在将要展示 markdown 内容的容器添加 markdown-body 样式 class，这是该样式库的默认选择
```ts
// src/utils/markdown-it.ts
import Markdown from 'markdown-it';
import 'github-markdown-css/github-markdown-light.css';
import '@/assets/css/markdown.less';
const markdown = new Markdown({
  linkify: true, // 支持识别文本连接
  typographer: true, // 智能引号
  breaks: true, // 转化段落里的 \n 成 <br>
  langPrefix: 'language-' // 增加css前缀，可补充样式
});
export default markdown;
```
```ts
<template>
  <div class="markdown-body">
    <div v-html="markdown.render(content)"></div>
  </div>
</template>

<script setup lang="ts">
import markdown from '@/utils/markdown-it.ts';

defineProps({
  content: String // 等待渲染成markdown格式的文字
});
</script>
```
