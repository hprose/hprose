# Hprose 3.0 插件设计指南

本文档以 Hprose 3.0 内置实现的插件为例，来讲解如何实现自定义插件。

Hprose 3.0 目前内置有负载均衡，集群容错，熔断降级，流量限速，并发限制，调试日志，单向调用，反向调用和推送插件。下面将会分别讲解这些插件的设计与实现。

## 单向调用

这是最简单的一个插件。可以作为 Hprose 插件的入门实例。

该插件的用途是当客户端发起调用后直接返回，不需要等待结果。

### TypeScript 版本

该插件代码非常简单，这里先以 TypeScript 版本为例来进行讲解：

```ts
import { Context, NextInvokeHandler } from '@hprose/rpc-core';

export class Oneway {
    public static async handler(name: string, args: any[], context: Context, next: NextInvokeHandler): Promise<any> {
        const result = next(name, args, context);
        if (context.oneway) {
            result.catch(() => { });
            return undefined;
        }
        return result;
    }
}
```

上面是该插件的全部代码。下面我们先来分析一下这段代码。

该插件实现了一个调用处理器（`InvokeHandler`），使用 `async` 函数来定义是为了让代码看上去更简洁。

该函数首先调用 `next` 方法，并将返回值保存在常量 `result` 中，在这里 `next` 的返回值跟 `handler` 方法的返回值一样，是一个 `Promise<any>` 类型，也就是说，在执行完这一句后，实际上调用还没有开始，只是把调用放入了请求队列中。

接下来，判断 `context.oneway` 是否为 `true`，如果为 `true`，则对 `result` 做错误忽略的处理，然后不需要等待远程调用执行完毕，直接返回 `undefined` 的结果。否则，返回 `result`。

那么在使用时，该如何将该插件应用于客户端，又该如何设置 `context.oneway` 的值呢？我们来看下面的使用代码：

```ts
import { Client, ClientContext } from '@hprose/rpc-core';
import { Oneway } from '@hprose/rpc-plugin-oneway';
import '@hprose/rpc-node';

async function main() {
    const client = new Client('http://127.0.0.1/');
    client.use(Oneway.handler);
    const context = new ClientContext({ oneway: true });
    await client.invoke('restart', [], context);
}

main();
```

在上面这段代码中，服务地址为 `http://127.0.0.1/`，该服务上有一个远程方法 `restart`，该方法用来重启服务器，不会有返回值，所以我们不需要等待它。

客户端使用 `use` 方法来设置 `Oneway.handler` 插件。

而 `context.oneway` 的值是通过直接创建一个 `context` 对象，并在调用 `invoke` 方法时，通过第三个参数传入该上下文对象即可。

在本例中，远程方法调用之后，`context` 中的值是否改变我们并不关心，在这种情况下，在 TypeScript 代码可以简化为：

```ts
import { Client } from '@hprose/rpc-core';
import { Oneway } from '@hprose/rpc-plugin-oneway';
import '@hprose/rpc-node';

async function main() {
    const client = new Client('http://127.0.0.1/');
    client.use(Oneway.handler);
    await client.invoke('restart', [], { oneway: true });
}

main();
```

不过这种简化写法，在其它语言中不一定都适用。

使用该插件后，如果在远程调用时，没有设置 `context.oneway` 为 `true`，那么远程调用跟没有设置该插件时行为相同，只有将 `context.oneway` 设为 `true` 时，远程调用才会变为单向调用，单向调用不但忽略远程调用的结果，连异常也会一起忽略。

