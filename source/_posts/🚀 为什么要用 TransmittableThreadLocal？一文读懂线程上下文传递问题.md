---
title: 为什么要用 TransmittableThreadLocal？一文读懂线程上下文传递问题
date: 2025-05-17 20:32:07
top_group_index: 1
categories: 后端
tags: [Java, 线程上下文, TransmittableThreadLocal]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/F2978672D981F1C36C0800ECDE7881C3.jpg
---
# 🚀 为什么要用 TransmittableThreadLocal？一文读懂线程上下文传递问题

在 Java Web 开发中，我们经常用 `ThreadLocal` 来保存每个请求的用户信息，例如 `userId`。但当我们使用线程池或异步方法（如 `@Async`）时，很多人会遇到这样的问题：

> ❗ 主线程设置了 ThreadLocal，子线程里却拿不到！

这是因为 **`ThreadLocal` 的值默认不会在线程池中传递到子线程**，会导致上下文丢失，比如 userId 为 null，traceId 无法传递等问题。

---

## 🧩 问题重现

```java
// 主线程中设置用户 ID
UserContext.setUserId(123L);

// 子线程中获取（比如线程池执行）
executorService.submit(() -> {
    System.out.println(UserContext.getUserId()); // 输出 null
});
```

## ✅ 解决方案：使用阿里开源的 TransmittableThreadLocal (TTL)

`TransmittableThreadLocal` 是阿里巴巴开源的增强版 `ThreadLocal`，可以将主线程中的上下文信息，**传递到子线程中（包括线程池中）**。

### 简单改造：

#### 1. 引入依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.14.2</version> <!-- 版本可选最新 -->
</dependency>
```

#### 2. 替换原有 ThreadLocal

```java
public class UserContext {
    private static final TransmittableThreadLocal<Long> userIdThreadLocal = new TransmittableThreadLocal<>();

    public static void setUserId(Long userId) {
        userIdThreadLocal.set(userId);
    }

    public static Long getUserId() {
        return userIdThreadLocal.get();
    }

    public static void clear() {
        userIdThreadLocal.remove();
    }
}
```

#### 3. 包装子线程任务

```java
Runnable task = () -> {
    System.out.println(UserContext.getUserId());
};

executorService.submit(TtlRunnable.get(task)); // TTL 包装，自动传递上下文
```

------

## ⚙ Spring 用户更方便

如果你使用 Spring 的 `@Async` 或线程池，可以配合 TTL 的 [插件支持](https://github.com/alibaba/transmittable-thread-local#integrate-with-frameworklibrary) 实现自动上下文传递，无需手动包装。

------

## 📌 总结对比

| 对比项             | ThreadLocal | TransmittableThreadLocal |
| ------------------ | ----------- | ------------------------ |
| 支持线程上下文传递 | ❌ 否        | ✅ 是                     |
| 在线程池中能获取值 | ❌ 否        | ✅ 是                     |
| 推荐场景           | 单线程      | 多线程、异步、线程池     |
| 是否侵入业务代码   | ❌ 少        | ✅ 极小，可自动适配       |



------

### 🧠 使用 TTL，你可以轻松实现：

- 日志 traceId 传递
- 用户上下文（userId、token）传递
- 请求头信息传播
- 分布式链路追踪

------

> 🔗 推荐阅读项目地址：https://github.com/alibaba/transmittable-thread-local

欢迎点赞、评论交流你在实际使用中的坑与经验 👇
