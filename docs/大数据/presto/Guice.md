**Guice：Google 开源的一款轻量级的依赖注入的框架，和 Spring 一样，只是更精炼，使用起来更自由。**

我们用代码来模拟用户点餐的场景：

```java
// 1. 餐桌服务
interface TableService {
    int provide();
}

class TableServiceImpl implements TableService {
    @Override
    public int provide() {
        return new Random().nextInt(10) + 1;
    }
}

// 2. 菜品服务
interface FoodService {
    String randomFood();
}

class FoodServiceImpl implements FoodService {
    private static final String[] FOODS = new String[]{
            "麻婆豆腐",
            "锅包肉",
            "蛋花汤",
            "酸菜汆白肉",
            "拍黄瓜"
    };

    @Override
    public String randomFood() {
        return FOODS[new Random().nextInt(FOODS.length)];
    }
}

// 3. 订餐服务
interface OrderService {
    int order(int table, String food, int count);
}

class OrderServiceImpl implements OrderService {
    private AtomicInteger index = new AtomicInteger();

    @Override
    public int order(int table, String food, int count) {
        return index.getAndIncrement();
    }
}

// 4.实际应用
public class OrderExample {
    public static void main(String[] args) {
        TableService tableService = new TableServiceImpl();
        FoodService foodService = new FoodServiceImpl();
        OrderService orderService = new OrderServiceImpl();

        int orderNo = orderService.order(tableService.provide(), foodService.randomFood());
        System.out.println("Wait for: " + orderNo);
    }
}
```

**代码的实现需要创建各种服务，同时需要维护服务的引用，对服务进行全生命周期管理，这样额外增加了应用类的负担，不利于代码的扩展。**

**如果我们使用 Guice：**

```java
// 1. 定义 Module，创建类的绑定关系
class OrderModule implements Module {
    @Override
    public void configure(Binder binder) {
        binder.bind(TableService.class).to(TableServiceImpl.class);
        binder.bind(FoodService.class).to(FoodServiceImpl.class);
        binder.bind(OrderService.class).to(OrderServiceImpl.class);
    }
}

// 2. 实际应用
class GuiceOrderExample {
    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new OrderModule());
        TableService tableService = injector.getInstance(TableService.class);
        FoodService foodService = injector.getInstance(FoodService.class);
        OrderService orderService = injector.getInstance(OrderService.class);

        int orderNo = orderService.order(tableService.provide(), foodService.randomFood());
        System.out.println("Wait for: " + orderNo);
    }
}
```

使用 Guice 通过创建 Module ，在 Module 内存绑定接口和实现类，实际应用时通过 Injector 获取实际的服务，每个类的分工明确，后续代码也很容易扩展。



## 1. 单实现绑定

Guice 中支持很多种绑定方式，下面绑定方式但最终结果是相同的：

```java
// 方式 1：通过类名绑定
class BinderExample implements Module {
    @Override
    public void configure(Binder binder) {
        binder.bind(TableService.class).to(TableServiceImpl.class);
    }
}

// 方式 2：直接绑定实例
class BinderExample implements Module {
    @Override
    public void configure(Binder binder) {
        binder.bind(TableService.class).toInstance(new TableServiceImpl());
    }
}

// 方式 3：通过注解 @Provides
class BinderExample implements Module {
    @Override
    public void configure(Binder binder) {
    }

    @Provides
    public TableService tableService(){
        return new TableServiceImpl();
    }
}

// 方式 4：通过实现 Provider<T> 接口
class TableServiceProvider implements Provider<TableService>{
    @Override
    public TableService get() {
        return new TableServiceImpl();
    }
}

class BinderExample implements Module {
    @Override
    public void configure(Binder binder) {
        binder.bind(TableService.class).toProvider(TableServiceProvider.class);
    }
}

```



## 2. 多实现绑定

如果服务有多个实现类，可以通过自定义注解或使用别名方式绑定。

### 2.1 通过自定义注解

```java
// 1. 定义不同的注解
@Qualifier
@Target({FIELD, PARAMETER, METHOD})
@Retention(RUNTIME)
@interface NormalTable {
}

@Qualifier
@Target({FIELD, PARAMETER, METHOD})
@Retention(RUNTIME)
@interface VipTable {
}

// 2. 实际不同的实现类
class NormalTableService implements TableService {
    @Override
    public int provide() {
        return new Random().nextInt(10);
    }
}

class VipTableService implements TableService {
    @Override
    public int provide() {
        return 100 + new Random().nextInt(10);
    }
}

// 3. 使用注解建立绑定关系
class CustomAnnotationModule implements Module {
    @Override
    public void configure(Binder binder) {
        binder.bind(TableService.class).annotatedWith(NormalTable.class).to(NormalTableService.class);
        binder.bind(TableService.class).annotatedWith(VipTable.class).to(VipTableService.class);
        binder.bind(CustomAnnotationExample.class);
    }
}

// 4. 通过使用自定义的注解进行依赖注入
class CustomAnnotationExample {
    private final TableService normalTableService;
    private final TableService vipTableService;

    @Inject
    public CustomAnnotationExample(@NormalTable TableService normalTableService, @VipTable TableService vipTableService) {
        this.normalTableService = normalTableService;
        this.vipTableService = vipTableService;
    }

    public void normal(){
        System.out.println(normalTableService.provide());
    }

    public void vip(){
        System.out.println(vipTableService.provide());
    }

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new CustomAnnotationModule());
        CustomAnnotationExample example = injector.getInstance(CustomAnnotationExample.class);
        example.normal();
        example.vip();
    }
}

```



### 2.2 通过内置的 @Named 注解

```java
// 1. 通过 Names.named 建立绑定关系
class NamedModule implements Module {
    @Override
    public void configure(Binder binder) {
        binder.bind(TableService.class).annotatedWith(Names.named("normal")).to(NormalTableService.class);
        binder.bind(TableService.class).annotatedWith(Names.named("vip")).to(VipTableService.class);
        binder.bind(NamedExample.class);
    }
}

// 2. 通过 Named 注解进行依赖注入
class NamedExample {
    private final TableService normalTableService;
    private final TableService vipTableService;

    @Inject
    public NamedExample(@Named("normal") TableService normalTableService, @Named("vip")  TableService vipTableService) {
        this.normalTableService = normalTableService;
        this.vipTableService = vipTableService;
    }

    public void normal(){
        System.out.println(normalTableService.provide());
    }

    public void vip(){
        System.out.println(vipTableService.provide());
    }

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new NamedModule());
        NamedExample example = injector.getInstance(NamedExample.class);
        example.normal();
        example.vip();
    }
}
```



## 3. 集合绑定

### 3.1 Set 绑定

需要使用 Multibinder<T> 进行绑定：

```java
class MutilsModule implements Module {
    @Override
    public void configure(Binder binder) {
        Multibinder<TableService> binders = Multibinder.newSetBinder(binder, TableService.class);
        binders.addBinding().to(NormalTableService.class);
        binders.addBinding().to(VipTableService.class);
    }
}

// 实际应用需要使用 Set 类型接收注入的所有实例
class MutilsExample {
    @Inject
    private Set<TableService> tableServices;

    public void provide() {
        tableServices.forEach(table -> System.out.println(table.provide()));
    }

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new MutilsModule());
        MutilsExample example = injector.getInstance(MutilsExample.class);
        example.provide();
    }
}

```



### 3.2 Map 绑定

需要使用 MapBinder<K, V> 进行绑定：

```java
class MapModule implements Module {
    @Override
    public void configure(Binder binder) {
        MapBinder<String, TableService> binders = MapBinder.newMapBinder(binder, String.class, TableService.class);
        binders.addBinding("normal").to(NormalTableService.class);
        binders.addBinding("vip").to(VipTableService.class);
    }
}

// 使用 Map 类型接收注入的键和值
class MapExample {
    @Inject
    private Map<String, TableService> map;

    public void provide() {
        map.entrySet().forEach(entry -> {
            System.out.println(entry.getKey() + ": " + entry.getValue().provide());
        });
    }

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new MapModule());
        MapExample example = injector.getInstance(MapExample.class);
        example.provide();
    }
}

```

## 4. AOP

Guice 也支持 AOP，首先声明注解，之后在标识注解的方法前后加入处理逻辑。

```java
// 1. 定义需要拦截的注解
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface AOP {
}

// 2. 对关心的类方法进行拦截
class AOPTableService implements TableService {
    @Override
    @AOP
    public int provide() {
        return new Random().nextInt(10);
    }
}

// 3. 定义拦截逻辑
class LogService implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        System.out.println("Invoking: " + methodInvocation.getMethod().getName());
        Object result = methodInvocation.proceed();
        System.out.println("Invoked: " + result);
        return result;
    }
}

// 4. 建立规则：识别拦截执行拦截逻辑
class AOPModule implements Module {
    public void configure(Binder binder) {
        binder.bindInterceptor(any(), annotatedWith(AOP.class), new LogService());
    }

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new AOPModule());
        AOPTableService tableService = injector.getInstance(AOPTableService.class);
        tableService.provide();
    }

```

## 5. Guice 在 Presto 中的使用

在 Preto 节点启动过程中，通过 NodeModule 加载本地配置、通过 DiscoveryModule 向 coordinator 发送注册信息、通过 HttpServerModule 提供 HTTP 服务或创建 HTTP Client，每个模块内都可以看到 Guice 的身影，想更深入的了解 Presto 的实现，以 Guice 的 Module 作为切入点是最佳的选择。

