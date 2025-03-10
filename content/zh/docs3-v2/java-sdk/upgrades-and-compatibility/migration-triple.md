---
type: docs
title: "Dubbo协议迁移至Triple协议指南"
linkTitle: "Dubbo协议迁移至Triple协议指南"
weight: 2
description: "Triple协议迁移指南"
---

## Triple 介绍

`Triple` 协议的格式和原理请参阅 [RPC 通信协议](/zh/docs/concepts/rpc-protocol/)

根据 Triple 设计的目标，`Triple` 协议有以下优势:

- 具备跨语言交互的能力，传统的多语言多 SDK 模式和 Mesh 化跨语言模式都需要一种更通用易扩展的数据传输协议。
- 提供更完善的请求模型，除了支持传统的 Request/Response 模型（Unary 单向通信），还支持 Stream（流式通信） 和 Bidirectional（双向通信）。
- 易扩展、穿透性高，包括但不限于 Tracing / Monitoring 等支持，也应该能被各层设备识别，网关设施等可以识别数据报文，对 Service Mesh 部署友好，降低用户理解难度。
- 完全兼容 grpc，客户端/服务端可以与原生grpc客户端打通。
- 可以复用现有 grpc 生态下的组件, 满足云原生场景下的跨语言、跨环境、跨平台的互通需求。

当前使用其他协议的 Dubbo 用户，框架提供了兼容现有序列化方式的迁移能力，在不影响线上已有业务的前提下，迁移协议的成本几乎为零。

需要新增对接 Grpc 服务的 Dubbo 用户，可以直接使用 Triple 协议来实现打通，不需要单独引入 grpc client 来完成，不仅能保留已有的 Dubbo 易用性，也能降低程序的复杂度和开发运维成本，不需要额外进行适配和开发即可接入现有生态。

对于需要网关接入的 Dubbo 用户，Triple 协议提供了更加原生的方式，让网关开发或者使用开源的 grpc 网关组件更加简单。网关可以选择不解析 payload ，在性能上也有很大提高。在使用 Dubbo 协议时，语言相关的序列化方式是网关的一个很大痛点，而传统的 HTTP 转 Dubbo 的方式对于跨语言序列化几乎是无能为力的。同时，由于 Triple 的协议元数据都存储在请求头中，网关可以轻松的实现定制需求，如路由和限流等功能。


## Dubbo2 协议迁移流程

Dubbo2 的用户使用 dubbo 协议 + 自定义序列化，如 hessian2 完成远程调用。

而 Grpc 的默认仅支持 Protobuf 序列化，对于 Java 语言中的多参数以及方法重载也无法支持。

Dubbo3的之初就有一条目标是完美兼容 Dubbo2，所以为了 Dubbo2 能够平滑升级， Dubbo 框架侧做了很多工作来保证升级的无感，目前默认的序列化和 Dubbo2 保持一致为`hessian2`。

所以，如果决定要升级到 Dubbo3 的 `Triple` 协议，只需要修改配置中的协议名称为 `tri` (注意: 不是triple)即可。

接下来我们我们以一个使用 Dubbo2 协议的[工程](https://github.com/apache/dubbo-samples/tree/master/dubbo-samples-triple/src/main/java/org/apache/dubbo/sample/tri/migration) 来举例，如何一步一步安全的升级。

1. 仅使用 `dubbo` 协议启动 `provider` 和 `consumer`，并完成调用。
2. 使用 `dubbo` 和 `tri` 协议 启动`provider`，以 `dubbo` 协议启动 `consumer`，并完成调用。
3. 仅使用 `tri` 协议 启动 `provider`和 `consumer`，并完成调用。

### 定义服务

1. 定义接口
```java
public interface IWrapperGreeter {

    //... 
    
    /**
     * 这是一个普通接口，没有使用 pb 序列化
     */
    String sayHello(String request);

}
```

2. 实现类如下
```java
public class IGreeter2Impl implements IWrapperGreeter {

    @Override
    public String sayHello(String request) {
        return "hello," + request;
    }
}
```

### 仅使用 dubbo 协议

为保证兼容性，我们先将部分 provider 升级到 `dubbo3` 版本并使用 `dubbo` 协议。

使用 `dubbo` 协议启动一个 [`Provider`](https://github.com/apache/dubbo-samples/tree/master/dubbo-samples-triple/src/main/java/org/apache/dubbo/sample/tri/migration/ApiMigrationDubboProvider.java) 和 [`Consumer`](https://github.com/apache/dubbo-samples/tree/master/dubbo-samples-triple/src/main/java/org/apache/dubbo/sample/tri/migration/ApiMigrationDubboConsumer.java) ,完成调用，输出如下:
![result](/imgs/v3/migration/tri/dubbo3-tri-migration-dubbo-dubbo-result.png)

###  同时使用 dubbo 和 triple 协议

对于线上服务的升级，不可能一蹴而就同时完成 provider 和 consumer 升级, 需要按步操作，保证业务稳定。
第二步, provider 提供双协议的方式同时支持 dubbo + tri 两种协议的客户端。

结构如图所示:
![strust](/imgs/v3/migration/tri/migrate-dubbo-tri-strust.png)

> 按照推荐升级步骤，provider 已经支持了tri协议，所以 dubbo3的 consumer 可以直接使用 tri 协议

使用`dubbo`协议和`triple`协议启动[`Provider`](https://github.com/apache/dubbo-samples/tree/master/dubbo-samples-triple/src/main/java/org/apache/dubbo/sample/tri/migration/ApiMigrationBothProvider.java)和[`Consumer`](https://github.com/apache/dubbo-samples/tree/master/dubbo-samples-triple/src/main/java/org/apache/dubbo/sample/tri/migration/ApiMigrationBothConsumer.java),完成调用，输出如下:

![result](/imgs/v3/migration/tri/dubbo3-tri-migration-both-dubbo-tri-result.png)


### 仅使用 triple 协议

当所有的 consuemr 都升级至支持 `Triple` 协议的版本后，provider 可切换至仅使用 `Triple` 协议启动 

结构如图所示:
![strust](/imgs/v3/migration/tri/migrate-only-tri-strust.png)

[Provider](https://github.com/apache/dubbo-samples/tree/master/dubbo-samples-triple/src/main/java/org/apache/dubbo/sample/tri/migration/ApiMigrationTriProvider.java)
和 [Consumer](https://github.com/apache/dubbo-samples/tree/master/dubbo-samples-triple/src/main/java/org/apache/dubbo/sample/tri/migration/ApiMigrationTriConsumer.java) 完成调用，输出如下:

![result](/imgs/v3/migration/tri/dubbo3-tri-migration-tri-tri-result.png)


### 实现原理

通过上面介绍的升级过程，我们可以很简单的通过修改协议类型来完成升级。框架是怎么帮我们做到这些的呢？

通过对 `Triple` 协议的介绍，我们知道Dubbo3的 `Triple` 的数据类型是 `protobuf` 对象，那为什么非 `protobuf` 的 java 对象也可以被正常传输呢。

这里 Dubbo3 使用了一个巧妙的设计，首先判断参数类型是否为 `protobuf` 对象，如果不是。用一个 `protobuf` 对象将 `request` 和 `response` 进行 wrapper，这样就屏蔽了其他各种序列化带来的复杂度。在 `wrapper` 对象内部声明序列化类型，来支持序列化的扩展。

wrapper 的`protobuf`的 IDL如下:
```proto
syntax = "proto3";

package org.apache.dubbo.triple;

message TripleRequestWrapper {
    // hessian4
    // json
    string serializeType = 1;
    repeated bytes args = 2;
    repeated string argTypes = 3;
}

message TripleResponseWrapper {
    string serializeType = 1;
    bytes data = 2;
    string type = 3;
}
```

对于请求，使用`TripleRequestWrapper`进行包装，对于响应使用`TripleResponseWrapper`进行包装。

> 对于请求参数，可以看到 args 被`repeated`修饰，这是因为需要支持 java 方法的多个参数。当然，序列化只能是一种。序列化的实现沿用 Dubbo2 实现的 spi


## 多语言用户 (正在使用 Protobuf) 
> 建议新服务均使用该方式

对于 Dubbo3 和 Triple 来说，主推的是使用 `protobuf` 序列化，并且使用 `proto` 定义的 `IDL` 来生成相关接口定义。以 `IDL` 做为多语言中的通用接口约定，加上 `Triple` 与 `Grpc` 的天然互通性，可以轻松地实现跨语言交互，例如 Go 语言等。


将编写好的 `.proto` 文件使用 `dubbo-compiler` 插件进行编译并编写实现类，完成方法调用：

![result](/imgs/v3/migration/tri/dubbo3-tri-migration-tri-tri-result.png)

从上面升级的例子我们可以知道，`Triple` 协议使用 `protbuf` 对象序列化后进行传输，所以对于本身就是 `protobuf` 对象的方法来说，没有任何其他逻辑。

使用 `protobuf` 插件编译后接口如下：
```java
public interface PbGreeter {

    static final String JAVA_SERVICE_NAME = "org.apache.dubbo.sample.tri.PbGreeter";
    static final String SERVICE_NAME = "org.apache.dubbo.sample.tri.PbGreeter";

    static final boolean inited = PbGreeterDubbo.init();

    org.apache.dubbo.sample.tri.GreeterReply greet(org.apache.dubbo.sample.tri.GreeterRequest request);

    default CompletableFuture<org.apache.dubbo.sample.tri.GreeterReply> greetAsync(org.apache.dubbo.sample.tri.GreeterRequest request){
        return CompletableFuture.supplyAsync(() -> greet(request));
    }

    void greetServerStream(org.apache.dubbo.sample.tri.GreeterRequest request, org.apache.dubbo.common.stream.StreamObserver<org.apache.dubbo.sample.tri.GreeterReply> responseObserver);

    org.apache.dubbo.common.stream.StreamObserver<org.apache.dubbo.sample.tri.GreeterRequest> greetStream(org.apache.dubbo.common.stream.StreamObserver<org.apache.dubbo.sample.tri.GreeterReply> responseObserver);
}
```

## 开启 Triple 新特性 —— Stream (流)
Stream 是 Dubbo3 新提供的一种调用类型，在以下场景时建议使用流的方式:

- 接口需要发送大量数据，这些数据无法被放在一个 RPC 的请求或响应中，需要分批发送，但应用层如果按照传统的多次 RPC 方式无法解决顺序和性能的问题，如果需要保证有序，则只能串行发送
- 流式场景，数据需要按照发送顺序处理, 数据本身是没有确定边界的
- 推送类场景，多个消息在同一个调用的上下文中被发送和处理

Stream 分为以下三种:
- SERVER_STREAM(服务端流)
![SERVER_STREAM](/imgs/v3/migration/tri/migrate-server-stream.png)
- CLIENT_STREAM(客户端流)
![CLIENT_STREAM](/imgs/v3/migration/tri/migrate-client-stream.png)
- BIDIRECTIONAL_STREAM(双向流)
![BIDIRECTIONAL_STREAM](/imgs/v3/migration/tri/migrate-bi-stream.png)

> 由于 `java` 语言的限制，BIDIRECTIONAL_STREAM 和 CLIENT_STREAM 的实现是一样的。

在 Dubbo3 中，流式接口以 `SteamObserver` 声明和使用，用户可以通过使用和实现这个接口来发送和处理流的数据、异常和结束。

> 对于 Dubbo2 用户来说，可能会对StreamObserver感到陌生，这是Dubbo3定义的一种流类型，Dubbo2 中并不存在 Stream 的类型，所以对于迁移场景没有任何影响。

流的语义保证
- 提供消息边界，可以方便地对消息单独处理
- 严格有序，发送端的顺序和接收端顺序一致
- 全双工，发送不需要等待
- 支持取消和超时

### 非 PB 序列化的流
1. api
```java
public interface IWrapperGreeter {

    StreamObserver<String> sayHelloStream(StreamObserver<String> response);

    void sayHelloServerStream(String request, StreamObserver<String> response);
}
```

> Stream 方法的方法入参和返回值是严格约定的，为防止写错而导致问题，Dubbo3 框架侧做了对参数的检查, 如果出错则会抛出异常。
> 对于 `双向流(BIDIRECTIONAL_STREAM)`, 需要注意参数中的 `StreamObserver` 是响应流，返回参数中的 `StreamObserver` 为请求流。

2. 实现类
```java
public class WrapGreeterImpl implements WrapGreeter {

    //...

    @Override
    public StreamObserver<String> sayHelloStream(StreamObserver<String> response) {
        return new StreamObserver<String>() {
            @Override
            public void onNext(String data) {
                System.out.println(data);
                response.onNext("hello,"+data);
            }

            @Override
            public void onError(Throwable throwable) {
                throwable.printStackTrace();
            }

            @Override
            public void onCompleted() {
                System.out.println("onCompleted");
                response.onCompleted();
            }
        };
    }

    @Override
    public void sayHelloServerStream(String request, StreamObserver<String> response) {
        for (int i = 0; i < 10; i++) {
            response.onNext("hello," + request);
        }
        response.onCompleted();
    }
}
```

3. 调用方式
```java
delegate.sayHelloServerStream("server stream", new StreamObserver<String>() {
    @Override
    public void onNext(String data) {
        System.out.println(data);
    }

    @Override
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
    }

    @Override
    public void onCompleted() {
        System.out.println("onCompleted");
    }
});


StreamObserver<String> request = delegate.sayHelloStream(new StreamObserver<String>() {
    @Override
    public void onNext(String data) {
        System.out.println(data);
    }

    @Override
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
    }

    @Override
    public void onCompleted() {
        System.out.println("onCompleted");
    }
});
for (int i = 0; i < n; i++) {
    request.onNext("stream request" + i);
}
request.onCompleted();
```

## 使用 Protobuf 序列化的流

对于 `Protobuf` 序列化方式，推荐编写 `IDL` 使用 `compiler` 插件进行编译生成。生成的代码大致如下:
```java
public interface PbGreeter {

    static final String JAVA_SERVICE_NAME = "org.apache.dubbo.sample.tri.PbGreeter";
    static final String SERVICE_NAME = "org.apache.dubbo.sample.tri.PbGreeter";

    static final boolean inited = PbGreeterDubbo.init();
    
    //...

    void greetServerStream(org.apache.dubbo.sample.tri.GreeterRequest request, org.apache.dubbo.common.stream.StreamObserver<org.apache.dubbo.sample.tri.GreeterReply> responseObserver);

    org.apache.dubbo.common.stream.StreamObserver<org.apache.dubbo.sample.tri.GreeterRequest> greetStream(org.apache.dubbo.common.stream.StreamObserver<org.apache.dubbo.sample.tri.GreeterReply> responseObserver);
}
```

### 流的实现原理

`Triple`协议的流模式是怎么支持的呢？

- 从协议层来说，`Triple` 是建立在 `HTTP2` 基础上的，所以直接拥有所有 `HTTP2` 的能力，故拥有了分 `stream` 和全双工的能力。

- 框架层来说，`StreamObserver` 作为流的接口提供给用户，用于入参和出参提供流式处理。框架在收发 stream data 时进行相应的接口调用, 从而保证流的生命周期完整。

## Triple 与应用级注册发现

关于 Triple 协议的应用级服务注册和发现和其他语言是一致的，可以通过下列内容了解更多。

- [服务发现](/zh/docs/concepts/service-discovery/)
- [应用级地址发现迁移指南](/zh/docs/migration/migration-service-discovery/)

## 与 GRPC 互通

通过对于协议的介绍，我们知道 `Triple` 协议是基于 `HTTP2` 并兼容 `GRPC`。为了保证和验证与`GRPC`互通能力，Dubbo3 也编写了各种从场景下的测试。详细的可以通过[这里](https://github.com/apache/dubbo-samples/tree/master/dubbo-samples-triple/README.MD) 了解更多。

##  未来: Everything on Stub

用过 `Grpc` 的同学应该对 `Stub` 都不陌生。
Grpc 使用 `compiler` 将编写的 `proto` 文件编译为相关的 protobuf 对象和相关 rpc 接口。默认的会同时生成几种不同的 `stub`

- blockingStub
- futureStub
- reactorStub
- ...

`stub` 用一种统一的使用方式帮我们屏蔽了不同调用方式的细节。不过目前 `Dubbo3` 暂时只支持传统定义接口并进行调用的使用方式。

在不久的未来，`Triple` 也将实现各种常用的 `Stub`，让用户写一份`proto`文件，通过 `comipler` 可以在任意场景方便的使用，请拭目以待。