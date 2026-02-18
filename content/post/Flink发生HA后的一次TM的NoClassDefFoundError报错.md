---
title: 'Flink发生HA后的一次TM的NoClassDefFoundError报错'
date: '2026-02-18T16:08:00+08:00'
lastmod: '2026-02-18T16:08:00+08:00'
keywords: ['flink']
categories: ['flink']
tags: ['flink']
author: '小十一狼'
---

# 〇、结论先行

**使用了已关闭的 ClassLoader 去加载类，但关闭的 ClassLoader 已丧失加载新类的能力。**

**已关闭的 ClassLoader 是被缓存的类实例持有并在类加载过程中使用的。**

以下简单说明一下过程，用 ChildFirstClassLoader1 代表 failover 前的 ClassLoader，用 ChildFirstClassLoader2 代表 failover 后的 ClassLoader：

Kafka Login 模块初始化时，提供了 MTPlainSaslServerProvider，该类由 ChildFirstClassLoader1 加载并在 sun.security.jca.Providers 中被缓存；

Failover 后，ChildFirstClassLoader1 被关闭，ChildFirstClassLoader2 诞生了；

ChildFirstClassLoader2 在执行 Hadoop 相关类的时候，获取所有的 SaslServerFactory，其中关于 Kafka 的 MTPlainSaslServerFactory 需要通过已被缓存的 MTPlainSaslServerProvider 对应的 ChildFirstClassLoader1 加载，然后调用 getMechanismNames，其中使用了外部类 MTPlainSaslServer 的 LOGGER 引用，所以需要去加载 MTPlainSaslServer 类，该类在 ChildFirstClassLoader1 关闭前未被加载，此时类加载器 ChildFirstClassLoader1 已关闭，再去加载就找不到类。

# 一、报错详情

```
java.lang.NoClassDefFoundError: org/apache/mtkafka/common/security/meituan/MTPlainSaslServer
	at org.apache.mtkafka.common.security.meituan.MTPlainSaslServer$MTPlainSaslServerFactory.<clinit>(MTPlainSaslServer.java:148) ~[?:?]
	at java.lang.Class.forName0(Native Method) ~[?:1.8.0_312]
	at java.lang.Class.forName(Class.java:348) ~[?:1.8.0_312]
	at javax.security.sasl.Sasl.loadFactory(Sasl.java:447) ~[?:1.8.0_312]
	at javax.security.sasl.Sasl.getFactories(Sasl.java:651) ~[?:1.8.0_312]
	at javax.security.sasl.Sasl.getSaslServerFactories(Sasl.java:608) ~[?:1.8.0_312]
	at org.apache.hadoop.security.SaslRpcServer$FastSaslServerFactory.<init>(SaslRpcServer.java:376) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.security.SaslRpcServer.init(SaslRpcServer.java:184) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.ipc.RPC.getProtocolProxy(RPC.java:595) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.hdfs.NameNodeProxies.createNNProxyWithClientProtocol(NameNodeProxies.java:432) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.hdfs.NameNodeProxies.createNonHAProxy(NameNodeProxies.java:318) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider$DefaultProxyFactory.createProxy(ConfiguredFailoverProxyProvider.java:68) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider.getProxy(ConfiguredFailoverProxyProvider.java:152) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.io.retry.RetryInvocationHandler.<init>(RetryInvocationHandler.java:75) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.io.retry.RetryInvocationHandler.<init>(RetryInvocationHandler.java:66) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.io.retry.RetryProxy.create(RetryProxy.java:58) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.hdfs.NameNodeProxies.createProxy(NameNodeProxies.java:185) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.hdfs.DFSClient.<init>(DFSClient.java:925) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.hdfs.DFSClient.<init>(DFSClient.java:855) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.hdfs.DistributedFileSystem.initialize(DistributedFileSystem.java:158) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.FileSystem.createFileSystem(FileSystem.java:2863) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.FileSystem.get(FileSystem.java:406) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.viewfs.ChRootedFileSystem.<init>(ChRootedFileSystem.java:109) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.viewfs.ViewFileSystem$1.getTargetFileSystem(ViewFileSystem.java:185) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.viewfs.ViewFileSystem$1.getTargetFileSystem(ViewFileSystem.java:179) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.viewfs.InodeTree.createLink(InodeTree.java:306) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.viewfs.InodeTree.<init>(InodeTree.java:417) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.viewfs.ViewFileSystem$1.<init>(ViewFileSystem.java:179) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.viewfs.ViewFileSystem.initialize(ViewFileSystem.java:179) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.FileSystem.createFileSystem(FileSystem.java:2863) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.FileSystem.get(FileSystem.java:406) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.FileSystem.get(FileSystem.java:192) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.FileSystem.get(FileSystem.java:389) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at org.apache.hadoop.fs.Path.getFileSystem(Path.java:295) ~[flink-shaded-hadoop-2-uber-2.7.1-mt-2.0.5-15.0.jar:2.7.1-mt-2.0.5-15.0]
	at com.meituan.datalinkstream.util.HiveUtil.getFileSystem(HiveUtil.java:567) ~[?:?]
	at com.meituan.datalinkstream.plugins.hive.writer.HiveBucketingProcessor.initFileSystem(HiveBucketingProcessor.java:369) ~[?:?]
	at com.meituan.datalinkstream.plugins.hive.writer.HiveBucketingProcessor.initializeState(HiveBucketingProcessor.java:713) ~[?:?]
	at org.apache.flink.streaming.util.functions.StreamingFunctionUtils.tryRestoreFunction(StreamingFunctionUtils.java:189) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.util.functions.StreamingFunctionUtils.restoreFunctionState(StreamingFunctionUtils.java:171) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.api.operators.AbstractUdfStreamOperator.initializeState(AbstractUdfStreamOperator.java:94) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.api.operators.StreamOperatorStateHandler.initializeOperatorState(StreamOperatorStateHandler.java:122) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.api.operators.AbstractStreamOperator.initializeState(AbstractStreamOperator.java:283) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.runtime.tasks.RegularOperatorChain.initializeStateAndOpenOperators(RegularOperatorChain.java:106) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.runtime.tasks.StreamTask.restoreGates(StreamTask.java:738) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.runtime.tasks.StreamTaskActionExecutor$SynchronizedStreamTaskActionExecutor.call(StreamTaskActionExecutor.java:100) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.runtime.tasks.StreamTask.restoreInternal(StreamTask.java:713) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.streaming.runtime.tasks.StreamTask.restore(StreamTask.java:679) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.runtime.taskmanager.Task.runWithSystemExitMonitoring(Task.java:937) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.runtime.taskmanager.Task.restoreAndInvoke(Task.java:906) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:730) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.runtime.taskmanager.Task.run(Task.java:550) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at java.lang.Thread.run(Thread.java:748) ~[?:1.8.0_312]
Caused by: java.lang.ClassNotFoundException: org.apache.mtkafka.common.security.meituan.MTPlainSaslServer
	at java.net.URLClassLoader.findClass(URLClassLoader.java:387) ~[?:1.8.0_312]
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418) ~[?:1.8.0_312]
	at org.apache.flink.util.FlinkUserCodeClassLoader.loadClassWithoutExceptionHandling(FlinkUserCodeClassLoader.java:67) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.util.ChildFirstClassLoader.loadClassWithoutExceptionHandling(ChildFirstClassLoader.java:74) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at org.apache.flink.util.FlinkUserCodeClassLoader.loadClass(FlinkUserCodeClassLoader.java:51) ~[flink-dist-1.16.1-mt001.jar:1.16.1-mt001]
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351) ~[?:1.8.0_312]
	... 52 more
```

# 二、深入问题

## 2.1 使用已关闭 ClassLoader

现象带来的直接启示是：加载 org/apache/mtkafka/common/security/meituan/MTPlainSaslServer 必然使用了一个已关闭的 ClassLoader。

原因：**ClassLoader close 只代表这个 ClassLoader 不能加载新的类了，但是已经加载过的类仍然可以被访问。**


```java
// UrlClassLoader#close 重点看注释

   /**
    * Closes this URLClassLoader, so that it can no longer be used to load
    * new classes or resources that are defined by this loader.
    * Classes and resources defined by any of this loader's parents in the
    * delegation hierarchy are still accessible. Also, any classes or resources
    * that are already loaded, are still accessible.
    * <p>
    * In the case of jar: and file: URLs, it also closes any files
    * that were opened by it. If another thread is loading a
    * class when the {@code close} method is invoked, then the result of
    * that load is undefined.
    * <p>
    * The method makes a best effort attempt to close all opened files,
    * by catching {@link IOException}s internally. Unchecked exceptions
    * and errors are not caught. Calling close on an already closed
    * loader has no effect.
    * <p>
    * @exception IOException if closing any file opened by this class loader
    * resulted in an IOException. Any such exceptions are caught internally.
    * If only one is caught, then it is re-thrown. If more than one exception
    * is caught, then the second and following exceptions are added
    * as suppressed exceptions of the first one caught, which is then re-thrown.
    *
    * @exception SecurityException if a security manager is set, and it denies
    *   {@link RuntimePermission}{@code ("closeClassLoader")}
    *
    * @since 1.7
    */
    public void close() throws IOException {
      	// ... ...
    }
```

## 2.2 类加载链路的前一个类必然使用了已关闭 ClassLoader

从 jdk src/hotspot/share/classfile/systemDictionary.cpp  中可以看到在 Find 和 Construct 一个对象实例时，**若没有显示指定 ClassLoader，则使用访问到该类的前一个类对应的类加载器**。

```cpp
// SystemDictionary::find_java_mirror_for_type

// Find or construct the Java mirror (java.lang.Class instance) for
// the given field type signature, as interpreted relative to the
// given class loader.  Handles primitives, void, references, arrays,
// and all other reflectable types, except method types.
// N.B.  Code in reflection should use this entry point.
Handle SystemDictionary::find_java_mirror_for_type(Symbol* signature,
                                                   Klass* accessing_klass,
                                                   SignatureStream::FailureMode failure_mode,
                                                   TRAPS) {

  Handle class_loader;

  // What we have here must be a valid field descriptor,
  // and all valid field descriptors are supported.
  // Produce the same java.lang.Class that reflection reports.
  if (accessing_klass != nullptr) {
    class_loader      = Handle(THREAD, accessing_klass->class_loader());
  }
  
  // ... ...
}
```

因此，推断 org.apache.mtkafka.common.security.meituan.MTPlainSaslServer$MTPlainSaslServerFactory 这个内部类 MTPlainSaslServerFactory 是已关闭 ClassLoader 加载。

**结合 2.1 和 2.2，大胆猜测并推断：某个类被已关闭 ClassLoader 加载，且被缓存，在 failover 后新的 ClassLoader 访问缓存，取到该类，并接着访问之前没访问过的字段引用或方法，导致使用已关闭 ClassLoader 去加载之前未加载的类，导致报错。**

## 2.3 MTPlainSaslServerFactory 是怎么被已关闭类加载的？

加载线程栈如下：
```
MTPlainSaslServerFactory loaded! Thread name: Source: kafkaReader -> LogParserProcessor -> hiveBucketingProcessor (4/10)#0
	java.lang.Thread.getStackTrace(Thread.java:1559)
	org.apache.mtkafka.common.security.meituan.MTPlainSaslServer$MTPlainSaslServerFactory.<clinit>(MTPlainSaslServer.java:150)
	java.lang.Class.forName0(Native Method)
	java.lang.Class.forName(Class.java:348)
	javax.security.sasl.Sasl.loadFactory(Sasl.java:447)
	javax.security.sasl.Sasl.getFactories(Sasl.java:651)
	javax.security.sasl.Sasl.getSaslServerFactories(Sasl.java:608)
	org.apache.hadoop.security.SaslRpcServer$FastSaslServerFactory.<init>(SaslRpcServer.java:376)
	org.apache.hadoop.security.SaslRpcServer.init(SaslRpcServer.java:184)
	org.apache.hadoop.ipc.RPC.getProtocolProxy(RPC.java:595)
	org.apache.hadoop.hdfs.NameNodeProxies.createNNProxyWithClientProtocol(NameNodeProxies.java:432)
	org.apache.hadoop.hdfs.NameNodeProxies.createNonHAProxy(NameNodeProxies.java:318)
	org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider$DefaultProxyFactory.createProxy(ConfiguredFailoverProxyProvider.java:68)
	org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider.getProxy(ConfiguredFailoverProxyProvider.java:152)
	org.apache.hadoop.io.retry.RetryInvocationHandler.<init>(RetryInvocationHandler.java:75)
	org.apache.hadoop.io.retry.RetryInvocationHandler.<init>(RetryInvocationHandler.java:66)
	org.apache.hadoop.io.retry.RetryProxy.create(RetryProxy.java:58)
	org.apache.hadoop.hdfs.NameNodeProxies.createProxy(NameNodeProxies.java:185)
	org.apache.hadoop.hdfs.DFSClient.<init>(DFSClient.java:925)
	org.apache.hadoop.hdfs.DFSClient.<init>(DFSClient.java:855)
	org.apache.hadoop.hdfs.DistributedFileSystem.initialize(DistributedFileSystem.java:158)
	org.apache.hadoop.fs.FileSystem.createFileSystem(FileSystem.java:2863)
	org.apache.hadoop.fs.FileSystem.get(FileSystem.java:406)
	org.apache.hadoop.fs.viewfs.ChRootedFileSystem.<init>(ChRootedFileSystem.java:109)
	org.apache.hadoop.fs.viewfs.ViewFileSystem$1.getTargetFileSystem(ViewFileSystem.java:185)
	org.apache.hadoop.fs.viewfs.ViewFileSystem$1.getTargetFileSystem(ViewFileSystem.java:179)
	org.apache.hadoop.fs.viewfs.InodeTree.createLink(InodeTree.java:306)
	org.apache.hadoop.fs.viewfs.InodeTree.<init>(InodeTree.java:417)
	org.apache.hadoop.fs.viewfs.ViewFileSystem$1.<init>(ViewFileSystem.java:179)
	org.apache.hadoop.fs.viewfs.ViewFileSystem.initialize(ViewFileSystem.java:179)
	org.apache.hadoop.fs.FileSystem.createFileSystem(FileSystem.java:2863)
	org.apache.hadoop.fs.FileSystem.get(FileSystem.java:406)
	org.apache.hadoop.fs.FileSystem.get(FileSystem.java:192)
	org.apache.hadoop.fs.FileSystem.get(FileSystem.java:389)
	org.apache.hadoop.fs.Path.getFileSystem(Path.java:295)
	com.meituan.datalinkstream.util.HiveUtil.getFileSystem(HiveUtil.java:567)
	com.meituan.datalinkstream.plugins.hive.writer.HiveBucketingProcessor.initFileSystem(HiveBucketingProcessor.java:369)
	com.meituan.datalinkstream.plugins.hive.writer.HiveBucketingProcessor.initializeState(HiveBucketingProcessor.java:713)
	org.apache.flink.streaming.util.functions.StreamingFunctionUtils.tryRestoreFunction(StreamingFunctionUtils.java:189)
	org.apache.flink.streaming.util.functions.StreamingFunctionUtils.restoreFunctionState(StreamingFunctionUtils.java:171)
	org.apache.flink.streaming.api.operators.AbstractUdfStreamOperator.initializeState(AbstractUdfStreamOperator.java:94)
	org.apache.flink.streaming.api.operators.StreamOperatorStateHandler.initializeOperatorState(StreamOperatorStateHandler.java:122)
	org.apache.flink.streaming.api.operators.AbstractStreamOperator.initializeState(AbstractStreamOperator.java:283)
	org.apache.flink.streaming.runtime.tasks.RegularOperatorChain.initializeStateAndOpenOperators(RegularOperatorChain.java:106)
	org.apache.flink.streaming.runtime.tasks.StreamTask.restoreGates(StreamTask.java:738)
	org.apache.flink.streaming.runtime.tasks.StreamTaskActionExecutor$SynchronizedStreamTaskActionExecutor.call(StreamTaskActionExecutor.java:100)
	org.apache.flink.streaming.runtime.tasks.StreamTask.restoreInternal(StreamTask.java:713)
	org.apache.flink.streaming.runtime.tasks.StreamTask.restore(StreamTask.java:679)
	org.apache.flink.runtime.taskmanager.Task.runWithSystemExitMonitoring(Task.java:937)
	org.apache.flink.runtime.taskmanager.Task.restoreAndInvoke(Task.java:906)
	org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:730)
	org.apache.flink.runtime.taskmanager.Task.run(Task.java:550)
	java.lang.Thread.run(Thread.java:748)
```

看以上线程栈，观察 Sasl#loadFactory 方法：

```java
// Sasl#loadFactory

    private static Object loadFactory(Provider p, String className)
        throws SaslException {
        // ... ...
            /*
             * Load the implementation class with the same class loader
             * that was used to load the provider.
             * In order to get the class loader of a class, the
             * caller's class loader must be the same as or an ancestor of
             * the class loader being returned. Otherwise, the caller must
             * have "getClassLoader" permission, or a SecurityException
             * will be thrown.
             */
            ClassLoader cl = p.getClass().getClassLoader();
            Class<?> implClass;
            implClass = Class.forName(className, true, cl);
            return implClass.newInstance();
        // ... ...
    }
```

这里使用了 Provider 的 ClassLoader。那么再去观察 Provider 的出处：

```java
// Sasl#getFactories

    private static Set<Object> getFactories(String serviceName) {
        // ... ...
      
      	// 重点！获取 providers
        Provider[] providers = Security.getProviders();
      
        for (int i = 0; i < providers.length; i++) {
          	// ... ...
            for (Enumeration<Object> e = providers[i].keys(); e.hasMoreElements(); ) {
                // ... ...
                if (currentKey.startsWith(serviceName)) {
                    // We should skip the currentKey if it contains a
                    // whitespace. The reason is: such an entry in the
                    // provider property contains attributes for the
                    // implementation of an algorithm. We are only interested
                    // in entries which lead to the implementation
                    // classes.
                    if (currentKey.indexOf(" ") < 0) {
                      	// 这里取了 providers
                        String className = providers[i].getProperty(currentKey);
                        				// 这里就是 loadFactory
                                fac = loadFactory(providers[i], className);
                                // ... ...
                        }
                    }
                }
            }
        }
        return Collections.unmodifiableSet(result);
    }
```

可以看到这里调用了 Security#getProviders 来获取所有 providers：

```java
// Security#getProviders

    public static Provider[] getProviders() {
        return Providers.getFullProviderList().toArray();
    }
```

调用 Providers#getFullProviderList：

```java
// Providers#getFullProviderList

    public static ProviderList getFullProviderList() {
        ProviderList list;
        synchronized (Providers.class) {
          	// 这是一处可以返回 ProviderList 的地方
            list = getThreadProviderList();
            if (list != null) {
                ProviderList newList = list.removeInvalid();
                if (newList != list) {
                    changeThreadProviderList(newList);
                    list = newList;
                }
                return list;
            }
        }
      	// 这是另外一处可以返回 ProviderList 的地方
        list = getSystemProviderList();
        ProviderList newList = list.removeInvalid();
        if (newList != list) {
            setSystemProviderList(newList);
            list = newList;
        }
        return list;
    }
```

可以看到有两处可以提供 providers 列表返回的地方：getThreadProviderList 和 getSystemProviderList。

我们应该关注哪个？

**先说结论：用的不是 getThreadProviderList，而是 getSystemProviderList。**

原因：

getThreadProviderList 是 ThreadLocal 类型

观察 Providers#beginThreadProviderList 注释说明了：仅用于 Jar 验证和 SunJSSE FIPS 模式。

看到 getSystemProviderList 直接返回了一个 static 成员，并在静态代码块中初始化，就看到了**类被缓存的迹象，也佐证了 2.2 小节后面的猜测。**

```java
    // current system-wide provider list
    // Note volatile immutable object, so no synchronization needed.
    private static volatile ProviderList providerList;

    static {
        // set providerList to empty list first in case initialization somehow
        // triggers a getInstance() call (although that should not happen)
        providerList = ProviderList.EMPTY;
        providerList = ProviderList.fromSecurityProperties();
    }
```

**而且 Providers 是被 BootstrapClassLoader 进行加载的，肯定是 Class&ClassLoader 全局唯一的。**

以上分析了 Provider 和 SaslServerFactory 的关系，现在来看看 MTPlainSaslServerFactory 是怎么把自己搞“坏”的？

通过开启 -verbose:class 查看类加载的详细信息：

```
// MTPlainSaslServerFactory 的加载

// ... ...
[Loaded org.apache.mtkafka.common.security.meituan.MTPlainLoginModule from file:/opt/flink/workdir/blobStore/job_000000000b0f12790000000000000001/blob_p-193f237956bdc195e9edcc0b6fda38196d19c7db-abbf5458a5961e489c6e41e2d7d21f92]
[Loaded org.apache.mtkafka.common.security.meituan.MTPlainSaslServerProvider from file:/opt/flink/workdir/blobStore/job_000000000b0f12790000000000000001/blob_p-193f237956bdc195e9edcc0b6fda38196d19c7db-abbf5458a5961e489c6e41e2d7d21f92]
[Loaded org.apache.mtkafka.common.security.meituan.MTPlainSaslServer$MTPlainSaslServerFactory from file:/opt/flink/workdir/blobStore/job_000000000b0f12790000000000000001/blob_p-193f237956bdc195e9edcc0b6fda38196d19c7db-abbf5458a5961e489c6e41e2d7d21f92]
// ... ...
```

只放了三行，这三行是距离加载到 MTPlainSaslServerFactory 路径上的“**最后一公里**”。因此从 MTPlainLoginModule 看起：

```java
// MTPlainLoginModule

public class MTPlainLoginModule implements LoginModule {
		// ... ...

    static {
        MTPlainSaslServerProvider.initialize();
    }
  
  	// ... ...
}
```

静态代码块初始化 MTPlainSaslServerProvider：

```java
// MTPlainSaslServerProvider

public class MTPlainSaslServerProvider extends Provider {
    private static final long serialVersionUID = 1L;

    protected MTPlainSaslServerProvider() {
        super("Simple SASL/PLAIN Server Provider", 1.0, "Simple SASL/PLAIN Server Provider for Kafka");
        super.put("SaslServerFactory." + MTPlainSaslServer.PLAIN_MECHANISM, MTPlainSaslServer.MTPlainSaslServerFactory.class.getName());
    }

    public static void initialize() {
        Security.addProvider(new MTPlainSaslServerProvider());
    }
}
```

initialize 方法 addProvider 最终会向上面提到的 ProviderList 中插入 MTPlainSaslServerProvider 实例，这就和本节前半段分析对上了。

另外实例化 MTPlainSaslServerProvider 是可以看到 <font color="#00dd00">MTPlainSaslServer.MTPlainSaslServerFactory.class.getName()</font>，这就是为什么 MTPlainSaslServerFactory 被加载但是没被实例化的原因，只访问了 class 信息。也是本小节一开始放的“加载线程栈”为什么只在 failover 后才会被调用打印的原因。

当然这里还访问了 MTPlainSaslServer.PLAIN_MECHANISM，那为什么不加载 MTPlainSaslServer，那按照 2.2 小节的理论也应该去加载 MTPlainSaslServer 才对啊。

不是这样的，因为 MTPlainSaslServer.PLAIN_MECHANISM 是**常量字符串，会被放到常量池中，编译阶段就完成了这个工作了。因此运行阶段直接去常量池获取即可，这是 JVM 的优化。**

好的，分析到这里，结合报错异常栈去看，已经很明晰了。

异常栈中 org.apache.hadoop.security.SaslRpcServer$FastSaslServerFactory 初始化时需要获取所有 SaslServerFactory，并调用 getMechanismNames：

```java
// SaslRpcServer$FastSaslServerFactory <init>

  private static class FastSaslServerFactory implements SaslServerFactory {
    
    FastSaslServerFactory(Map<String,?> props) {
      // 获取所有 SaslServerFactory
      final Enumeration<SaslServerFactory> factories =
          Sasl.getSaslServerFactories();
      while (factories.hasMoreElements()) {
        SaslServerFactory factory = factories.nextElement();
        // 调用 getMechanismNames
        for (String mech : factory.getMechanismNames(props)) {
          // ... ...
        }
      }
    }
```

所有的 SaslServerFactory 中就包括 MTPlainSaslServerFactory，而它就是通过被缓存的 MTPlainSaslServerProvider 对应的已关闭的 ClassLoader 进行加载，但是 MTPlainSaslServerFactory 已被加载过了，所以可以直接被找到。问题就出在接下来的这一步：**去调用 getMechanismNames 时访问 MTPlainSaslServer 的 LOGGER，需要加载 MTPlainSaslServer，该类在 ClassLoader 被关闭前没被加载，现在关闭后去加载，就导致找不到类。**

