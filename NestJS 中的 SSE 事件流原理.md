阅读这篇文章你可以了解到：
- 如何在 NestJS 中使用 @Sse 装饰器来实现服务器发送事件（SSE）。
- 在 NestJS 中如何利用 RxJS 的 Subject 来创建一个 Observable，用于流式传输数据。
- 通过异步生成器函数模拟分段的、逐步的数据流输出方法，以适应 SSE 的需求。
- 当无法调用 AI API 接口时，如何模拟一个符合 SSE 标准的消息流接口，确保客户端能够接收到连续更新的数据。
# 在 nestjs 中创建 sse 接口
为了支持服务器发送事件 (SSE)，NestJS 提供了 @Sse 装饰器，它要求被装饰的方法返回一个 Observable 对象。这使得开发者可以轻松地构建实时应用，其中服务器能够主动推送信息到客户端

可以使用subject.pipe 或 subject.asObservable 返回一个只读的 Observable，它是 rxjs 库的核心类型，表示一个可被观察的异步数据流，实现观察者模式。
# 选择 subject 的原因
Subject 是一种特殊的 Observable，允许我们手动控制何时发出新值或结束流，这将有利于我们在 ai 更新消息时，通过调用 subject.next() 触发事件流更新
# 输出的过程要保持分段异步数据流的形式
当我们调用 openai 的 chat.completions.create 时，会返回一个包含了异步生成器函数（async generator function） 的可迭代对象

每次迭代器输出内容时，调用 subject.next() 将最新的内容派发出去
# 无ai时的流式输出模拟
当我们无法及时调用任何ai的api接口时，我们可以利用以上原理完成满足sse输出形式的模拟接口
```ts
// 创建一个异步可迭代对象, 在数字达到 max 之前每1秒输出当前数字
function createAsyncCounter(max) {
  return {
    async *[Symbol.asyncIterator]() {
      for (let i = 1; i <= max; i++) {
        await waiting(1000); // 模拟异步延迟
        yield i;
      }
    }
  };
}
```
```ts
@Controller('sse')
export class SseController {
  constructor(private readonly AiService: AiService) {}
  // 模拟在无 ai 的情况下发送最基本的 sse 消息
  @Post('test')
  @Sse('test') // 支持通过 POST + SSE 混合方式
  test(@Body() body: any) {
    const subject = new Subject<ResponseMessage>();
    // 每1秒输出1条最多输出3条消息
    const maxCount = 3;
    // 创建虚拟的聊天消息，
    const streamTest = createAsyncCounter(maxCount);
    console.log('body ===>', body);
    async function startStream() {
      for await (const chunk of streamTest) {
        subject.next({
          data: JSON.stringify({ content: '这是测试内容 ==>' + chunk }),
        });
        if (chunk === maxCount) {
          subject.next({ data: '[DONE]' });
          subject.complete();
        }
      }
    }
    startStream().catch(() => {});
    return subject.asObservable();
  }
}
```
# 流程图
1. 建立连接时，创建 subject 监听器，并返回 observable
2. subject 可等待 ai 更新
3. ai 更新数据时，调用 subject.next(msg) 触发 sse 消息传输
4. ai 更新完毕，调用 subject.complete() 结束监听
![流程图](https://github.com/user-attachments/assets/bce86d6b-6539-4572-9f6b-682f33738b57)
