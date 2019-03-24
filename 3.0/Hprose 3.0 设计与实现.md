# Hprose 3.0 设计与实现

本文档重点讲述 Hprose 3.0 的 RPC 核心构架的设计，该设计是通用的，与具体的序列化格式和 RPC 协议编码方式无关。其次，会讲解在具体实现时可能遇到的问题以及解决方法。

## 设计目标

* 跨语言跨平台
* 编码层可替换
* 传输层可替换
* 具有扩展机制

## 构架图

```
            +------+                         +--------+          
            |Invoke|                         |Execute |          
            +------+                         +--------+          
             |    ^                            |    ^            
             v    |                            v    |            
        +--------------+                  +--------------+       
        |InvokeHandlers|                  |InvokeHandlers|       
        +--------------+                  +--------------+       
          |          ^                      |          ^         
          v          |                      v          |         
       +------+  +------+                +------+  +------+      
       |Encode|  |Decode|                |Encode|  |Decode|      
       +------+  +------+                +------+  +------+      
            |      ^                          |      ^           
            v      |                          v      |           
          +----------+                      +----------+         
          |IOHandlers|                      |IOHandlers|         
          +----------+                      +----------+         
            |      ^                          |      ^           
            v      |                          v      |           
          +----------+                       +--------+          
          |Transports|---------------------->|Handlers|          
          +----------+                       +--------+          
               ^                                 |              
               |                                 |              
               +---------------------------------+              
                                                                 
  ____ _    _ ____ _  _ ___     ____ ____ ____ _  _ _ ____ ____  
  |    |    | |___ |\ |  |      [__  |___ |__/ |  | | |    |___  
  |___ |___ | |___ | \|  |      ___] |___ |  \  \/  | |___ |___  
                                                                 
```

## 构架简介

通过上面的构架图，我们可以大致了解 Hprose RPC 的工作流程。上图左面部分是客户端，右面部分是服务端。客户端和服务端的执行流程是对称且相反的两个过程。

客户端首先发起调用，原始的调用请求会通过调用处理器进行处理，之后经过处理的调用请求被编码为序列化后的 RPC 请求，编码之后的请求再通过输入输出处理器进行处理，最后通过传输层发送到服务端。

服务端的处理器将接收到的客户端发来的序列化后的 RPC 请求转发给输入输出处理器进行处理，然后处理过后的序列化 RPC 请求被解码为原始调用请求再转发给调用处理器，调用处理器处理后，请求被执行，并将执行结果按原路层层返回给客户端。

在该调用过程中，客户端和服务端的各自存在一个上下文对象（`Context`）。也就是说，除了请求和结果会在以上过程中被传递以外，上下文对象也会在以上过程中被传递。但是上下文对象并不会从客户端传递给服务端，也不会从服务端传递给客户端，上下文对象在客户端和服务端是各自独立的。

客户端上下文对象（`ClientContext`）和服务端的上下文对象（`ServiceContext`）继承自同一个上下文对象（`Context`）的结构。通过这种抽象，就可以将调用处理器（`InvokeHandler`）和输入输出处理器（`IOHandler`）设计为在客户端和服务端通用的形式了。

调用处理器（`InvokeHandler`）和输入输出处理器（`IOHandler`）用来实现可扩展的插件机制。

它们具有相同的工作模式，只是接口和所处理的数据有所不同。

客户端和服务的各自拥有独立的编解码器（`Codec`），默认的编解码器使用 Hprose 序列化和 Hprose RPC 协议进行编解码处理。

编解码器是可替换的，通过替换编解码器，Hprose 客户端或服务端可以变身为其它 RPC 的客户端或服务端。例如，如果实现了 JSONRPC 编解码器，Hprose 客户端和服务端就完全可以作为 JSONRPC 的客户端和服务端来使用，并且可以与其它方式实现的 JSONRPC 客户端或服务端进行互通。

在上面的构架图中，客户端的传输层被抽象为一个 `Transport` 对象，而服务的传输层处理器被抽象为一个 `Handler` 对象。可以为不同的传输协议提供不同的实现，将实现的传输层对象在客户端和服务端的进行注册后，就可以使用该传输层协议进行通讯了。客户端和服务端都可以注册多种传输协议，并可以增加替换，还能混合使用。

## 上下文对象

上下文对象（`Context`）是一个类似于 `Map<String, Object>` 的结构，用户可以用字符串做 `Key`，放入任何类型的数据。因为该对象不需要在客户端和服务端进行传输，所以它当中存放的数据，不必是可序列化类型。

上下文对象具有判断某个 `Key` 是否存在的功能，但是在不同语言实现时，可以根据各自语言的特性在语法或方法命名上有所不同。

上下文对象还应具有一个 `clone` 方法，该方法名在大小写上，可以根据不同语言有所区别。

在调用中，每个调用都具有自己独立的上下文对象，

客户端上下文对象（`ClientContext`）继承自上下文对象（`Context`），它在客户端进行调用前由用户创建，或者在客户端进行调用时被自动创建，也就是从客户端调用的入口处被创建。它增加了一些专属于客户端的属性，例如：

* 客户端对象（`client`)
* 服务地址（`uri`）
* 调用返回值类型（`returnType`）
* 调用超时时长（`timeout`）
* 请求头部（`requestHeaders`）
* 响应头部（`responseHeaders`）

每种语言在实现时，这些属性在命名上可能会有大小写上的区别。

在插件处理器中，可以通过调用原始上下文对象上的 `clone` 方法创建多个上下文对象副本，然后通过修改副本上下文对象上的服务地址（`uri`）属性，将一个调用变为对多个不同服务的调用。

服务端上下文对象（`ServiceContext`）也继承自上下文对象（`Context`），它在服务端由 `Handler` 对象创建，也就是在服务端的入口处被创建。它增加了一些专属于服务端的属性，例如：

* 服务对象（`service`）
* 服务处理器对象（`handler`)
* 服务方法（`method`）
* 客户端地址（`address`）
* 请求头部（`requestHeaders`）
* 响应头部（`responseHeaders`）

每种语言在实现时，这些属性在命名上可能会有大小写上的区别。其中客户端地址属性在不同的语言中因为类型差别较大，所以命名上没有统一要求。

从上面的描述中我们可以看到，不论客户端还是服务端的上下文对象中，都包含有请求头部（`requestHeaders`）和响应头部（`responseHeaders`）这两个属性。这两个属性比较特殊，它们也是一个类似于 `Map<String, Object>` 的类型，但是客户端的请求头部（`requestHeaders`）会跟随请求一起发送给服务端，而服务端的响应头部（`responseHeaders`）会跟随响应一起发送给客户端，因此，这两个属性中添加的数据必须是可序列化的类型。

用户也可以自定义上下文对象类型，但是必须继承自客户端上下文对象（`ClientContext`）或服务端上下文对象（`ServiceContext`）。自定义上下文对象类型可以在插件处理器中对现有的上下文对象进行包装然后传递到下一层去。但是通常不需要这样做，直接使用 `key-value` 方式在现有上下文对象中存取用户数据是更通用的做法。

## 服务方法管理器

在上面介绍上下文对象时，我们提到在服务端上下文对象（`ServiceContext`）包含有一个服务方法（`method`）属性。该属性对应服务端发布的服务方法，该属性的类型在不同语言中的定义会有所不同，它其中包含了关于服务方法的一些必要信息，比如发布名称（区别于方法定义的名称），方法本体（可以反射调用或直接调用的方法对象），方法所属对象，参数类型，是否是通用方法，参数中是否包含有上下文对象等等。

服务方法（`Method`）由服务方法管理器（`MethodManager`）来统一管理，服务方法管理器包含以下一些必须的方法：

* `getNames`
* `add`
* `get`
* `remove`
* `addMissingMethod`

每种语言在实现时，这些方法在命名上可能会有大小写上的区别。

其中 `getNames` 方法返回所有已发布的方法名列表，该方法本身也会作为一个特殊的服务方法被发布，为了避免跟用户发布的方法有命名冲突，在发布时，该方法是以特殊字符 `~` 来作为发布名称来发布的。

`add` 方法用来添加发布的服务方法。该方法在不同的语言中，可以根据具体情况提供多个重载或不同命名的多个实现，例如：`addMethod`，`addMethods`，`addFunction`，`addFunctions`，`addStaticMethods`，`addInstanceMethods` 等等。

`get` 方法用来通过发布名称获取已发布的服务方法对象。在某些语言中（比如 C#），允许发布相同名称但参数个数不同的重载方法，因此，在这些语言中，`get` 方法本身会有 2 个参数（发布名称，参数个数）。

`remove` 方法用来删除已发布的服务方法。该方法的参数与 `get` 方法相同。


`addMissingMethod` 方法用来添加一个通用的服务方法，当客户端调用的服务方法不存在时，该服务方法才会被调用。该方法的名称在服务方法管理器中，以一个特殊字符 `*` 来表示，客户端不应直接调用此方法。

当调用 `get` 方法时，如果查找的具体服务方法不存在，但是通用服务方法存在的话，则会返回通用服务方法，否则返回 `null`。

用户通常不需要直接使用服务方法管理器（`MethodManager`），因为在服务端 `Service` 类型中它是作为一个内部属性出现的，它所包含的方法也是由 `Service` 类型对象所暴露的。

但是当开发插件时，如果需要用到服务发布管理的功能，用户可以在插件中使用服务方法管理器（`MethodManager`），例如在反向调用插件的实现中，就使用了服务方法管理器（`MethodManager`）。

## 插件处理器

在 hprose 2.0 中，插件处理器被称为“中间件”。在 hprose 3.0 中，为了便于跟其它框架中的“中间件”进行区分，我们将它改称为“插件处理器”。

因为插件处理器不但可以实现之前版本中调用拦截器和过滤器的功能，而且还能实现它们完成不了功能。因此在 hprose 3.0 的设计和实现中，取消了调用拦截器和过滤器，只保留了插件处理器。

插件处理器分为两种，一种是调用处理器（`InvokeHandler`），一种是输入输出处理器（`IOHandler`）。

不论客户端还是服务端，这两种类型的处理器都可以添加任意多个并可进行自由组合。

在客户端，调用处理器先被执行，在服务端，输入输出处理器先被执行。

每种类型的处理器都按照添加顺序执行，假设添加的调用处理器分别为：invokeHandler1, invokeHandler2 ... invokeHandlerN，那么执行顺序就是 invokeHandler1, invokeHandler2 ... invokeHandlerN。

### 调用处理器

调用处理器的形式在不同的语言中有所不同，但参数都是一样的。

例如在 C# 中，它的形式如下：

```csharp
delegate Task<object> NextInvokeHandler(string name, object[] args, Context context);
delegate Task<object> InvokeHandler(string name, object[] args, Context context, NextInvokeHandler next)
```

在 TypeScript 中，它的形式如下：

```ts
type NextInvokeHandler = (name: string, args: any[], context: Context) => Promise<any>;
type InvokeHandler = (name: string, args: any[], context: Context, next: NextInvokeHandler) => Promise<any>;
```

在 Dart 中，它的形式如下：

```dart
typedef Future NextInvokeHandler(String name, List args, Context context);
typedef Future InvokeHandler(String name, List args, Context context, NextInvokeHandler next);
```

要定义一个调用处理器，只需要实现 `InvokeHandler` 即可，你可以将它实现为一个函数，方法或者匿名函数，例如：

```csharp
InvokeHandler myInvokeHandler = (name, args, context, next) => {
    ...
    var result = next(name, args, context);
    ...
    return result;
}
```

`name` 是服务方法的发布名称。

`args` 是调用参数。

`context` 是调用的上下文对象，在客户端你可以将它转换为一个 `ClientContext` 类型的对象，在服务端你可以将它转换为一个 `ServiceContext` 类型的对象。

`next` 表示下一个调用处理器，它是由调用管理器（`InvokeManager`）自动生成的。在调用处理器中，通过调用 `next` 将各个调用处理器串联起来。

在调用 `next` 之前的操作在调用发生前执行，在调用 `next` 之后的操作在调用发生后执行，如果你不想修改返回结果，你应该将 `next` 的返回值作为该调用处理器的返回值返回。

如果你在调用处理器中跳过对 `next` 的调用，则在该调用处理器之后添加的调用处理器以及后面的步骤都会跳过执行。

如果在调用处理器中发生异常，异常信息会作为结果返回给上一层调用处理器。

### 调用管理器

调用管理器（`InvokeManager`）用来管理调用处理器（`InvokeHandler`）的添加和删除。它主要包含两个方法：

* `use`
* `unuse`

每种语言在实现时，这两个方法在命名上可能会有大小写上的区别。有些语言中，`use` 可能会是一个关键字而不能作为方法名使用，这种情况下，这两个方法可以被命名为：

* `addHandler`
* `removeHandler`

调用管理器中还包含一个 `handler` 属性，用来返回第一个调用处理器。

用户通常不需要直接使用调用管理器（`InvokeManager`），因为在客户端 `Client` 和服务端 `Service` 类型中它是作为一个内部属性出现的，它所包含的方法也是由 `Client` 和 `Service` 类型对象所暴露的。

但是如果有必要，用户可以在插件中使用调用管理器（`InvokeManager`），例如在反向调用插件的实现中，就使用了调用管理器（`InvokeManager`）。

### 输入输出处理器

输入输出处理器的形式在不同的语言中有所不同，但参数都是一样的。

例如在 C# 中，它的形式如下：

```csharp
delegate Task<Stream> NextIOHandler(Stream request, Context context);
delegate Task<Stream> IOHandler(Stream request, Context context, NextIOHandler next);
```

在 TypeScript 中，它的形式如下：

```ts
type NextIOHandler = (request: Uint8Array, context: Context) => Promise<Uint8Array>;
type IOHandler = (request: Uint8Array, context: Context, next: NextIOHandler) => Promise<Uint8Array>;
```

在 Dart 中，它的形式如下：

```dart
typedef Future<Uint8List> NextIOHandler(Uint8List request, Context context);
typedef Future<Uint8List> IOHandler(Uint8List request, Context context, NextIOHandler next);
```

要定义一个输入输出处理器，只需要实现 `IOHandler` 即可，你可以将它实现为一个函数，方法或者匿名函数，例如：

```csharp
IOHandler myIOHandler = (request, context, next) => {
    ...
    var response = next(request, context);
    ...
    return response;
}
```

`request` 是序列化后的请求数据。

`context` 是调用的上下文对象，在客户端你可以将它转换为一个 `ClientContext` 类型的对象，在服务端你可以将它转换为一个 `ServiceContext` 类型的对象。

`next` 表示下一个输入输出处理器，它是由输入输出管理器（`IOManager`）自动生成的。在输入输出处理器中，通过调用 `next` 将各个输入输出处理器串联起来。

在调用 `next` 之前的操作在调用发生前执行，在调用 `next` 之后的操作在调用发生后执行，如果你不想修改返回结果，你应该将 `next` 的返回值作为该输入输出处理器的返回值返回。

如果你在输入输出处理器中跳过对 `next` 的调用，则在该输入输出处理器之后添加的输入输出处理器以及后面的步骤都会跳过执行。

如果在输入输出处理器中发生异常，异常信息会作为结果返回给上一层输入输出处理器。

### 输入输出管理器

输入输出管理器（`IOManager`）用来管理输入输出处理器（`IOHandler`）的添加和删除。它主要包含两个方法：

* `use`
* `unuse`

每种语言在实现时，这两个方法在命名上可能会有大小写上的区别。有些语言中，`use` 可能会是一个关键字而不能作为方法名使用，这种情况下，这两个方法可以被命名为：

* `addHandler`
* `removeHandler`

输入输出管理器中还包含一个 `handler` 属性，用来返回第一个输入输出处理器。

用户通常不需要直接使用输入输出管理器（`IOManager`），因为在客户端 `Client` 和服务端 `Service` 类型中它是作为一个内部属性出现的，它所包含的方法也是由 `Client` 和 `Service` 类型对象所暴露的。

## 编解码器

服务端和客户端拥有各自的编解码器接口定义。虽然在形式上，不同的语言有所不同，但参数都是一样的。

例如在 C# 中，接口定义为：

```csharp
public interface IServiceCodec {
    Stream Encode(object result, ServiceContext context);
    Task<(string, object[])> Decode(Stream request, ServiceContext context);
}

public interface IClientCodec {
    Stream Encode(string name, object[] args, ClientContext context);
    Task<object> Decode(Stream response, ClientContext context);
}
```

在 TypeScript，接口定义为：

```ts
interface ServiceCodec {
    encode(result: any, context: ServiceContext): Uint8Array;
    decode(request: Uint8Array, context: ServiceContext): [string, any[]];
}

interface ClientCodec {
    encode(name: string, args: any[], context: ClientContext): Uint8Array;
    decode(response: Uint8Array, context: ClientContext): any;
}
```

在 Dart 中，接口定义为：

```dart
class RequestInfo {
  final String name;
  final List args;
  RequestInfo(this.name, this.args);
}

abstract class ServiceCodec {
  Uint8List encode(result, ServiceContext context);
  RequestInfo decode(Uint8List request, ServiceContext context);
}

abstract class ClientCodec {
  Uint8List encode(String name, List args, ClientContext context);
  dynamic decode(Uint8List response, ClientContext context);
}
```

Hprose 3.0 中内置的默认编解码器是使用 hprose 序列化和 hprose RPC 编码协议实现的。另外，还提供了 JSONRPC 2.0 编解码器。它们都是上面这两个接口的具体实现。用户可以通过自定义编解码器来支持更多的 RPC 协议。

客户端和服务端各有一个 `codec` 属性，该属性的默认值为 hprose 编解码器。通过给该属性赋值为其它编解码器的实例对象，可以将客户端或服务端变为其它类型的 RPC 客户端或服务端。

## 传输接口

在客户端，传输层被设计为一个 `Transport` 接口，它只有两个方法：`transport` 和 `abort`。该接口在不语言中虽然定义有所不同，但在形式和参数上大致是一致的。

例如在 C# 中，该接口定义为：

```csharp
public interface ITransport {
    Task<Stream> Transport(Stream request, Context context);
    Task Abort();
}
```

在 TypeScript 中，该接口定义为：

```ts
interface Transport {
    transport(request: Uint8Array, context: Context): Promise<Uint8Array>;
    abort(): Promise<void>;
}

interface TransportConstructor {
    readonly schemes: string[];
    new(): Transport
}
```

在 Dart 中，该接口定义为：

```dart
abstract class Transport {
  Future<Uint8List> transport(Uint8List request, Context context);
  Future<void> abort();
}

abstract class TransportCreator<T extends Transport> {
  List<String> schemes;
  T create();
}
```

该接口的实现还应有一个名为 `schemes` 的静态属性和一个无参构造函数。但大部分语言的接口没有办法对静态属性和构造函数做出约定，因此在接口定义中，体现不出这一点来。

`schemes` 属性用来描述该接口的实现支持那些传输协议，例如：`http`，`https`，`tcp`，`tcp4`，`tcp6`，`tls`，`udp`，`websocket`，`unix` 等。因为一个传输接口实现可能支持多种传输协议，因此该属性为一个字符串数组或字符串列表。

`transport` 方法用于实际传输的实现，`request` 是请求数据，返回值是响应数据。与请求相关的其它数据，比如服务地址，超时间隔等则在 `context` 参数中。

`abort` 方法用于中断与服务器的连接或请求。该方法在某些情况下不一定有实际实现的代码。

客户端有一个 `register` 静态方法，用来注册传输接口实现。

例如在 C# 中，该方法定义为：

```csharp
public static void Register<T>(string name) where T : ITransport, new();
```

在 TypeScript 中，该方法被定义为：

```ts
public static register(name: string, ctor: TransportConstructor): void;
```

在 Dart 中，该方法被定义为：

```dart
static void register<T extends Transport>(String name, TransportCreator<T> creator);
```

虽然在不同语言中，形式和参数上有些差异，但是实现的功能是一样的。

但用户几乎不需要用到 `register` 静态方法，因为默认提供的传输接口的实现都已经注册过了。除非用户实现自己的传输接口时，才需要用该方法来进行注册。

## 处理器接口

在服务端，处理器接口用来把服务跟具体的服务器绑定在一起。在这里，服务是指独立于具体传输协议的 RPC 服务，它是由 `Service` 类型实现的。而具体的服务器是指 `http`，`tcp`，`udp`，`websocket` 等服务器，这些服务器是由语言本身所提供的标准库、第三方类库或框架实现的。

在 C# 中，该接口定义如下：

```csharp
public interface IHandler<T> {
    Task Bind(T server);
}
```

在 TypeScript 中，该接口定义为：

```ts
interface Handler {
    bind(server: any): void;
}

interface HandlerConstructor {
    new(service: Service): Handler;
}
```

在 Dart 中，该接口定义为：

```dart
abstract class Handler<T> {
  void bind(T server);
}

abstract class HandlerCreator<T extends Handler> {
  List<String> serverTypes;
  T create(Service service);
}
```

该接口的实现应包含一个有参构造函数，参数为 `Service` 类型。即通过构造函数，将该处理器与服务绑定，然后通过 `bind` 方法，将该处理器与具体的服务器绑定。

与客户端类似，服务端也包含一个 `register` 静态方法，用来将注册处理器接口的实现。

例如在 C# 中，该方法定义为：

```csharp
public static void Register<T, S>(string name) where T : IHandler<S>;
```

在 TypeScript 中，该方法定义为：

```ts
public static register(name: string, ctor: HandlerConstructor, serverTypes: Function[]): void;
```

在 Dart 中，该方法定义为：

```dart
static void register<T extends Handler>(String name, HandlerCreator<T> creator);
```

虽然在不同语言中，形式和参数上有些差异，但是实现的功能是一样的。

一般来说，语言标准库中提供的服务器在默认实现中都已经自动注册了。扩展库，第三方类库或框架中提供的服务器所对应的处理器则需要用户自己注册。如果某个第三方类库或框架所提供的服务器没有默认对应的处理器实现，用户也可以自行实现和注册。

另外，在 `Handler` 接口被实现时，除了接口中声明的 `bind` 方法外，一般还会实现一个 `handler` 方法，但是该方法的具体参数类型和个数会因为服务器不同而不同。因此该接口中并没有声明该方法。

`handler` 方法是用来处理实际连接和请求的。对于某些服务器或服务器框架来说，该方法可以直接作为回调函数被传入。因此，它是非常特别又非常重要的。

## 服务端

服务端统一为一个 `Service` 类型。下面来分别介绍它所包含的属性和方法。

### 属性

* `timeout`
* `codec`
* `maxRequestLength`
* `names`

`timeout` 属性是用来限制服务代码执行时间的，如果服务执行超时，则返回超时异常。需要注意的是，即使服务执行超时，也不意味着在超时后，服务代码本身会被强行中断，实际上服务代码仍然会继续执行，只是执行结果不会作为调用结果被返回。因此，如果客户端调用某个服务发生超时异常，应该检查服务本身是否编写错误，如果某个服务因为死锁而造成超时，服务器最终将会被该服务耗尽资源。该设置默认值为 30 秒。

`codec` 属性用来设置编解码器。默认设置是 hprose 编解码器，如果替换为 JSONRPC 编解码器，则服务器不但可以处理 hprose 请求，也可以同时处理 JSONRPC 请求。

`maxRequestLength` 属性用来设置请求编码后的最大字节数。如果请求字节数超过该设置，则返回异常。该设置默认值为 0x7FFFFFFF（2147483647），相当于没有限制。为了保证不会因为请求太大造成内存溢出的错误，最好根据具体业务将该值设置为一个更为合理的值。

`names` 只读属性。返回服务端已发布的所有方法名列表。

### 方法

* `bind`
* `handle`
* `process`
* `execute`
* `use`
* `unuse`
* `get`
* `remove`
* `add`
* `addMissingMethod`

`bind` 方法的功能跟处理器接口的 `bind` 方法功能一致，但是处理器接口的 `bind` 方法是针对具体某个类型的服务器的。而服务对象上的 `bind` 方法，可以跟已经注册过的任何服务器进行绑定。因此，用户不需要调用处理器接口上的 `bind` 方法，而应该使用服务对象上的 `bind` 方法来统一操作。

`handle` 方法接受编码后的请求并返回服务执行后已编码响应结果。该方法是留给处理器实现 `handler` 方法时使用的，用户如果不需要编写自定义处理器，则不需要关心该方法。

`process` 方法是服务端的输入输出处理器链中最后执行的方法。如果在实现服务端的输入输出处理器时，希望跳过输入输出处理器链上后面所有的输入输出处理器，可以通过直接调用该方法来代替调用 `next`。其他情况下，用户不需要关心该方法。

`execute` 方法是服务端的调用处理器链中最后执行的方法。如果在实现服务端的调用处理器时，希望跳过调用处理器链上后面所有的调用处理器，可以通过直接调用该方法来代替调用 `next`。其他情况下，用户不需要关心该方法。

`use` 方法用来添加输入输出处理器或调用处理器，`unuse` 方法用来删除输入输出处理器或调用处理器。对于不支持使用 `use` 作为方法名的语言，使用 `addHandler` 和 `removeHandler` 代替。

`get` 方法用来通过发布名称获取已发布的服务方法对象。在某些语言中（比如 C#），允许发布相同名称但参数个数不同的重载方法，因此，在这些语言中，`get` 方法本身会有 2 个参数（发布名称，参数个数）。

`remove` 方法用来删除已发布的服务方法。该方法的参数与 `get` 方法相同。

`add` 方法用来添加发布的服务方法。该方法在不同的语言中，可以根据具体情况提供多个重载或不同命名的多个实现，例如：`addMethod`，`addMethods`，`addFunction`，`addFunctions`，`addStaticMethods`，`addInstanceMethods` 等等。

`addMissingMethod` 方法用来添加一个通用的服务方法，当客户端调用的服务方法不存在时，该服务方法才会被调用。

当调用 `get` 方法时，如果查找的具体服务方法不存在，但是通用服务方法存在的话，则会返回通用服务方法，否则返回 `null`。

另外，还可以通过 `service[name]` 的方式来存取处理器对象，例如通过 `service["http"]` 可以获得已经注册的 `HttpHandler` 对象实例。

## 客户端

服务端统一为一个 `Client` 类型。下面来分别介绍它所包含的属性和方法。

### 属性

* `timeout`
* `codec`
* `requestHeaders`
* `uris`

`timeout` 属性用来限制远程调用的执行时间，如果远程调用执行超时，则返回超时异常。需要注意的是，如果远程调用服务执行超时，那么客户端和服务端之间的连接可能会断开。该设置默认值为 30 秒。

`codec` 属性用来设置编解码器。默认设置是 hprose 编解码器，如果替换为 JSONRPC 编解码器，则客户端会变成 JSONRPC 客户端。

`requestHeaders` 属性用来设置所有远程调用共同的请求头部信息。

`uris` 用来设置服务地址列表。需要注意，当为该属性赋值时，所设置的属性中服务地址列表的顺序会自动随机排序，默认情况下，调用只使用第一个服务地址。因此对于不同到客户端来说，调用的服务地址可能是不同的。当有众多的客户端时，虽然默认情况下，每个客户端所发起的调用都会发往某一个特定的服务地址，但是所有客户端发起的调用会被平均的发送到设置的所有服务地址上。

### 方法

* `useService`
* `use`
* `unuse`
* `invoke`
* `call`
* `transport`
* `abort`

`useService` 方法用来返回远程调用的代理对象。对于不同的语言，该方法的使用也会有所区别，有些语言需要定义接口，而有些语言则不需要定义接口。

`use` 方法用来添加输入输出处理器或调用处理器，`unuse` 方法用来删除输入输出处理器或调用处理器。对于不支持使用 `use` 作为方法名的语言，使用 `addHandler` 和 `removeHandler` 代替。

`invoke` 方法用来直接调用远程方法，对于不同的语言，该方法可能是表示同步调用，也可能表示异步调用。但是如果在某个语言中同时支持同步和异步调用，通常同步版本使用 `invoke` 这个方法名，而异步版本使用 `invokeAsync` 这个方法名。

`call` 方法是客户端的调用处理器链中最后执行的方法。如果在实现客户端的调用处理器时，希望跳过调用处理器链上后面所有的调用处理器，可以通过直接调用该方法来代替调用 `next`。其他情况下，用户不需要关心该方法。

`transport` 方法是客户端的输入输出处理器链中最后执行的方法。如果在实现客户端的输入输出处理器时，希望跳过输入输出处理器链上后面所有的输入输出处理器，可以通过直接调用该方法来代替调用 `next`。其他情况下，用户不需要关心该方法。

`abort` 方法用来中断客户端和服务端的连接。但是对于某些传输协议，该方法可能没有实际效果。

另外，`transport` 和 `abort` 方法跟注册的传输接口实现有关。可以通过 `client[name]` 的方式来存取实际的传输接口对象，例如通过 `client["http"]` 可以获得已经注册的 `HttpTransport` 对象实例。