---
title: Elasticsearch 6.3.2 启动过程
categories:
  - 大数据
tags:
  - elasticsearch
keywords: elasticsearch,elasticsearch启动流程
date: 2018-09-01 20:25:45
---



### 前言

本文探究Elasticsearch 6.3.2的启动流程

#### 环境准备

使用工具：IDEA，XMind

关于ES调试环境的搭建，可以参考前面的文章 《[教你编译调试Elasticsearch 6.3.2源码](https://mp.weixin.qq.com/s?__biz=MzI1NDU0MTE1NA==&mid=2247483676&idx=1&sn=1d88a883ce21d7dcacd073a8fa85dbfc&chksm=e9c2ed11deb56407879ba0b22a4ef96916f8a9e7931e1efb99df57991966a3dc475eb3e23101&mpshare=1&scene=1&srcid=0901DM6ZqcQqSqyujfIApsDj#rd)》

然后通过设置断点，从 `org.elasticsearch.bootstrap.ElasticSearch` 的入口函数开始，一步一步调试

![IDEA 2018.2 调试按钮](http://image.laijianfeng.org/20180901_131938.png)

上图为使用 IDEA 2018.2 进行调试的一个截图，左上角84行出红点为一个断点，1、2、3编号的3个按钮是较为常用的按钮，作用如下：

- 按钮1：step over，执行到下一行，遇到方法**不进入**方法内部
- 按钮2：step into，执行到下一句代码，遇到方法则**进入**方法内部
- 按钮3：Run to cursor，执行到下一个断点处，后面没有断点则执行到结束


#### 通过XMind记录ES启动流程的整个过程

![ES 6.3.2 启动流程](http://image.laijianfeng.org/ES_startup_process.jpg)


根据上图，作者大概地把ES启动流程分为四个阶段：

- Elasticsearch 解析 Command，加载配置
- Bootstrap 初始化，资源检查
- Node 创建节点
- Bootstrap 启动节点和保活线程




### Elasticsearch 解析 Command，加载配置

首先可以看一下入口方法 `Elasticsearch.main`：

```
    public static void main(final String[] args) throws Exception {
        System.setSecurityManager(new SecurityManager() {
            @Override
            public void checkPermission(Permission perm) {
                // grant all permissions so that we can later set the security manager to the one that we want
            }
        });
        LogConfigurator.registerErrorListener();
        final Elasticsearch elasticsearch = new Elasticsearch();
        int status = main(args, elasticsearch, Terminal.DEFAULT);
        if (status != ExitCodes.OK) {
            exit(status);
        }
    }
```

1.1, 创建 SecurityManager 安全管理器

> 关于 SecurityManager:   
> 安全管理器在Java语言中的作用就是**检查操作是否有权限执行**，通过则顺序进行，否则抛出一个异常   
> 网上一篇文章：[Java安全——安全管理器、访问控制器和类装载器](https://blog.csdn.net/wwwdc1012/article/details/82287474)

1.2, LogConfigurator.registerErrorListener() 注册侦听器

1.3, 创建Elasticsearch对象

Elasticsearch 入口类的继承关系如下：

![Elasticsearch 入口类的继承关系](http://image.laijianfeng.org/20180901_143515.png)

可以看到Elasticsearch继承了EnvironmentAwareCommand，Command，这几个类的功能简要介绍如下：

- Elasticsearch: This class starts elasticsearch.
- EnvironmentAwareCommand: A cli command which requires an `org.elasticsearch.env.Environment` to use current paths and settings
- Command: An action to execute within a cli.

可以看出Elasticsearch的一个重要作用是解析命令参数

执行带 `-h` 参数的Elasticsearch启动命令

![带参数的Elasticsearch启动命令](http://image.laijianfeng.org/20180901_144410.png)


可以发现这几个参数与 Cammand 类 和 Elasticsearch 的几个私有变量是对应的

Elasticsearch的构造函数如下：

```
Elasticsearch() {
    super("starts elasticsearch", () -> {}); // we configure logging later so we override the base class from configuring logging
    versionOption = parser.acceptsAll(Arrays.asList("V", "version"), "Prints elasticsearch version information and exits");
    daemonizeOption = parser.acceptsAll(Arrays.asList("d", "daemonize"), "Starts Elasticsearch in the background")
    	.availableUnless(versionOption);
    pidfileOption = parser.acceptsAll(Arrays.asList("p", "pidfile"), "Creates a pid file in the specified path on start")
        .availableUnless(versionOption).withRequiredArg().withValuesConvertedBy(new PathConverter());
    quietOption = parser.acceptsAll(Arrays.asList("q", "quiet"), "Turns off standard output/error streams logging in console")
        .availableUnless(versionOption).availableUnless(daemonizeOption);
}
```


1.4, 接着进入 `Command.main` 方法

该方法给当前Runtime类添加一个hook线程，该线程作用是：当Runtime异常关闭时打印异常信息

1.5, `Command.mainWithoutErrorHandling` 方法，根据命令行参数，打印或者设置参数，然后执行命令，有异常则抛出所有异常

1.6, `EnvironmentAwareCommand.execute`，确保 `es.path.data`, `es.path.home`, `es.path.logs` 等参数已设置，否则从 `System.properties` 中读取

```
putSystemPropertyIfSettingIsMissing(settings, "path.data", "es.path.data");
putSystemPropertyIfSettingIsMissing(settings, "path.home", "es.path.home");
putSystemPropertyIfSettingIsMissing(settings, "path.logs", "es.path.logs");

execute(terminal, options, createEnv(terminal, settings));
```

1.7, `EnvironmentAwareCommand.createEnv`，读取config下的配置文件`elasticsearch.yml`内容，收集plugins，bin，lib，modules等目录下的文件信息

createEnv最后返回一个 Environment 对象，执行结果如下

![EnvironmentAwareCommand.createEnv](http://image.laijianfeng.org/20180901_160825.png)

1.8, `Elasticsearch.execute` ，读取daemonize， pidFile，quiet 的值，并 确保配置的临时目录(temp)是有效目录


进入Bootstrap初始化阶段

```
Bootstrap.init(!daemonize, pidFile, quiet, initialEnv);
```

### Bootstrap初始化阶段 

#### Bootstrap.init

2.1, 进入 `Bootstrap.init`, This method is invoked by `Elasticsearch#main(String[])` to startup elasticsearch.

`INSTANCE = new Bootstrap();`, 创建一个Bootstrap对象作为类对象，该类构造函数会创建一个用户线程，添加到Runtime Hook中，进行 countDown 操作

```
 private final CountDownLatch keepAliveLatch = new CountDownLatch(1);

 /** creates a new instance */
    Bootstrap() {
        keepAliveThread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    keepAliveLatch.await();
                } catch (InterruptedException e) {
                }
            }
        }, "elasticsearch[keepAlive/" + Version.CURRENT + "]");
        keepAliveThread.setDaemon(false);
        // keep this thread alive (non daemon thread) until we shutdown
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                keepAliveLatch.countDown();
            }
        });
    }
```

> CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程执行完后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有框架服务之后执行。   
> CountDownLatch是通过一个计数器来实现的，计数器的初始化值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就相应得减1。当计数器到达0时，表示所有的线程都已完成任务，然后在闭锁上等待的线程就可以恢复执行任务。
> 更多介绍请看文章：[并发工具类 CountDownLatch](https://blog.csdn.net/wwwdc1012/article/details/82288473)

2.2, 加载 keystore 安全配置，keystore文件不存在则创建，保存；存在则解密，更新keystore

2.3, 根据已有的配置信息，创建一个Environment对象

2.4, LogConfigurator log4j日志配置

2.5, 检查pid文件是否存在，不存在则创建

> 关于 pid 文件：   
> (1) **pid文件的内容**：pid文件为文本文件，内容只有一行，记录了该进程的ID，用cat命令可以看到。   
> (2) **pid文件的作用**：防止进程启动多个副本。只有获得pid文件(固定路径固定文件名)写入权限(F_WRLCK)的进程才能正常启动并把自身的PID写入该文件中，其它同一个程序的多余进程则自动退出。

2.6, 检查Lucene版本与实际的Lucene Jar文件的版本是否一致，不一致则抛异常

2.7, 设置未捕获异常的处理 Thread.setDefaultUncaughtExceptionHandler

在Thread ApI中提供了UncaughtExceptionHandle，它能检测出某个由于未捕获的异常而终结的情况

> 朱小厮 [JAVA多线程之UncaughtExceptionHandler——处理非正常的线程中止](https://blog.csdn.net/u013256816/article/details/50417822)


#### INSTANCE.setup(true, environment); 

3.1，` spawner.spawnNativeControllers(environment);`

遍历每个模块，生成本机控制类（native Controller）：读取modules文件夹下所有的文件夹中的模块信息，保存为一个 PluginInfo  对象，为合适的模块生成控制类，通过 `Files.isRegularFile(spawnPath)` 来判断

尝试为给定模块生成控制器(native Controller)守护程序。    生成的进程将通过其stdin，stdout和stderr流保持与此JVM的连接，但对此包之外的代码不能使用对这些流的引用。


3.2， `initializeNatives(Path tmpFile, boolean mlockAll, boolean systemCallFilter, boolean ctrlHandler) `初始化本地资源

检查用户是否为root用户，是则抛异常;   
尝试启用 系统调用过滤器 system call filter;    
如果设置了则进行 mlockall    
Windows关闭事件监听器   
init lucene random seed.   

这个过程中使用到了 Natives 类:   
Natives类是一个包装类，用于检查调用本机方法所需的类是否在启动时可用。如果它们不可用，则此类将避免调用加载这些类的代码

3.3, 添加一个Hook： Runtime.getRuntime().addShutdownHook，当ES退出时用于关闭必要的IO流，日志器上下文和配置器等

3.4, 使用 JarHell 检查重复的 jar 文件

3.5, 初始化 SecurityManager

```
// install SM after natives, shutdown hooks, etc.
Security.configure(environment, BootstrapSettings.SECURITY_FILTER_BAD_DEFAULTS_SETTING.get(settings));
```

### 创建 node 节点

```
node = new Node(environment) {
    @Override
    protected void validateNodeBeforeAcceptingRequests(
        final BootstrapContext context,
        final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {
        BootstrapChecks.check(context, boundTransportAddress, checks);
    }
};
```

4.1, 这里直接贴一下代码（前半部分）

```
    protected Node(final Environment environment, Collection<Class<? extends Plugin>> classpathPlugins) {
        final List<Closeable> resourcesToClose = new ArrayList<>(); // register everything we need to release in the case of an error
        boolean success = false;
        {
            // use temp logger just to say we are starting. we can't use it later on because the node name might not be set
            Logger logger = Loggers.getLogger(Node.class, NODE_NAME_SETTING.get(environment.settings()));
            logger.info("initializing ...");
        }
        try {
            originalSettings = environment.settings();
            Settings tmpSettings = Settings.builder().put(environment.settings())
                .put(Client.CLIENT_TYPE_SETTING_S.getKey(), CLIENT_TYPE).build();

            // create the node environment as soon as possible, to recover the node id and enable logging
            try {
                nodeEnvironment = new NodeEnvironment(tmpSettings, environment);
                resourcesToClose.add(nodeEnvironment);
            } catch (IOException ex) {
                throw new IllegalStateException("Failed to create node environment", ex);
            }
            final boolean hadPredefinedNodeName = NODE_NAME_SETTING.exists(tmpSettings);
            final String nodeId = nodeEnvironment.nodeId();
            tmpSettings = addNodeNameIfNeeded(tmpSettings, nodeId);
            final Logger logger = Loggers.getLogger(Node.class, tmpSettings);
            // this must be captured after the node name is possibly added to the settings
            final String nodeName = NODE_NAME_SETTING.get(tmpSettings);
            if (hadPredefinedNodeName == false) {
                logger.info("node name derived from node ID [{}]; set [{}] to override", nodeId, NODE_NAME_SETTING.getKey());
            } else {
                logger.info("node name [{}], node ID [{}]", nodeName, nodeId);
            }

            final JvmInfo jvmInfo = JvmInfo.jvmInfo();
            logger.info(
                "version[{}], pid[{}], build[{}/{}/{}/{}], OS[{}/{}/{}], JVM[{}/{}/{}/{}]",
                Version.displayVersion(Version.CURRENT, Build.CURRENT.isSnapshot()),
                jvmInfo.pid(),
                Build.CURRENT.flavor().displayName(),
                Build.CURRENT.type().displayName(),
                Build.CURRENT.shortHash(),
                Build.CURRENT.date(),
                Constants.OS_NAME,
                Constants.OS_VERSION,
                Constants.OS_ARCH,
                Constants.JVM_VENDOR,
                Constants.JVM_NAME,
                Constants.JAVA_VERSION,
                Constants.JVM_VERSION);
            logger.info("JVM arguments {}", Arrays.toString(jvmInfo.getInputArguments()));
            warnIfPreRelease(Version.CURRENT, Build.CURRENT.isSnapshot(), logger);

            if (logger.isDebugEnabled()) {
                logger.debug("using config [{}], data [{}], logs [{}], plugins [{}]",
                    environment.configFile(), Arrays.toString(environment.dataFiles()), environment.logsFile(), environment.pluginsFile());
            }

            this.pluginsService = new PluginsService(tmpSettings, environment.configFile(), environment.modulesFile(), environment.pluginsFile(), classpathPlugins);
            this.settings = pluginsService.updatedSettings();
            localNodeFactory = new LocalNodeFactory(settings, nodeEnvironment.nodeId());

            // create the environment based on the finalized (processed) view of the settings
            // this is just to makes sure that people get the same settings, no matter where they ask them from
            this.environment = new Environment(this.settings, environment.configFile());
            Environment.assertEquivalent(environment, this.environment);

            final List<ExecutorBuilder<?>> executorBuilders = pluginsService.getExecutorBuilders(settings);

            final ThreadPool threadPool = new ThreadPool(settings, executorBuilders.toArray(new ExecutorBuilder[0]));
            resourcesToClose.add(() -> ThreadPool.terminate(threadPool, 10, TimeUnit.SECONDS));
            // adds the context to the DeprecationLogger so that it does not need to be injected everywhere
            DeprecationLogger.setThreadContext(threadPool.getThreadContext());
            resourcesToClose.add(() -> DeprecationLogger.removeThreadContext(threadPool.getThreadContext()));

            final List<Setting<?>> additionalSettings = new ArrayList<>(pluginsService.getPluginSettings());
            final List<String> additionalSettingsFilter = new ArrayList<>(pluginsService.getPluginSettingsFilter());
            for (final ExecutorBuilder<?> builder : threadPool.builders()) {
                additionalSettings.addAll(builder.getRegisteredSettings());
            }
            client = new NodeClient(settings, threadPool);
    ...
```

这里进行的主要操作有:

1. 生命周期Lifecycle设置为 初始化状态 INITIALIZED
2. 创建一个 NodeEnvironment 对象保存节点环境信息，如各种数据文件的路径
3. 读取JVM信息
4. 创建 PluginsService 对象，创建过程中会读取并加载所有的模块和插件
5. 创建一个最终的 Environment 对象
6. 创建线程池 ThreadPool 后面各类对象基本都是通过线程来提供服务，这个线程池可以管理各类线程
7. 创建 节点客户端 NodeClient


**这里重点介绍 PluginsService 和 ThreadPool 这两个类**

#### PluginsService

在构造该类对象是传入的参数如下：

![PluginsService 构造方法的参数](http://image.laijianfeng.org/20180901_175542.png)

在构造方法中加载所有的模块

```
Set<Bundle> seenBundles = new LinkedHashSet<>();
List<PluginInfo> modulesList = new ArrayList<>();

Set<Bundle> modules = getModuleBundles(modulesDirectory); 

for (Bundle bundle : modules) {
   modulesList.add(bundle.plugin);
}
seenBundles.addAll(modules);

/** Get bundles for plugins installed in the given modules directory. */
static Set<Bundle> getModuleBundles(Path modulesDirectory) throws IOException {
    return findBundles(modulesDirectory, "module").stream().flatMap(b -> b.bundles().stream()).collect(Collectors.toSet());
}
```

其中的 Bundle是一个内部类（a "bundle" is a group of plugins in a single classloader）   
而 PluginInfo 则是 An in-memory representation of the plugin descriptor. 存在内存中的用来描述一个 plugin 的类


插件加载的实际代码如下：

```
    /**
     * Reads the plugin descriptor file.
     *
     * @param path           the path to the root directory for the plugin
     * @return the plugin info
     * @throws IOException if an I/O exception occurred reading the plugin descriptor
     */
    public static PluginInfo readFromProperties(final Path path) throws IOException {
        final Path descriptor = path.resolve(ES_PLUGIN_PROPERTIES);

        final Map<String, String> propsMap;
        {
            final Properties props = new Properties();
            try (InputStream stream = Files.newInputStream(descriptor)) {
                props.load(stream);
            }
            propsMap = props.stringPropertyNames().stream().collect(Collectors.toMap(Function.identity(), props::getProperty));
        }

        final String name = propsMap.remove("name");
        if (name == null || name.isEmpty()) {
            throw new IllegalArgumentException(
                    "property [name] is missing in [" + descriptor + "]");
        }
        final String description = propsMap.remove("description");
        if (description == null) {
            throw new IllegalArgumentException(
                    "property [description] is missing for plugin [" + name + "]");
        }
        final String version = propsMap.remove("version");
        if (version == null) {
            throw new IllegalArgumentException(
                    "property [version] is missing for plugin [" + name + "]");
        }

        final String esVersionString = propsMap.remove("elasticsearch.version");
        if (esVersionString == null) {
            throw new IllegalArgumentException(
                    "property [elasticsearch.version] is missing for plugin [" + name + "]");
        }
        final Version esVersion = Version.fromString(esVersionString);
        final String javaVersionString = propsMap.remove("java.version");
        if (javaVersionString == null) {
            throw new IllegalArgumentException(
                    "property [java.version] is missing for plugin [" + name + "]");
        }
        JarHell.checkVersionFormat(javaVersionString);
        final String classname = propsMap.remove("classname");
        if (classname == null) {
            throw new IllegalArgumentException(
                    "property [classname] is missing for plugin [" + name + "]");
        }

        final String extendedString = propsMap.remove("extended.plugins");
        final List<String> extendedPlugins;
        if (extendedString == null) {
            extendedPlugins = Collections.emptyList();
        } else {
            extendedPlugins = Arrays.asList(Strings.delimitedListToStringArray(extendedString, ","));
        }

        final String hasNativeControllerValue = propsMap.remove("has.native.controller");
        final boolean hasNativeController;
        if (hasNativeControllerValue == null) {
            hasNativeController = false;
        } else {
            switch (hasNativeControllerValue) {
                case "true":
                    hasNativeController = true;
                    break;
                case "false":
                    hasNativeController = false;
                    break;
                default:
                    final String message = String.format(
                            Locale.ROOT,
                            "property [%s] must be [%s], [%s], or unspecified but was [%s]",
                            "has_native_controller",
                            "true",
                            "false",
                            hasNativeControllerValue);
                    throw new IllegalArgumentException(message);
            }
        }

        if (esVersion.before(Version.V_6_3_0) && esVersion.onOrAfter(Version.V_6_0_0_beta2)) {
            propsMap.remove("requires.keystore");
        }

        if (propsMap.isEmpty() == false) {
            throw new IllegalArgumentException("Unknown properties in plugin descriptor: " + propsMap.keySet());
        }

        return new PluginInfo(name, description, version, esVersion, javaVersionString,
                              classname, extendedPlugins, hasNativeController);
    }
```

其中的两个常量的值

```
    public static final String ES_PLUGIN_PROPERTIES = "plugin-descriptor.properties";
    public static final String ES_PLUGIN_POLICY = "plugin-security.policy";
```

从以上代码可以看出**模块的加载过程**：

1. 读取模块的配置文件 `plugin-descriptor.properties`，解析出内容并存储到 Map 中
2. 分别校验 `name`, `description`, `version`, `elasticsearch.version`, `java.version`, `classname`, `extended.plugins`, `has.native.controller`, `requires.keystore` 这些配置项，缺失或者不按要求则抛出异常
3. 根据配置项构造一个 PluginInfo 对象返回

举例：读取出的 aggs-matrix-stats 模块的配置项信息如下

![读取插件配置文件并解析文件内容](http://image.laijianfeng.org/20180901_181500.png)


加载插件与加载模块调用的是相同的方法



### ThreadPool 线程池

线程池的构造方法如下：

```
    public ThreadPool(final Settings settings, final ExecutorBuilder<?>... customBuilders) {
        super(settings);

        assert Node.NODE_NAME_SETTING.exists(settings);

        final Map<String, ExecutorBuilder> builders = new HashMap<>();
        final int availableProcessors = EsExecutors.numberOfProcessors(settings);
        final int halfProcMaxAt5 = halfNumberOfProcessorsMaxFive(availableProcessors);
        final int halfProcMaxAt10 = halfNumberOfProcessorsMaxTen(availableProcessors);
        final int genericThreadPoolMax = boundedBy(4 * availableProcessors, 128, 512);
        
        builders.put(Names.GENERIC, new ScalingExecutorBuilder(Names.GENERIC, 4, genericThreadPoolMax, TimeValue.timeValueSeconds(30)));
        builders.put(Names.INDEX, new FixedExecutorBuilder(settings, Names.INDEX, availableProcessors, 200, true));
        builders.put(Names.WRITE, new FixedExecutorBuilder(settings, Names.WRITE, "bulk", availableProcessors, 200));
        builders.put(Names.GET, new FixedExecutorBuilder(settings, Names.GET, availableProcessors, 1000));
        builders.put(Names.ANALYZE, new FixedExecutorBuilder(settings, Names.ANALYZE, 1, 16));
        builders.put(Names.SEARCH, new AutoQueueAdjustingExecutorBuilder(settings,
                        Names.SEARCH, searchThreadPoolSize(availableProcessors), 1000, 1000, 1000, 2000));
        builders.put(Names.MANAGEMENT, new ScalingExecutorBuilder(Names.MANAGEMENT, 1, 5, TimeValue.timeValueMinutes(5)));
        // no queue as this means clients will need to handle rejections on listener queue even if the operation succeeded
        // the assumption here is that the listeners should be very lightweight on the listeners side
        builders.put(Names.LISTENER, new FixedExecutorBuilder(settings, Names.LISTENER, halfProcMaxAt10, -1));
        builders.put(Names.FLUSH, new ScalingExecutorBuilder(Names.FLUSH, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
        builders.put(Names.REFRESH, new ScalingExecutorBuilder(Names.REFRESH, 1, halfProcMaxAt10, TimeValue.timeValueMinutes(5)));
        builders.put(Names.WARMER, new ScalingExecutorBuilder(Names.WARMER, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
        builders.put(Names.SNAPSHOT, new ScalingExecutorBuilder(Names.SNAPSHOT, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
        builders.put(Names.FETCH_SHARD_STARTED, new ScalingExecutorBuilder(Names.FETCH_SHARD_STARTED, 1, 2 * availableProcessors, TimeValue.timeValueMinutes(5)));
        builders.put(Names.FORCE_MERGE, new FixedExecutorBuilder(settings, Names.FORCE_MERGE, 1, -1));
        builders.put(Names.FETCH_SHARD_STORE, new ScalingExecutorBuilder(Names.FETCH_SHARD_STORE, 1, 2 * availableProcessors, TimeValue.timeValueMinutes(5)));
        
        for (final ExecutorBuilder<?> builder : customBuilders) {
            if (builders.containsKey(builder.name())) {
                throw new IllegalArgumentException("builder with name [" + builder.name() + "] already exists");
            }
            builders.put(builder.name(), builder);
        }
        this.builders = Collections.unmodifiableMap(builders);

        threadContext = new ThreadContext(settings);

        final Map<String, ExecutorHolder> executors = new HashMap<>();
        for (@SuppressWarnings("unchecked") final Map.Entry<String, ExecutorBuilder> entry : builders.entrySet()) {
            final ExecutorBuilder.ExecutorSettings executorSettings = entry.getValue().getSettings(settings);
            final ExecutorHolder executorHolder = entry.getValue().build(executorSettings, threadContext);
            if (executors.containsKey(executorHolder.info.getName())) {
                throw new IllegalStateException("duplicate executors with name [" + executorHolder.info.getName() + "] registered");
            }
            logger.debug("created thread pool: {}", entry.getValue().formatInfo(executorHolder.info));
            executors.put(entry.getKey(), executorHolder);
        }

        executors.put(Names.SAME, new ExecutorHolder(DIRECT_EXECUTOR, new Info(Names.SAME, ThreadPoolType.DIRECT)));
        this.executors = unmodifiableMap(executors);
        this.scheduler = Scheduler.initScheduler(settings);
        TimeValue estimatedTimeInterval = ESTIMATED_TIME_INTERVAL_SETTING.get(settings);
        this.cachedTimeThread = new CachedTimeThread(EsExecutors.threadName(settings, "[timer]"), estimatedTimeInterval.millis());
        this.cachedTimeThread.start();
    }
```

参考着文档来理解这里的代码：[Elasticsearch Reference [6.4] » Modules » Thread Pool](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html) 和 [apachecn 线程池](http://cwiki.apachecn.org/pages/viewpage.action?pageId=9405389)

#### 线程池类型 ThreadPoolType

**fixed**（固定）：fixed线程池拥有固定数量的线程来处理请求，在没有空闲线程时请求将被挂在队列中。queue_size参数可以控制在没有空闲线程时，能排队挂起的请求数

**fixed\_auto\_queue_size**：此类型为实验性的，将被更改或删除，不关注

**scaling**（弹性）：scaling线程池拥有的线程数量是动态的，这个数字介于core和max参数的配置之间变化。keep_alive参数用来控制线程在线程池中空闲的最长时间

**direct**：此类线程是一种不支持关闭的线程,就意味着一旦使用,则会一直存活下去.

#### 一些重要的线程池


**generic**：用于通用的请求（例如：后台节点发现），线程池类型为 scaling。

**index**：用于index/delete请求，线程池类型为 fixed， 大小的为处理器数量，队列大小为200，最大线程数为 1 + 处理器数量。

**search**：用于count/search/suggest请求。线程池类型为 fixed， 大小的为 int((处理器数量 3) / 2) +1，队列大小为1000。*

**get**：用于get请求。线程池类型为 fixed，大小的为处理器数量，队列大小为1000。

**analyze**：用于analyze请求。线程池类型为 fixed，大小的1，队列大小为16

**write**：用于单个文档的 index/delete/update 请求以及 **bulk 请求**，线程池类型为 fixed，大小的为处理器数量，队列大小为200，最大线程数为 1 + 处理器数量。

**snapshot**：用于snaphost/restore请求。线程池类型为 scaling，线程保持存活时间为5分钟，最大线程数为min(5, (处理器数量)/2)。

**warmer**：用于segment warm-up请求。线程池类型为 scaling，线程保持存活时间为5分钟，最大线程数为min(5, (处理器数量)/2)。

**refresh**：用于refresh请求。线程池类型为 scaling，线程空闲保持存活时间为5分钟，最大线程数为min(10, (处理器数量)/2)。

**listener**：主要用于Java客户端线程监听器被设置为true时执行动作。线程池类型为 scaling，最大线程数为min(10, (处理器数量)/2)。


ThreadPool 类中除了以上线程队列，还可以看到有 CachedTimeThread（缓存系统时间）、ExecutorService（在当前线程上执行提交的任务）、ThreadContext（线程上下文）、ScheduledThreadPoolExecutor（Java任务调度）等

> 参考文章：[Java并发编程14-ScheduledThreadPoolExecutor详解](https://my.oschina.net/u/3145136/blog/848079)   
> [Java线程池原理分析ScheduledThreadPoolExecutor篇](https://www.jianshu.com/p/4b8a257f1b90)   
> 关于 ScheduledThreadPoolExecutor 更多的细节应该看书或者官方文档

#### 关于线程

了解了线程池，继续深究ES线程是什么样子的

在 `ScalingExecutorBuilder.build` 中可以发现 `ExecutorService` 对象是由 `EsExecutors.newScaling` 创建的

```
public static EsThreadPoolExecutor newScaling(String name, int min, int max, long keepAliveTime, TimeUnit unit, ThreadFactory threadFactory, ThreadContext contextHolder) {
    ExecutorScalingQueue<Runnable> queue = new ExecutorScalingQueue<>();
    EsThreadPoolExecutor executor = new EsThreadPoolExecutor(name, min, max, keepAliveTime, unit, queue, threadFactory, new ForceQueuePolicy(), contextHolder);
    queue.executor = executor;
    return executor;
}
```

再看看 `EsThreadPoolExecutor` 这个类的继承关系，其是扩展自Java的线程池 `ThreadPoolExecutor`

![EsThreadPoolExecutor的继承链](http://image.laijianfeng.org/20180901_192545.png)

```
    EsThreadPoolExecutor(String name, int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,
            BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, XRejectedExecutionHandler handler,
            ThreadContext contextHolder) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
        this.name = name;
        this.contextHolder = contextHolder;
    }
```


### 回到 Node 节点的创建

4.2, 创建各种服务类对象 ResourceWatcherService、NetworkService、ClusterService、IngestService、ClusterInfoService、UsageService、MonitorService、CircuitBreakerService、MetaStateService、IndicesService、MetaDataIndexUpgradeService、TemplateUpgradeService、TransportService、ResponseCollectorService、SearchTransportService、NodeService、SearchService、PersistentTasksClusterService

这些服务类是的功能可以根据名称做一个大概的判断，具体还需要看文档和源码，限于篇幅，在此不做探究

4.3, ModulesBuilder类加入各种模块 ScriptModule、AnalysisModule、SettingsModule、pluginModule、ClusterModule、IndicesModule、SearchModule、GatewayModule、RepositoriesModule、ActionModule、NetworkModule、DiscoveryModule


4.4, guice 绑定依赖以及依赖注入

> 关于 guice 可以参考之前的文章:   
> [Google Guice 快速入门](https://mp.weixin.qq.com/s?__biz=MzI1NDU0MTE1NA==&mid=2247483683&idx=1&sn=0d77085a0234b2c5b7c679e62200e6f5&chksm=e9c2ed2edeb56438010b5f5d487bcb7f0529c85d50ac7c858e1a8e3a9279c15007341170c5ac&mpshare=1&scene=1&srcid=0901SWQiIdjHZ3endUHZqjkP#rd)   
> [Elasticsearch 中的 Guice](https://mp.weixin.qq.com/s?__biz=MzI1NDU0MTE1NA==&mid=2247483691&idx=1&sn=3c7175d318bce6728c2105d27ae6bafe&chksm=e9c2ed26deb56430289edabd15cef1a0cf777c5dfe4f4ad5013655e9d3607958e0fe16ac5436&mpshare=1&scene=1&srcid=0901u1aslOFhg6UmeBkLw8BY#rd)

elasticsearch里面的组件基本都进行进行了模块化管理，elasticsearch对guice进行了封装，通过ModulesBuilder类构建es的模块（一般包括的模块在 4.3 中列举了）

```
// 依赖绑定
modules.add(b -> {
        b.bind(Node.class).toInstance(this);
        b.bind(NodeService.class).toInstance(nodeService);
        b.bind(NamedXContentRegistry.class).toInstance(xContentRegistry);
        b.bind(PluginsService.class).toInstance(pluginsService);
        b.bind(Client.class).toInstance(client);
        b.bind(NodeClient.class).toInstance(client);
        b.bind(Environment.class).toInstance(this.environment);
        b.bind(ThreadPool.class).toInstance(threadPool);
        b.bind(NodeEnvironment.class).toInstance(nodeEnvironment);
        b.bind(ResourceWatcherService.class).toInstance(resourceWatcherService);
        b.bind(CircuitBreakerService.class).toInstance(circuitBreakerService);
        b.bind(BigArrays.class).toInstance(bigArrays);
        b.bind(ScriptService.class).toInstance(scriptModule.getScriptService());
        b.bind(AnalysisRegistry.class).toInstance(analysisModule.getAnalysisRegistry());
        b.bind(IngestService.class).toInstance(ingestService);
        b.bind(UsageService.class).toInstance(usageService);
        b.bind(NamedWriteableRegistry.class).toInstance(namedWriteableRegistry);
        b.bind(MetaDataUpgrader.class).toInstance(metaDataUpgrader);
        b.bind(MetaStateService.class).toInstance(metaStateService);
        b.bind(IndicesService.class).toInstance(indicesService);
        b.bind(SearchService.class).toInstance(searchService);
        b.bind(SearchTransportService.class).toInstance(searchTransportService);
        b.bind(SearchPhaseController.class).toInstance(new SearchPhaseController(settings,
            searchService::createReduceContext));
        b.bind(Transport.class).toInstance(transport);
        b.bind(TransportService.class).toInstance(transportService);
        b.bind(NetworkService.class).toInstance(networkService);
        b.bind(UpdateHelper.class).toInstance(new UpdateHelper(settings, scriptModule.getScriptService()));
        b.bind(MetaDataIndexUpgradeService.class).toInstance(metaDataIndexUpgradeService);
        b.bind(ClusterInfoService.class).toInstance(clusterInfoService);
        b.bind(GatewayMetaState.class).toInstance(gatewayMetaState);
        b.bind(Discovery.class).toInstance(discoveryModule.getDiscovery());
        {
            RecoverySettings recoverySettings = new RecoverySettings(settings, settingsModule.getClusterSettings());
            processRecoverySettings(settingsModule.getClusterSettings(), recoverySettings);
            b.bind(PeerRecoverySourceService.class).toInstance(new PeerRecoverySourceService(settings, transportService,
                    indicesService, recoverySettings));
            b.bind(PeerRecoveryTargetService.class).toInstance(new PeerRecoveryTargetService(settings, threadPool,
                    transportService, recoverySettings, clusterService));
        }
        httpBind.accept(b);
        pluginComponents.stream().forEach(p -> b.bind((Class) p.getClass()).toInstance(p));
        b.bind(PersistentTasksService.class).toInstance(persistentTasksService);
        b.bind(PersistentTasksClusterService.class).toInstance(persistentTasksClusterService);
        b.bind(PersistentTasksExecutorRegistry.class).toInstance(registry);
    }
);
injector = modules.createInjector();
```


### Bootstrap 启动

5.1， 通过 `injector` 获取各个类的对象，调用 `start()` 方法启动（实际进入各个类的中 `doStart` 方法）: LifecycleComponent、IndicesService、IndicesClusterStateService、SnapshotsService、SnapshotShardsService、RoutingService、SearchService、MonitorService、NodeConnectionsService、ResourceWatcherService、GatewayService、Discovery、TransportService


这里简要介绍一下各个服务类的职能：

IndicesService：索引管理   
IndicesClusterStateService：跨集群同步   
SnapshotsService：负责创建快照   
SnapshotShardsService：此服务在数据和主节点上运行，并控制这些节点上当前快照的分片。 它负责启动和停止分片级别快照   
RoutingService：侦听集群状态，当它收到ClusterChangedEvent（集群改变事件）将验证集群状态，路由表可能会更新   
SearchService：搜索服务   
MonitorService：监控   
NodeConnectionsService：此组件负责在节点添加到群集状态后连接到节点，并在删除它们时断开连接。 此外，它会定期检查所有连接是否仍处于打开状态，并在需要时还原它们。 请注意，如果节点断开/不响应ping，则此组件不负责从群集中删除节点。 这是由NodesFaultDetection完成的。 主故障检测由链接MasterFaultDetection完成。     
ResourceWatcherService：通用资源观察器服务   
GatewayService：网关



如果该节点是主节点或数据节点，还需要进行相关的职能操作

5.2, 集群发现与监控等，启动 HttpServerTransport， 绑定服务端口

```
validateNodeBeforeAcceptingRequests(new BootstrapContext(settings, onDiskMetadata), transportService.boundAddress(), pluginsService
    .filterPlugins(Plugin
    .class)
    .stream()
    .flatMap(p -> p.getBootstrapChecks().stream()).collect(Collectors.toList()));

clusterService.addStateApplier(transportService.getTaskManager());
// start after transport service so the local disco is known
discovery.start(); // start before cluster service so that it can set initial state on ClusterApplierService
clusterService.start();
assert clusterService.localNode().equals(localNodeFactory.getNode())
    : "clusterService has a different local node than the factory provided";
transportService.acceptIncomingRequests();
discovery.startInitialJoin();
// tribe nodes don't have a master so we shouldn't register an observer         s
final TimeValue initialStateTimeout = DiscoverySettings.INITIAL_STATE_TIMEOUT_SETTING.get(settings);
if (initialStateTimeout.millis() > 0) {
    final ThreadPool thread = injector.getInstance(ThreadPool.class);
    ClusterState clusterState = clusterService.state();
    ClusterStateObserver observer = new ClusterStateObserver(clusterState, clusterService, null, logger, thread.getThreadContext());
    if (clusterState.nodes().getMasterNodeId() == null) {
        logger.debug("waiting to join the cluster. timeout [{}]", initialStateTimeout);
        final CountDownLatch latch = new CountDownLatch(1);
        observer.waitForNextChange(new ClusterStateObserver.Listener() {
            @Override
            public void onNewClusterState(ClusterState state) { latch.countDown(); }

            @Override
            public void onClusterServiceClose() {
                latch.countDown();
            }

            @Override
            public void onTimeout(TimeValue timeout) {
                logger.warn("timed out while waiting for initial discovery state - timeout: {}",
                    initialStateTimeout);
                latch.countDown();
            }
        }, state -> state.nodes().getMasterNodeId() != null, initialStateTimeout);

        try {
            latch.await();
        } catch (InterruptedException e) {
            throw new ElasticsearchTimeoutException("Interrupted while waiting for initial discovery state");
        }
    }
}


if (NetworkModule.HTTP_ENABLED.get(settings)) {
    injector.getInstance(HttpServerTransport.class).start();
}

if (WRITE_PORTS_FILE_SETTING.get(settings)) {
    if (NetworkModule.HTTP_ENABLED.get(settings)) {
        HttpServerTransport http = injector.getInstance(HttpServerTransport.class);
        writePortsFile("http", http.boundAddress());
    }
    TransportService transport = injector.getInstance(TransportService.class);
    writePortsFile("transport", transport.boundAddress());
}

```

5.3, 启动保活线程 keepAliveThread.start 进行心跳检测



### 小结

过程很漫长，后面很多类的功能未了解，之后补上

有理解错误的地方请大家多多指教

-----

打开微信扫一扫，关注【小旋锋】微信公众号，及时接收博文推送

![小旋锋的微信公众号](http://image.laijianfeng.org/%E5%B0%8F%E6%97%8B%E9%94%8B%E7%9A%84%E5%BE%AE%E4%BF%A1%E5%85%AC%E4%BC%97%E5%8F%B7_%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)