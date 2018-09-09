---
title: Elasticsearch 中的 Guice
categories:
  - 大数据
tags:
  - elasticsearch
keywords: elasticsearch,guice
date: 2018-08-31 00:39:17
---

### 前言

Elasticsearch 源代码中使用了Guice框架进行依赖注入. 为了方便阅读源码, 此处我先通过模仿ES guice的使用方式简单写了一个基本Demo 方便理解, 之后再来理一下ES的Guice使用. 编写的测试类原理图如下:

![ES Guice Demo](http://image.laijianfeng.org/2778947-7847f61cf5180bdc.webp)


总共有两个Module，一个是ToolModule，**用于绑定**IAnimal接口、ITool接口以及Map对象. 另一个是HumanModule 用于绑定Person对象。   

其中Person的构造函数通过 `@Inject` 注解注入其他实例

gradle 需要引入的 Jar 包

```
compile group: 'com.google.inject.extensions', name: 'guice-multibindings', version: '4.2.0'
compile group: 'com.google.inject', name: 'guice', version: '4.2.0'
```

### 1、Demo

#### iTool接口与实现类

```
public interface ITool {
    public void doWork();
}
```

```
import com.whirly.guice.example.ITool;

public class IToolImpl implements ITool {
    @Override
    public void doWork() {
        System.out.println("use tool to work");
    }
}
```

#### IAnimal 接口与实现类

```
public interface IAnimal {
    void work();
}
```

```
public class IAnimalImpl implements IAnimal {
    @Override
    public void work() {
        System.out.println("animals can also do work");
    }
}
```

#### ToolModule的实现, 它绑了三个实例

```
public class ToolModule extends AbstractModule {

    @Override
    protected void configure() {
        //此处注入的实例可以注入到其他类的构造函数中, 只要那个类使用@Inject进行注入即可
        bind(IAnimal.class).to(IAnimalImpl.class);
        bind(ITool.class).to(IToolImpl.class);

        // 注入Map实例
        MapBinder<String, String> mapBinder = MapBinder.newMapBinder(binder(), String.class, String.class);
        mapBinder.addBinding("test1").toInstance("test1");
        mapBinder.addBinding("test2").toInstance("test2");
    }
}
```

`bind(IAnimal.class).to(IAnimalImpl.class);bind(ITool.class).to(IToolImpl.class);`  是将接口与其具体实现绑定起来

`MapBinder<String,String> mapBinder =MapBinder.newMapBinder(binder(), String.class, String.class);mapBinder.addBinding("test1").toInstance("test1");mapBinder.addBinding("test2").toInstance("test2");` 则是完成Map的绑定. 

后面来看看Person类和HumanModule

#### Person 类

```
public class Person {

    private IAnimal iAnimal;
    private ITool iTool;
    private Map<String, String> map;

    @Inject
    public Person(IAnimal iAnimal, ITool iTool, Map<String, String> map) {
        this.iAnimal = iAnimal;
        this.iTool = iTool;
        this.map = map;
    }

    public void startwork() {
        iTool.doWork();
        iAnimal.work();
        for (Map.Entry entry : map.entrySet()) {
            System.out.println("注入的map 是 " + entry.getKey() + " value " + entry.getValue());
        }
    }
}
```

Person 类中由 `IAnimal`、`ITool` 和 `Map<String, String>` 这三个接口定义的变量，对象将通过 `@Inject` 从构造方法中注入进来

```
public class HumanModule extends AbstractModule {
    @Override
    protected void configure() {
        bind(Person.class).asEagerSingleton();
    }
}
```

Person类的构造函数是通过注入的方式，注入对象实例的

最后 `CustomModuleBuilder` 进行**统一管理所有的Module**，实例化所有Module中的对象. 完成依赖注入。

这里的CustomModuleBuilder是修改自Elasticsearch中的ModulesBuilder，其原理是一样的。

就是一个迭代器，**内部封装的是Module集合, 统一管理所有的Module**


#### CustomModuleBuilder 统一管理 Module

```
public class CustomModuleBuilder implements Iterable<Module> {

    private final List<Module> modules = new ArrayList<>();

    public CustomModuleBuilder add(Module... newModules) {
        for (Module module : newModules) {
            modules.add(module);
        }
        return this;
    }

    @Override
    public Iterator<Module> iterator() {
        return modules.iterator();
    }

    public Injector createInjector() {
        Injector injector = Guice.createInjector(modules);
        return injector;
    }
}
```

这样就可以从Main方法是如何进行使用的

#### Main 方法

```
public class Main {
    public static void main(String[] args) {
        CustomModuleBuilder moduleBuilder = new CustomModuleBuilder();
        moduleBuilder.add(new ToolModule());
        moduleBuilder.add(new HumanModule());
        Injector injector = moduleBuilder.createInjector();
        Person person = injector.getInstance(Person.class);
        person.startwork();
    }
}
```

运行结果

```
use tool to work
animals can also do work
注入的map 是 test1 value test1
注入的map 是 test2 value test2
```

通过CustomModuleBuilder 的createInjector获取Injector 对象, 根据Injector 对象取相应的具体实例对象.

### 2、ES 中Guice的使用

ES中TransportClient初始化时的Guice的使用是这样的, 如下图所示


![ES中TransportClient初始化时的Guice的使用（ES版本不是6.3.2）](http://image.laijianfeng.org/2778947-ab2035e865492a2b.png)

#### TransportClient的初始化代码

Elasticsearch 6.3.2 

```
private static ClientTemplate buildTemplate(Settings providedSettings, Settings defaultSettings,
                                            Collection<Class<? extends Plugin>> plugins, HostFailureListener failureListner) {
    // 省略 ...
    try {
        // 省略 ...

        // 创建一个迭代器, 然后将各个Module通过add方法加入进去
        ModulesBuilder modules = new ModulesBuilder();
        // plugin modules must be added here, before others or we can get crazy injection errors...
        for (Module pluginModule : pluginsService.createGuiceModules()) {
            modules.add(pluginModule);
        }
        modules.add(b -> b.bind(ThreadPool.class).toInstance(threadPool));
        ActionModule actionModule = new ActionModule(true, settings, null, settingsModule.getIndexScopedSettings(),
                settingsModule.getClusterSettings(), settingsModule.getSettingsFilter(), threadPool,
                pluginsService.filterPlugins(ActionPlugin.class), null, null, null);
        modules.add(actionModule);

        CircuitBreakerService circuitBreakerService = Node.createCircuitBreakerService(settingsModule.getSettings(),
            settingsModule.getClusterSettings());
        resourcesToClose.add(circuitBreakerService);
        PageCacheRecycler pageCacheRecycler = new PageCacheRecycler(settings);
        BigArrays bigArrays = new BigArrays(pageCacheRecycler, circuitBreakerService);
        resourcesToClose.add(bigArrays);
        modules.add(settingsModule);
        
        NetworkModule networkModule = new NetworkModule(settings, true, pluginsService.filterPlugins(NetworkPlugin.class), threadPool,
            bigArrays, pageCacheRecycler, circuitBreakerService, namedWriteableRegistry, xContentRegistry, networkService, null);
        final Transport transport = networkModule.getTransportSupplier().get();
        final TransportService transportService = new TransportService(settings, transport, threadPool,
            networkModule.getTransportInterceptor(),
            boundTransportAddress -> DiscoveryNode.createLocal(settings, new TransportAddress(TransportAddress.META_ADDRESS, 0),
                UUIDs.randomBase64UUID()), null, Collections.emptySet());
        modules.add((b -> {
            b.bind(BigArrays.class).toInstance(bigArrays);
            b.bind(PluginsService.class).toInstance(pluginsService);
            b.bind(CircuitBreakerService.class).toInstance(circuitBreakerService);
            b.bind(NamedWriteableRegistry.class).toInstance(namedWriteableRegistry);
            b.bind(Transport.class).toInstance(transport);
            b.bind(TransportService.class).toInstance(transportService);
            b.bind(NetworkService.class).toInstance(networkService);
        }));

        // 注入所有module下的实例
        Injector injector = modules.createInjector();
        final TransportClientNodesService nodesService =
            new TransportClientNodesService(settings, transportService, threadPool, failureListner == null
                ? (t, e) -> {} : failureListner);

        // construct the list of client actions
        final List<ActionPlugin> actionPlugins = pluginsService.filterPlugins(ActionPlugin.class);
        final List<GenericAction> clientActions =
                actionPlugins.stream().flatMap(p -> p.getClientActions().stream()).collect(Collectors.toList());
        // add all the base actions
        final List<? extends GenericAction<?, ?>> baseActions =
                actionModule.getActions().values().stream().map(ActionPlugin.ActionHandler::getAction).collect(Collectors.toList());
        clientActions.addAll(baseActions);
        final TransportProxyClient proxy = new TransportProxyClient(settings, transportService, nodesService, clientActions);

        List<LifecycleComponent> pluginLifecycleComponents = new ArrayList<>(pluginsService.getGuiceServiceClasses().stream()
            .map(injector::getInstance).collect(Collectors.toList()));
        resourcesToClose.addAll(pluginLifecycleComponents);

        // 启动服务
        transportService.start();
        transportService.acceptIncomingRequests();

        ClientTemplate transportClient = new ClientTemplate(injector, pluginLifecycleComponents, nodesService, proxy, namedWriteableRegistry);
        resourcesToClose.clear();
        return transportClient;
    } finally {
        IOUtils.closeWhileHandlingException(resourcesToClose);
    }
}
```

可以看到确实是先通 `过ModulesBuilder modules = new ModulesBuilder()` 创建一个迭代器, 然后将各个Module通过add方法加入进去, 最后通过 `Injector injector = modules.createInjector();` 创建Injector对象, **之后便可根据Injector对象去获取实例了**. 

各个Module会绑定自己所需要的实例, 这里以 SettingsModule 举例:

```
public class SettingsModule extends AbstractModule {
    private final Settings settings;
    private final Set<String> settingsFilterPattern = new HashSet<>();
    private final Map<String, Setting<?>> nodeSettings = new HashMap<>();
    private final Map<String, Setting<?>> indexSettings = new HashMap<>();
    private final Logger logger;
    private final IndexScopedSettings indexScopedSettings;
    private final ClusterSettings clusterSettings;
    private final SettingsFilter settingsFilter;

    public SettingsModule(Settings settings, Setting<?>... additionalSettings) {
        this(settings, Arrays.asList(additionalSettings), Collections.emptyList());
    }

    @Override
    public void configure(Binder binder) {
        binder.bind(Settings.class).toInstance(settings);
        binder.bind(SettingsFilter.class).toInstance(settingsFilter);
        binder.bind(ClusterSettings.class).toInstance(clusterSettings);
        binder.bind(IndexScopedSettings.class).toInstance(indexScopedSettings);
    }
    
    //...
}
```

可以看到它绑定了四个,分别是 Settings.class，SettingsFilter.class，ClusterSettings.class，IndexScopedSettings.class

它们的实例对象都可以通过Injector来获取

### 小结

示例代码可在 https://github.com/whirlys/elastic-example/tree/master/guice 处下载

> 参考：   
> kason_zhang [Elasticsearch Guice 的使用](https://www.jianshu.com/p/0a1e6267b46f)



-----

打开微信扫一扫，关注【小旋锋】微信公众号，及时接收博文推送

![小旋锋的微信公众号](http://image.laijianfeng.org/%E5%B0%8F%E6%97%8B%E9%94%8B%E7%9A%84%E5%BE%AE%E4%BF%A1%E5%85%AC%E4%BC%97%E5%8F%B7_%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)