---
title: SpringBoot单元测试报java.lang.IllegalStateException Failed to load ApplicationContext错误的解决方法
date: 2024-12-31 16:50:38
top_group_index: 1
categories: 后端
tags: [spring, 单元测试]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/31714DB3BBFB0F94C8BA2FF720A1D04C.jpg
---
# SpringBoot单元测试报java.lang.IllegalStateException: Failed to load ApplicationContext错误的解决方法

 大致原因：使用了WebSocket使得在测试环境也得需要启动嵌入式的Servlet容器

解决方案：将@SpringBootTest的属性webEnvironment改为SpringBootTest.WebEnvironment.RANDOM_PORT即可

> @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)

以下为详细解释：

在使用单元测试时报以下

> java.lang.IllegalStateException: Failed to load ApplicationContext

一开始很疑惑，因为我之前使用过单元测试是能使用的，然后看了看完整的报错信息

```
java.lang.IllegalStateException: Failed to load ApplicationContext for [WebMergedContextConfiguration@19bfbe28 testClass = com.example.demo.service.impl.TeamServiceImplTest, locations = [], classes = [com.example.demo.DemoApplication], contextInitializerClasses = [], activeProfiles = [], propertySourceDescriptors = [], propertySourceProperties = ["org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true"], contextCustomizers = [org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@537f60bf, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@229c6181, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@d86a6f, org.springframework.boot.test.web.reactor.netty.DisableReactorResourceFactoryGlobalResourcesContextCustomizerFactory$DisableReactorResourceFactoryGlobalResourcesContextCustomizerCustomizer@6e6f2380, org.springframework.boot.test.autoconfigure.OnFailureConditionReportContextCustomizerFactory$OnFailureConditionReportContextCustomizer@53fe15ff, org.springframework.boot.test.autoconfigure.actuate.observability.ObservabilityContextCustomizerFactory$DisableObservabilityContextCustomizer@1f, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizer@4be29ed9, org.springframework.boot.test.context.SpringBootTestAnnotation@88aa156c], resourceBasePath = "src/main/webapp", contextLoader = org.springframework.boot.test.context.SpringBootContextLoader, parent = null]

	at org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate.loadContext(DefaultCacheAwareContextLoaderDelegate.java:180)
	at org.springframework.test.context.support.DefaultTestContext.getApplicationContext(DefaultTestContext.java:130)
	at org.springframework.test.context.web.ServletTestExecutionListener.setUpRequestContextIfNecessary(ServletTestExecutionListener.java:191)
	at org.springframework.test.context.web.ServletTestExecutionListener.prepareTestInstance(ServletTestExecutionListener.java:130)
	at org.springframework.test.context.TestContextManager.prepareTestInstance(TestContextManager.java:260)
	at org.springframework.test.context.junit.jupiter.SpringExtension.postProcessTestInstance(SpringExtension.java:163)
	at java.base/java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:197)
	at java.base/java.util.stream.ReferencePipeline$2$1.accept(ReferencePipeline.java:179)
	at java.base/java.util.ArrayList$ArrayListSpliterator.forEachRemaining(ArrayList.java:1708)
	at java.base/java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:509)
	at java.base/java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:499)
	at java.base/java.util.stream.StreamSpliterators$WrappingSpliterator.forEachRemaining(StreamSpliterators.java:310)
	at java.base/java.util.stream.Streams$ConcatSpliterator.forEachRemaining(Streams.java:735)
	at java.base/java.util.stream.Streams$ConcatSpliterator.forEachRemaining(Streams.java:734)
	at java.base/java.util.stream.ReferencePipeline$Head.forEach(ReferencePipeline.java:762)
	at java.base/java.util.Optional.orElseGet(Optional.java:364)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'serviceEndpointExporter' defined in class path resource [com/example/demo/config/WebSocketConfiguration.class]: jakarta.websocket.server.ServerContainer not available
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1806)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:600)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:522)
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:337)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:335)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:200)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:975)
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:971)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:625)
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:754)
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:456)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:335)
	at org.springframework.boot.test.context.SpringBootContextLoader.lambda$loadContext$3(SpringBootContextLoader.java:137)
	at org.springframework.util.function.ThrowingSupplier.get(ThrowingSupplier.java:58)
	at org.springframework.util.function.ThrowingSupplier.get(ThrowingSupplier.java:46)
	at org.springframework.boot.SpringApplication.withHook(SpringApplication.java:1463)
	at org.springframework.boot.test.context.SpringBootContextLoader$ContextLoaderHook.run(SpringBootContextLoader.java:553)
	at org.springframework.boot.test.context.SpringBootContextLoader.loadContext(SpringBootContextLoader.java:137)
	at org.springframework.boot.test.context.SpringBootContextLoader.loadContext(SpringBootContextLoader.java:108)
```

似乎是使用了WebSocket之后产生的问题，在网上查了查资料才知道，@SpringBootTest的`webEnvironment`属性

默认是`MOCK`值，它会使用Mock的Servlet环境，而不会启动嵌入式的Servlet容器，这意味着不能进行真实网络请求。当将属性值改为`RANDOM_PORT`时，它会启动嵌入式的Servlet容器（如Tomcat），并在一个随机端口上监听，这在WebSocket测试是必要的。

WebSocket是一种在单个TCP连接上进行全双工通信的协议。这意味着客户端和服务器之间需要持续的、双向的数据流。在单元测试中，如果使用了Mock的Servlet环境（即`webEnvironment = MOCK`），则不会启动嵌入式的Servlet容器，因此WebSocket连接无法建立。

而设置`webEnvironment = RANDOM_PORT`时，Spring Boot会启动一个嵌入式的Servlet容器（如Tomcat），并在一个随机端口上监听。这允许你的测试代码通过WebSocket客户端连接到这个容器，从而进行实际的网络通信测试。这是WebSocket集成测试所必需的。

所以我们只需要将`webEnvironment `的`webEnvironment`属性改`SpringBootTest.WebEnvironment.RANDOM_PORT`即可
