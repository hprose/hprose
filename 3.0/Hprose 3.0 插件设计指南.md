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

## 调试日志

接下来的我们介绍的第二个插件是调试日志插件，该插件相对第一个复杂一些，但是因为不涉及到什么复杂的算法，还是很容易看懂的。

该插件针对输入输出处理器和调用处理器都有实现。

## TypeScript 版本

```ts
import { ByteStream } from '@hprose/io';
import { Context, NextIOHandler, NextInvokeHandler } from '@hprose/rpc-core';

export class Log {
    private static readonly instance: Log = new Log();
    public constructor(public enabled: boolean = true) {}
    public static ioHandler(request: Uint8Array, context: Context, next: NextIOHandler): Promise<Uint8Array> {
        return Log.instance.ioHandler(request, context, next);
    }
    public static invokeHandler(name: string, args: any[], context: Context, next: NextInvokeHandler): Promise<any> {
        return Log.instance.invokeHandler(name, args, context, next);
    }
    public async ioHandler(request: Uint8Array, context: Context, next: NextIOHandler): Promise<Uint8Array> {
        const enabled = (context.log === undefined) ? this.enabled : context.log;
        if (!enabled) return next(request, context);
        try {
            console.log(ByteStream.toString(request));
        }
        catch(e) {
            console.error(e);
        }
        const response = next(request, context);
        response.then(
            (value) => console.log(ByteStream.toString(value))
        ).catch(
            (reason) => console.error(reason)
        );
        return response;
    }
    public async invokeHandler(name: string, args: any[], context: Context, next: NextInvokeHandler): Promise<any> {
        const enabled = (context.log === undefined) ? this.enabled : context.log;
        if (!enabled) return next(name, args, context);
        let a: string = '';
        try {
            a = JSON.stringify((args.length > 0 && context.method && context.method.passContext && !context.method.missing) ? args.slice(0, args.length - 1) : args);
        }
        catch(e) {
            console.error(e);
        }
        const result = next(name, args, context);
        result.then(
            (value) => console.log(`${name}(${a.substring(1, a.length - 1)}) = ${JSON.stringify(value)}`)
        ).catch(
            (reason) => console.error(reason)
        );
        return result;
    }
}
```

上面是 TypeScript 版本的全部代码。

该类的 `ioHandler` 和 `invokeHandler` 各有两个，一个是静态方法，一个是实例方法。静态方法是默认开启打印日志功能的。如果需要默认关闭打印日志功能，则需要自己创建一个 `Log` 对象实例。

该方法在客户端，可以通过使用跟第一个插件同样的方式来设置 `context.log` 为 `true` 或 `false`，以决定某个方法是否要打印调试日志。

对于 `ioHandler` 来说，请求和响应结果都是 `Uint8Array` 这种二进制数据类型，因此需要转换为 `String` 类型再输出，否则可读性会很差。在二进制数据转换为字符串时，可能会发生异常，为了避免这些异常中断正常的调用，因此都做了捕获异常的处理。在上面的代码中，在调用 `next` 之前使用的是同步的 `try-catch` 异常捕获方式，在调用 `next` 之后使用的是异步的 `then-catch` 异常捕获方式。其实在调用之后，也可以使用同步的 `try-catch` + `await` 方式来捕获异常，只不过使用 `then-catch` 方式会似的代码更简洁。

`invokeHandler` 是用来把调用的方法名、参数和结果按照 `name(arg1, arg2, ...argn) = result` 的形式打印出来。方法名直接作为字符串打印就可以了，结果直接按照 json 序列化打印就可以了。这两个都很简单。对于参数来说稍微复杂一些。因为在服务端，参数中可能会包含 `context` 对象，但是在打印调试信息时，该参数不应该被显示出来，因此需要判断它是否存在，如果存在需要将它从打印的参数列表中移除。参数和结果转换为 json 序列化形式时，也可能会出错，因此也需要进行异常捕获，这部分在处理上跟 `ioHandler` 类似，这里不在重复。

将二进制数据转换为字符串，将参数和结果进行 json 序列化，以及在判断参数中是否包含有 `context` 参数时，不同的语言的具体操作可能差别较大。但在总体的处理流程上是大同小异的，因此这里就不再对其它语言的实现进行这方面的分析了。

该插件使用非常简单，下面是一个例子：

```ts
import * as http from 'http';
import { Client, Service } from '@hprose/rpc-core';
import { Log } from '@hprose/rpc-plugin-log';
import '@hprose/rpc-node';

function hello(name: string): string {
    return 'hello ' + name;
}

async function main() {
    const service = new Service();
    service.use(Log.ioHandler);
    service.addFunction(hello);
    const server = http.createServer();
    service.bind(server);
    server.listen(8000);

    const client = new Client('http://127.0.0.1:8000/');
    client.use(Log.invokeHandler);
    const proxy = await client.useServiceAsync();
    await proxy.hello('world');

    server.close();
}

main();
```

该程序运行结果为：

```
Cu~z
Ra2{u~s5"hello"}z
~() = ["~","hello"]
Cs5"hello"a1{s5"world"}z
Rs11"hello world"z
hello("world") = "hello world"
```

其中：

```
Cu~z
Ra2{u~s5"hello"}z

Cs5"hello"a1{s5"world"}z
Rs11"hello world"z

```

这四行是服务端的 `Log.ioHandler` 插件打印的，这些就是 hprose 协议通讯传输的内容。第一行 `Cu~z` 表示获取服务方法名列表，第二行 `Ra2{u~s5"hello"}z` 是服务器端返回的结果。`Cs5"hello"a1{s5"world"}z` 是 `hello('world')` 的调用请求，`Rs11"hello world"z` 是该请求的结果。

```

~() = ["~","hello"]


hello("world") = "hello world"
```

这两行是客户端的 `Log.invokeHandler` 插件打印的。从这两行我们可以更直观的看到这里的两个调用。第一个调用名为 `~`，这是 hprose 服务中的一个特殊服务，用来返回服务方法名列表，从返回的结果中，我们也可以看到，服务方法中也包含有这个特殊的服务方法名。第二个调用就是对 `hello` 方法的调用，因为很直观，这里就不做解释了。

虽然上面的例子中，我们将 `Log.ioHandler` 插件用于服务端，将 `Log.invokeHandler` 插件用于客户端。实际上这两个插件，都是既可以用于服务端也可以用于客户端的。如果把两个插件在客户端和服务端调换，或者都放在服务端，或者都放在客户端时，日志的打印顺序会有所不同。这一点用户可以亲自尝试一下。

## C# 版本

C# 版本的代码较长，这里就不全部贴出来了，下面只贴出一部分进行说明：

```csharp
public class Log {
    private static readonly Log instance = new Log();
    public bool Enabled { get; set; }
    public Log(bool enabled = true) {
        Enabled = enabled;
    }
    public static Task<Stream> IOHandler(Stream request, Context context, NextIOHandler next) {
        return LogExtensions.IOHandler(instance, request, context, next);
    }
    public static Task<object> InvokeHandler(string name, object[] args, Context context, NextInvokeHandler next) {
        return LogExtensions.InvokeHandler(instance, name, args, context, next);
    }
}
```

这里，我们定义了一个 `Log` 类，该类有一个静态字段 `instance` 是该类自己的实例对象。然后定义了两个静态方法 `IOHandler` 和 `InvokeHandler`。但是这两个方法的实现有点特别，它是调用了 `LogExtensions` 类上的对应的两个静态方法 `IOHandler` 和 `InvokeHandler` 来实现的。

`LogExtenstions` 是一个特殊的静态类，它上面的两个静态方法 `IOHandler` 和 `InvokeHandler` 签名是这样的：

```csharp
public static class LogExtensions {
    public static async Task<Stream> IOHandler(this Log log, Stream request, Context context, NextIOHandler next);
    public static async Task<object> InvokeHandler(this Log log, string name, object[] args, Context context, NextInvokeHandler next);
}
```

这两个方法第一个参数都是 `this Log log`，也就是说，这两个方法其实都是 `Log` 类的扩展方法。

该类之所以这样写是为了让 `Log` 类上可以同时存在同名的静态方法和实例方法，如果直接在 `Log` 类上定义 `IOHandler` 和 `InvokeHandler` 的静态方法和实例方法的话，那么编译时会报告错误，不能正常编译通过。为了实现跟 TypeScript 版本一样的效果，这里使用了扩展方法这个技巧。

至于其它部分代码功能上跟 TypeScript 版本相同，就不再重复叙述了。

C# 版本在打印日志时，使用的是 `Trace` 方式，使用该方法主要考虑了两个方面，一方面是 `Trace` 方式是 .NET 自带的方式，不依赖任何第三方库。另一方面是 `Trace` 的灵活性很高，它可以配置自定义日志方式。你甚至可以通过定制 `TraceListener` 让 `Trace` 使用 `log4net` 来做日志。

下面我们来看一下 C# 下使用该插件的示例：

```csharp
using Hprose.RPC;
using Hprose.RPC.Plugins.Log;
using System;
using System.Diagnostics;
using System.Net;
using System.Threading.Tasks;

class MyService {
    public int Sum(int x, int y) {
        return x + y;
    }
    public string Hello(string name, ServiceContext context) {
        var endPoint = context.RemoteEndPoint as IPEndPoint;
        return "Hello " + name + " from " + endPoint.Address + ":" + endPoint.Port;
    }
}

public interface IMyService {
    [Log(false)]
    Task<int> Sum(int x, int y);
    Task<string> Hello(string name);
}

class Program {
    static async Task RunClient() {
        var client = new Client("http://127.0.0.1:8080/");
        client.Use(Log.InvokeHandler);
        var proxy = client.UseService<IMyService>();
        Console.WriteLine(await proxy.Hello("world"));
        Console.WriteLine(await proxy.Sum(1, 2));
    }
    static void Main(string[] args) {
        Trace.Listeners.Add(new TextWriterTraceListener(Console.Out));
        HttpListener server = new HttpListener();
        server.Prefixes.Add("http://127.0.0.1:8080/");
        server.Start();
        var service = new Service();
        service.Use(Log.IOHandler)
            .AddInstanceMethods(new MyService())
            .Bind(server);
        RunClient().Wait();
        server.Stop();
    }
}
```

`MyService` 是我们的远程服务实现。`IMyService` 是我们用来调用远程服务的接口。如果仔细观察你会发现，`MyService` 并没有实现 `IMyService` 接口，甚至里面的方法签名都不一致，在 `MyService` 中方法都是同步的，而 `IMyService` 中的方法返回的是 `Task<T>` 的异步结果。这是 hprose 的一个特征，服务不必实现任何接口。客户端接口仅用于调用，不必与服务端定义一致。通过这种方式，不但实现了客户端与服务端的解耦，而且增加了服务编写的灵活性，也方便了跨语言调用。

另外，我们在 `IMyService` 中，特别对 `Sum` 方法标注了 `[Log(false)]`，因此客户端对该方法不进行日志打印。

为了便于查看日志，我们在 `Main` 函数开头加了一句：

```csharp
Trace.Listeners.Add(new TextWriterTraceListener(Console.Out));
```

目的是让日志直接打印到控制台上。否则该程序被编译后，单独执行时，我们会看不到日志输出，只能在 Virtual Stuido 中调试时，才能从调试信息窗口中看到日志输出。

```csharp
        HttpListener server = new HttpListener();
        server.Prefixes.Add("http://127.0.0.1:8080/");
        server.Start();
```

这三句是用来单独启动一个 Http 服务器。

```csharp
        var service = new Service();
        service.Use(Log.IOHandler)
            .AddInstanceMethods(new MyService())
            .Bind(server);
```

这两句是创建一个 Hprose 服务，设置 `Log.IOHandler` 插件，创建一个服务实例并发布，然后将服务绑定到已经启动的 Http 服务器上。

`RunClient` 方法是创建并运行客户端。因为代码很简单直观，就不多做解释了。

下面看一下该程序运行之后的输出结果：

```
Log Information: 0 : Cs5"Hello"a1{s5"world"}z
Log Information: 0 : Rs32"Hello world from 127.0.0.1:50731"z
Log Information: 0 : Hello("world") = "Hello world from 127.0.0.1:50731"
Hello world from 127.0.0.1:50731
Log Information: 0 : Cs3"Sum"a2{12}z
Log Information: 0 : R3z
3
```

其中：

```



Hello world from 127.0.0.1


3
```

这两行是上面的 `RunClient` 中的两句 `Console.WriteLine` 输出的。

```
Log Information: 0 : Cs5"Hello"a1{s5"world"}z
Log Information: 0 : Rs32"Hello world from 127.0.0.1:50731"z


Log Information: 0 : Cs3"Sum"a2{12}z
Log Information: 0 : R3z

```

这四行是服务端的 `Log.IOHandler` 插件输出的。

最后剩下的这一句：

```
Log Information: 0 : Hello("world") = "Hello world from 127.0.0.1:50731"
```

是由客户端的 `Log.InvokeHandler` 插件输出的。因为接口 `IMyService` 中的标注 `[Log(false)]` 只对客户端起作用，因此我们从上面的日志中可以看出，对于 `Sum` 方法的调用，客户端没有打印日志，而服务端则打印了关于 `Sum` 方法调用的日志。如果将服务端的 `Log.IOHandler` 插件设置去掉，设置并将它设置到客户端上，那么原来服务器输出的那四行，在客户端输出时，也会变为只有 `Hello` 调用的两行日志，而关于 `Sum` 方法调用的日志不会再输出。

在上面服务定义中，需要特别注意一下 `Hello` 方法：

```csharp
    public string Hello(string name, ServiceContext context) {
        var endPoint = context.RemoteEndPoint as IPEndPoint;
        return "Hello " + name + " from " + endPoint.Address + ":" + endPoint.Port;
    }
```

它的参数列表中，声明了一个 `ServiceContext context` 参数，但是客户端调用时，并没有传入它，因为这个参数是服务器端自动传入的。我们可以通过它得到许多有用的信息，比如例子中，通过 `context.RemoteEndPoint` 我们得到了客户端的地址。

关于调试日志插件的讲解就介绍这么多，接下来我们介绍负载均衡插件。