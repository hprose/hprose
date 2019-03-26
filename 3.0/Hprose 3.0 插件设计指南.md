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

## C# 版本

接下来我们来看看 C# 版本如何实现：

```csharp
using System;
using System.Threading.Tasks;

namespace Hprose.RPC.Plugins.Oneway {
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = false)]
    public class OnewayAttribute : ContextAttribute {
        public OnewayAttribute() : base("Oneway", true) { }
    }

    public static class Oneway {
        public static async Task<object> Handler(string name, object[] args, Context context, NextInvokeHandler next) {
            var result = next(name, args, context);
            if (context.Contains("Oneway")) {
                return null;
            }
            return await result.ConfigureAwait(false);
        }
    }
}
```

这段代码跟 TypeScript 版本相比，多了一个 `OnewayAttribute`，该属性用于在客户端定义调用接口时，标注接口方法为单向调用模式。该属性并不是必须的，`OnewayAttribute` 属性继承自 `ContextAttribute`，它的效果跟直接使用 `[Context("Oneway", true)]` 对接口方法进行标注的效果是一样的，但是直接用 `[Oneway]` 来做标注，会让代码更简洁清晰。

 C# 版本的 `Oneway` 静态类跟 TypeScript 版本 `Oneway` 类代码几乎是一样的。只不过 C# 版本不需要对 `result` 做错误忽略的处理，因为只要不对该结果执行 `Wait` 或 `await` 操作，错误就会自动被忽略。这跟 .NET 的 `Task` 和 JavaScript/TypeScript 的 `Promise` 的内部处理机制有关，这里就不做详细讨论了。

下面我们来看一下在 .NET 中如何使用：

```cs
using Hprose.RPC;
using Hprose.RPC.Plugins.Oneway;

class Program {
    static void Main(string[] args) {
        var client = new Client("http://127.0.0.1/");
        client.Use(Oneway.Handler);
        client.Invoke("restart", null, new ClientContext() {
            ["Oneway"] = true
        });
    }
}
```

这段代码跟上面的 TypeScript 版本的代码几乎一模一样，只是这里用的 `Invoke` 方法是使用的同步版本，使用异步版本效果也是一样的。在这里创建 `ClientContext` 对象时，利用了 C# 新版本的对象属性初始化的功能，这让代码变的更加简洁。

下面我们来看一下使用代理对象如何使用该插件：

```cs
using System.Threading.Tasks;
using Hprose.RPC;
using Hprose.RPC.Plugins.Oneway;

public interface IRestart {
    [Oneway]
    Task Restart();
}
class Program {
    static async Task Example() {
        var client = new Client("http://127.0.0.1/");
        client.Use(Oneway.Handler);
        var proxy = client.UseService<IRestart>();
        await proxy.Restart();
    }
    static void Main(string[] args) {
        Example().Wait();
    }
}
```

虽然代码上，比直接使用 `Invoke` 要多一些，但是代码结构还是很清晰的。`IRestart` 接口中，定义了一个 `Restart` 方法，返回值为 `Task` 类型，表示该方法时异步的，该方法上面加了 `[Oneway]` 属性标注，表示该方法是单向调用。后面的 `Example` 方法中，使用 `UseService` 生成代理对象，然后通过代理对象就可以直接调用 `Restart` 方法了，该方法在使用时，形式上跟使用本地方法没有什么区别。因为控制台程序的入口函数 `Main` 不能定义为 `async` 方法，所以单独定义了一个 `async` 方法 `Example` 并在 `Main` 入口函数中同步调用它。

上面的代码，不论服务器有没有打开，代码执行都不会出错。这是因为 `Oneway` 模式把异常忽略了。如果去掉 `[Oneway]` 标注，再执行上面的代码，就会看到一串异常信息了。

其它语言的实现和使用方式跟这两个都是类似的，这里就不再举例说明了。

在下面介绍其它插件的实现和用法时，我们仍然使用这两种语言来举例说明，但是在代码较长的情况下，我们可能会省略掉一些无关紧要代码，只保留核心算法的代码。