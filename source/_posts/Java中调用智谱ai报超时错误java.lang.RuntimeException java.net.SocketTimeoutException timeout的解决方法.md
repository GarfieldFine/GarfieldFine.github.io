---
title: Java中调用智谱ai报超时错误java.lang.RuntimeException java.net.SocketTimeoutException timeout的解决方法.md
date: 2025-01-03 13:53:52
top_group_index: 1
categories: 后端
tags: [java, 智谱ai]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/BC9EF0E658CEE8B551B9CB1401DBD16D.jpeg
---
# Java中调用智谱ai报超时错误java.lang.RuntimeException: java.net.SocketTimeoutException: timeout的解决方法 

主要问题：调用智谱ai时报：java.net.SocketTimeoutException: timeout，超时错误

解决方法：

> 在创建Client时将
>
> ```java
> ClientV4 client = new ClientV4.Builder("{Your ApiSecretKey}").build();  
> ```
>
> 改成
>
> ```java
> new ClientV4.Builder(aiProperites.getKeySecret())
>      .enableTokenCache()
>      .networkConfig(60, 60, 60, 60, TimeUnit.SECONDS)
>      .connectionPool(new okhttp3.ConnectionPool(8, 1, TimeUnit.SECONDS))
>      .build();
> ```
>
> `.networkConfig(60, 60, 60, 60, TimeUnit.SECONDS)`当中的四个时间不要设置的太小（单位秒），不然可能还会报`timeout`错误

详细解释：

在使用官方的示例代码时

```java
private static void testInvoke() {
   List<ChatMessage> messages = new ArrayList<>();
   ChatMessage chatMessage = new ChatMessage(ChatMessageRole.USER.value(), {Your message});
   messages.add(chatMessage);
   String requestId = String.format(requestIdTemplate, System.currentTimeMillis());
   
   ChatCompletionRequest chatCompletionRequest = ChatCompletionRequest.builder()
           .model(Constants.ModelChatGLM4)
           .stream(Boolean.FALSE)
           .invokeMethod(Constants.invokeMethod)
           .messages(messages)
           .requestId(requestId)
           .build();
   ModelApiResponse invokeModelApiResp = client.invokeModelApi(chatCompletionRequest);
   try {
       System.out.println("model output:" + mapper.writeValueAsString(invokeModelApiResp));
   } catch (JsonProcessingException e) {
       e.printStackTrace();
   }
}
```

报以下错误

```
java.lang.RuntimeException: java.net.SocketTimeoutException: timeout
```

在我尝试了几次后发现， `client.invokeModelApi(chatCompletionRequest)`请求的等待时长是10s，当超过这个时间，则会报错，所以我们不能使用这个最基本的创建方式，而选择在构造时加一些自己的配置，所以我们使用

```
new ClientV4.Builder(aiProperites.getKeySecret())
        .enableTokenCache()
        .networkConfig(60, 60, 60, 60, TimeUnit.SECONDS)
        .connectionPool(new okhttp3.ConnectionPool(8, 1, TimeUnit.SECONDS))
        .build();
```

来构造客户端。

在这里我主要解释下`.networkConfig(60, 60, 60, 60, TimeUnit.SECONDS)`连接配置，前面四个时间分别是`requestTimeOut`（请求超时）、`connectTimeOut`（连接超时）、`readTimeOut`（读取超时）和`writeTimeOut`（写入超时）；

1. requestTimeOut（请求超时）
   - 通常指的是从客户端发起请求到收到服务器响应的整个过程的最大等待时间。
   - 如果在这个时间内没有收到服务器的任何响应（无论是成功还是失败的响应），则客户端会认为请求已经超时，并可能会抛出异常或错误。
   - `requestTimeOut`涵盖了连接建立、数据发送、数据接收等整个请求-响应周期的时间。
2. connectTimeOut（连接超时）
   - 指的是客户端尝试与服务器建立TCP连接的最大等待时间。
   - 在这个时间内，客户端会尝试与服务器进行三次握手，以建立可靠的TCP连接。
   - 如果在这个时间内无法成功建立连接（例如，因为服务器没有响应或网络问题），则客户端会认为连接超时，并可能会放弃连接尝试。
3. readTimeOut（读取超时）
   - 指的是客户端从服务器读取数据的最大等待时间。
   - 一旦客户端与服务器建立了连接，并开始从服务器读取数据，`readTimeOut`就开始计时。
   - 如果在这个时间内客户端没有从服务器接收到任何数据（即数据读取操作被阻塞或进展缓慢），则客户端会认为读取超时，并可能会中断连接或抛出异常。
4. writeTimeOut（写入超时）
   - 指的是客户端向服务器发送数据的最大等待时间。
   - 不同于`readTimeOut`，`writeTimeOut`关注的是数据发送过程。
   - 如果在这个时间内客户端无法将数据成功发送到服务器（例如，因为网络拥堵或服务器处理缓慢），则客户端可能会认为写入超时，并可能会采取相应的措施（如重试发送、中断连接等）。

以上是四个时间的有关概念，最后第五个参数是时间单位。

`.connectionPool(new okhttp3.ConnectionPool(8, 1, TimeUnit.SECONDS))`用来创建连接池，`new okhttp3.ConnectionPool(8, 1, TimeUnit.SECONDS)`的三个参数分别是连接池中最大空闲连接数、空闲连接在连接池中保持存活的最长时间，第二个参数的时间单位。

以上是有关介绍
