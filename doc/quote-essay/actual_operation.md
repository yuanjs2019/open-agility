### 将Bean放入Spring容器中的五种方式

- 1、@Configuration + @Bean
- 2、@Componet + @ComponentScan]
- 3、@Import注解导入
- 4、使用FactoryBean接口
- 5、使用 BeanDefinitionRegistryPostProcessor

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcdwRvicQw2a2aasfdAsMCvpUKciblImy753lAKROqGboV9CCGLiaKP2brVGpibA4yot2xI6pNfmKe0lQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



------

将bean放入Spring容器中有哪些方式？

#### 1、@Configuration + @Bean

这种方式其实，在上一篇文章已经介绍过了，也是我们最常用的一种方式，@Configuration用来声明一个配置类，然后使用 @Bean 注解，用于声明一个bean，将其加入到Spring容器中。

具体代码如下:

```java
@Configuration
public class MyConfiguration {

    @Bean
    public Person person() {
        Person person = new Person();
        person.setName("spring");
        return person;
    }
}
```

#### 2、@Componet + @ComponentScan

这种方式也是我们用的比较多的方式，@Componet中文译为组件，放在类名上面，然后@ComponentScan放置在我们的配置类上，然后可以指定一个路径，进行扫描带有@Componet注解的bean，然后加至容器中。

具体代码如下:

```java
@Component
public class Person {
    private String name;
 
    public String getName() {
 
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                '}';
    }
}
 
@ComponentScan(basePackages = "com.springboot.initbean.*")
public class Demo1 {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Demo1.class);
        Person bean = applicationContext.getBean(Person.class);
        System.out.println(bean);
    }
}
```

结果输出:

```
Person{name='null'}
```

表示成功将Person放置在了IOC容器中。

#### 3、@Import注解导入]

前两种方式，大家用的可能比较多，也是平时开发中必须要知道的，@Import注解用的可能不是特别多了，但是也是非常重要的，在进行Spring扩展时经常会用到，它经常搭配自定义注解进行使用，然后往容器中导入一个配置文件。

关于@Import注解，我会多介绍一点，它有四种使用方式。这是@Import注解的源码，表示只能放置在类上。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
 
    /**
   * 用于导入一个class文件
     * {@link Configuration @Configuration}, {@link ImportSelector},
     * {@link ImportBeanDefinitionRegistrar}, or regular component classes to import.
     */
    Class<?>[] value();
 
}
```

##### 3.1 @Import直接导入类

```java
public class Person {
    private String name;
 
    public String getName() {
 
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                '}';
    }
}
/**
* 直接使用@Import导入person类，然后尝试从applicationContext中取，成功拿到
**/
@Import(Person.class)
public class Demo1 {
 
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Demo1.class);
        Person bean = applicationContext.getBean(Person.class);
        System.out.println(bean);
    }
}
```

上述代码直接使用@Import导入了一个类，然后自动的就被放置在IOC容器中了。

> 注意：我们的Person类上 就不需要任何的注解了，直接导入即可。

##### 3.2 @Import + ImportSelector]

其实在@Import注解的源码中，说的已经很清楚了，感兴趣的可以看下，我们实现一个ImportSelector的接口，然后实现其中的方法，进行导入。

代码如下:

```java
@Import(MyImportSelector.class)
public class Demo1 {
 
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Demo1.class);
        Person bean = applicationContext.getBean(Person.class);
        System.out.println(bean);
    }
}
 
class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"com.springboot.pojo.Person"};
    }
}
```

我自定义了一个 MyImportSelector 实现了 ImportSelector 接口，重写selectImports 方法，然后将我们要导入的类的全限定名写在里面即可，实现起来也是非常简单。

##### 3.3 @Import + ImportBeanDefinitionRegistrar

这种方式也需要我们实现 ImportBeanDefinitionRegistrar 接口中的方法，具体代码如下:

```java
@Import(MyImportBeanDefinitionRegistrar.class)
public class Demo1 {
 
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Demo1.class);
        Person bean = applicationContext.getBean(Person.class);
        System.out.println(bean);
    }
}
 
class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
 
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 构建一个beanDefinition, 关于beanDefinition我后续会介绍，可以简单理解为bean的定义.
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(Person.class).getBeanDefinition();
        // 将beanDefinition注册到Ioc容器中.
        registry.registerBeanDefinition("person", beanDefinition);
    }
}
```

上述实现其实和Import的第二种方式差不多，都需要去实现接口，然后进行导入。接触到了一个新的概念，BeanDefinition，可以简单理解为bean的定义(bean的元数据)，也是需要放在IOC容器中进行管理的，先有bean的元数据，applicationContext再根据bean的元数据去创建Bean。

##### 3.4 @Import + DeferredImportSelector

这种方式也需要我们进行实现接口，其实它和@Import的第二种方式差不多，DeferredImportSelector 它是 ImportSelector 的子接口，所以实现的方法和第二种无异。只是Spring的处理方式不同，它和Spring Boot中的自动导入配置文件 延迟导入有关，非常重要。使用方式如下:

```java
@Import(MyDeferredImportSelector.class)
public class Demo1 {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Demo1.class);
        Person bean = applicationContext.getBean(Person.class);
        System.out.println(bean);
    }
}
class MyDeferredImportSelector implements DeferredImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // 也是直接将Person的全限定名放进去
        return new String[]{Person.class.getName()};
    }
}
```

关于@Import注解的使用方式，大概就以上三种，当然它还可以搭配@Configuration注解使用，用于导入一个配置类。

#### 4、使用FactoryBean接口

FactoryBean接口和BeanFactory千万不要弄混了，从名字其实可以大概的区分开，FactoryBean, 后缀为bean，那么它其实就是一个bean, BeanFactory，顾名思义 bean工厂，它是IOC容器的顶级接口，这俩接口都很重要。

代码示例:

```java
@Configuration
public class Demo1 {
    @Bean
    public PersonFactoryBean personFactoryBean() {
        return new PersonFactoryBean();
    }
 
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Demo1.class);
        Person bean = applicationContext.getBean(Person.class);
        System.out.println(bean);
    }
}
 
class PersonFactoryBean implements FactoryBean<Person> {
 
    /**
     *  直接new出来Person进行返回.
     */
    @Override
    public Person getObject() throws Exception {
        return new Person();
    }
    /**
     *  指定返回bean的类型.
     */
    @Override
    public Class<?> getObjectType() {
        return Person.class;
    }
}
```

上述代码，我使用@Configuration + @Bean的方式将 PersonFactoryBean 加入到容器中，注意，我没有向容器中注入 Person, 而是直接注入的 PersonFactoryBean 然后从容器中拿Person这个类型的bean,成功运行。

#### 5、使用 BeanDefinitionRegistryPostProcessor

其实这种方式也是利用到了 BeanDefinitionRegistry，在Spring容器启动的时候会执行 BeanDefinitionRegistryPostProcessor 的 postProcessBeanDefinitionRegistry 方法，大概意思就是等beanDefinition加载完毕之后，对beanDefinition进行后置处理，可以在此进行调整IOC容器中的beanDefinition，从而干扰到后面进行初始化bean。

具体代码如下:

```java
public class Demo1 {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        MyBeanDefinitionRegistryPostProcessor beanDefinitionRegistryPostProcessor = new MyBeanDefinitionRegistryPostProcessor();
        applicationContext.addBeanFactoryPostProcessor(beanDefinitionRegistryPostProcessor);
        applicationContext.refresh();
        Person bean = applicationContext.getBean(Person.class);
        System.out.println(bean);
    }
}
 
class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
 
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(Person.class).getBeanDefinition();
        registry.registerBeanDefinition("person", beanDefinition);
    }
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
 
    }
}
```

上述代码中，我们手动向beanDefinitionRegistry中注册了person的BeanDefinition。最终成功将person加入到applicationContext中，上述的几种方式的具体原理，我后面会进行介绍。

#### 小结

向spring容器中加入bean的几种方式：

- @Configuration + @Bean

- @ComponentScan + @Component

- @Import 配合接口进行导入

- 使用FactoryBean。

- 实现BeanDefinitionRegistryPostProcessor进行后置处理。

  

### CompletableFuture:让你的代码免受阻塞之苦

#### 写在前面

通过阅读本篇文章你将了解到：

- CompletableFuture的使用
- CompletableFure异步和同步的性能测试
- 已经有了Future为什么仍需要在JDK1.8中引入CompletableFuture
- CompletableFuture的应用场景
- 对CompletableFuture的使用优化

#### 场景说明

查询所有商店某个商品的价格并返回，并且查询商店某个商品的价格的API为同步 一个Shop类，提供一个名为getPrice的同步方法

- 店铺类：Shop.java

```java
public class Shop {
    private Random random = new Random();
    /**
     * 根据产品名查找价格
     * */
    public double getPrice(String product) {
        return calculatePrice(product);
    }

    /**
     * 计算价格
     *
     * @param product
     * @return
     * */
    private double calculatePrice(String product) {
        delay();
        //random.nextDouble()随机返回折扣
        return random.nextDouble() * product.charAt(0) + product.charAt(1);
    }

    /**
     * 通过睡眠模拟其他耗时操作
     * */
    private void delay() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

查询商品的价格为同步方法，并通过sleep方法模拟其他操作。这个场景模拟了当需要调用第三方API，但第三方提供的是同步API，在无法修改第三方API时如何设计代码调用提高应用的性能和吞吐量，这时候可以使用CompletableFuture类

#### CompletableFuture使用

Completable是Future接口的实现类，在JDK1.8中引入

- **CompletableFuture的创建：**

  说明：

- - 两个重载方法之间的区别 => 后者可以传入自定义Executor，前者是默认的，使用的ForkJoinPool

  - supplyAsync和runAsync方法之间的区别 => 前者有返回值，后者无返回值

  - Supplier是函数式接口，因此该方法需要传入该接口的实现类，追踪源码会发现在run方法中会调用该接口的方法。因此使用该方法创建CompletableFuture对象只需重写Supplier中的get方法，在get方法中定义任务即可。又因为函数式接口可以使用Lambda表达式，和new创建CompletableFuture对象相比代码会**简洁**不少

  - 使用new方法

    ```java
    CompletableFuture<Double> futurePrice = new CompletableFuture<>();
    ```

  - 使用CompletableFuture#completedFuture静态方法创建

    ```java
    public static <U> CompletableFuture<U> completedFuture(U value) {
        return new CompletableFuture<U>((value == null) ? NIL : value);
    }
    ```

    参数的值为任务执行完的结果，一般该方法在实际应用中较少应用

  - 使用 CompletableFuture#supplyAsync静态方法创建 supplyAsync有两个重载方法：

    ```java
    //方法一
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
    }
    //方法二
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                       Executor executor) {
        return asyncSupplyStage(screenExecutor(executor), supplier);
    }
    ```

  - 使用CompletableFuture#runAsync静态方法创建 runAsync有两个重载方法

    ```java
    //方法一
    public static CompletableFuture<Void> runAsync(Runnable runnable) {
        return asyncRunStage(asyncPool, runnable);
    }
    //方法二
    public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor) {
        return asyncRunStage(screenExecutor(executor), runnable);
    }
    ```

- **结果的获取：** 对于结果的获取CompltableFuture类提供了四种方式

  ```java
  //方式一
  public T get()
  //方式二
  public T get(long timeout, TimeUnit unit)
  //方式三
  public T getNow(T valueIfAbsent)
  //方式四
  public T join()
  ```

  说明：

  示例：

- - get()和get(long timeout, TimeUnit unit) => 在Future中就已经提供了，后者提供超时处理，如果在指定时间内未获取结果将抛出超时异常
  - getNow => 立即获取结果不阻塞，结果计算已完成将返回结果或计算过程中的异常，如果未计算完成将返回设定的valueIfAbsent值
  - join => 方法里不会抛出异常

```java
public class AcquireResultTest {
  public static void main(String[] args) throws ExecutionException, InterruptedException {
      //getNow方法测试
      CompletableFuture<String> cp1 = CompletableFuture.supplyAsync(() -> {
          try {
              Thread.sleep(60 * 1000 * 60 );
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
  
          return "hello world";
      });
  
      System.out.println(cp1.getNow("hello h2t"));
  
      //join方法测试
      CompletableFuture<Integer> cp2 = CompletableFuture.supplyAsync((()-> 1 / 0));
      System.out.println(cp2.join());
  
      //get方法测试
      CompletableFuture<Integer> cp3 = CompletableFuture.supplyAsync((()-> 1 / 0));
      System.out.println(cp3.get());
  }
}
```

说明：

- 第一个执行结果为hello h2t，因为要先睡上1分钟结果不能立即获取
- join方法获取结果方法里不会抛异常，但是执行结果会抛异常，抛出的异常为CompletionException
- get方法获取结果方法里将抛出异常，执行结果抛出的异常为ExecutionException
- **异常处理：** 使用静态方法创建的CompletableFuture对象无需显示处理异常，使用new创建的对象需要调用completeExceptionally方法设置捕获到的异常，举例说明：

```java
CompletableFuture completableFuture = new CompletableFuture();
new Thread(() -> {
   try {
       //doSomething，调用complete方法将其他方法的执行结果记录在completableFuture对象中
       completableFuture.complete(null);
   } catch (Exception e) {
       //异常处理
       completableFuture.completeExceptionally(e);
    }
}).start();
```

#### 同步方法Pick异步方法查询所有店铺某个商品价格

店铺为一个列表：

```java
private static List<Shop> shopList = Arrays.asList(
        new Shop("BestPrice"),
        new Shop("LetsSaveBig"),
        new Shop("MyFavoriteShop"),
        new Shop("BuyItAll")
);
```

**同步方法：**

```java
private static List<String> findPriceSync(String product) {
    return shopList.stream()
            .map(shop -> String.format("%s price is %.2f",
                    shop.getName(), shop.getPrice(product)))  //格式转换
            .collect(Collectors.toList());
}
```

**异步方法：**

```java
private static List<String> findPriceAsync(String product) {
    List<CompletableFuture<String>> completableFutureList = shopList.stream()
            //转异步执行
            .map(shop -> CompletableFuture.supplyAsync(
                    () -> String.format("%s price is %.2f",
                            shop.getName(), shop.getPrice(product))))  //格式转换
            .collect(Collectors.toList());

    return completableFutureList.stream()
            .map(CompletableFuture::join)  //获取结果不会抛出异常
            .collect(Collectors.toList());
}
```

**性能测试结果：**

```java
Find Price Sync Done in 4141
Find Price Async Done in 1033
```

**异步**执行效率**提高四倍**

#### 为什么仍需要CompletableFuture

在JDK1.8以前，通过调用线程池的submit方法可以让任务以异步的方式运行，该方法会返回一个Future对象，通过调用get方法获取异步执行的结果：

```java
private static List<String> findPriceFutureAsync(String product) {
    ExecutorService es = Executors.newCachedThreadPool();
    List<Future<String>> futureList = shopList.stream().map(shop -> es.submit(() -> String.format("%s price is %.2f",
            shop.getName(), shop.getPrice(product)))).collect(Collectors.toList());

    return futureList.stream()
            .map(f -> {
                String result = null;
                try {
                    result = f.get();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }

                return result;
            }).collect(Collectors.toList());
}
```

既生瑜何生亮，为什么仍需要引入CompletableFuture？对于简单的业务场景使用Future完全没有，但是想将多个异步任务的计算结果组合起来，后一个异步任务的计算结果需要前一个异步任务的值等等，使用Future提供的那点API就囊中羞涩，处理起来不够优雅，这时候还是让CompletableFuture以**声明式**的方式优雅的处理这些需求。而且在Future编程中想要拿到Future的值然后拿这个值去做后续的计算任务，只能通过轮询的方式去判断任务是否完成这样非常占CPU并且代码也不优雅，用伪代码表示如下：

```java
while(future.isDone()) {
    result = future.get();
    doSomrthingWithResult(result);
} 
```

但CompletableFuture提供了API帮助我们实现这样的需求

#### 其他API介绍

#### whenComplete计算结果的处理：

对前面计算结果进行处理，无法返回新值 提供了三个方法：

```java
//方法一
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
//方法二
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
//方法三
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
```

说明：

- BiFunction<? super T,? super U,? extends V> fn参数 => 定义对结果的处理
- Executor executor参数 => 自定义线程池
- 以async结尾的方法将会在一个新的线程中执行组合操作

示例：

```java
public class WhenCompleteTest {
    public static void main(String[] args) {
        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> "hello");
        CompletableFuture<String> cf2 = cf1.whenComplete((v, e) ->
                System.out.println(String.format("value:%s, exception:%s", v, e)));
        System.out.println(cf2.join());
    }
}
```

#### thenApply转换：

将前面计算结果的的CompletableFuture传递给thenApply，返回thenApply处理后的结果。可以认为通过thenApply方法实现`CompletableFuture<T>`至`CompletableFuture<U>`的转换。白话一点就是将CompletableFuture的计算结果作为thenApply方法的参数，返回thenApply方法处理后的结果 提供了三个方法：

```java
//方法一
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}

//方法二
public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(asyncPool, fn);
}

//方法三
public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn, Executor executor) {
    return uniApplyStage(screenExecutor(executor), fn);
}
```

说明：

- Function<? super T,? extends U> fn参数 => 对前一个CompletableFuture 计算结果的转化操作
- Executor executor参数 => 自定义线程池
- 以async结尾的方法将会在一个新的线程中执行组合操作 示例：

```java
public class ThenApplyTest {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<Integer> result = CompletableFuture.supplyAsync(ThenApplyTest::randomInteger).thenApply((i) -> i * 8);
        System.out.println(result.get());
    }

    public static Integer randomInteger() {
        return 10;
    }
}
```

这里将前一个CompletableFuture计算出来的结果扩大八倍

#### thenAccept结果处理：

thenApply也可以归类为对结果的处理，thenAccept和thenApply的区别就是没有返回值 提供了三个方法：

```java
//方法一
public CompletableFuture<Void> thenAccept(Consumer<? super T> action) {
    return uniAcceptStage(null, action);
}

//方法二
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action) {
    return uniAcceptStage(asyncPool, action);
}

//方法三
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action,
                                               Executor executor) {
    return uniAcceptStage(screenExecutor(executor), action);
}
```

说明：

- Consumer<? super T> action参数 => 对前一个CompletableFuture计算结果的操作
- Executor executor参数 => 自定义线程池
- 同理以async结尾的方法将会在一个新的线程中执行组合操作 示例：

```java
public class ThenAcceptTest {
    public static void main(String[] args) {
        CompletableFuture.supplyAsync(ThenAcceptTest::getList).thenAccept(strList -> strList.stream()
                .forEach(m -> System.out.println(m)));
    }

    public static List<String> getList() {
        return Arrays.asList("a", "b", "c");
    }
}
```

将前一个CompletableFuture计算出来的结果打印出来

#### thenCompose异步结果流水化：

thenCompose方法可以将两个异步操作进行流水操作 提供了三个方法：

```java
//方法一
public <U> CompletableFuture<U> thenCompose(
    Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(null, fn);
}

//方法二
public <U> CompletableFuture<U> thenComposeAsync(
    Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(asyncPool, fn);
}

//方法三
public <U> CompletableFuture<U> thenComposeAsync(
    Function<? super T, ? extends CompletionStage<U>> fn,
    Executor executor) {
    return uniComposeStage(screenExecutor(executor), fn);
}
```

说明：

- `Function<? super T, ? extends CompletionStage<U>> fn`参数 => 当前CompletableFuture计算结果的执行
- Executor executor参数 => 自定义线程池
- 同理以async结尾的方法将会在一个新的线程中执行组合操作 示例：

```java
public class ThenComposeTest {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<Integer> result = CompletableFuture.supplyAsync(ThenComposeTest::getInteger)
                .thenCompose(i -> CompletableFuture.supplyAsync(() -> i * 10));
        System.out.println(result.get());
    }

    private static int getInteger() {
        return 666;
    }

    private static int expandValue(int num) {
        return num * 10;
    }
}
```

执行流程图：

![图片](https://mmbiz.qpic.cn/mmbiz/N34tfh8WYkiahg74DUViaraK9PSjAvycy1Zx0KmIBMazZfcCzsEAsCrsOclmklaK06078KZ8X8RLB8vytgB3Xia6w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### thenCombine组合结果：

thenCombine方法将两个无关的CompletableFuture组合起来，第二个Completable并不依赖第一个Completable的结果 提供了三个方法：

```java
//方法一
public <U,V> CompletableFuture<V> thenCombine( 
    CompletionStage<? extends U> other,
    BiFunction<? super T,? super U,? extends V> fn) {
    return biApplyStage(null, other, fn);
}
  //方法二
  public <U,V> CompletableFuture<V> thenCombineAsync(
      CompletionStage<? extends U> other,
      BiFunction<? super T,? super U,? extends V> fn) {
      return biApplyStage(asyncPool, other, fn);
  }

  //方法三
  public <U,V> CompletableFuture<V> thenCombineAsync(
      CompletionStage<? extends U> other,
      BiFunction<? super T,? super U,? extends V> fn, Executor executor) {
      return biApplyStage(screenExecutor(executor), other, fn);
  }
```

说明：

- CompletionStage<? extends U> other参数 => 新的CompletableFuture的计算结果
- BiFunction<? super T,? super U,? extends V> fn参数 => 定义了两个CompletableFuture对象**完成计算后**如何合并结果，该参数是一个函数式接口，因此可以使用Lambda表达式
- Executor executor参数 => 自定义线程池
- 同理以async结尾的方法将会在一个新的线程中执行组合操作

示例：

```java
public class ThenCombineTest {
    private static Random random = new Random();
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<Integer> result = CompletableFuture.supplyAsync(ThenCombineTest::randomInteger).thenCombine(
                CompletableFuture.supplyAsync(ThenCombineTest::randomInteger), (i, j) -> i * j
        );

        System.out.println(result.get());
    }

    public static Integer randomInteger() {
        return random.nextInt(100);
    }
}
```

将两个线程计算出来的值做一个乘法在返回 执行流程图：

![图片](https://mmbiz.qpic.cn/mmbiz/N34tfh8WYkiahg74DUViaraK9PSjAvycy1Zx0KmIBMazZfcCzsEAsCrsOclmklaK06078KZ8X8RLB8vytgB3Xia6w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### allOf&anyOf组合多个CompletableFuture：

方法介绍：

```java
//allOf
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {
    return andTree(cfs, 0, cfs.length - 1);
}
//anyOf
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) {
    return orTree(cfs, 0, cfs.length - 1);
}
```

说明：

- allOf => 所有的CompletableFuture都执行完后执行计算。
- anyOf => 任意一个CompletableFuture执行完后就会执行计算

示例：

- allOf方法测试

```java
public class AllOfTest {
  public static void main(String[] args) throws ExecutionException, InterruptedException {
      CompletableFuture<Void> future1 = CompletableFuture.supplyAsync(() -> {
          System.out.println("hello");
          return null;
      });
      CompletableFuture<Void> future2 = CompletableFuture.supplyAsync(() -> {
          System.out.println("world"); return null;
      });
      CompletableFuture<Void> result = CompletableFuture.allOf(future1, future2);
      System.out.println(result.get());
  }
}
```

allOf方法没有返回值，适合没有返回值并且需要前面所有任务执行完毕才能执行后续任务的应用场景

- anyOf方法测试

```java
public class AnyOfTest {
  private static Random random = new Random();
  public static void main(String[] args) throws ExecutionException, InterruptedException {
      CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
          randomSleep();
          System.out.println("hello");
          return "hello";});
      CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
          randomSleep();
          System.out.println("world");
          return "world";
      });
      CompletableFuture<Object> result = CompletableFuture.anyOf(future1, future2);
      System.out.println(result.get());
 }
  
  private static void randomSleep() {
      try {
          Thread.sleep(random.nextInt(10));
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
  }
}
```

两个线程都会将结果打印出来，但是get方法只会返回最先完成任务的结果。该方法比较适合只要有一个返回值就可以继续执行其他任务的应用场景

#### 注意点

很多方法都提供了异步实现【带async后缀】，但是需小心谨慎使用这些异步方法，因为异步意味着存在上下文切换，可能性能不一定比同步好。如果需要使用异步的方法，**先做测试**，用测试数据说话！！！

#### CompletableFuture的应用场景

存在IO密集型的任务可以选择CompletableFuture，IO部分交由另外一个线程去执行。Logback、Log4j2异步日志记录的实现原理就是新起了一个线程去执行IO操作，这部分可以以CompletableFuture.runAsync(()->{ioOperation();})的方式去调用。如果是CPU密集型就不推荐使用了推荐使用并行流

#### 优化空间

supplyAsync执行任务底层实现：

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
    return asyncSupplyStage(asyncPool, supplier);
}
static <U> CompletableFuture<U> asyncSupplyStage(Executor e, Supplier<U> f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<U> d = new CompletableFuture<U>();
    e.execute(new AsyncSupply<U>(d, f));
    return d;
}
```

底层调用的是线程池去执行任务，而CompletableFuture中默认线程池为ForkJoinPool

```java
private static final Executor asyncPool = useCommonPool ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```

ForkJoinPool线程池的大小取决于CPU的核数。CPU密集型任务线程池大小配置为CPU核心数就可以了，但是IO密集型，线程池的大小由**CPU数量 * CPU利用率 * (1 + 线程等待时间/线程CPU时间)**确定。而CompletableFuture的应用场景就是IO密集型任务，因此默认的ForkJoinPool一般无法达到最佳性能，我们需自己根据业务创建线程池

### 20个使用JavaCompletableFuture的例子

这篇文章介绍 Java 8 的 CompletionStage API和它的标准库的实现 CompletableFuture。API通过例子的方式演示了它的行为，每个例子演示一到两个行为。

既然`CompletableFuture`类实现了`CompletionStage`接口，首先我们需要理解这个接口的契约。它代表了一个特定的计算的阶段，可以同步或者异步的被完成。你可以把它看成一个计算流水线上的一个单元，最终会产生一个最终结果，这意味着几个`CompletionStage`可以串联起来，一个完成的阶段可以触发下一阶段的执行，接着触发下一次，接着……

除了实现`CompletionStage`接口， `CompletableFuture`也实现了`future`接口, 代表一个未完成的异步事件。`CompletableFuture`提供了方法，能够显式地完成这个future,所以它叫`CompletableFuture`。

#### 1、 创建一个完成的CompletableFuture

最简单的例子就是使用一个预定义的结果创建一个完成的CompletableFuture,通常我们会在计算的开始阶段使用它。

```java
static void completedFutureExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message");
    assertTrue(cf.isDone());
    assertEquals("message", cf.getNow(null));
}
```

`getNow(null)`方法在future完成的情况下会返回结果，就比如上面这个例子，否则返回null (传入的参数)。

#### 2、运行一个简单的异步阶段

这个例子创建一个一个异步执行的阶段：

```java
static void runAsyncExample() {
    CompletableFuture cf = CompletableFuture.runAsync(() -> {
        assertTrue(Thread.currentThread().isDaemon());
        randomSleep();
    });
    assertFalse(cf.isDone());
    sleepEnough();
    assertTrue(cf.isDone());
}
```

通过这个例子可以学到两件事情：

CompletableFuture的方法如果以`Async`结尾，它会异步的执行(没有指定executor的情况下)， 异步执行通过ForkJoinPool实现， 它使用守护线程去执行任务。注意这是CompletableFuture的特性， 其它CompletionStage可以override这个默认的行为。

#### 3、在前一个阶段上应用函数

下面这个例子使用前面 **#1** 的完成的CompletableFuture， #1返回结果为字符串`message`,然后应用一个函数把它变成大写字母。

```java
static void thenApplyExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApply(s -> {
        assertFalse(Thread.currentThread().isDaemon());
        return s.toUpperCase();
    });
    assertEquals("MESSAGE", cf.getNow(null));
}
```

注意`thenApply`方法名称代表的行为。

`then`意味着这个阶段的动作发生当前的阶段正常完成之后。本例中，当前节点完成，返回字符串`message`。

`Apply`意味着返回的阶段将会对结果前一阶段的结果应用一个函数。

函数的执行会被**阻塞**，这意味着`getNow()`只有打斜操作被完成后才返回。

#### 4、在前一个阶段上异步应用函数

通过调用异步方法(方法后边加Async后缀)，串联起来的CompletableFuture可以异步地执行（使用ForkJoinPool.commonPool()）。

```java
static void thenApplyAsyncExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(s -> {
        assertTrue(Thread.currentThread().isDaemon());
        randomSleep();
        return s.toUpperCase();
    });
    assertNull(cf.getNow(null));
    assertEquals("MESSAGE", cf.join());
}
```

#### 5、使用定制的Executor在前一个阶段上异步应用函数

异步方法一个非常有用的特性就是能够提供一个Executor来异步地执行CompletableFuture。这个例子演示了如何使用一个固定大小的线程池来应用大写函数。

```java
static ExecutorService executor = Executors.newFixedThreadPool(3, new ThreadFactory() {
    int count = 1;
 
    @Override
    public Thread newThread(Runnable runnable) {
        return new Thread(runnable, "custom-executor-" + count++);
    }
});
 
static void thenApplyAsyncWithExecutorExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(s -> {
        assertTrue(Thread.currentThread().getName().startsWith("custom-executor-"));
        assertFalse(Thread.currentThread().isDaemon());
        randomSleep();
        return s.toUpperCase();
    }, executor);
 
    assertNull(cf.getNow(null));
    assertEquals("MESSAGE", cf.join());
}
```

#### 6、消费前一阶段的结果

如果下一阶段接收了当前阶段的结果，但是在计算的时候不需要返回值(它的返回类型是void)， 那么它可以不应用一个函数，而是一个消费者， 调用方法也变成了`thenAccept`:

```java
static void thenAcceptExample() {
    StringBuilder result = new StringBuilder();
    CompletableFuture.completedFuture("thenAccept message")
            .thenAccept(s -> result.append(s));
    assertTrue("Result was empty", result.length() > 0);
}
```

本例中消费者同步地执行，所以我们不需要在CompletableFuture调用`join`方法。

#### 7、异步地消费迁移阶段的结果

同样，可以使用`thenAcceptAsync`方法， 串联的CompletableFuture可以异步地执行。

```java
static void thenAcceptAsyncExample() {
    StringBuilder result = new StringBuilder();
    CompletableFuture cf = CompletableFuture.completedFuture("thenAcceptAsync message")
            .thenAcceptAsync(s -> result.append(s));
    cf.join();
    assertTrue("Result was empty", result.length() > 0);
}
```

#### 8、完成计算异常

现在我们来看一下异步操作如何显式地返回异常，用来指示计算失败。我们简化这个例子，操作处理一个字符串，把它转换成答谢，我们模拟延迟一秒。

我们使用`thenApplyAsync(Function, Executor)`方法，第一个参数传入大写函数， executor是一个delayed executor,在执行前会延迟一秒。

```java
static void completeExceptionallyExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(String::toUpperCase,
            CompletableFuture.delayedExecutor(1, TimeUnit.SECONDS));
    CompletableFuture exceptionHandler = cf.handle((s, th) -> { return (th != null) ? "message upon cancel" : ""; });
    cf.completeExceptionally(new RuntimeException("completed exceptionally"));
assertTrue("Was not completed exceptionally", cf.isCompletedExceptionally());
    try {
        cf.join();
        fail("Should have thrown an exception");
    } catch(CompletionException ex) { // just for testing
        assertEquals("completed exceptionally", ex.getCause().getMessage());
    }
 
    assertEquals("message upon cancel", exceptionHandler.join());
}
```

让我们看一下细节。

首先我们创建了一个CompletableFuture, 完成后返回一个字符串`message`,接着我们调用`thenApplyAsync`方法，它返回一个CompletableFuture。这个方法在第一个函数完成后，异步地应用转大写字母函数。

这个例子还演示了如何通过`delayedExecutor(timeout, timeUnit)`延迟执行一个异步任务。

我们创建了一个分离的`handler`阶段：exceptionHandler， 它处理异常异常，在异常情况下返回`message upon cancel`。

下一步我们显式地用异常完成第二个阶段。在阶段上调用`join`方法，它会执行大写转换，然后抛出CompletionException（正常的join会等待1秒，然后得到大写的字符串。不过我们的例子还没等它执行就完成了异常）， 然后它触发了handler阶段。

#### 9、取消计算

和完成异常类似，我们可以调用`cancel(boolean mayInterruptIfRunning)`取消计算。对于CompletableFuture类，布尔参数并没有被使用，这是因为它并没有使用中断去取消操作，相反，`cancel`等价于`completeExceptionally(new CancellationException())`。

```java
static void cancelExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(String::toUpperCase,
            CompletableFuture.delayedExecutor(1, TimeUnit.SECONDS));
    CompletableFuture cf2 = cf.exceptionally(throwable -> "canceled message");
    assertTrue("Was not canceled", cf.cancel(true));
    assertTrue("Was not completed exceptionally", cf.isCompletedExceptionally());
    assertEquals("canceled message", cf2.join());
}
```

#### 10、在两个完成的阶段其中之一上应用函数

下面的例子创建了`CompletableFuture`, `applyToEither`处理两个阶段， 在其中之一上应用函数(包保证哪一个被执行)。本例中的两个阶段一个是应用大写转换在原始的字符串上， 另一个阶段是应用小些转换。

```java
static void applyToEitherExample() {
    String original = "Message";
    CompletableFuture cf1 = CompletableFuture.completedFuture(original)
            .thenApplyAsync(s -> delayedUpperCase(s));
    CompletableFuture cf2 = cf1.applyToEither(
            CompletableFuture.completedFuture(original).thenApplyAsync(s -> delayedLowerCase(s)),
            s -> s + " from applyToEither");
    assertTrue(cf2.join().endsWith(" from applyToEither"));
}
```

#### 11、在两个完成的阶段其中之一上调用消费函数

和前一个例子很类似了，只不过我们调用的是消费者函数 (Function变成Consumer):

```java
static void acceptEitherExample() {
    String original = "Message";
    StringBuilder result = new StringBuilder();
    CompletableFuture cf = CompletableFuture.completedFuture(original)
            .thenApplyAsync(s -> delayedUpperCase(s))
            .acceptEither(CompletableFuture.completedFuture(original).thenApplyAsync(s -> delayedLowerCase(s)),
                    s -> result.append(s).append("acceptEither"));
    cf.join();
    assertTrue("Result was empty", result.toString().endsWith("acceptEither"));
}
```

#### 12、在两个阶段都执行完后运行一个 `Runnable`

这个例子演示了依赖的CompletableFuture如果等待两个阶段完成后执行了一个Runnable。注意下面所有的阶段都是同步执行的，第一个阶段执行大写转换，第二个阶段执行小写转换。

```java
static void runAfterBothExample() {
    String original = "Message";
    StringBuilder result = new StringBuilder();
    CompletableFuture.completedFuture(original).thenApply(String::toUpperCase).runAfterBoth(
            CompletableFuture.completedFuture(original).thenApply(String::toLowerCase),
            () -> result.append("done"));
    assertTrue("Result was empty", result.length() > 0);
}
```

#### 13、 使用BiConsumer处理两个阶段的结果

上面的例子还可以通过BiConsumer来实现:

```java
static void thenAcceptBothExample() {
    String original = "Message";
    StringBuilder result = new StringBuilder();
    CompletableFuture.completedFuture(original).thenApply(String::toUpperCase).thenAcceptBoth(
            CompletableFuture.completedFuture(original).thenApply(String::toLowerCase),
            (s1, s2) -> result.append(s1 + s2));
    assertEquals("MESSAGEmessage", result.toString());
}
```

#### 14、使用BiFunction处理两个阶段的结果

如果CompletableFuture依赖两个前面阶段的结果， 它复合两个阶段的结果再返回一个结果，我们就可以使用`thenCombine()`函数。整个流水线是同步的，所以`getNow()`会得到最终的结果，它把大写和小写字符串连接起来。

```java
static void thenCombineExample() {
    String original = "Message";
    CompletableFuture cf = CompletableFuture.completedFuture(original).thenApply(s -> delayedUpperCase(s))
            .thenCombine(CompletableFuture.completedFuture(original).thenApply(s -> delayedLowerCase(s)),
                    (s1, s2) -> s1 + s2);
    assertEquals("MESSAGEmessage", cf.getNow(null));
}
```

#### 15、异步使用BiFunction处理两个阶段的结果

类似上面的例子，但是有一点不同：依赖的前两个阶段异步地执行，所以`thenCombine()`也异步地执行，即时它没有`Async`后缀。

Javadoc中有注释：

> Actions supplied for dependent completions of non-async methods may be performed by the thread that completes the current CompletableFuture, or by any other caller of a completion method

所以我们需要`join`方法等待结果的完成。

```java
static void thenCombineAsyncExample() {
    String original = "Message";
    CompletableFuture cf = CompletableFuture.completedFuture(original)
            .thenApplyAsync(s -> delayedUpperCase(s))
            .thenCombine(CompletableFuture.completedFuture(original).thenApplyAsync(s -> delayedLowerCase(s)),
                    (s1, s2) -> s1 + s2);
    assertEquals("MESSAGEmessage", cf.join());
}
```

#### 16、组合 CompletableFuture

我们可以使用`thenCompose()`完成上面两个例子。这个方法等待第一个阶段的完成(大写转换)， 它的结果传给一个指定的返回CompletableFuture函数，它的结果就是返回的CompletableFuture的结果。

有点拗口，但是我们看例子来理解。函数需要一个大写字符串做参数，然后返回一个CompletableFuture, 这个CompletableFuture会转换字符串变成小写然后连接在大写字符串的后面。

```java
static void thenComposeExample() {
    String original = "Message";
    CompletableFuture cf = CompletableFuture.completedFuture(original).thenApply(s -> delayedUpperCase(s))
            .thenCompose(upper -> CompletableFuture.completedFuture(original).thenApply(s -> delayedLowerCase(s))
                    .thenApply(s -> upper + s));
    assertEquals("MESSAGEmessage", cf.join());
}
```

#### 17、当几个阶段中的一个完成，创建一个完成的阶段

下面的例子演示了当任意一个CompletableFuture完成后， 创建一个完成的CompletableFuture.

待处理的阶段首先创建， 每个阶段都是转换一个字符串为大写。因为本例中这些阶段都是同步地执行(thenApply), 从`anyOf`中创建的CompletableFuture会立即完成，这样所有的阶段都已完成，我们使用`whenComplete(BiConsumer<? super Object, ? super Throwable> action)`处理完成的结果。

```java
static void anyOfExample() {
    StringBuilder result = new StringBuilder();
    List messages = Arrays.asList("a", "b", "c");
    List<CompletableFuture> futures = messages.stream()
            .map(msg -> CompletableFuture.completedFuture(msg).thenApply(s -> delayedUpperCase(s)))
            .collect(Collectors.toList());
    CompletableFuture.anyOf(futures.toArray(new CompletableFuture[futures.size()])).whenComplete((res, th) -> {
        if(th == null) {
            assertTrue(isUpperCase((String) res));
            result.append(res);
        }
    });
    assertTrue("Result was empty", result.length() > 0);
}
```

#### 18、当所有的阶段都完成后创建一个阶段

上一个例子是当任意一个阶段完成后接着处理，接下来的两个例子演示当所有的阶段完成后才继续处理, 同步地方式和异步地方式两种。

```java
static void allOfExample() {
    StringBuilder result = new StringBuilder();
    List messages = Arrays.asList("a", "b", "c");
    List<CompletableFuture> futures = messages.stream()
            .map(msg -> CompletableFuture.completedFuture(msg).thenApply(s -> delayedUpperCase(s)))
            .collect(Collectors.toList());
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[futures.size()])).whenComplete((v, th) -> {
        futures.forEach(cf -> assertTrue(isUpperCase(cf.getNow(null))));
        result.append("done");
    });
    assertTrue("Result was empty", result.length() > 0);
}
```

#### 19、当所有的阶段都完成后异步地创建一个阶段

使用`thenApplyAsync()`替换那些单个的CompletableFutures的方法，`allOf()`会在通用池中的线程中异步地执行。所以我们需要调用`join`方法等待它完成。

```java
static void allOfAsyncExample() {
    StringBuilder result = new StringBuilder();
    List messages = Arrays.asList("a", "b", "c");
    List<CompletableFuture> futures = messages.stream()
            .map(msg -> CompletableFuture.completedFuture(msg).thenApplyAsync(s -> delayedUpperCase(s)))
            .collect(Collectors.toList());
    CompletableFuture allOf = CompletableFuture.allOf(futures.toArray(new CompletableFuture[futures.size()]))
            .whenComplete((v, th) -> {
                futures.forEach(cf -> assertTrue(isUpperCase(cf.getNow(null))));
                result.append("done");
            });
    allOf.join();
    assertTrue("Result was empty", result.length() > 0);
}
```

#### 20、真实的例子

Now that the functionality of CompletionStage and specifically CompletableFuture is explored, the below example applies them in a practical scenario:

现在你已经了解了CompletionStage 和 CompletableFuture 的一些函数的功能，下面的例子是一个实践场景：

1. 首先异步调用`cars`方法获得Car的列表，它返回CompletionStage场景。`cars`消费一个远程的REST API。
2. 然后我们复合一个CompletionStage填写每个汽车的评分，通过`rating(manufacturerId)`返回一个CompletionStage, 它会异步地获取汽车的评分(可能又是一个REST API调用)
3. 当所有的汽车填好评分后，我们结束这个列表，所以我们调用`allOf`得到最终的阶段， 它在前面阶段所有阶段完成后才完成。
4. 在最终的阶段调用`whenComplete()`,我们打印出每个汽车和它的评分。

```java
cars().thenCompose(cars -> {
    List<CompletionStage> updatedCars = cars.stream()
            .map(car -> rating(car.manufacturerId).thenApply(r -> {
                car.setRating(r);
                return car;
            })).collect(Collectors.toList());
 
    CompletableFuture done = CompletableFuture
            .allOf(updatedCars.toArray(new CompletableFuture[updatedCars.size()]));
    return done.thenApply(v -> updatedCars.stream().map(CompletionStage::toCompletableFuture)
            .map(CompletableFuture::join).collect(Collectors.toList()));
}).whenComplete((cars, th) -> {
    if (th == null) {
        cars.forEach(System.out::println);
    } else {
        throw new RuntimeException(th);
    }
}).toCompletableFuture().join();
```

因为每个汽车的实例都是独立的，得到每个汽车的评分都可以异步地执行，这会提高系统的性能(延迟)，而且，等待所有的汽车评分被处理使用的是`allOf`方法，而不是手工的线程等待(Thread#join() 或 a CountDownLatch)。

这些例子可以帮助你更好的理解相关的API,你可以在github上得到所有的例子的代码。

### 编程老司机带你玩转CompletableFuture异步编程

本文从实例出发，介绍 `CompletableFuture` 基本用法。不过讲的再多，不如亲自上手练习一下。所以建议各位小伙伴看完，上机练习一把，快速掌握 `CompletableFuture`。

**全文摘要：**

- `Future` VS `CompletableFuture`
- `CompletableFuture` 基本用法

#### 0x00. 前言

一些业务场景我们需要使用多线程异步执行任务，加快任务执行速度。Java 提供 `Runnable` `Future<V>` 两个接口用来实现异步任务逻辑。

虽然 `Future<V>` 可以获取任务执行结果，但是获取方式十方不变。我们不得不使用`Future#get` 阻塞调用线程，或者使用轮询方式判断 `Future#isDone` 任务是否结束，再获取结果。

这两种处理方式都不是很优雅，JDK8 之前并发类库没有提供相关的异步回调实现方式。没办法，我们只好借助第三方类库，如 `Guava`，扩展 `Future`，增加支持回调功能。相关代码如下:

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbsXYiaPGen0hSeK3DjIgnrqE3wknS6djYcibNQY6xQp85ribOyPNG0PTQQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

虽然这种方式增强了 Java 异步编程能力，但是还是无法解决多个异步任务需要相互依赖的场景。

举一个生活上的例子，假如我们需要出去旅游，需要完成三个任务：

- 任务一：订购航班
- 任务二：订购酒店
- 任务三：订购租车服务

很显然任务一和任务二没有相关性，可以单独执行。但是任务三必须等待任务一与任务二结束之后，才能订购租车服务。

为了使任务三时执行时能获取到任务一与任务二执行结果，我们还需要借助 `CountDownLatch` 。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbiaadEnfTkBflicbDd08285zRbLnl1L52a6v9bEfiaVB0rWW0sDDYEc5LA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### 0x01. CompletableFuture

JDK8 之后，Java 新增一个功能十分强大的类：`CompletableFuture`。单独使用这个类就可以轻松的完成上面的需求:

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbOZY8ZWFXGu6wjDXUFfJiarfE56CUiabdl7hVdu8eLte8NvmbLu3Qa4sA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

> 大家可以先不用管 `CompletableFuture` 相关 `API`，下面将会具体讲解。

对比 `Future<V>`，`CompletableFuture` 优点在于：

- 不需要手工分配线程，JDK 自动分配
- 代码语义清晰，异步任务链式调用
- 支持编排异步任务

##### 1.1 方法一览

首先来通过 IDE 查看下这个类提供的方法：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbWe46A5HDUpFcVegITT4oRiaDU1aeZNVpX0F0nsYdTeZe8URzwgctIPQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbvrf6AicibwVqf90ogyGdfpoicyATsD7q9wtCqoSNMp7WxN4ouJZaT10Ag/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

##### 1.2 创建 CompletableFuture 实例

创建 `CompletableFuture` 对象实例我们可以使用如下几个方法：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbdVsialr8HcPdmjOT1MZPEIicxicSZRXNCrvNaUSk55Y9IMbibEMDibUWWmw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

第一个方法创建一个具有默认结果的 `CompletableFuture`，这个没啥好讲。我们重点讲述下下面四个异步方法。

前两个方法 `runAsync` 不支持返回值，而 `supplyAsync`可以支持返回结果。

这个两个方法默认将会使用公共的 `ForkJoinPool` 线程池执行，这个线程池默认线程数是 **CPU** 的核数。

> 可以设置 JVM option:-Djava.util.concurrent.ForkJoinPool.common.parallelism 来设置 ForkJoinPool 线程池的线程数

使用共享线程池将会有个弊端，一旦有任务被阻塞，将会造成其他任务没机会执行。所以**强烈**建议使用后两个方法，根据任务类型不同，主动创建线程池，进行资源隔离，避免互相干扰。

##### 1.3 设置任务结果

`CompletableFuture` 提供以下方法，可以主动设置任务结果。

```java
1 boolean complete(T value)
2 boolean completeExceptionally(Throwable ex)
```

第一个方法，主动设置 `CompletableFuture` 任务执行结果，若返回 `true`，表示设置成功。如果返回 `false`，设置失败，这是因为任务已经执行结束，已经有了执行结果。

示例代码如下：

```java
 // 执行异步任务
 CompletableFuture cf = CompletableFuture.supplyAsync(() -> {
   System.out.println("cf 任务执行开始");
   sleep(10, TimeUnit.SECONDS);
   System.out.println("cf 任务执行结束");
   return "楼下小黑哥";
 });
 //
 Executors.newSingleThreadScheduledExecutor().execute(() -> {
  sleep(5, TimeUnit.SECONDS);
  System.out.println("主动设置 cf 任务结果");
  // 设置任务结果，由于 cf 任务未执行结束，结果返回 true
  cf.complete("程序通事");
});
// 由于 cf 未执行结束，将会被阻塞。5 秒后，另外一个线程主动设置任务结果
System.out.println("get:" + cf.get());
// 等待 cf 任务执行结束
sleep(10, TimeUnit.SECONDS);
// 由于已经设置任务结果，cf 执行结束任务结果将会被抛弃
System.out.println("get:" + cf.get());
 /***
   * cf 任务执行开始
   * 主动设置 cf 任务结果
   * get:程序通事
   * cf 任务执行结束
   * get:程序通事
   */
```

这里需要注意一点，一旦 `complete` 设置成功，`CompletableFuture` 返回结果就不会被更改，即使后续 `CompletableFuture` 任务执行结束。

第二个方法，给 `CompletableFuture` 设置异常对象。若设置成功，如果调用 `get` 等方法获取结果，将会抛错。

示例代码如下：

```java
 // 执行异步任务
 CompletableFuture cf = CompletableFuture.supplyAsync(() -> {
     System.out.println("cf 任务执行开始");
     sleep(10, TimeUnit.SECONDS);
     System.out.println("cf 任务执行结束");
     return "楼下小黑哥";
 });
 //
 Executors.newSingleThreadScheduledExecutor().execute(() -> {
    sleep(5, TimeUnit.SECONDS);
    System.out.println("主动设置 cf 异常");
    // 设置任务结果，由于 cf 任务未执行结束，结果返回 true
    cf.completeExceptionally(new RuntimeException("啊，挂了"));
});
// 由于 cf 未执行结束，前 5 秒将会被阻塞。后续程序抛出异常，结束
System.out.println("get:" + cf.get());
/***
 * cf 任务执行开始
 * 主动设置 cf 异常
 * java.util.concurrent.ExecutionException: java.lang.RuntimeException: 啊，挂了
 * ......
 */
```

##### 1.4 CompletionStage

`CompletableFuture` 分别实现两个接口 `Future`与 `CompletionStage`。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbe0H0LPb2ahiaflxicAfOiceqQvOHArzt9D0BZZueCnib5aETIEASNSNGsA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

`Future` 接口大家都比较熟悉，这里主要讲讲 `CompletionStage`。

`CompletableFuture` 大部分方法来自`CompletionStage` 接口，正是因为这个接口，`CompletableFuture`才有如此强大功能。

想要理解 `CompletionStage` 接口，我们需要先了解任务的时序关系的。我们可以将任务时序关系分为以下几种：

- 串行执行关系
- 并行执行关系
- AND 汇聚关系
- OR 汇聚关系

##### 1.5 串行执行关系

任务串行执行，下一个任务必须等待上一个任务完成才可以继续执行。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbuEA21mbaxibvuqBjbMtRv6eNPJ2O59diaiaHxSE9fvck0rIOl2kWKh3xA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

`CompletionStage` 有四组接口可以描述串行这种关系，分别为:

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbQicNjSxfj4jbodsUtdy2Ws9tffz9AKy6fwIdLHcOQ7J7oib4jCeA521w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

`thenApply` 方法需要传入核心参数为 `Function<T,R>`类型。这个类核心方法为：

```
1 R apply(T t)
```

所以这个接口将会把上一个任务返回结果当做入参，执行结束将会返回结果。

`thenAccept` 方法需要传入参数对象为 `Consumer<T>`类型，这个类核心方法为：

```
1void accept(T t)
```

返回值 `void` 可以看出，这个方法不支持返回结果，但是需要将上一个任务执行结果当做参数传入。

`thenRun` 方法需要传入参数对象为 `Runnable` 类型，这个类大家应该都比较熟悉，核心方法既不支持传入参数，也不会返回执行结果。

`thenCompose` 方法作用与 `thenApply` 一样，只不过 `thenCompose` 需要返回新的  `CompletionStage`。这么理解比较抽象，可以集合代码一起理解。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbgQdbzWnaHnUiamXYjrjFl2dViaZT0Uqa6ibQu3EhNztxyLicjSSR9cKN8w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

方法中带有 **Async** ，代表可以异步执行，这个系列还有重载方法，可以传入自定义的线程池，上图未展示，读者只可以自行查看 API。

最后我们通过代码展示 `thenApply` 使用方式：

```java
CompletableFuture<String> cf
        = CompletableFuture.supplyAsync(() -> "hello,楼下小黑哥")// 1
        .thenApply(s -> s + "@程序通事") // 2
        .thenApply(String::toUpperCase); // 3
System.out.println(cf.join());
// 输出结果 HELLO,楼下小黑哥@程序通事
```

这段代码比较简单，首先我们开启一个异步任务，接着串行执行后续两个任务。任务 2 需要等待任务1 执行完成，任务 3 需要等待任务 2。

> 上面方法，大家需要记住了  `Function<T，R>`，`Consumer<T>`，`Runnable` 三者区别，根据场景选择使用。

##### 1.6 AND 汇聚关系

AND 汇聚关系代表所有任务完成之后，才能进行下一个任务。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbibGsYVMAJKYAFUGdSJiajEmP5pGo3aBhYk0xHcx74ibIY8hQIcP41F2pw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

如上所示，只有任务 A 与任务 B 都完成之后，任务 C 才会开始执行。

`CompletionStage` 有以下接口描述这种关系。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbMMLpUa5GQrqc1SFC73MtPRic2xt7blhQGKV7nqibczyghHzRGcysNCUw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

`thenCombine` 方法核心参数 `BiFunction` ，作用与 `Function`一样，只不过 `BiFunction` 可以接受两个参数，而 `Function` 只能接受一个参数。

`thenAcceptBoth` 方法核心参数`BiConsumer` 作用也与 `Consumer`一样，不过其需要接受两个参数。

`runAfterBoth` 方法核心参数最简单，上面已经介绍过，不再介绍。

这三组方法只能完成两个任务 AND 汇聚关系，如果需要完成多个任务汇聚关系，需要使用 `CompletableFuture#allOf`，不过这里需要注意，这个方法是不支持返回任务结果。

AND 汇聚关系相关示例代码，开头已经使用过了，这里再粘贴一下，方便大家理解：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbOZY8ZWFXGu6wjDXUFfJiarfE56CUiabdl7hVdu8eLte8NvmbLu3Qa4sA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

##### 1.7 OR 汇聚关系

有 AND 汇聚关系，当然也存在 OR 汇聚关系。OR 汇聚关系代表只要多个任务中任一任务完成，就可以接着接着执行下一任务。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwb51Yric2RkJvGjn6mibbCeibZyDh6lOhpMlQIFYnjxETzzqqPVZibJoUVbA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

`CompletionStage` 有以下接口描述这种关系：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwblHiauqaB32bicwuOS5vYreAursXxUXCTpVHxce3icuYHRzor8tweApicBg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

前面三组接口方法传参与 AND 汇聚关系一致，这里也不再详细解释了。

当然 OR 汇聚关系可以使用 `CompletableFuture#anyOf` 执行多个任务。

下面示例代码展示如何使用 `applyToEither` 完成 OR 关系。

```java
 CompletableFuture<String> cf
         = CompletableFuture.supplyAsync(() -> {
     sleep(5, TimeUnit.SECONDS);
     return "hello,楼下小黑哥";
 });// 1
 
 CompletableFuture<String> cf2 = cf.supplyAsync(() -> {
     sleep(3, TimeUnit.SECONDS);
     return "hello，程序通事";
});
// 执行 OR 关系
CompletableFuture<String> cf3 = cf2.applyToEither(cf, s -> s);

// 输出结果，由于 cf2 只休眠 3 秒，优先执行完毕
System.out.println(cf2.join());
// 结果：hello，程序通事
```

##### 1.8 异常处理

`CompletableFuture` 方法执行过程若产生异常，当调用 `get`，`join`获取任务结果才会抛出异常。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbbvwScJAZMLtQXIZGfKLjYbj2zaibO00vGloibZoaK7ZmSBbQaBhVjmzg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

上面代码我们显示使用 `try..catch` 处理上面的异常。不过这种方式不太优雅，`CompletionStage` 提供几个方法，可以优雅处理异常。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbrQ00M3KMG0oIcnChfzQicRBFibps8KXVaazTmWK86HE9diaeERjwvo1Hw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

`exceptionally` 使用方式类似于 `try..catch` 中 `catch`代码块中异常处理。

`whenComplete` 与 `handle` 方法就类似于 `try..catch..finanlly` 中 `finally` 代码块。无论是否发生异常，都将会执行的。这两个方法区别在于  `handle` 支持返回结果。

下面示例代码展示 `handle` 用法：

```java
CompletableFuture<Integer>
         f0 = CompletableFuture.supplyAsync(() -> (7 / 0))
         .thenApply(r -> r * 10)
         .handle((integer, throwable) -> {
             // 如果异常存在,打印异常，并且返回默认值
             if (throwable != null) {
                throwable.printStackTrace();
                return 0;
            } else {
                // 如果
                return integer;
            }
        });


System.out.println(f0.join());
/**
 *java.util.concurrent.CompletionException: java.lang.ArithmeticException: / by zero
 * .....
 * 
 * 0
 */
```

#### 0x02. 总结

JDK8 提供 `CompletableFuture` 功能非常强大，可以编排异步任务，完成串行执行，并行执行，AND 汇聚关系，OR 汇聚关系。

不过这个类方法实在太多，且方法还需要传入各种函数式接口，新手刚开始使用会直接会被弄懵逼。这里帮大家在总结一下三类核心参数的作用

- `Function` 这类函数接口既支持接收参数，也支持返回值
- `Consumer` 这类接口函数只支持接受参数，不支持返回值
- `Runnable` 这类接口不支持接受参数，也不支持返回值

搞清楚函数参数作用以后，然后根据串行，AND 汇聚关系，OR 汇聚关系归纳一下相关方法，这样就比较好理解了

最后再贴一下，文章开头的思维导图，希望对你有帮助。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/LEFcpfxrbq6bHhbMiaQtFEvKibl3icxVDwbvrf6AicibwVqf90ogyGdfpoicyATsD7q9wtCqoSNMp7WxN4ouJZaT10Ag/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### 0x03. 帮助文档

1. 极客时间-并发编程专栏
2. https://colobu.com/2016/02/29/Java-CompletableFuture
3. https://www.ibm.com/developerworks/cn/java/j-cf-of-jdk8/index.html

### 异步并发利器:实际项目中使用CompletionService提升系统性能的一次实践

#### 场景

随着互联网应用的深入，很多传统行业也都需要接入到互联网。我们公司也是这样，保险核心需要和很多保险中介对接，比如阿里、京东等等。这些公司对于接口服务的性能有些比较高的要求，传统的核心无法满足要求，所以信息技术部领导高瞻远瞩，决定开发互联网接入服务，满足来自性能的需求。

#### 概念

**CompletionService**将**Executor**和**BlockingQueue**的功能融合在一起，将**Callable**任务提交给**CompletionService**来执行，然后使用类似于队列操作的take和poll等方法来获得已完成的结果，而这些结果会在完成时被封装为Future。对于更多的概念，请参阅其他网络文档。

线程池的设计，阿里开发手册说过不要使用Java Executors 提供的默认线程池，因此需要更接近实际的情况来自定义一个线程池，根据多次压测，采用的线程池如下：

```java
  public ExecutorService getThreadPool(){
          return new ThreadPoolExecutor(75,
                  125,
                  180000,
                  TimeUnit.MILLISECONDS,
                  new LinkedBlockingDeque<>(450),
                  new ThreadPoolExecutor.CallerRunsPolicy());
      }
```

**说明：**公司的业务为低频交易，对于单次调用性能要求高，但是并发压力根本不大，所以 阻塞队列已满且线程数达到最大值时所采取的饱和策略为调用者执行。

#### 实现

##### 业务

投保业务主要涉及这几个大的方面：投保校验、核保校验、保费试算

- **投保校验：**最主要的是要查询客户黑名单和风险等级，都是千万级的表。而且投保人和被保人都需要校验
- **核保校验：**除了常规的核保规则校验，查询千万级的大表，还需要调用外部智能核保接口获得用户的风险等级，投保人和被保人都需要校验
- **保费试算：**需要计算每个险种的保费

##### 设计

根据上面的业务，如果串行执行的话，单次性能肯定不高，所以考虑多线程异步执行获得校验结果，再对结果综合判断

- **投保校验：**采用一个线程(也可以根据投保人和被保人数量来采用几个线程)

- **核保校验：**

- - **常规校验：**采用一个线程
  - **外部调用：**有几个用户(指投保人和被保人)就采用几个线程

- **保费计算：**有几个险种就采用几个线程，最后合并得到整个的保费

##### 代码

**以下代码是样例，实际逻辑已经去掉**

**先创建投保、核保（常规、外部调用）、保费计算4个业务服务类：**

投保服务类：**InsuranceVerificationServiceImpl**，假设耗时50ms

```java
    @Service
    public class InsuranceVerificationServiceImpl implements InsuranceVerificationService {
        private static final Logger logger = LoggerFactory.getLogger(InsuranceVerificationServiceImpl.class);
        @Override
        public TaskResponseModel<Object> insuranceCheck(String key, PolicyModel policyModel) {
            try {
                //假设耗时50ms
                Thread.sleep(50);            
                return TaskResponseModel.success().setKey(key).setData(policyModel);
            } catch (InterruptedException e) {
                logger.warn(e.getMessage());            
                return TaskResponseModel.failure().setKey(key).setResultMessage(e.getMessage());
            }
        }
    }
```

核保常规校验服务类：**UnderwritingCheckServiceImpl**，假设耗时50ms

```java
    @Service
    public class UnderwritingCheckServiceImpl implements UnderwritingCheckService {
        private static final Logger logger = LoggerFactory.getLogger(UnderwritingCheckServiceImpl.class);
        @Override
        public TaskResponseModel<Object> underwritingCheck(String key, PolicyModel policyModel) {
            try {
                //假设耗时50ms
                Thread.sleep(50);            
                return TaskResponseModel.success().setKey(key).setData(policyModel);
            } catch (InterruptedException e) {
                logger.warn(e.getMessage());            
                return TaskResponseModel.failure().setKey(key).setResultMessage(e.getMessage());
            }
        }
    }
```

核保外部调用服务类：**ExternalCallServiceImpl**，假设耗时200ms

```java
    @Service
    public class ExternalCallServiceImpl implements ExternalCallService {
        private static final Logger logger = LoggerFactory.getLogger(ExternalCallServiceImpl.class);
        @Override
        public TaskResponseModel<Object> externalCall(String key, Insured insured) {
            try {
                //假设耗时200ms
                Thread.sleep(200);
                ExternalCallResultModel externalCallResultModel = new ExternalCallResultModel();
                externalCallResultModel.setIdcard(insured.getIdcard());
                externalCallResultModel.setScore(200);
                return TaskResponseModel.success().setKey(key).setData(externalCallResultModel);
            } catch (InterruptedException e) {
                logger.warn(e.getMessage());
                return TaskResponseModel.failure().setKey(key).setResultMessage(e.getMessage());
            }
        }
    }
```

试算服务类：**TrialCalculationServiceImpl**，假设耗时50ms

```java
    @Service
    public class TrialCalculationServiceImpl implements TrialCalculationService {
        private static final Logger logger = LoggerFactory.getLogger(TrialCalculationServiceImpl.class);
        @Override
        public TaskResponseModel<Object> trialCalc(String key, Risk risk) {
            try {
                //假设耗时50ms
                Thread.sleep(50);
                return TaskResponseModel.success().setKey(key).setData(risk);
            } catch (InterruptedException e) {
                logger.warn(e.getMessage());
                return TaskResponseModel.failure().setKey(key).setResultMessage(e.getMessage());
            }
        }
    }
```

统一返回接口类：**TaskResponseModel**， 上面4个服务的方法统一返回**TaskResponseModel**

```java
  @Data
  @ToString
  @NoArgsConstructor
  @AllArgsConstructor
  @EqualsAndHashCode
  @Accessors(chain = true)
  public class TaskResponseModel<T extends Object> implements Serializable {
      private String key;           //唯一调用标志
      private String resultCode;    //结果码
      private String resultMessage; //结果信息
      private T data;               //业务处理结果

      public static TaskResponseModel<Object> success() {
          TaskResponseModel<Object> taskResponseModel = new TaskResponseModel<>();
          taskResponseModel.setResultCode("200");
          return taskResponseModel;
      }
      public static TaskResponseModel<Object> failure() {
          TaskResponseModel<Object> taskResponseModel = new TaskResponseModel<>();
          taskResponseModel.setResultCode("400");
          return taskResponseModel;
      }
  }
```

**注：**

1. **key**为这次调用的唯一标识，由调用者传进来
2. **resultCode**结果码，200为成功，400表示有异常
3. **resultMessage**信息，表示不成功或者异常信息
4. **data**业务处理结果，如果成功的话
5. 这些服务类都是单例模式

要使用用**CompletionService**的话，需要创建实现了**Callable**接口的线程

投保**Callable**:

```java
    @Data
    @AllArgsConstructor
    public class InsuranceVerificationCommand implements Callable<TaskResponseModel<Object>> {
        private String key;
        private PolicyModel policyModel;
        private final InsuranceVerificationService insuranceVerificationService;
        @Override
        public TaskResponseModel<Object> call() throws Exception {
            return insuranceVerificationService.insuranceCheck(key, policyModel);
        }
    }
```

核保常规校验**Callable**:

```java
    @Data
    @AllArgsConstructor
    public class UnderwritingCheckCommand implements Callable<TaskResponseModel<Object>> {
        private String key;
        private PolicyModel policyModel;
        private final UnderwritingCheckService underwritingCheckService;
        @Override
        public TaskResponseModel<Object> call() throws Exception {
            return underwritingCheckService.underwritingCheck(key, policyModel);
        }
    }
```

核保外部调用**Callable**:

```java
    @Data
    @AllArgsConstructor
    public class ExternalCallCommand implements Callable<TaskResponseModel<Object>> {
        private String key;
        private Insured insured;
        private final ExternalCallService externalCallService;
        @Override
        public TaskResponseModel<Object> call() throws Exception {
            return externalCallService.externalCall(key, insured);
        }
    }
```

试算调用**Callable**:

```java
    @Data
    @AllArgsConstructor
    public class TrialCalculationCommand implements Callable<TaskResponseModel<Object>> {
        private String key;
        private Risk risk;
        private final TrialCalculationService trialCalculationService;
        @Override
        public TaskResponseModel<Object> call() throws Exception {
            return trialCalculationService.trialCalc(key, risk);
        }
    }
```

**注**：

1. 每一次调用，需要创建这4种**Callable**
2. 返回统一接口**TaskResopnseModel**

异步执行的类：**TaskExecutor**

```java
  @Component
  public class TaskExecutor {
      private static final Logger logger = LoggerFactory.getLogger(TaskExecutor.class);
      //线程池
      private final ExecutorService executorService;

      public TaskExecutor(ExecutorService executorService) {
          this.executorService = executorService;
      }

      //异步执行，获取所有结果后返回
      public List<TaskResponseModel<Object>> execute(List<Callable<TaskResponseModel<Object>>> commands) {
          //创建异步执行对象
          CompletionService<TaskResponseModel<Object>> completionService = new ExecutorCompletionService<>(executorService);
          for (Callable<TaskResponseModel<Object>> command : commands) {
              completionService.submit(command);
          }
          //获取所有异步执行线程的结果
          int taskCount = commands.size();
          List<TaskResponseModel<Object>> params = new ArrayList<>(taskCount);
          try {
              for (int i = 0; i < taskCount; i++) {
                  Future<TaskResponseModel<Object>> future = completionService.take();
                  params.add(future.get());
              }
          } catch (InterruptedException | ExecutionException e) {
              //异常处理
              params.clear();
              params.add(TaskResponseModel.failure().setKey("error").setResultMessage("异步执行线程错误"));
          }
          //返回，如果执行中发生error, 则返回相应的key值：error
          return params;
      }
  }
```

**注**：

1. 为单例模式
2. 接收参数为List<Callable<TaskResponseModel<Object>>>，也就是上面定义的4种Callable的列表
3. 返回List<TaskResponseModel<Object>>，也就是上面定义4种Callable返回的结果列表
4. 我们的业务是对返回结果统一判断，业务返回结果有因果关系
5. 如果线程执行有异常，也返回List<TaskResponseModel>，这个时候列表中只有一个**TaskResponseModel**，**key**为error, 后续调用者可以通过这个来判断线程是否执行成功；

调用方：CompletionServiceController

```java
  @RestController
  public class CompletionServiceController {
      //投保key
      private static final String INSURANCE_KEY = "insurance_";
      //核保key
      private static final String UNDERWRITING_KEY = "underwriting_";
      //外部调用key
      private static final String EXTERNALCALL_KEY = "externalcall_";
      //试算key
      private static final String TRIA_KEY = "trial_";

      private static final Logger logger = LoggerFactory.getLogger(CompletionServiceController.class);

      private final ExternalCallService externalCallService;
      private final InsuranceVerificationService insuranceVerificationService;
      private final TrialCalculationService trialCalculationService;
      private final UnderwritingCheckService underwritingCheckService;
      private final TaskExecutor taskExecutor;

      public CompletionServiceController(ExternalCallService externalCallService, InsuranceVerificationService insuranceVerificationService, TrialCalculationService trialCalculationService, UnderwritingCheckService underwritingCheckService, TaskExecutor taskExecutor) {
          this.externalCallService = externalCallService;
          this.insuranceVerificationService = insuranceVerificationService;
          this.trialCalculationService = trialCalculationService;
          this.underwritingCheckService = underwritingCheckService;
          this.taskExecutor = taskExecutor;
      }

      //多线程异步并发接口
      @PostMapping(value = "/async", headers = "Content-Type=application/json;charset=UTF-8")
      public String asyncExec(@RequestBody PolicyModel policyModel) {
          long start = System.currentTimeMillis();

          asyncExecute(policyModel);
          logger.info("异步总共耗时：" + (System.currentTimeMillis() - start));
          return "ok";
      }

      //串行调用接口
      @PostMapping(value = "/sync", headers = "Content-Type=application/json;charset=UTF-8")
      public String syncExec(@RequestBody PolicyModel policyModel) {
          long start = System.currentTimeMillis();
          syncExecute(policyModel);
          logger.info("同步总共耗时：" + (System.currentTimeMillis() - start));
          return "ok";
      }
      private void asyncExecute(PolicyModel policyModel) {
          List<Callable<TaskResponseModel<Object>>> baseTaskCallbackList = new ArrayList<>();
          //根据被保人外部接口调用
          for (Insured insured : policyModel.getInsuredList()) {
              ExternalCallCommand externalCallCommand = new ExternalCallCommand(EXTERNALCALL_KEY + insured.getIdcard(), insured, externalCallService);
              baseTaskCallbackList.add(externalCallCommand);
          }
          //投保校验
          InsuranceVerificationCommand insuranceVerificationCommand = new InsuranceVerificationCommand(INSURANCE_KEY, policyModel, insuranceVerificationService);
          baseTaskCallbackList.add(insuranceVerificationCommand);
          //核保校验
          UnderwritingCheckCommand underwritingCheckCommand = new UnderwritingCheckCommand(UNDERWRITING_KEY, policyModel, underwritingCheckService);
          baseTaskCallbackList.add(underwritingCheckCommand);
          //根据险种进行保费试算
          for(Risk risk : policyModel.getRiskList()) {
              TrialCalculationCommand trialCalculationCommand = new TrialCalculationCommand(TRIA_KEY + risk.getRiskcode(), risk, trialCalculationService);
              baseTaskCallbackList.add(trialCalculationCommand);
          }
          List<TaskResponseModel<Object>> results = taskExecutor.execute(baseTaskCallbackList);
          for (TaskResponseModel<Object> t : results) {
              if (t.getKey().equals("error")) {
                  logger.warn("线程执行失败");
                  logger.warn(t.toString());
              }
              logger.info(t.toString());
          }

      }
      private void syncExecute(PolicyModel policyModel) {
          //根据被保人外部接口调用
          for (Insured insured : policyModel.getInsuredList()) {
              TaskResponseModel<Object> externalCall = externalCallService.externalCall(insured.getIdcard(), insured);
              logger.info(externalCall.toString());
          }
          //投保校验
          TaskResponseModel<Object> insurance = insuranceVerificationService.insuranceCheck(INSURANCE_KEY, policyModel);
          logger.info(insurance.toString());
          //核保校验
          TaskResponseModel<Object> underwriting = underwritingCheckService.underwritingCheck(UNDERWRITING_KEY, policyModel);
          logger.info(underwriting.toString());
          //根据险种进行保费试算
          for(Risk risk : policyModel.getRiskList()) {
              TaskResponseModel<Object> risktrial = trialCalculationService.trialCalc(risk.getRiskcode(), risk);
              logger.info(risktrial.toString());
          }

      }
  }
```

**注**：

1.为测试方便，提供两个接口调用：一个是串行执行，一个是异步并发执行

2.在异步并发执行函数**asyncExecute**中：

1. 根据有多少个被保人，创建多少个外部调用的Callable实例，**key**值为**EXTERNALCALL_KEY + insured.getIdcard()**，在一次保单投保调用中，每一个被保人**Callable**的**key**是不一样的。
2. 根据有多少个险种，创建多少个试算的**Callable**实例，**key**为**TRIA_KEY + risk.getRiskcode()**，在一次保单投保调用中，每一个险种的**Callable**的key是不一样的
3. 创建投保校验的**Callable**实例，业务上只需要一个
4. 创建核保校验的**Callable**实例，业务上只需要一个
5. 将Callable列表传入到**TaskExecutor**执行异步并发调用
6. 根据返回结果来判断，通过判断返回的**TaskResponseModel**的**key**值可以知道是哪类业务校验，分别进行判断，还可以交叉判断（公司的业务就是要交叉判断）

#### 验证

验证数据：

```json
{
"insuredList":
[{"idcard":"laza","name":"320106"},
{"idcard":"ranran","name":"120102"}],
"policyHolder":"lazasha","policyNo":"345000987","riskList":
[{"mainFlag":1,"premium":300,"riskcode":"risk001","riskname":"险种一"},
{"mainFlag":0,"premium":400,"riskcode":"risk002","riskname":"险种二"}]
}
```

上面数据表明：有两个被保人，两个险种。按照我们上面的定义，会调用两次外部接口，两次试算，一次投保，一次核保。而在样例代码中，一次外部接口调用耗时为200ms, 其他都为50ms.

**本地开发的配置为8C16G:**

- 同步串行接口调用计算：2 * 200 + 2 * 50 + 50 + 50 = 600ms
- 多线程异步执行调用计算：按照多线程并发执行原理，取耗时最长的200ms

**验证：同步接口**

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufOBllTqKzsfEESCxFScnicJkwFByic5Kf6DoMibyaEdbmYr1SyFAbRNEpDlFXXkGmdwLXNq04f6PUDA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

输出耗时：可以看到耗时**601ms**

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufOBllTqKzsfEESCxFScnicJJCFNYJTxHsDCM8opFxqL6Z3Ds0749kkXG7nFNOpEOvpEN3ibQYXNviaQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**验证：多线程异步执行接口**

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufOBllTqKzsfEESCxFScnicJqTKjPicBrabYlJDrqmJfPbovcbL7nDZ2IeWQZXZ9u2CbVubU2Hf5GSg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

输出耗时：可以看到为**204ms**

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufOBllTqKzsfEESCxFScnicJLhzibdZlSQBG0xH07OAAhjRA9NCUIp9E7csjWTUUIiacj1jibyYUWzsVA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

结果：基本和我们的预期相符合。

#### 结束

这是将实际生产中的例子简化出来，具体生产的业务比较复杂，不便于展示。

实际情况下，原来的接口需要1000ms以上才能完成单次调用，有的需要2000-3000ms。现在的接口，在生产两台8c16g的虚拟机, 经过4个小时的简单压测能够支持2000用户并发，单次返回时长为350ms左右，服务很稳定，完全能够满足公司的业务发展需求。

> 提供的这个是可以运行的列子，代码在：https://github.com/lazasha111211/completionservice-demo.git

### 大文件上传:秒传、断点续传、分片上传

#### 前言

文件上传是一个老生常谈的话题了，在文件相对比较小的情况下，可以直接把文件转化为字节流上传到服务器，但在文件比较大的情况下，用普通的方式进行上传，这可不是一个好的办法，毕竟很少有人会忍受，当文件上传到一半中断后，继续上传却只能重头开始上传，这种让人不爽的体验。那有没有比较好的上传体验呢，答案有的，就是下边要介绍的几种上传方式

#### 详细教程

##### 秒传

###### 1、什么是秒传

通俗的说，你把要上传的东西上传，服务器会先做MD5校验，如果服务器上有一样的东西，它就直接给你个新地址，其实你下载的都是服务器上的同一个文件，想要不秒传，其实只要让MD5改变，就是对文件本身做一下修改（改名字不行），例如一个文本文件，你多加几个字，MD5就变了，就不会秒传了.

###### 2、本文实现的秒传核心逻辑

a、利用redis的set方法存放文件上传状态，其中key为文件上传的md5，value为是否上传完成的标志位，

b、当标志位true为上传已经完成，此时如果有相同文件上传，则进入秒传逻辑。如果标志位为false，则说明还没上传完成，此时需要在调用set的方法，保存块号文件记录的路径，其中key为上传文件md5加一个固定前缀，value为块号文件记录路径

##### 分片上传

###### 1.什么是分片上传

分片上传，就是将所要上传的文件，按照一定的大小，将整个文件分隔成多个数据块（我们称之为Part）来进行分别上传，上传完之后再由服务端对所有上传的文件进行汇总整合成原始的文件。

###### 2.分片上传的场景

1.大文件上传

2.网络环境环境不好，存在需要重传风险的场景

##### 断点续传

###### 1、什么是断点续传

断点续传是在下载或上传时，将下载或上传任务（一个文件或一个压缩包）人为的划分为几个部分，每一个部分采用一个线程进行上传或下载，如果碰到网络故障，可以从已经上传或下载的部分开始继续上传或者下载未完成的部分，而没有必要从头开始上传或者下载。本文的断点续传主要是针对断点上传场景。

###### 2、应用场景

断点续传可以看成是分片上传的一个衍生，因此可以使用分片上传的场景，都可以使用断点续传。

###### 3、实现断点续传的核心逻辑

在分片上传的过程中，如果因为系统崩溃或者网络中断等异常因素导致上传中断，这时候客户端需要记录上传的进度。在之后支持再次上传时，可以继续从上次上传中断的地方进行继续上传。

为了避免客户端在上传之后的进度数据被删除而导致重新开始从头上传的问题，服务端也可以提供相应的接口便于客户端对已经上传的分片数据进行查询，从而使客户端知道已经上传的分片数据，从而从下一个分片数据开始继续上传。

###### 4、实现流程步骤

a、方案一，常规步骤

- 将需要上传的文件按照一定的分割规则，分割成相同大小的数据块；
- 初始化一个分片上传任务，返回本次分片上传唯一标识；
- 按照一定的策略（串行或并行）发送各个分片数据块；
- 发送完成后，服务端根据判断数据上传是否完整，如果完整，则进行数据块合成得到原始文件。

b、方案二、本文实现的步骤

- 前端（客户端）需要根据固定大小对文件进行分片，请求后端（服务端）时要带上分片序号和大小
- 服务端创建conf文件用来记录分块位置，conf文件长度为总分片数，每上传一个分块即向conf文件中写入一个127，那么没上传的位置就是默认的0,已上传的就是Byte.MAX_VALUE 127（这步是实现断点续传和秒传的核心步骤）
- 服务器按照请求数据中给的分片序号和每片分块大小（分片大小是固定且一样的）算出开始位置，与读取到的文件片段数据，写入文件。

###### 5、分片上传/断点上传代码实现

a、前端采用百度提供的webuploader的插件，进行分片。因本文主要介绍服务端代码实现，webuploader如何进行分片，具体实现可以查看如下链接:

> http://fex.baidu.com/webuploader/getting-started.html

b、后端用两种方式实现文件写入，一种是用RandomAccessFile，如果对RandomAccessFile不熟悉的朋友，可以查看如下链接:

> https://blog.csdn.net/dimudan2015/article/details/81910690

另一种是使用MappedByteBuffer，对MappedByteBuffer不熟悉的朋友，可以查看如下链接进行了解:

> https://www.jianshu.com/p/f90866dcbffc

#### 后端进行写入操作的核心代码

a、RandomAccessFile实现方式

```java
@UploadMode(mode = UploadModeEnum.RANDOM_ACCESS)  
@Slf4j  
public class RandomAccessUploadStrategy extends SliceUploadTemplate {  
  
  @Autowired  
  private FilePathUtil filePathUtil;  
  
  @Value("${upload.chunkSize}")  
  private long defaultChunkSize;  
  
  @Override  
  public boolean upload(FileUploadRequestDTO param) {  
    RandomAccessFile accessTmpFile = null;  
    try {  
      String uploadDirPath = filePathUtil.getPath(param);  
      File tmpFile = super.createTmpFile(param);  
      accessTmpFile = new RandomAccessFile(tmpFile, "rw");  
      //这个必须与前端设定的值一致  
      long chunkSize = Objects.isNull(param.getChunkSize()) ? defaultChunkSize * 1024 * 1024  
          : param.getChunkSize();  
      long offset = chunkSize * param.getChunk();  
      //定位到该分片的偏移量  
      accessTmpFile.seek(offset);  
      //写入该分片数据  
      accessTmpFile.write(param.getFile().getBytes());  
      boolean isOk = super.checkAndSetUploadProgress(param, uploadDirPath);  
      return isOk;  
    } catch (IOException e) {  
      log.error(e.getMessage(), e);  
    } finally {  
      FileUtil.close(accessTmpFile);  
    }  
   return false;  
  }  
  
}  
```

b、MappedByteBuffer实现方式

```java
@UploadMode(mode = UploadModeEnum.MAPPED_BYTEBUFFER)  
@Slf4j  
public class MappedByteBufferUploadStrategy extends SliceUploadTemplate {  
  
  @Autowired  
  private FilePathUtil filePathUtil;  
  
  @Value("${upload.chunkSize}")  
  private long defaultChunkSize;  
  
  @Override  
  public boolean upload(FileUploadRequestDTO param) {  
  
    RandomAccessFile tempRaf = null;  
    FileChannel fileChannel = null;  
    MappedByteBuffer mappedByteBuffer = null;  
    try {  
      String uploadDirPath = filePathUtil.getPath(param);  
      File tmpFile = super.createTmpFile(param);  
      tempRaf = new RandomAccessFile(tmpFile, "rw");  
      fileChannel = tempRaf.getChannel();  
  
      long chunkSize = Objects.isNull(param.getChunkSize()) ? defaultChunkSize * 1024 * 1024  
          : param.getChunkSize();  
      //写入该分片数据  
      long offset = chunkSize * param.getChunk();  
      byte[] fileData = param.getFile().getBytes();  
      mappedByteBuffer = fileChannel  
.map(FileChannel.MapMode.READ_WRITE, offset, fileData.length);  
      mappedByteBuffer.put(fileData);  
      boolean isOk = super.checkAndSetUploadProgress(param, uploadDirPath);  
      return isOk;  
  
    } catch (IOException e) {  
      log.error(e.getMessage(), e);  
    } finally {  
  
      FileUtil.freedMappedByteBuffer(mappedByteBuffer);  
      FileUtil.close(fileChannel);  
      FileUtil.close(tempRaf);  
  
    }  
  
    return false;  
  }  
  
}  
```

c、文件操作核心模板类代码

```java
@Slf4j  
public abstract class SliceUploadTemplate implements SliceUploadStrategy {  
  
  public abstract boolean upload(FileUploadRequestDTO param);  
  
  protected File createTmpFile(FileUploadRequestDTO param) {  
  
    FilePathUtil filePathUtil = SpringContextHolder.getBean(FilePathUtil.class);  
    param.setPath(FileUtil.withoutHeadAndTailDiagonal(param.getPath()));  
    String fileName = param.getFile().getOriginalFilename();  
    String uploadDirPath = filePathUtil.getPath(param);  
    String tempFileName = fileName + "_tmp";  
    File tmpDir = new File(uploadDirPath);  
    File tmpFile = new File(uploadDirPath, tempFileName);  
    if (!tmpDir.exists()) {  
      tmpDir.mkdirs();  
    }  
    return tmpFile;  
  }  
  
  @Override  
  public FileUploadDTO sliceUpload(FileUploadRequestDTO param) {  
  
    boolean isOk = this.upload(param);  
    if (isOk) {  
      File tmpFile = this.createTmpFile(param);  
      FileUploadDTO fileUploadDTO = this.saveAndFileUploadDTO(param.getFile().getOriginalFilename(), tmpFile);  
      return fileUploadDTO;  
    }  
    String md5 = FileMD5Util.getFileMD5(param.getFile());  
  
    Map<Integer, String> map = new HashMap<>();  
    map.put(param.getChunk(), md5);  
    return FileUploadDTO.builder().chunkMd5Info(map).build();  
  }  
  
  /**  
   * 检查并修改文件上传进度  
   */  
  public boolean checkAndSetUploadProgress(FileUploadRequestDTO param, String uploadDirPath) {  
  
    String fileName = param.getFile().getOriginalFilename();  
    File confFile = new File(uploadDirPath, fileName + ".conf");  
    byte isComplete = 0;  
    RandomAccessFile accessConfFile = null;  
    try {  
      accessConfFile = new RandomAccessFile(confFile, "rw");  
      //把该分段标记为 true 表示完成  
      System.out.println("set part " + param.getChunk() + " complete");  
      //创建conf文件文件长度为总分片数，每上传一个分块即向conf文件中写入一个127，那么没上传的位置就是默认0,已上传的就是Byte.MAX_VALUE 127  
      accessConfFile.setLength(param.getChunks());  
      accessConfFile.seek(param.getChunk());  
      accessConfFile.write(Byte.MAX_VALUE);  
  
      //completeList 检查是否全部完成,如果数组里是否全部都是127(全部分片都成功上传)  
      byte[] completeList = FileUtils.readFileToByteArray(confFile);  
      isComplete = Byte.MAX_VALUE;  
      for (int i = 0; i < completeList.length && isComplete == Byte.MAX_VALUE; i++) {  
        //与运算, 如果有部分没有完成则 isComplete 不是 Byte.MAX_VALUE  
        isComplete = (byte) (isComplete & completeList[i]);  
        System.out.println("check part " + i + " complete?:" + completeList[i]);  
      }  
  
    } catch (IOException e) {  
      log.error(e.getMessage(), e);  
    } finally {  
      FileUtil.close(accessConfFile);  
    }  
 boolean isOk = setUploadProgress2Redis(param, uploadDirPath, fileName, confFile, isComplete);  
    return isOk;  
  }  
  
  /**  
   * 把上传进度信息存进redis  
   */  
  private boolean setUploadProgress2Redis(FileUploadRequestDTO param, String uploadDirPath,  
      String fileName, File confFile, byte isComplete) {  
  
    RedisUtil redisUtil = SpringContextHolder.getBean(RedisUtil.class);  
    if (isComplete == Byte.MAX_VALUE) {  
      redisUtil.hset(FileConstant.FILE_UPLOAD_STATUS, param.getMd5(), "true");  
      redisUtil.del(FileConstant.FILE_MD5_KEY + param.getMd5());  
      confFile.delete();  
      return true;  
    } else {  
      if (!redisUtil.hHasKey(FileConstant.FILE_UPLOAD_STATUS, param.getMd5())) {  
        redisUtil.hset(FileConstant.FILE_UPLOAD_STATUS, param.getMd5(), "false");  
        redisUtil.set(FileConstant.FILE_MD5_KEY + param.getMd5(),  
            uploadDirPath + FileConstant.FILE_SEPARATORCHAR + fileName + ".conf");  
      }  
  
      return false;  
    }  
  }  
/**  
   * 保存文件操作  
   */  
  public FileUploadDTO saveAndFileUploadDTO(String fileName, File tmpFile) {  
  
    FileUploadDTO fileUploadDTO = null;  
  
    try {  
  
      fileUploadDTO = renameFile(tmpFile, fileName);  
      if (fileUploadDTO.isUploadComplete()) {  
        System.out  
            .println("upload complete !!" + fileUploadDTO.isUploadComplete() + " name=" + fileName);  
        //TODO 保存文件信息到数据库  
  
      }  
  
    } catch (Exception e) {  
      log.error(e.getMessage(), e);  
    } finally {  
  
    }  
    return fileUploadDTO;  
  }  
/**  
   * 文件重命名  
   *  
   * @param toBeRenamed 将要修改名字的文件  
   * @param toFileNewName 新的名字  
   */  
  private FileUploadDTO renameFile(File toBeRenamed, String toFileNewName) {  
    //检查要重命名的文件是否存在，是否是文件  
    FileUploadDTO fileUploadDTO = new FileUploadDTO();  
    if (!toBeRenamed.exists() || toBeRenamed.isDirectory()) {  
      log.info("File does not exist: {}", toBeRenamed.getName());  
      fileUploadDTO.setUploadComplete(false);  
      return fileUploadDTO;  
    }  
    String ext = FileUtil.getExtension(toFileNewName);  
    String p = toBeRenamed.getParent();  
    String filePath = p + FileConstant.FILE_SEPARATORCHAR + toFileNewName;  
    File newFile = new File(filePath);  
    //修改文件名  
    boolean uploadFlag = toBeRenamed.renameTo(newFile);  
  
    fileUploadDTO.setMtime(DateUtil.getCurrentTimeStamp());  
    fileUploadDTO.setUploadComplete(uploadFlag);  
    fileUploadDTO.setPath(filePath);  
    fileUploadDTO.setSize(newFile.length());  
    fileUploadDTO.setFileExt(ext);  
    fileUploadDTO.setFileId(toFileNewName);  
  
    return fileUploadDTO;  
  }  
}  
```

#### 总结

在实现分片上传的过程，需要前端和后端配合，比如前后端的上传块号的文件大小，前后端必须得要一致，否则上传就会有问题。其次文件相关操作正常都是要搭建一个文件服务器的，比如使用fastdfs、hdfs等。

本示例代码在电脑配置为4核内存8G情况下，上传24G大小的文件，上传时间需要30多分钟，主要时间耗费在前端的md5值计算，后端写入的速度还是比较快。如果项目组觉得自建文件服务器太花费时间，且项目的需求仅仅只是上传下载，那么推荐使用阿里的oss服务器，其介绍可以查看官网:

> https://help.aliyun.com/product/31815.html

阿里的oss它本质是一个对象存储服务器，而非文件服务器，因此如果有涉及到大量删除或者修改文件的需求，oss可能就不是一个好的选择。

### MyBatis中使用流式查询避免数据量过大导致OOM

今天mybatis查询数据库中大量的数据，程序抛出：

```
java.lang.OutOfMemoryError: Java heap space
```

看下日志，是因为一次查询数据量过大导致JVM内存溢出了，虽然可以配置JVM大小，但是指标不治本，还是需要优化代码。网上查看大家都是流式查询，这里记录下解决的过程。

#### 1、Mapper.xml配置

select语句需要`增加fetchSize属性`，底层是调用jdbc的setFetchSize方法，查询时从结果集里面每次取设置的行数，循环去取，直到取完。默认size是0，也就是默认会一次性把结果集的数据全部取出来，当结果集数据量很大时就容易造成内存溢出。

```
<select id="listTaskResultIpInfo" fetchSize="1000" resultType="String">
   select info from task_result_info where project_id = #{projectId} and task_id = #{taskId}
</select>
```

> 注意：此时需要在mysql连接URL中增加`useCursorFetch=true`。

```
jdbc.url=jdbc:mysql://127.0.0.1:3306/test?useCursorFetch=true
```

#### 2、自定义ResultHandler

```java
package com.iie.test.handler.result;
 
import com.iie.test.entity.po.custom.CustTaskResultInfo;
import org.apache.ibatis.session.ResultContext;
import org.apache.ibatis.session.ResultHandler;
 
import java.util.ArrayList;
import java.util.List;
 
/**
 * MyBatis中使用流式查询避免数据量过大导致OOM
 */
public class ResultInfoHandler implements ResultHandler<CustTaskResultInfo> {
    // 存储每批数据的临时容器
    private List<CustTaskResultInfo> resultInfoList = new ArrayList<>();
 
    public List<CustTaskResultInfo> getResultInfoList() {
        return resultInfoList;
    }
 
    @Override
    public void handleResult(ResultContext<? extends CustTaskResultInfo> resultContext) {
        CustTaskResultInfo custTaskResultInfo = resultContext.getResultObject();
        resultInfoList.add(CustTaskResultInfo);
    }
 
}
```

#### 3、spring中配置sqlSessionTemplate

```xml
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="configLocation" value="classpath:mybatis/mybatis-config.xml"/>
        <!-- mapper扫描 -->
        <property name="mapperLocations" value="classpath:mybatis/mapper/*/*.xml"/>
    </bean>
    <bean id="sqlSessionTemplate" class="org.mybatis.spring.SqlSessionTemplate">
        <constructor-arg index="0" ref="sqlSessionFactory" />
    </bean>
```

#### 4、service中使用

```java
    @Autowired
    private SqlSessionTemplate sqlSessionTemplate;
  
public List<CustTaskResultInfo> listTaskResultInfo(Long projectId, Long taskId) {
        Map<String, Object> param = new HashMap<>();
        param.put("projectId", projectId);
        param.put("taskId", taskId);
        ResultInfoHandler handler = new ResultInfoHandler();
        sqlSessionTemplate.select("com.iie.cyberpecker.dao.custom.CustTaskResultInfoMapper.listTaskResultInfo", param, handler);
        return handler.getResultInfoList();
    }
```

#### 5、疑问

上面这种方案必须要定义一个sqlSessionTemplate，我想着能不能直接在mapper.xml中配置，网上说的是这样实现：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhoPXh6Z4JcYATrKXbjdkCWERXVXwTXN0vn1qdoUO0rxZRSvsnvhE5ez5rjvC2eFYAs3icOVoGkkX2w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

但是我这样实现一直没有成功，查询数据为空。

### SpringBoot项目鉴权的4种方式

文章介绍spring-boot中实现通用auth的四种方式

包括传统**AOP**、**拦截器**、**参数解析器**和**过滤器**，并提供了对应的实例代码，最后简单总结他们的执行顺序。

#### 传统AOP

对于这种需求，首先想到的当然是 Spring-boot 提供的 AOP 接口，只需要在 Controller 方法前添加切点，然后再对切点进行处理即可。

**实现**

其使用步骤如下：

- 使用 @Aspect 声明一下切面类 WhitelistAspect；
- 在切面类内添加一个切点 whitelistPointcut()，为了实现此切点灵活可装配的能力，这里不使用 execution 全部拦截，而是添加一个注解 @Whitelist，被注解的方法才会校验白名单。
- 在切面类中使用 spring 的 AOP 注解 @Before 声明一个通知方法 checkWhitelist() 在 Controller 方法被执行之前校验白名单。

切面类伪代码如下：

```java
@Aspect
public class WhitelistAspect {
   
 @Before(value = "whitelistPointcut() && @annotation(whitelist)")
 public void checkAppkeyWhitelist(JoinPoint joinPoint, Whitelist whitelist) {
     checkWhitelist();
     // 可使用 joinPoint.getArgs() 获取Controller方法的参数
     // 可以使用 whitelist 变量获取注解参数
 }
   
 @Pointcut("@annotation(com.zhenbianshu.Whitelist)")
 public void whitelistPointCut() {
 }
}
```

在Controller方法上添加 @Whitelist 注解实现功能。

**扩展**

本例中使用了 注解 来声明切点，并且我实现了通过注解参数来声明要校验的白名单，如果之后还需要添加其他白名单的话，如通过 UID 来校验，则可以为此注解添加 uid() 等方法，实现自定义校验。此外，spring 的 AOP 还支持 execution（执行方法） 、bean（匹配特定名称的 Bean 对象的执行方法）等切点声明方法和 @Around（在目标函数执行中执行） 、@After（方法执行后） 等通知方法。如此，功能已经实现了，但领导并不满意=_=，原因是项目中 AOP 用得太多了，都用滥了，建议我换一种方式。嗯，只好搞起。

#### Interceptor

Spring 的 拦截器(Interceptor) 实现这个功能也非常合适。顾名思义，拦截器用于在 Controller 内 Action 被执行前通过一些参数判断是否要执行此方法，要实现一个拦截器，可以实现 Spring 的 HandlerInterceptor 接口。

**实现**

实现步骤如下：

- 定义拦截器类 AppkeyInterceptor 类并实现 HandlerInterceptor 接口。
- 实现其 preHandle() 方法；
- 在 preHandle 方法内通过注解和参数判断是否需要拦截请求，拦截请求时接口返回 false；
- 在自定义的 WebMvcConfigurerAdapter 类内注册此拦截器；

AppkeyInterceptor 类如下：

```java
@Component
public class WhitelistInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        Whitelist whitelist = ((HandlerMethod) handler).getMethodAnnotation(Whitelist.class);
        // whitelist.values(); 通过 request 获取请求参数，通过 whitelist 变量获取注解参数
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
  // 方法在Controller方法执行结束后执行
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
  // 在view视图渲染完成后执行
    }
}
```

**扩展**

要启用 拦截器还要显式配置它启用，这里我们使用 WebMvcConfigurerAdapter 对它进行配置。需要注意，继承它的的 MvcConfiguration 需要在 ComponentScan 路径下。

```java
@Configuration
public class MvcConfiguration extends WebMvcConfigurerAdapter {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new WhitelistInterceptor()).addPathPatterns("/*").order(1);
        // 这里可以配置拦截器启用的 path 的顺序，在有多个拦截器存在时，任一拦截器返回 false 都会使后续的请求方法不再执行
    }
}
```

还需要注意，拦截器执行成功后响应码为 200，但响应数据为空。当使用拦截器实现功能后，领导终于祭出大招了：我们已经有一个 Auth 参数了，appkey 可以从 Auth 参数里取到，可以把在不在白名单作为 Auth 的一种方式，为什么不在 Auth 时校验。

#### ArgumentResolver

参数解析器是 Spring 提供的用于解析自定义参数的工具，我们常用的 @RequestParam 注解就有它的影子，使用它，我们可以将参数在进入Controller Action之前就组合成我们想要的样子。Spring 会维护一个 ResolverList， 在请求到达时，Spring 发现有自定义类型参数（非基本类型）， 会依次尝试这些 Resolver，直到有一个 Resolver 能解析需要的参数。要实现一个参数解析器，需要实现 HandlerMethodArgumentResolver 接口。

**实现**

- 定义自定义参数类型 AuthParam，类内有 appkey 相关字段；
- 定义 AuthParamResolver 并实现 HandlerMethodArgumentResolver 接口；
- 实现 supportsParameter() 接口方法将 AuthParam 与 AuthParamResolver 适配起来；
- 实现 resolveArgument() 接口方法解析 reqest 对象生成 AuthParam 对象，并在此校验 AuthParam ，确认 appkey 是否在白名单内；
- 在 Controller Action 方法上签名内添加 AuthParam 参数以启用此 Resolver;

实现的 AuthParamResolver 类如下：

```java
@Component
public class AuthParamResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().equals(AuthParam.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        Whitelist whitelist = parameter.getMethodAnnotation(Whitelist.class);
        // 通过 webRequest 和 whitelist 校验白名单
        return new AuthParam();
    }
}
```

**扩展**

当然，使用参数解析器也需要单独配置，我们同样在WebMvcConfigurerAdapter内配置：

```java
@Configuration
public class MvcConfiguration extends WebMvcConfigurerAdapter {

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(new AuthParamResolver());
    }
}
```

这次实现完了，我还有些不放心，于是在网上查找是否还有其他方式可以实现此功能，发现常见的还有 Filter。

#### Filter

Filter 并不是 Spring 提供的，它是在 Servlet 规范中定义的，是 Servlet 容器支持的。被 Filter 过滤的请求，不会派发到 Spring 容器中。它的实现也比较简单，实现 javax.servlet.Filter接口即可。由于不在 Spring 容器中，Filter 获取不到 Spring 容器的资源，只能使用原生 Java 的 ServletRequest 和 ServletResponse 来获取请求参数。另外，在一个 Filter 中要显示调用 FilterChain 的 doFilter 方法，不然认为请求被拦截。实现类似：

```java
public class WhitelistFilter implements javax.servlet.Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
  // 初始化后被调用一次
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
     // 判断是否需要拦截
       chain.doFilter(request, response); // 请求通过要显示调用
    }

    @Override
    public void destroy() {
     // 被销毁时调用一次
    }
}
```

**扩展**

Filter 也需要显示配置：

```java
@Configuration
public class FilterConfiguration {

    @Bean
    public FilterRegistrationBean someFilterRegistration() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new WhitelistFilter());
        registration.addUrlPatterns("/*");
        registration.setName("whitelistFilter");
        registration.setOrder(1); // 设置过滤器被调用的顺序
        return registration;
    }
}
```

#### 小结

四种实现方式都有其适合的场 景，那么它们之间的调用顺序如何呢？Filter 是 Servlet 实现的，自然是最先被调用，后续被调用的是 Interceptor 被拦截了自然不需要后续再进行处理，然后是 参数解析器，最后才是切面的切点。

### 类型转换神器Mapstruct新出的Spring插件

Mapstruct这个神器，它可以代替`BeanUtil`来进行**DTO**、**VO**、**PO**之间的转换。它使用的是Java编译期的  annotation processor 机制，说白了它就是一个代码生成器，代替你手工进行类型转换期间的取值赋值操作。

```java
@Mapper(componentModel = "spring")
public interface AreaMapping {

    List<AreaInfoListVO> toVos(List<Area> areas);
}
```

就这么几行就把一个**PO**的集合转换成了对应**VO**的集合。

```java
// spring bean 
@Autowired
AreaMapping areaMapping
    
// 转换源 areas    
List<Area> areas = ……;
// 转换目标 vos 
List<AreaInfoListVO> vos = areaMapping.toVos(areas)
```

#### Converter

**Spring framework**提供了一个`Converter<S,T>`接口：

```java
@FunctionalInterface
public interface Converter<S, T> {
    @Nullable
    T convert(S source);

    default <U> Converter<S, U> andThen(Converter<? super T, ? extends U> after) {
        Assert.notNull(after, "After Converter must not be null");
        return (s) -> {
            T initialResult = this.convert(s);
            return initialResult != null ? after.convert(initialResult) : null;
        };
    }
}
```

它的作用是将`S`转换为`T`，这和**Mapstruct**的作用不谋而合。

`Converter`会通过`ConverterRegistry`这个注册接口注册到`ConversionService`，然后你就可以通过`ConversionService`的`convert`方法来进行转换：

```java
 <T> T convert(@Nullable Object source, Class<T> targetType);
```

#### MapStruct Spring Extensions

根据上面的机制官方推出了**MapStruct Spring Extensions**插件， 它实现了一种机制，所有的**Mapstruct**映射接口(**Mapper**)只要实现了`Converter`，都会自动注册到`ConversionService`，我们只需要通过`ConversionService`就能完成任何转换操作。

```java
/**
 * @since 1.0.0
 */
@Mapper(componentModel = "spring")
public interface CarMapper extends Converter<Car, CarDto> {

    @Mapping(target = "seats", source = "seatConfiguration")
    CarDto convert(Car car);
}
```

调用时：

```java
@Autowired
private ConversionService conversionService;

Car car = ……;
CarDto carDto = conversionService.convert(car,CarDto.class);
```

**MapStruct Spring Extensions** 会自动生成一个适配类处理**Mapper**注册：

```java
package org.mapstruct.extensions.spring.converter;

import cn.felord.mapstruct.entity.Car;
import cn.felord.mapstruct.entity.CarDto;
import org.springframework.context.annotation.Lazy;
import org.springframework.core.convert.ConversionService;
import org.springframework.stereotype.Component;
/**
 * @since 1.0.0
 */
@Component
public class ConversionServiceAdapter {
    private final ConversionService conversionService;

    public ConversionServiceAdapter(@Lazy final ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    public CarDto mapCarToCarDto(final Car source) {
        return (CarDto)this.conversionService.convert(source, CarDto.class);
    }
}
```

#### 自定义

##### 自定义适配类的包路径和名称

默认情况下，生成的适配类将位于包`org.mapstruct.extensions.spring.converter`中，名称固定为`ConversionServiceAdapter`。如果你希望修改包路径或者名称，你可以这样：

```java
import org.mapstruct.MapperConfig;
import org.mapstruct.extensions.spring.SpringMapperConfig;

/**
 * @since 1.0.0
 */
@MapperConfig(componentModel = "spring")
@SpringMapperConfig(conversionServiceAdapterPackage = "cn.xx.mapstruct.config",
        conversionServiceAdapterClassName = "MapStructConversionServiceAdapter")
public class MapperSpringConfig {
}
```

> ❝
>
> 不指定`conversionServiceAdapterPackage`元素，生成的 Adapter 类将与注解的 Config 驻留在同一个包中，所以上面的路径是可以省略的。

##### 指定ConversionService

如果你的**Spring IoC**容器中有多个`ConversionService`，你可以通过`@SpringMapperConfig`注解的`conversionServiceBeanName `参数指定。

```java
import org.mapstruct.MapperConfig;
import org.mapstruct.extensions.spring.SpringMapperConfig;

/**
 * @since 1.0.0
 */
@MapperConfig(componentModel = "spring")
@SpringMapperConfig(conversionServiceAdapterPackage = "cn.xx.mapstruct.config",
        conversionServiceAdapterClassName = "MapStructConversionServiceAdapter",
                   conversionServiceBeanName = "myConversionService")
public class MapperSpringConfig {
}
```

##### 集成Spring的内置转换

**Spring**内部提供了很多好用的`Converter<S,T>`实现，有的并不直接开放，如果你想用**Mapstruct**的机制使用它们，可以通过`@SpringMapperConfig`注解的 `externalConversions`注册它们。

```java
@MapperConfig(componentModel = "spring")
@SpringMapperConfig(
   externalConversions = @ExternalConversion(sourceType = String.class, targetType = Locale.class))
public interface MapstructConfig {}
```

会在适配器中自动生成相应的转换：

```java
@Component
public class ConversionServiceAdapter {
  private final ConversionService conversionService;

  public ConversionServiceAdapter(@Lazy final ConversionService conversionService) {
    this.conversionService = conversionService;
  }

  public Locale mapStringToLocale(final String source) {
    return conversionService.convert(source, Locale.class);
  }
}
```

#### 总结

**mapstruct-spring-annotations** 使开发人员能够通过`ConversionService`使用定义的 **Mapstruct** 映射器，而不必单独导入每个 **Mapper**，从而允许 **Mapper** 之间的松散耦合。，它本身不会影响**Mapstruct**的机制。相关的**DEMO**可以通过公众号回复 mapstructspring 获取。

### 如何写复杂业务的代码

- 一个复杂业务的处理过程]

- - 业务背景
  - 过程分解
  - 过程分解后的两个问题
  - 过程分解+对象模型

- 写复杂业务的方法论

- - 上下结合
  - 能力下沉

- 业务技术要怎么做

------

看零售通商品域的代码。面对零售通如此复杂的业务场景，如何在架构和代码层面进行应对，是一个新课题。针对该命题，我进行了比较细致的思考和研究。结合实际的业务场景，我沉淀了一套“如何写复杂业务代码”的方法论，在此分享给大家。

#### 一个复杂业务的处理过程

##### 业务背景

简单的介绍下业务背景，零售通是给线下小店供货的B2B模式，我们希望通过数字化重构传统供应链渠道，提升供应链效率，为新零售助力。阿里在中间是一个平台角色，提供的是Bsbc中的service的功能。[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLOjQtS5yXSS3U2icye7oibHybyKmu3MeXy8DCbhWiagTia6cQFSFGIKJVJaA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

在商品域，运营会操作一个“上架”动作，上架之后，商品就能在零售通上面对小店进行销售了。**是零售通业务非常关键的业务操作之一，因此涉及很多的数据校验和关联操作** 。

针对上架，一个简化的业务流程如下所示：[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLO3qEGKicTSXVQIWzurMl33ubLZCqVoaIvullnBHqbSfC3qWWiapiccoW8w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

##### 过程分解

像这么复杂的业务，我想应该没有人会写在一个service方法中吧。一个类解决不了，那就分治吧。

说实话，能想到分而治之的工程师，已经做的不错了，至少比没有分治思维要好很多。我也见过复杂程度相当的业务，连分解都没有，就是一堆方法和类的堆砌。

不过，这里存在一个问题：即很多同学过度的依赖工具或是辅助手段来实现分解。比如在我们的商品域中，类似的分解手段至少有3套以上，有自制的流程引擎，有依赖于数据库配置的流程处理：[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLOaupPzjzybBDvphhBEhcHLJ41w3sTZpElPoXxzbkMeVic3XSpKcFicf5A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

本质上来讲，这些辅助手段做的都是一个pipeline的处理流程，没有其它。因此，我建议此处最好保持KISS（Keep It Simple and Stupid），即**最好是什么工具都不要用，次之是用一个极简的Pipeline模式，最差是使用像流程引擎这样的重方法** 。

除非你的应用有极强的流程可视化和编排的诉求，否则我非常不推荐使用流程引擎等工具。第一，它会引入额外的复杂度，特别是那些需要持久化状态的流程引擎；第二，它会割裂代码，导致阅读代码的不顺畅。**大胆断言一下，全天下估计80%对流程引擎的使用都是得不偿失的** 。

回到商品上架的问题，这里问题核心是工具吗？是设计模式带来的代码灵活性吗？显然不是，**问题的核心应该是如何分解问题和抽象问题** ，知道金字塔原理的应该知道，此处，我们可以使用结构化分解将问题解构成一个有层级的金字塔结构：[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLO2CuR7G8Rj0WM1iaLhSZGSroOkic00aWicGZLjjq9kaHVUeRtyPb9KnZ7w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

按照这种分解写的代码，就像一本书，目录和内容清晰明了。

以商品上架为例，程序的入口是一个上架命令（OnSaleCommand）, 它由三个阶段（Phase）组成。

```java
@Command
public class OnSaleNormalItemCmdExe {

    @Resource
    private OnSaleContextInitPhase onSaleContextInitPhase;
    @Resource
    private OnSaleDataCheckPhase onSaleDataCheckPhase;
    @Resource
    private OnSaleProcessPhase onSaleProcessPhase;

    @Override
    public Response execute(OnSaleNormalItemCmd cmd) {

        OnSaleContext onSaleContext = init(cmd);

        checkData(onSaleContext);

        process(onSaleContext);

        return Response.buildSuccess();
    }

    private OnSaleContext init(OnSaleNormalItemCmd cmd) {
        return onSaleContextInitPhase.init(cmd);
    }

    private void checkData(OnSaleContext onSaleContext) {
        onSaleDataCheckPhase.check(onSaleContext);
    }

    private void process(OnSaleContext onSaleContext) {
        onSaleProcessPhase.process(onSaleContext);
    }
}
```

每个Phase又可以拆解成多个步骤（Step），以 `OnSaleProcessPhase`为例，它是由一系列Step组成的：

```java
@Phase
public class OnSaleProcessPhase {

    @Resource
    private PublishOfferStep publishOfferStep;
    @Resource
    private BackOfferBindStep backOfferBindStep;
    //省略其它step

    public void process(OnSaleContext onSaleContext){
        SupplierItem supplierItem = onSaleContext.getSupplierItem();

        // 生成OfferGroupNo
        generateOfferGroupNo(supplierItem);

        // 发布商品
        publishOffer(supplierItem);

        // 前后端库存绑定 backoffer域
        bindBackOfferStock(supplierItem);

        // 同步库存路由 backoffer域
        syncStockRoute(supplierItem);

        // 设置虚拟商品拓展字段
        setVirtualProductExtension(supplierItem);

        // 发货保障打标 offer域
        markSendProtection(supplierItem);

        // 记录变更内容ChangeDetail
        recordChangeDetail(supplierItem);

        // 同步供货价到BackOffer
        syncSupplyPriceToBackOffer(supplierItem);

        // 如果是组合商品打标，写扩展信息
        setCombineProductExtension(supplierItem);

        // 去售罄标
        removeSellOutTag(offerId);

        // 发送领域事件
        fireDomainEvent(supplierItem);

        // 关闭关联的待办事项
        closeIssues(supplierItem);
    }
}
```

看到了吗，这就是商品上架这个复杂业务的业务流程。需要流程引擎吗？不需要，需要设计模式支撑吗？也不需要。对于这种业务流程的表达，简单朴素的组合方法模式（Composed Method）是再合适不过的了。

因此，在做过程分解的时候，我建议工程师不要把太多精力放在工具上，放在设计模式带来的灵活性上。而是应该多花时间在对问题分析，结构化分解，最后通过合理的抽象，形成合适的阶段（Phase）和步骤（Step）上。z[![图片](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLOjUGpEU32stKyLeDm4Fqp6eVw9G6KlHibp5iaNWeUa1ibtJec2Fg74aZmw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

##### 过程分解后的两个问题

的确，使用过程分解之后的代码，已经比以前的代码更清晰、更容易维护了。不过，还有两个问题值得我们去关注一下：

**1、领域知识被割裂肢解**]

什么叫被肢解？因为我们到目前为止做的都是过程化拆解，导致没有一个聚合领域知识的地方。每个Use Case的代码只关心自己的处理流程，知识没有沉淀。

相同的业务逻辑会在多个Use Case中被重复实现，导致代码重复度高，即使有复用，最多也就是抽取一个util，代码对业务语义的表达能力很弱，从而影响代码的可读性和可理解性。

**2、代码的业务表达能力缺失**

试想下，在过程式的代码中，所做的事情无外乎就是取数据--做计算--存数据，在这种情况下，要如何通过代码显性化的表达我们的业务呢？说实话，很难做到，因为我们缺失了模型，以及模型之间的关系。脱离模型的业务表达，是缺少韵律和灵魂的。

举个例子，在上架过程中，有一个校验是检查库存的，其中对于组合品（CombineBackOffer）其库存的处理会和普通品不一样。原来的代码是这么写的：

```java
Boolean isCombineProduct = supplierItem.getSign().isCombProductQuote();
// supplier.usc warehouse needn't check
if (WarehouseTypeEnum.isAliWarehouse(supplierItem.getWarehouseType())) {
    // quote warehosue check
    if (CollectionUtil.isEmpty(supplierItem.getWarehouseIdList()) && !isCombineProduct) {
        throw ExceptionFactory.makeFault(ServiceExceptionCode.SYSTEM_ERROR, "亲，不能发布Offer，请联系仓配运营人员，建立品仓关系！");
    }
    // inventory amount check
    long sellableAmount = 0L;
    if (!isCombineProduct) {
        sellableAmount = normalBiz.acquireSellableAmount(supplierItem.getBackOfferId(), supplierItem.getWarehouseIdList());
    } else {
        //组套商品
        OfferModel backOffer = backOfferQueryService.getBackOffer(supplierItem.getBackOfferId());
        if (backOffer != null) {
            sellableAmount = backOffer.getOffer().getTradeModel().getTradeCondition().getAmountOnSale();
        }
    }
    if (sellableAmount < 1) {
        throw ExceptionFactory.makeFault(ServiceExceptionCode.SYSTEM_ERROR, "亲，实仓库存必须大于0才能发布，请确认已补货.r[id:" + supplierItem.getId() + "]");
    }
}
```

然而，如果我们在系统中引入领域模型之后，其代码会简化为如下：

```java
if(backOffer.isCloudWarehouse()){   
    return;
}
if (backOffer.isNonInWarehouse()){  
    throw new BizException("亲，不能发布Offer，请联系仓配运营人员，建立品仓关系！");
}
if (backOffer.getStockAmount() < 1){    
    throw new BizException("亲，实仓库存必须大于0才能发布，请确认已补货.\r[id:" + backOffer.getSupplierItem().getCspuCode() + "]");
}
```

有没有发现，使用模型的表达要清晰易懂很多，而且也不需要做关于组合品的判断了，因为我们在系统中引入了更加贴近现实的对象模型（CombineBackOffer继承BackOffer），通过对象的多态可以消除我们代码中的大部分的if-else。[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLOLuic2DCjUibG3ev0x6iaD382VMCibR2IL2mRPAu3ptZGgxAxak2onrjr5Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

##### 过程分解+对象模型

通过上面的案例，我们可以看到**有过程分解要好于没有分解** ，**过程分解+对象模型要好于仅仅是过程分解** 。对于商品上架这个case，如果采用过程分解+对象模型的方式，最终我们会得到一个如下的系统结构：[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLOtIcPhhEM9bO5eBblicC8XctAKNNZIxJMlX13Qp9P09M6icaHzOCtbrLQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

#### 写复杂业务的方法论

通过上面案例的讲解，我想说，我已经交代了复杂业务代码要怎么写：**即自上而下的结构化分解+自下而上的面向对象分析** 。

接下来，让我们把上面的案例进行进一步的提炼，形成一个可落地的方法论，从而可以泛化到更多的复杂业务场景。

##### 上下结合

所谓上下结合，是指我们要**结合自上而下的过程分解和自下而上的对象建模** ，螺旋式的构建我们的应用系统。这是一个动态的过程，两个步骤可以交替进行、也可以同时进行。

这两个步骤是相辅相成的，**上面的分析可以帮助我们更好的理清模型之间的关系，而下面的模型表达可以提升我们代码的复用度和业务语义表达能力** 。

其过程如下图所示：[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLOFrDq1xQNibzRmQY4f175C4lN4fg1KQbYiaJcZjWMFnPEVucmzPribLm4w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

使用这种上下结合的方式，我们就有可能在面对任何复杂的业务场景，都能写出干净整洁、易维护的代码。

##### 能力下沉

**1. 套概念阶段**

了解了一些DDD的概念，然后在代码中“使用”Aggregation Root，Bonded Context，Repository等等这些概念。跟进一步，也会使用一定的分层策略。然而这种做法一般对复杂度的治理并没有多大作用。

**2. 融会贯通阶段**

术语已经不再重要，理解DDD的本质是统一语言、边界划分和面向对象分析的方法。

大体上而言，我大概是在1.7的阶段，因为有一个问题一直在困扰我，就是哪些能力应该放在Domain层，是不是按照传统的做法，将所有的业务都收拢到Domain上，这样做合理吗？说实话，这个问题我一直没有想清楚。

因为在现实业务中，很多的功能都是用例特有的（Use case specific）的，如果“盲目”的使用Domain收拢业务并不见得能带来多大的益处。相反，这种收拢会导致Domain层的膨胀过厚，不够纯粹，反而会影响复用性和表达能力。

鉴于此，我最近的思考是我们应该采用**能力下沉** 的策略。

所谓的能力下沉，是指我们不强求一次就能设计出Domain的能力，也不需要强制让把所有的业务功能都放到Domain层，而是采用实用主义的态度，即只对那些需要在多个场景中需要被复用的能力进行抽象下沉，而不需要复用的，就暂时放在App层的Use Case里就好了。

注：Use Case是《架构整洁之道》里面的术语，简单理解就是响应一个Request的处理过程

通过实践，**我发现这种循序渐进的能力下沉策略，应该是一种更符合实际、更敏捷的方法。因为我们承认模型不是一次性设计出来的，而是迭代演化出来的。**

下沉的过程如下图所示，假设两个use case中，我们发现uc1的step3和uc2的step1有类似的功能，我们就可以考虑让其下沉到Domain层，从而增加代码的复用性。[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLOv6W07Vgg1RsNK6lAXDU4n2COf3UhwOqoMRbL0Iia4C5hM6N9tZX9RKg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

**指导下沉有两个关键指标：代码的复用性和内聚性** 。

复用性是告诉我们When（什么时候该下沉了），即有重复代码的时候。内聚性是告诉我们How（要下沉到哪里），功能有没有内聚到恰当的实体上，有没有放到合适的层次上（因为Domain层的能力也是有两个层次的，一个是Domain Service这是相对比较粗的粒度，另一个是Domain的Model这个是最细粒度的复用）。

比如，在我们的商品域，经常需要判断一个商品是不是最小单位，是不是中包商品。像这种能力就非常有必要直接挂载在Model上。

```java
public class CSPU {
    private String code;
    private String baseCode;
    //省略其它属性

    /**
     * 单品是否为最小单位。
     *
     */
    public boolean isMinimumUnit(){
        return StringUtils.equals(code, baseCode);
    }

    /**
     * 针对中包的特殊处理
     *
     */
    public boolean isMidPackage(){
        return StringUtils.equals(code, midPackageCode);
    }
}
```

之前，因为老系统中没有领域模型，没有CSPU这个实体。你会发现像判断单品是否为最小单位的逻辑是以 `StringUtils.equals(code,baseCode)`的形式散落在代码的各个角落。这种代码的可理解性是可想而知的，至少在第一眼看到的时候，是完全不知道什么意思。

#### 业务技术要怎么做

写到这里，我想顺便回答一下很多业务技术同学的困惑，也是我之前的困惑：**即业务技术到底是在做业务，还是做技术？业务技术的技术性体现在哪里？**

通过上面的案例，我们可以看到业务所面临的复杂性并不亚于底层技术，要想写好业务代码也不是一件容易的事情。业务技术和底层技术人员唯一的区别是他们所面临的问题域不一样。

业务技术面对的问题域变化更多、面对的人更加庞杂。而底层技术面对的问题域更加稳定、但对技术的要求更加深。比如，如果你需要去开发Pandora，你就要对Classloader有更加深入的了解才行。

但是，不管是业务技术还是底层技术人员，有一些思维和能力都是共通的。比如，**分解问题的能力，抽象思维，结构化思维** 等等。[![图片](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcItBCG6BUDw22XxJmTNhLOYvzPKrJbckQ7jEXAG9zhYBLibfcqck3RibicEzLSdIFy3fX35iad7Fzkbw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=551460673&lang=zh_CN#wechat_redirect)

用我的话说就是：**“做不好业务开发的，也做不好技术底层开发，反之亦然。业务开发一点都不简单，只是我们很多人把它做“简单”了**

因此，如果从变化的角度来看，业务技术的难度一点不逊色于底层技术，其面临的挑战甚至更大。因此，我想对广大的从事业务技术开发说：**沉下心来，夯实自己的基础技术能力、OO能力、建模能力... 不断提升抽象思维、结构化思维、思辨思维... 持续学习精进，写好代码。我们可以在业务技术岗做的很”技术“！**

### 并发编程的11种业务场景

#### 前言

并发编程是一项非常重要的技术，无论在面试，还是工作中出现的频率非常高。

并发编程说白了就是多线程编程，但多线程一定比单线程效率更高？

答：不一定，要看具体业务场景。

毕竟如果使用了多线程，那么线程之间的竞争和抢占 CPU 资源，线程的上下文切换，也是相对来说比较耗时的操作。

下面这几个问题在面试中，你必定遇到过：

- 你在哪来业务场景中使用过多线程？
- 怎么用的？
- 踩过哪些坑？

今天聊聊我之前在项目中用并发编程的 12 种业务场景，给有需要的朋友一个参考。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5jOIBX6rOGfeJG9lIWN024tDV5fhA7w3HcZmAPpGhKX9dkficb7B5RbRWqE1bicR9vmLkCFA3gNju4w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

#### 1. 简单定时任务

各位亲爱的朋友，你没看错，Thread 类真的能做定时任务。如果你看过一些定时任务框架的源码，你最后会发现，它们的底层也会使用 Thread 类。

实现这种定时任务的具体代码如下：

```java
  public static void init() {
        new Thread(() -> {
            while (true) {
                try {
                    System.out.println("下载文件");
                    Thread.sleep(1000 * 60 * 5);
                } catch (Exception e) {
                    log.error(e);
                }
            }
        }).start();
    }
```

使用 Thread 类可以做最简单的定时任务，在 run 方法中有个 while 的死循环（当然还有其他方式），执行我们自己的任务。有个需要特别注意的地方是，需要用 try...catch 捕获异常。否则，如果出现异常，就直接退出循环，下次将无法继续执行了。

但这种方式做的定时任务，只能周期性执行，不能支持定时在某个时间点执行。

特别提醒一下，该线程建议定义成守护线程，可以通过 setDaemon 方法设置，让它在后台默默执行就好。

使用场景：比如项目中有时需要每隔 5 分钟去下载某个文件，或者每隔 10 分钟去读取模板文件生成静态 HTML 页面等等，一些简单的周期性任务场景。

使用 Thread 类做定时任务的优缺点：

- **优点**：这种定时任务非常简单，学习成本低，容易入手，对于那些简单的周期性任务，是个不错的选择；
- **缺点**：不支持指定某个时间点执行任务，不支持延迟执行等操作，功能过于单一，无法应对一些较为复杂的场景。

#### 2. 监听器

有时候，我们需要写个监听器，去监听某些数据的变化。

比如：我们在使用 Canal 的时候，需要监听 binlog 的变化，能够及时把数据库中的数据，同步到另外一个业务数据库中。



![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5jOIBX6rOGfeJG9lIWN024tedzurrHYLUvjC6XzZRPEaukSLZt1eTVVCaAohdCkFZ2PtehKHVr1ZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

如果直接写一个监听器去监听数据就太没意思了，我们想实现这样一个功能：在配置中心有个开关，配置监听器是否开启，如果开启了使用单线程异步执行。

主要代码如下：

```java
    @Service 
    public CanalService {
        private volatile boolean running = false;
        private Thread thread;
        @Autowired private CanalConnector canalConnector;
        public void handle () {        //连接canal
            while (running) {           //业务处理
            }
        }
        public void start () {
            thread = new Thread(this::handle, "name");
            running = true;
            thread.start();
        }
        public void stop () {
            if (!running) {
                return;
            }
            running = false;
        }
    }
```



在 start 方法中开启了一个线程，在该线程中异步执行 handle 方法的具体任务。然后通过调用 stop 方法，可以停止该线程。

其中，使用 volatile 关键字控制的 running 变量作为开关，它可以控制线程中的状态。

接下来,有个比较关键的点是：如何通过配置中心的配置，控制这个开关呢？

以 Apollo 配置为例，我们在配置中心的后台，修改配置之后，自动获取最新配置的核心代码如下：

```java
 public class CanalConfig {
        @Autowired
        private CanalService canalService;

        @ApolloConfigChangeListener
        public void change(ConfigChangeEvent event) {
            String value = event.getChange("test.canal.enable").getNewValue();
            if (BooleanUtils.toBoolean(value)) {
                canalService.start();
            } else {
                canalService.stop();
            }
        }
    }
```

通过 Apollo 的 ApolloConfigChangeListener 注解，可以监听配置参数的变化。

如果 test.canal.enable 开关配置为 true，则调用 canalService 类的 start 方法开启 Canal 数据同步功能。如果开关配置为 false，则调用 canalService 类的 stop 方法，自动停止 Canal 数据同步功能。

#### 3. 收集日志

在某些高并发的场景中，我们需要收集部分用户的日志（比如：用户登录的日志），写到数据库中，以便于做分析。

但由于项目中，还没有引入消息中间件，比如 Kafka、RocketMQ 等。

如果直接将日志同步写入数据库，可能会影响接口性能。

所以，大家很自然想到了异步处理。

实现这个需求最简单的做法是，开启一个线程，异步写入数据到数据库即可。

这样做，可以是可以。

但如果用户登录操作的耗时，比异步写入数据库的时间要少得多。这样导致的结果是：生产日志的速度，比消费日志的速度要快得多，最终的性能瓶颈在消费端。

其实，还有更优雅的处理方式，虽说没有使用消息中间件，但借用了它的思想。

这套记录登录日志的功能，分为日志生产端、日志存储端和日志消费端。

如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5jOIBX6rOGfeJG9lIWN024t3LIpTBH15jFbic90S8A2ZC2jGvibpMOicOgiaCic9NGI5ibVpBHkzlMdsDTQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)先定义了一个阻塞队列。

```java
   @Component
    public class LoginLogQueue {
        private static final int QUEUE_MAX_SIZE = 1000;
        private BlockingQueueblockingQueue queue = new LinkedBlockingQueue<>(QUEUE_MAX_SIZE);
        //生成消息
        public boolean push(LoginLog loginLog) {
            return this.queue.add(loginLog);
        }

        //消费消息
        public LoginLog poll() {
            LoginLog loginLog = null;
            try {
                loginLog = this.queue.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return result;
        }
    }
```

然后定义了一个日志的生产者。

```java
   @Service
    public class LoginSerivce {
        @Autowired
        private LoginLogQueue loginLogQueue;

        public int login(UserInfo userInfo) {        //业务处理        
            LoginLog loginLog = convert(userInfo);
            loginLogQueue.push(loginLog);
        }
    }
```



接下来，定义了日志的消费者。

```java
 @Service
    public class LoginInfoConsumer {
        @Autowired
        private LoginLogQueue queue;
        
        @PostConstruct
        public voit init {
            new Thread(() -> {
                while (true) {
                    LoginLog loginLog = queue.take();              //写入数据库         
                }
            }).start();
        }
    }
```

当然，这个例子中使用单线程接收登录日志。为了提升性能，也可以使用线程池来处理业务逻辑（比如写入数据库）等。

####  4. Excel 导入

我们可能会经常收到运营同学提过来的 Excel 数据导入需求，比如将某一大类下的所有子类一次性导入系统，或者导入一批新的供应商数据等等。

我们以导入供应商数据为例，它所涉及的业务流程很长，比如：

1. 调用天眼查接口校验企业名称和统一社会信用代码；
2. 写入供应商基本表；
3. 写入组织表；
4. 给供应商自动创建一个用户；
5. 给该用户分配权限；
6. 自定义域名；
7. 发站内通知。

等等。

如果在程序中，解析完 Excel，读取了所有数据之后。用单线程一条条处理业务逻辑，可能耗时会非常长。

为了提升 Excel 数据导入效率，非常有必要使用多线程来处理。

当然在 Java 中实现多线程的手段有很多种。下面重点聊聊 Java 8 中最简单的实现方式 parallelStream。

伪代码如下：

```java
supplierList.parallelStream().forEach(x -> importSupplier(x));
```

parallelStream 是一个并行执行的流，它默认通过 ForkJoinPool 实现的，能提高你的多线程任务的速度。

ForkJoinPool 处理的过程会分而治之，它的核心思想是将一个大任务切分成多个小任务。每个小任务都能单独执行，最后它会把所用任务的执行结果进行汇总。

下面用一张图简单介绍一下 ForkJoinPool 的原理：



![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5jOIBX6rOGfeJG9lIWN024tqH2crKgzL8lGHs7nVJU5Pic4D6DmJVx95ibK6TF1cCUAh5D0uO3RRMOA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



当然除了 Excel 导入之外，还有类似的读取文本文件，也可以用类似的方法处理。

温馨的提醒一下：如果一次性导入的数据非常多，用多线程处理，可能会使系统的 CPU 使用率飙升，需要特别关注。

#### 5. 查询接口

很多时候，我们需要在某个查询接口中，调用其他服务的接口，组合数据之后，一起返回。

比如有这样的业务场景：

在用户信息查询接口中需要返回用户名称、性别、等级、头像、积分、成长值等信息。

而用户名称、性别、等级、头像在用户服务中，积分在积分服务中，成长值在成长值服务中。为了汇总这些数据统一返回，需要另外提供一个对外接口服务。

于是，用户信息查询接口需要调用用户查询接口、积分查询接口 和 成长值查询接口，然后汇总数据统一返回。

调用过程如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5jOIBX6rOGfeJG9lIWN024tiaQAcj9hk2YjEmMiaN2vveBEqe73KHfWWLULzelK0ISn7nFUfL1Cut8w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

调用远程接口总耗时 530ms = 200ms + 150ms + 180ms

显然这种串行调用远程接口性能是非常不好的，调用远程接口总的耗时为所有的远程接口耗时之和。

那么如何优化远程接口性能呢？

既然串行调用多个远程接口性能很差，为什么不改成并行呢？

如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5jOIBX6rOGfeJG9lIWN024tfLS0mahgDtUQvBjXSIXMp2ibvicv60ToqYTkDGUbasOowDiavw38YHcUA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

调用远程接口总耗时 200ms = 200ms（即耗时最长的那次远程接口调用）

在 Java 8 之前可以通过实现 Callable 接口，获取线程返回结果。

Java 8 以后通过 CompleteFuture 类实现该功能。我们这里以 CompleteFuture 为例：

```java
 public UserInfo getUserInfo(Long id) throws InterruptedException, ExecutionException {
        final UserInfo userInfo = new UserInfo();
        CompletableFuture userFuture = CompletableFuture.supplyAsync(() -> {
            getRemoteUserAndFill(id, userInfo);
            return Boolean.TRUE;
        }, executor);
        CompletableFuture bonusFuture = CompletableFuture.supplyAsync(() -> {
            getRemoteBonusAndFill(id, userInfo);
            return Boolean.TRUE;
        }, executor);
        CompletableFuture growthFuture = CompletableFuture.supplyAsync(() -> {
            getRemoteGrowthAndFill(id, userInfo);
            return Boolean.TRUE;
        }, executor);
        CompletableFuture.allOf(userFuture, bonusFuture, growthFuture).join();
        userFuture.get();
        bonusFuture.get();
        growthFuture.get();
        return userInfo;
    }
```

温馨提醒一下，这两种方式别忘了使用线程池。示例中我用到了 executor，表示自定义的线程池，为了防止高并发场景下，出现线程过多的问题。

#### 6. 获取用户上下文

不知道你在项目开发时，有没有遇到过这样的需求：用户登录之后，在所有的请求接口中，通过某个公共方法，就能获取到当前登录用户的信息？

获取的用户上下文，我们以 CurrentUser 为例。

CurrentUser 内部包含了一个 ThreadLocal 对象，它负责保存当前线程的用户上下文信息。当然为了保证在线程池中，也能从用户上下文中获取到正确的用户信息，这里用了阿里的 TransmittableThreadLocal。

伪代码如下：

```java
@Datapublic
    class CurrentUser {
        private static final TransmittableThreadLocal<CurrentUser> THREA_LOCAL = new TransmittableThreadLocal<>();
        private String id;
        private String userName;
        private String password;
        private String phone;    ...
        public statis

        void set(CurrentUser user) {
            THREA_LOCAL.set(user);
        }

        public static void getCurrent() {
            return THREA_LOCAL.get();
        }
    }
```

这里为什么用了阿里的 TransmittableThreadLocal，而不是普通的 ThreadLocal 呢？

在线程池中，由于线程会被多次复用，导致从普通的 ThreadLocal 中无法获取正确的用户信息。父线程中的参数，没法传递给子线程，而 TransmittableThreadLocal 很好解决了这个问题。

然后在项目中定义一个全局的 Spring MVC 拦截器，专门设置用户上下文到 ThreadLocal 中。

伪代码如下：

```java
 public class UserInterceptor extends HandlerInterceptorAdapter {
        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            CurrentUser user = getUser(request);
            if (Objects.nonNull(user)) {
                CurrentUser.set(user);
            }
        }
    }
```

用户在请求我们接口时，会先触发该拦截器，它会根据用户 Cookie 中的 token，调用调用接口获取 Redis 中的用户信息。如果能获取到，说明用户已经登录，则把用户信息设置到 CurrentUser 类的 ThreadLocal 中。

接下来，在 API 服务的下层，即 Business 层的方法中，就能轻松通过 CurrentUser.getCurrent(); 方法获取到想要的用户上下文信息了。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/eZzl4LXykQyJibtuaVia21VmsgpRXKN7zb1TkDSUPaaohDictBtVV0w8r7Yr2yiaiaS4oIWF8Du9uQy256kdacuxqhw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

这套用户体系的想法是很好的。但深入使用后，发现了一个小插曲：

 API 服务和 MQ 消费者服务都引用了 Business 层，Business 层中的方法两个服务都能直接调用。

我们都知道在 API 服务中用户是需要登录的，而 MQ 消费者服务则不需要登录。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5jOIBX6rOGfeJG9lIWN024tLBAwPjwCkfhLiaQXRzaxW0R8reL1LJGzOVEHkk14RTKxZic9B7HRe1GA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)如果 Business 中的某个方法刚开始是给 API 开发的，在方法深处使用了 CurrentUser.getCurrent(); 获取用户上下文。但后来，某位新来的帅哥在 MQ 消费者中也调用了那个方法，并未发觉这个小机关，就会中招，出现找不到用户上下文的问题。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5jOIBX6rOGfeJG9lIWN024tZjoQPCFibsyXaUiboickOGOsHF9kfaQwiaWiaA4v0rDdKibOJYks6WRxo5jg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

所以我当时的第一个想法是：代码没做兼容处理，因为之前这类问题偶尔会发生一次。

想要解决这个问题，其实也很简单。只需先判断一下能否从 CurrentUser 中获取用户信息。如果不能，则取配置的系统用户信息。

伪代码如下：

```java
    @Autowiredprivate
    BusinessConfig businessConfig;
    
    CurrentUser user = CurrentUser.getCurrent();
    
    if(Objects.nonNull(user)){
        entity.setUserId(user.getUserId());
        entity.setUserName(user.getUserName());
    } else {
        entity.setUserId(businessConfig.getDefaultUserId());
        entity.setUserName(businessConfig.getDefaultUserName());
    }
```

这种简单无公害的代码，如果只是在一两个地方加还 OK。

此外，众所周知 SimpleDateFormat 在 Java 8 以前，是用来处理时间的工具类，它是非线程安全的。也就是说，用该方法解析日期会有线程安全问题。

为了避免线程安全问题的出现，我们可以把 SimpleDateFormat 对象定义成局部变量。但如果你一定要把它定义成静态变量，可以使用 ThreadLocal 保存日期，也能解决线程安全问题。

#### 7. 传递参数

之前见过有些同事写代码时，一个非常有趣的用法，即使用 MDC 传递参数。

MDC 是什么？

MDC 是 org.slf4j 包下的一个类，它的全称是 Mapped Diagnostic Context，我们可以认为它是一个线程安全的存放诊断日志的容器。

MDC 的底层是用了 ThreadLocal 来保存数据的。

例如现在有这样一种场景：我们使用 RestTemplate 调用远程接口时，有时需要在 header 中传递信息，比如 traceId，source 等，便于在查询日志时能够串联一次完整的请求链路，快速定位问题。

这种业务场景就能通过 ClientHttpRequestInterceptor 接口实现，具体做法如下：

第一步，定义一个 LogFilter 拦截所有接口请求，在 MDC 中设置 traceId：

```java
  public class LogFilter implements Filter {
        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
        }

        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
            MdcUtil.add(UUID.randomUUID().toString());
            System.out.println("记录请求日志");
            chain.doFilter(request, response);
            System.out.println("记录响应日志");
        }

        @Override
        public void destroy() {
        }
    }
```

第二步，实现 ClientHttpRequestInterceptor 接口，MDC 中获取当前请求的 traceId，然后设置到 Header 中：

```java
public class RestTemplateInterceptor implements ClientHttpRequestInterceptor {
        @Override
        public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
            request.getHeaders().set("traceId", MdcUtil.get());
            return execution.execute(request, body);
        }
    }
```

第三步，定义配置类，配置上面定义的 RestTemplateInterceptor 类：

```java
@Configurationpublic
    class RestTemplateConfiguration {
        @Bean
        public RestTemplate restTemplate() {
            RestTemplate restTemplate = new RestTemplate();
            restTemplate.setInterceptors(Collections.singletonList(restTemplateInterceptor()));
            return restTemplate;
        }

        @Bean
        public RestTemplateInterceptor restTemplateInterceptor() {
            return new RestTemplateInterceptor();
        }
    }
```

其中 MdcUtil 其实是利用 MDC 工具在 ThreadLocal 中存储和获取 traceId。

```java
  public class MdcUtil {
        private static final String TRACE_ID = "TRACE_ID";

        public static String get() {
            return MDC.get(TRACE_ID);
        }

        public static void add(String value) {
            MDC.put(TRACE_ID, value);
        }
    }
```

当然，这个例子中没有演示 MdcUtil 类的 add 方法具体调的地方，我们可以在 filter 中执行接口方法之前，生成 traceId，调用 MdcUtil 类的 add 方法添加到 MDC 中，然后在同一个请求的其他地方就能通过 MdcUtil 类的 get 方法获取到该 traceId。

能使用 MDC 保存 traceId 等参数的根本原因是，用户请求到应用服务器，Tomcat 会从线程池中分配一个线程去处理该请求。

那么该请求的整个过程中，保存到 MDC 的 ThreadLocal 中的参数，也是该线程独享的，所以不会有线程安全问题。

#### 8. 模拟高并发

有时候我们写的接口，在低并发的场景下，一点问题都没有。

但如果一旦出现高并发调用，该接口可能会出现一些意想不到的问题。

为了防止类似的事情发生，一般在项目上线前，我们非常有必要对接口做一下压力测试。

当然，现在已经有比较成熟的压力测试工具，比如：Jmeter、LoadRunner等。

如果你觉得下载压测工具比较麻烦，也可以手写一个简单的模拟并发操作的工具，用 CountDownLatch 就能实现，例如：

```java
   public static void concurrenceTest() {    /**     * 模拟高并发情况代码     */
        final AtomicInteger atomicInteger = new AtomicInteger(0);
        final CountDownLatch countDownLatch = new CountDownLatch(1000); // 相当于计数器，当所有都准备好了，再一起执行，模仿多并发，保证并发量
        final CountDownLatch countDownLatch2 = new CountDownLatch(1000); // 保证所有线程执行完了再打印atomicInteger的值
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        try {
            for (int i = 0; i < 1000; i++) {
                executorService.submit(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            countDownLatch.await(); //一直阻塞当前线程，直到计时器的值为0,保证同时并发
                        } catch (InterruptedException e) {
                            log.error(e.getMessage(), e);
                        }                    //每个线程增加1000次，每次加1
                        for (int j = 0; j < 1000; j++) {
                            atomicInteger.incrementAndGet();
                        }
                        countDownLatch2.countDown();
                    }
                });
                countDownLatch.countDown();
            }
            countDownLatch2.await();// 保证所有线程执行完
            executorService.shutdown();
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
    }
```

#### 9. 处理 MQ 消息

在高并发的场景中，消息积压问题可以说如影随形，真的没办法从根本上解决。表面上看已经解决了，但后面不知道什么时候就会冒出一次。

比如这次：

有天下午，产品过来说：有几个商户投诉过来了，他们说菜品有延迟，快查一下原因。

这次问题出现得有点奇怪。

为什么这么说？

首先这个时间点就有点奇怪，平常出问题，不都是中午或者晚上用餐高峰期吗？怎么这次问题出现在下午？

根据以往积累的经验，我直接看了 Kafka 的 topic 的数据，果然上面消息有积压。但这次每个 partition 都积压了十几万的消息没有消费，比以往加压的消息数量增加了几百倍。这次消息积压得极不寻常。

我赶紧查服务监控看看消费者挂了没，还好没挂。又查服务日志没有发现异常，这时我有点迷茫。碰运气问了问订单组下午发生了什么事情没？他们说下午有个促销活动，跑了一个 Job 批量更新过有些商户的订单信息。

这时我一下子如梦初醒，是他们在 Job 中批量发消息导致的问题。怎么没有通知我们呢？实在太坑了。

虽说知道问题的原因了，倒是眼前积压的这十几万的消息该如何处理呢？

此时，如果直接调大 partition 数量是不行的，历史消息已经存储到 4 个固定的 partition，只有新增的消息才会到新的 partition。我们重点需要处理的是已有的 partition。

直接加服务节点也不行，因为 Kafka 允许同组的多个 partition 被一个 consumer 消费，但不允许一个 partition 被同组的多个 consumer 消费，可能会造成资源浪费。

看来只有用多线程处理了。

为了紧急解决问题，我改成了用线程池处理消息，核心线程和最大线程数都配置成了 50。

大致用法如下：

1) 先定义一个线程池：

```java
 @Configurationpublic
    class ThreadPoolConfig {
        @Value("${thread.pool.corePoolSize:5}")
        private int corePoolSize;
        @Value("${thread.pool.maxPoolSize:10}")
        private int maxPoolSize;
        @Value("${thread.pool.queueCapacity:200}")
        private int queueCapacity;
        @Value("${thread.pool.keepAliveSeconds:30}")
        private int keepAliveSeconds;
        @Value("${thread.pool.threadNamePrefix:ASYNC_}")
        private String threadNamePrefix;

        @Bean("messageExecutor")
        public Executor messageExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(corePoolSize);
            executor.setMaxPoolSize(maxPoolSize);
            executor.setQueueCapacity(queueCapacity);
            executor.setKeepAliveSeconds(keepAliveSeconds);
            executor.setThreadNamePrefix(threadNamePrefix);
            executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
            executor.initialize();
            return executor;
        }
    }

```

2) 再定义一个消息的 consumer：

```java
 @Service
    public class MyConsumerService {
        @Autowired
        private Executor messageExecutor;

        @KafkaListener(id = "test", topics = {"topic-test"})
        public void listen(String message) {
            System.out.println("收到消息：" + message);
            messageExecutor.submit(new MyWork(message);
        }
    }
```



3) 在定义的 Runable 实现类中处理业务逻辑：

```java
 public class MyWork implements Runnable {
        private String message;

        public MyWork(String message) {
            this.message = message;
        }

        @Override
        public void run() {
            System.out.println(message);
        }
    }
```

果然，调整之后消息积压数量确实下降的非常快。大约半小时后，积压的消息就非常顺利的处理完了。

但此时有个更严重的问题出现：我收到了报警邮件，有两个订单系统的节点 宕机了。

#### 10. 统计数量

在多线程的场景中，有时候需要统计数量，比如用多线程导入供应商数据时，统计导入成功的供应商数有多少。

如果这时候用 count++ 统计次数，最终的结果可能会不准。因为 count++ 并非原子操作，如果多个线程同时执行该操作，则统计的次数可能会出现异常。

为了解决这个问题，就需要使用 concurent 的 atomic 包下面的类，比如 AtomicInteger、AtomicLong 等。

```java
   @Servcie
    public class ImportSupplierService {
        private static AtomicInteger count = new AtomicInteger(0);

        public int importSupplier(List<SupplierInfo> supplierList) {
            if (CollectionUtils.isEmpty(supplierList)) {
                return 0;
            }
            supplierList.parallelStream().forEach(x -> {
                try {
                    importSupplier(x);
                    count.addAndGet(1);
                } catch (Exception e) {
                    log.error(e.getMessage(), e);
                }       );
                return count.get();
            }
        }
    }
```

AtomicInteger 的底层说白了使用 自旋锁 + CAS。

```java
 public final int incrementAndGet() {
        for (; ; ) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next)) return next;
        }
    }
```

自旋锁说白了就是一个死循环。而 CAS 是比较和交换的意思。它的实现逻辑是将内存位置处的旧值与预期值进行比较，若相等，则将内存位置处的值替换为新值。若不相等，则不做任何操作。

#### 11. 延迟定时任务

我们经常有延迟处理数据的需求，比如如果用户下单后，超过 30 分钟还未完成支付，则系统自动将该订单取消。

这里需求就可以使用延迟定时任务实现。

ScheduledExecutorService 是 JDK1.5+ 版本引进的定时任务，该类位于 java.util.concurrent 并发包下。

ScheduledExecutorService 是基于多线程的，设计的初衷是为了解决 Timer 单线程执行，多个任务之间会互相影响的问题。

它主要包含四个方法：

- **schedule(Runnable command,long delay,TimeUnit unit)**：带延迟时间的调度，只执行一次。调度之后可通过 Future.get() 阻塞直至任务执行完毕；
- **schedule(Callable****callable,long delay,TimeUnit unit)**：带延迟时间的调度，只执行一次。调度之后可通过 Future.get() 阻塞直至任务执行完毕，并且可以获取执行结果；
- **scheduleAtFixedRate**：表示以固定频率执行的任务。如果当前任务耗时较多，超过定时周期 period，则当前任务结束后会立即执行；
- **scheduleWithFixedDelay**：表示以固定延时执行任务，延时是相对当前任务结束为起点计算开始时间。

现这种定时任务的具体代码如下：

```java
public class ScheduleExecutorTest {
        public static void main(String[] args) {
            ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(5);
            scheduledExecutorService.scheduleAtFixedRate(() -> {
                System.out.println("doSomething");
            }, 1000, 1000, TimeUnit.MILLISECONDS);
        }
    }
```

调用S cheduledExecutorService 类的 scheduleAtFixedRate 方法实现周期性任务，每隔 1 秒钟执行一次，每次延迟 1 秒再执行。

这种定时任务是阿里巴巴开发者规范中用来替代 Timer 类的方案，对于多线程执行周期性任务是个不错的选择。

使用 ScheduledExecutorService 类做延迟定时任务的优缺点：

- **优点**：基于多线程的定时任务，多个任务之间不会相关影响，支持周期性的执行任务，并且带延迟功能。
- **缺点**：不支持一些较复杂的定时规则。

当然，你也可以使用分布式定时任务，比如 xxl-job 或者 elastic-job 等等。

其实，在实际工作中我使用多线程的场景远远不只这 12 种，在这里只是抛砖引玉，介绍了一些我认为比较常见的业务场景。 

### Stream操作总结

#### toMap

```
Map<String, String> collect = objList.stream().collect(Collectors.toMap(
        // StudentScore::stuName, 
        //Function.identity(),   
         x -> {
           return x + "_";
         },
         obj -> {
                    return obj + "_test";
         }
         //, (u, v) -> v,TreeMap::new //可指定Map
 ));
```

#### map中的merge

**使用场景**

   这个使用场景相对来说还是比较多的，比如`分组求和`这类的操作，虽然 stream 中有相关 groupingBy() 方法，但如果你想在循环中做一些其他操作的时候，merge() 还是一个挺不错的选择的。

```java
  private static List<StudentScore> buildATestList() {
        List<StudentScore> studentScoreList = new ArrayList<>();
        StudentScore studentScore1 = new StudentScore() {{
            setStuName("张三");
            setSubject("语文");
            setScore(70);
        }};
        StudentScore studentScore2 = new StudentScore() {{
            setStuName("张三");
            setSubject("数学");
            setScore(80);
        }};
        StudentScore studentScore3 = new StudentScore() {{
            setStuName("张三");
            setSubject("英语");
            setScore(65);
        }};
        StudentScore studentScore4 = new StudentScore() {{
            setStuName("李四");
            setSubject("语文");
            setScore(68);
        }};
        StudentScore studentScore5 = new StudentScore() {{
            setStuName("李四");
            setSubject("数学");
            setScore(70);
        }};
        StudentScore studentScore6 = new StudentScore() {{
            setStuName("李四");
            setSubject("英语");
            setScore(90);
        }};
        StudentScore studentScore7 = new StudentScore() {{
            setStuName("王五");
            setSubject("语文");
            setScore(80);
        }};
        StudentScore studentScore8 = new StudentScore() {{
            setStuName("王五");
            setSubject("数学");
            setScore(85);
        }};
        StudentScore studentScore9 = new StudentScore() {{
            setStuName("王五");
            setSubject("英语");
            setScore(70);
        }};

        studentScoreList.add(studentScore1);
        studentScoreList.add(studentScore2);
        studentScoreList.add(studentScore3);
        studentScoreList.add(studentScore4);
        studentScoreList.add(studentScore5);
        studentScoreList.add(studentScore6);
        studentScoreList.add(studentScore7);
        studentScoreList.add(studentScore8);
        studentScoreList.add(studentScore9);
        return studentScoreList;
    }

```

分组求和：

```java
 buildATestList().forEach(studentScore -> studentScoreMap2.merge(
                studentScore.getStuName(),
                studentScore.getScore(),
                Integer::sum));
        System.out.println(new ObjectMapper().writeValueAsString(studentScoreMap2));
```

stream进行分组求和

```java
 Map<String, Integer> collect = buildATestList().stream()
                .collect(Collectors.groupingBy(StudentScore::getStuName,
                        Collectors.summingInt(StudentScore::getScore)));
        System.out.println(new ObjectMapper().writeValueAsString(collect));
```

按照分数排序分组求和:   参考地址: https://zhuanlan.zhihu.com/p/144228930

```java
 Map<Integer, StudentScore> collect1 = buildATestList().stream()
                .collect(Collectors.toMap(
                        StudentScore::getScore,
                        Function.identity(),
                        BinaryOperator
                                .maxBy(Comparator.comparing(StudentScore::getScore))));
        System.out.println(new ObjectMapper().writeValueAsString(collect1));
```

#### Collectors.groupingBy

  参考地址: https://blog.csdn.net/u014231523/article/details/102535902

#### map中的compute

需要统计一个字符串中各个单词出现的频率，然后从中找出频率最高的单词。 当于put,只不过返回的是新值。

如果不存在，再put：putIfAbsent  computeIfAbsent

```java
HashMap<Character, Integer> result2 = new HashMap<>(32);
        for (int i = 0; i < str.length(); i++) {
            char curChar = str.charAt(i);
            result2.compute(curChar, (k, v) -> {
                if (v == null) {
                    v = 1;
                } else {
                    v += 1;
                }
                return v;
            });
        }
 System.out.println(new ObjectMapper().writeValueAsString(result2));
```

#### Stream API中使用的函数式接口

| 函数式接口         | 参数类型 | 返回类型 | 描述                       |
| ------------------ | -------- | -------- | -------------------------- |
| Supplier           | 无       | T        | 提供一个T类型的值          |
| Consumer           | T        | void     | 处理一个T类型的值          |
| BiConsumer<T, U>   | T, U     | void     | 处理T类型和U类型的值       |
| Predicate          | T        | boolean  | 一个计算Boolean值的函数    |
| ToIntFunction      | T        | int      | 分别计算 int值的函数       |
| ToLongFunction     | T        | long     | 分别计算 long值的函数      |
| ToDoubleFunction   | T        | double   | 分别计算double值的函数     |
| IntFunction        | int      | R        | 参数分别为int类型的函数    |
| LongFunction       | long     | R        | 参数分别为long类型的函数   |
| DoubleFunction     | double   | R        | 参数分别为double类型的函数 |
| Function<T, R>     | T        | R        | 一个参数类型为T的函数      |
| BiFunction<T,U, R> | T, U     | R        | 一个参数类型为T和U的函数   |
| UnaryOperator      | T        | T        | 对类型T进行的一元操作      |
| BinaryOperator     | T, T     | T        | 对类型T进行的二元操作      |

```java
 Supplier<String> supplier = () -> {
            LocalDate now = LocalDate.now();
            if ("2022-06-29".equals(now.toString())) {
                return "ok";
            }
            return "error";
        };
        System.out.println(supplier.get());
        
Consumer xx = (t) -> {
      System.out.println(t + "_");
    };
```

参考地址:  https://yuan625.gitee.io/yn-read-notes/#/docs/JavaSE8forReallyImpatient

基础：

1. 方法引用：  `String::compareToIgnoreCase`
2. 构造器引用：`Button::new`
3. 变量作用域：` final p `
4. 接口默认方法：` default void method(){}`
5. 接口静态方法：`public static void xx(){}`
6. 函数式接口： `Function`
7. lambda表达式：`->`

Stream API:

1. filter   过滤
2. map   执行转换
3. flatMap 扁平化 `.flatMap(Arrays::stream)`
4. limit   指定长度
5. skip(n)   丢弃掉前面的n个元素
6. peek 产生另一个与原始流具有相同元素的流，但是在每次获取一个元素时，都会调用一个函数，这样便于调试 。 `.peek(e -> System.out.println("Debug: " + e)).count();`
7. distinct 去重
8. parallel 并行流
9. Sorted排序：`sorted(Comparator.comparing(String::length).reversed());`
10. groupingBy 分组
11. reduce 归约   `reduce(Integer::sum)、reduce((x, y)->x + y`)
12. optional：  `orElseGet(()->{})/orElseThrow(Exception::new)`
13. join：`.collect(groupingBy(City::getState, mapping(City::getName,joining (","))))`/`.collect(Collectors.joining(","))`

### SpringBoot使用Caffeine本地缓存

##### 目录

一、本地缓存介绍

二、缓存组件 Caffeine 介绍

1. Caffeine 性能
2. Caffeine 配置说明
3. 软引用与弱引用

三、SpringBoot 集成 Caffeine 两种方式

四、SpringBoot 集成 Caffeine 方式一

1. Maven 引入相关依赖
2. 配置缓存配置类
3. 定义测试的实体对象
4. 定义服务接口类和实现类
5. 测试的 Controller 类

五、SpringBoot 集成 Caffeine 方式二

1. Maven 引入相关依赖
2. 配置缓存配置类
3. 定义测试的实体对象
4. 定义服务接口类和实现类
5. 测试的 Controller 类

#### 环境配置： 

- JDK 版本：1.8
- Caffeine 版本：2.8.0
- SpringBoot 版本：2.2.2.RELEASE

##### 参考地址：

> https://www.jianshu.com/p/c72fb0c787fc
> https://www.cnblogs.com/rickiyang/p/11074158.html
> 博文示例项目 Github 地址：https://github.com/my-dlq/blog-example/tree/master/springboot/springboot-caffeine-cache-example

#### 一、本地缓存介绍

缓存在日常开发中启动至关重要的作用，由于是存储在内存中，数据的读取速度是非常快的，能大量减少对数据库的访问，减少数据库的压力。

之前介绍过 Redis 这种 NoSql 作为缓存组件，它能够很好的作为分布式缓存组件提供多个服务间的缓存，但是 Redis 这种还是需要网络开销，增加时耗。本地缓存是直接从本地内存中读取，没有网络开销，例如秒杀系统或者数据量小的缓存等，比远程缓存更合适。

#### 二、缓存组件 Caffeine 介绍

按 Caffeine Github 文档描述，Caffeine 是基于 JAVA 8 的高性能缓存库。并且在 spring5 (springboot 2.x) 后，spring 官方放弃了 Guava，而使用了性能更优秀的 Caffeine 作为默认缓存组件。

##### 1、Caffeine 性能

可以通过下图观测到，在下面缓存组件中 Caffeine 性能是其中最好的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbuc4ksRNP1YoibxDzgNRqfNYVCX4dCqicykaZbxoQEEQcGlMAH3PX6CBB9icxSWkgG0MBsFAobhO0J5pQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### 2、Caffeine 配置说明

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbuc4ksRNP1YoibxDzgNRqfNYVrJfg8abic5Gu7h5IAyNhCicJ1vCTEP5S3NvAMSibxldpdFAp4GHglj1tQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

注意：

- weakValues 和 softValues 不可以同时使用。
- maximumSize 和 maximumWeight 不可以同时使用。
- expireAfterWrite 和 expireAfterAccess 同事存在时，以 expireAfterWrite 为准。

##### 3、软引用与弱引用

软引用：如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。

弱引用：弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存

```java
// 软引用
Caffeine.newBuilder().softValues().build();

// 弱引用
Caffeine.newBuilder().weakKeys().weakValues().build();
```

#### 三、SpringBoot 集成 Caffeine 两种方式

SpringBoot 有俩种使用 Caffeine 作为缓存的方式：

- 方式一：直接引入 Caffeine 依赖，然后使用 Caffeine 方法实现缓存。
- 方式二：引入 Caffeine 和 Spring Cache 依赖，使用 SpringCache 注解方法实现缓存。

下面将介绍下，这俩中集成方式都是如何实现的。

#### 四、SpringBoot 集成 Caffeine 方式一

##### 1、Maven 引入相关依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.2.RELEASE</version>
    </parent>

    <groupId>mydlq.club</groupId>
    <artifactId>springboot-caffeine-cache-example-1</artifactId>
    <version>0.0.1</version>
    <name>springboot-caffeine-cache-example-1</name>
    <description>Demo project for Spring Boot Cache</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

##### 2、配置缓存配置类

```java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.concurrent.TimeUnit;

@Configuration
public class CacheConfig {

    @Bean
    public Cache<String, Object> caffeineCache() {
        return Caffeine.newBuilder()
                // 设置最后一次写入或访问后经过固定时间过期
                .expireAfterWrite(60, TimeUnit.SECONDS)
                // 初始的缓存空间大小
                .initialCapacity(100)
                // 缓存的最大条数
                .maximumSize(1000)
                .build();
    }

}
```

##### 3、定义测试的实体对象

```java
import lombok.Data;
import lombok.ToString;

@Data
@ToString
public class UserInfo {
    private Integer id;
    private String name;
    private String sex;
    private Integer age;
}
```

##### 4、定义服务接口类和实现类

UserInfoService

```java
import mydlq.club.example.entity.UserInfo;

public interface UserInfoService {

    /**
     * 增加用户信息
     *
     * @param userInfo 用户信息
     */
    void addUserInfo(UserInfo userInfo);

    /**
     * 获取用户信息
     *
     * @param id 用户ID
     * @return 用户信息
     */
    UserInfo getByName(Integer id);

    /**
     * 修改用户信息
     *
     * @param userInfo 用户信息
     * @return 用户信息
     */
    UserInfo updateUserInfo(UserInfo userInfo);

    /**
     * 删除用户信息
     *
     * @param id 用户ID
     */
    void deleteById(Integer id);

}
```

UserInfoServiceImpl

```java
import com.github.benmanes.caffeine.cache.Cache;
import lombok.extern.slf4j.Slf4j;
import mydlq.club.example.entity.UserInfo;
import mydlq.club.example.service.UserInfoService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;
import java.util.HashMap;

@Slf4j
@Service
public class UserInfoServiceImpl implements UserInfoService {

    /**
     * 模拟数据库存储数据
     */
    private HashMap<Integer, UserInfo> userInfoMap = new HashMap<>();

    @Autowired
    Cache<String, Object> caffeineCache;

    @Override
    public void addUserInfo(UserInfo userInfo) {
        log.info("create");
        userInfoMap.put(userInfo.getId(), userInfo);
        // 加入缓存
        caffeineCache.put(String.valueOf(userInfo.getId()),userInfo);
    }

    @Override
    public UserInfo getByName(Integer id) {
        // 先从缓存读取
        caffeineCache.getIfPresent(id);
        UserInfo userInfo = (UserInfo) caffeineCache.asMap().get(String.valueOf(id));
        if (userInfo != null){
            return userInfo;
        }
        // 如果缓存中不存在，则从库中查找
        log.info("get");
        userInfo = userInfoMap.get(id);
        // 如果用户信息不为空，则加入缓存
        if (userInfo != null){
            caffeineCache.put(String.valueOf(userInfo.getId()),userInfo);
        }
        return userInfo;
    }

    @Override
    public UserInfo updateUserInfo(UserInfo userInfo) {
        log.info("update");
        if (!userInfoMap.containsKey(userInfo.getId())) {
            return null;
        }
        // 取旧的值
        UserInfo oldUserInfo = userInfoMap.get(userInfo.getId());
        // 替换内容
        if (!StringUtils.isEmpty(oldUserInfo.getAge())) {
            oldUserInfo.setAge(userInfo.getAge());
        }
        if (!StringUtils.isEmpty(oldUserInfo.getName())) {
            oldUserInfo.setName(userInfo.getName());
        }
        if (!StringUtils.isEmpty(oldUserInfo.getSex())) {
            oldUserInfo.setSex(userInfo.getSex());
        }
        // 将新的对象存储，更新旧对象信息
        userInfoMap.put(oldUserInfo.getId(), oldUserInfo);
        // 替换缓存中的值
        caffeineCache.put(String.valueOf(oldUserInfo.getId()),oldUserInfo);
        return oldUserInfo;
    }

    @Override
    public void deleteById(Integer id) {
        log.info("delete");
        userInfoMap.remove(id);
        // 从缓存中删除
        caffeineCache.asMap().remove(String.valueOf(id));
    }

}
```

##### 5、测试的 Controller 类

```java
import mydlq.club.example.entity.UserInfo;
import mydlq.club.example.service.UserInfoService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping
public class UserInfoController {

    @Autowired
    private UserInfoService userInfoService;

    @GetMapping("/userInfo/{id}")
    public Object getUserInfo(@PathVariable Integer id) {
        UserInfo userInfo = userInfoService.getByName(id);
        if (userInfo == null) {
            return "没有该用户";
        }
        return userInfo;
    }

    @PostMapping("/userInfo")
    public Object createUserInfo(@RequestBody UserInfo userInfo) {
        userInfoService.addUserInfo(userInfo);
        return "SUCCESS";
    }

    @PutMapping("/userInfo")
    public Object updateUserInfo(@RequestBody UserInfo userInfo) {
        UserInfo newUserInfo = userInfoService.updateUserInfo(userInfo);
        if (newUserInfo == null){
            return "不存在该用户";
        }
        return newUserInfo;
    }

    @DeleteMapping("/userInfo/{id}")
    public Object deleteUserInfo(@PathVariable Integer id) {
        userInfoService.deleteById(id);
        return "SUCCESS";
    }

}
```

#### 五、SpringBoot 集成 Caffeine 方式二

##### 1、Maven 引入相关依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.2.RELEASE</version>
    </parent>

    <groupId>mydlq.club</groupId>
    <artifactId>springboot-caffeine-cache-example-2</artifactId>
    <version>0.0.1</version>
    <name>springboot-caffeine-cache-example-2</name>
    <description>Demo project for Spring Boot caffeine</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

##### 2、配置缓存配置类

```java
@Configuration
public class CacheConfig {

    /**
     * 配置缓存管理器
     *
     * @return 缓存管理器
     */
    @Bean("caffeineCacheManager")
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
                // 设置最后一次写入或访问后经过固定时间过期
                .expireAfterAccess(60, TimeUnit.SECONDS)
                // 初始的缓存空间大小
                .initialCapacity(100)
                // 缓存的最大条数
                .maximumSize(1000));
        return cacheManager;
    }

}
```

##### 3、定义测试的实体对象

```java
@Data
@ToString
public class UserInfo {
    private Integer id;
    private String name;
    private String sex;
    private Integer age;
}
```

##### 4、定义服务接口类和实现类

服务接口

```java
import mydlq.club.example.entity.UserInfo;

public interface UserInfoService {

    /**
     * 增加用户信息
     *
     * @param userInfo 用户信息
     */
    void addUserInfo(UserInfo userInfo);

    /**
     * 获取用户信息
     *
     * @param id 用户ID
     * @return 用户信息
     */
    UserInfo getByName(Integer id);

    /**
     * 修改用户信息
     *
     * @param userInfo 用户信息
     * @return 用户信息
     */
    UserInfo updateUserInfo(UserInfo userInfo);

    /**
     * 删除用户信息
     *
     * @param id 用户ID
     */
    void deleteById(Integer id);

}
```

服务实现类

```java
import lombok.extern.slf4j.Slf4j;
import mydlq.club.example.entity.UserInfo;
import mydlq.club.example.service.UserInfoService;
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;
import java.util.HashMap;

@Slf4j
@Service
@CacheConfig(cacheNames = "caffeineCacheManager")
public class UserInfoServiceImpl implements UserInfoService {

    /**
     * 模拟数据库存储数据
     */
    private HashMap<Integer, UserInfo> userInfoMap = new HashMap<>();

    @Override
    @CachePut(key = "#userInfo.id")
    public void addUserInfo(UserInfo userInfo) {
        log.info("create");
        userInfoMap.put(userInfo.getId(), userInfo);
    }

    @Override
    @Cacheable(key = "#id")
    public UserInfo getByName(Integer id) {
        log.info("get");
        return userInfoMap.get(id);
    }

    @Override
    @CachePut(key = "#userInfo.id")
    public UserInfo updateUserInfo(UserInfo userInfo) {
        log.info("update");
        if (!userInfoMap.containsKey(userInfo.getId())) {
            return null;
        }
        // 取旧的值
        UserInfo oldUserInfo = userInfoMap.get(userInfo.getId());
        // 替换内容
        if (!StringUtils.isEmpty(oldUserInfo.getAge())) {
            oldUserInfo.setAge(userInfo.getAge());
        }
        if (!StringUtils.isEmpty(oldUserInfo.getName())) {
            oldUserInfo.setName(userInfo.getName());
        }
        if (!StringUtils.isEmpty(oldUserInfo.getSex())) {
            oldUserInfo.setSex(userInfo.getSex());
        }
        // 将新的对象存储，更新旧对象信息
        userInfoMap.put(oldUserInfo.getId(), oldUserInfo);
        // 返回新对象信息
        return oldUserInfo;
    }

    @Override
    @CacheEvict(key = "#id")
    public void deleteById(Integer id) {
        log.info("delete");
        userInfoMap.remove(id);
    }

}
```

##### 5、测试的 Controller 类

```java
import mydlq.club.example.entity.UserInfo;
import mydlq.club.example.service.UserInfoService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping
public class UserInfoController {

    @Autowired
    private UserInfoService userInfoService;

    @GetMapping("/userInfo/{id}")
    public Object getUserInfo(@PathVariable Integer id) {
        UserInfo userInfo = userInfoService.getByName(id);
        if (userInfo == null) {
            return "没有该用户";
        }
        return userInfo;
    }

    @PostMapping("/userInfo")
    public Object createUserInfo(@RequestBody UserInfo userInfo) {
        userInfoService.addUserInfo(userInfo);
        return "SUCCESS";
    }

    @PutMapping("/userInfo")
    public Object updateUserInfo(@RequestBody UserInfo userInfo) {
        UserInfo newUserInfo = userInfoService.updateUserInfo(userInfo);
        if (newUserInfo == null){
            return "不存在该用户";
        }
        return newUserInfo;
    }

    @DeleteMapping("/userInfo/{id}")
    public Object deleteUserInfo(@PathVariable Integer id) {
        userInfoService.deleteById(id);
        return "SUCCESS";
    }

}
```

### 十个最常用的JVM配置参数

1. **-Xms**：初始堆大小。只要启动，就占用的堆大小。

2. **-Xmx**：最大堆大小。java.lang.OutOfMemoryError：Java heap这个错误可以通过配置-Xms和-Xmx参数来设置。

3. **-Xss**：栈大小分配。栈是每个线程私有的区域，通常只有几百K大小，决定了函数调用的深度，而局部变量、参数都分配到栈上。当出现大量局部变量，递归时，会发生栈空间OOM（java.lang.StackOverflowError）之类的错误。

4. **XX:NewSize**：设置新生代大小的绝对值。

5. **-XX:NewRatio**：设置年轻代和年老代的比值。比如设置为3，则新生代：老年代=1:3，新生代占总heap的1/4。

6. **-XX:MaxPermSiz**e：设置持久代大小。java.lang.OutOfMemoryError:PermGenspace这个OOM错误需要合理调大PermSize和MaxPermSize大小。

7. **-XX:SurvivorRatio**：年轻代中Eden区与两个Survivor区的比值。注意，Survivor区有form和to两个。比如设置为8时，那么eden:form:to=8:1:1。

8. **-XX:HeapDumpOnOutOfMemoryError**：发生OOM时转储堆到文件，这是一个非常好的诊断方法。

9. **-XX:HeapDumpPath：**导出堆的转储文件路径。

10. **-XX:OnOutOfMemoryError**：OOM时，执行一个脚本，比如发送邮件报警，重启程序。后面跟着一个脚本的路径。

### 如何设计一个安全的对外接口

最近有个项目需要对外提供一个接口，提供公网域名进行访问，而且接口和交易订单有关，所以安全性很重要；这里整理了一下常用的一些安全措施以及具体如何去实现。

#### 安全措施

个人觉得安全措施大体来看主要在两个方面，一方面就是如何保证数据在传输过程中的安全性，另一个方面是数据已经到达服务器端，服务器端如何识别数据，如何不被攻击；下面具体看看都有哪些安全措施。

##### 1. 数据加密

我们知道数据在传输过程中是很容易被抓包的，如果直接传输比如通过 http 协议，那么用户传输的数据可以被任何人获取；所以必须对数据加密，常见的做法对关键字段加密比如用户密码直接通过 md5 加密；现在主流的做法是使用 https 协议，在 http 和 tcp 之间添加一层加密层 (SSL 层)，这一层负责数据的加密和解密；

##### 2. 数据加签

数据加签就是由发送者产生一段无法伪造的一段数字串，来保证数据在传输过程中不被篡改；你可能会问数据如果已经通过 https 加密了，还有必要进行加签吗？数据在传输过程中经过加密，理论上就算被抓包，也无法对数据进行篡改；但是我们要知道加密的部分其实只是在外网，现在很多服务在内网中都需要经过很多服务跳转，所以这里的加签可以防止内网中数据被篡改；

##### 3. 时间戳机制

数据是很容易被抓包的，但是经过如上的加密，加签处理，就算拿到数据也不能看到真实的数据；但是有不法者不关心真实的数据，而是直接拿到抓取的数据包进行恶意请求；这时候可以使用时间戳机制，在每次请求中加入当前的时间，服务器端会拿到当前时间和消息中的时间相减，看看是否在一个固定的时间范围内比如 5 分钟内；这样恶意请求的数据包是无法更改里面时间的，所以 5 分钟后就视为非法请求了；

##### 4.AppId 机制

大部分网站基本都需要用户名和密码才能登录，并不是谁来能使用我的网站，这其实也是一种安全机制；对应的对外提供的接口其实也需要这么一种机制，并不是谁都可以调用，需要使用接口的用户需要在后台开通 appid，提供给用户相关的密钥；在调用的接口中需要提供 appid + 密钥，服务器端会进行相关的验证；

##### 5. 限流机制

本来就是真实的用户，并且开通了 appid，但是出现频繁调用接口的情况；这种情况需要给相关 appid 限流处理，常用的限流算法有令牌桶和漏桶算法；

##### 6. 黑名单机制

如果此 appid 进行过很多非法操作，或者说专门有一个中黑系统，经过分析之后直接将此 appid 列入黑名单，所有请求直接返回错误码；

##### 7. 数据合法性校验

这个可以说是每个系统都会有的处理机制，只有在数据是合法的情况下才会进行数据处理；每个系统都有自己的验证规则，当然也可能有一些常规性的规则，比如身份证长度和组成，电话号码长度和组成等等；

#### 如何实现

以上大体介绍了一下常用的一些接口安全措施，当然可能还有其他我不知道的方式，希望大家补充，下面看看以上这些方法措施，具体如何实现；

##### 1. 数据加密

现在主流的加密方式有对称加密和非对称加密；

对称加密：对称密钥在加密和解密的过程中使用的密钥是相同的，常见的对称加密算法有 DES，AES；优点是计算速度快，缺点是在数据传送前，发送方和接收方必须商定好秘钥，然后使双方都能保存好秘钥，如果一方的秘钥被泄露，那么加密信息也就不安全了；

非对称加密：服务端会生成一对密钥，私钥存放在服务器端，公钥可以发布给任何人使用；优点就是比起对称加密更加安全，但是加解密的速度比对称加密慢太多了；广泛使用的是 RSA 算法；

两种方式各有优缺点，而 https 的实现方式正好是结合了两种加密方式，整合了双方的优点，在安全和性能方面都比较好；

对称加密和非对称加密代码实现，jdk 提供了相关的工具类可以直接使用，此处不过多介绍；

##### 2. 数据加签

数据签名使用比较多的是 md5 算法，将需要提交的数据通过某种方式组合和一个字符串，然后通过 md5 生成一段加密字符串，这段加密字符串就是数据包的签名，可以看一个简单的例子：

str：参数1={参数1}&参数2={参数2}&......&参数n={参数n}$key={用户密钥}; MD5.encrypt(str);

注意最后的用户密钥，客户端和服务端都有一份，这样会更加安全；

##### 3. 时间戳机制

解密后的数据，经过签名认证后，我们拿到数据包中的客户端时间戳字段，然后用服务器当前时间去减客户端时间，看结果是否在一个区间内，伪代码如下：

```java
long interval=5*60*1000；//超时时间

long clientTime=request.getparameter("clientTime");

long serverTime=System.currentTimeMillis();

if(serverTime-clientTime>interval){

    returnnew Response("超过处理时长")

}
```

##### 4.AppId 机制

生成一个唯一的 AppId 即可，密钥使用字母、数字等特殊字符随机生成即可；生成唯一 AppId 根据实际情况看是否需要全局唯一；但是不管是否全局唯一最好让生成的 Id 有如下属性：

- 趋势递增：这样在保存数据库的时候，使用索引性能更好；
- 信息安全：尽量不要连续的，容易发现规律；
- 关于全局唯一 Id 生成的方式常见的有类 snowflake 方式等；

##### 5. 限流机制

常用的限流算法包括： `令牌桶限流`， `漏桶限流`， `计数器限流`；

1. 令牌桶限流 令牌桶算法的原理是系统以一定速率向桶中放入令牌，填满了就丢弃令牌；请求来时会先从桶中取出令牌，如果能取到令牌，则可以继续完成请求，否则等待或者拒绝服务；令牌桶允许一定程度突发流量，只要有令牌就可以处理，支持一次拿多个令牌；

2. 漏桶限流 漏桶算法的原理是按照固定常量速率流出请求，流入请求速率任意，当请求数超过桶的容量时，新的请求等待或者拒绝服务；可以看出漏桶算法可以强制限制数据的传输速度；

3. 计数器限流 计数器是一种比较简单粗暴的算法，主要用来限制总并发数，比如数据库连接池、线程池、秒杀的并发数；计数器限流只要一定时间内的总请求数超过设定的阀值则进行限流；

具体基于以上算法如何实现，Guava 提供了 RateLimiter 工具类基于基于令牌桶算法：

```java
RateLimiter rateLimiter = RateLimiter.create(5);
```

以上代码表示一秒钟只允许处理五个并发请求，以上方式只能用在单应用的请求限流，不能进行全局限流；这个时候就需要分布式限流，可以基于 redis+lua 来实现；

##### 6. 黑名单机制

如何为什么中黑我们这边不讨论，我们可以给每个用户设置一个状态比如包括：初始化状态，正常状态，中黑状态，关闭状态等等；或者我们直接通过分布式配置中心，直接保存黑名单列表，每次检查是否在列表中即可；

##### 7. 数据合法性校验

合法性校验包括：常规性校验以及业务校验；

常规性校验：包括签名校验，必填校验，长度校验，类型校验，格式校验等；

业务校验：根据实际业务而定，比如订单金额不能小于 0 等；

#### 总结

本文大致列举了几种常见的安全措施机制包括：数据加密、数据加签、时间戳机制、AppId 机制、限流机制、黑名单机制以及数据合法性校验；当然肯定有其他方式，欢迎补充。

https://mp.weixin.qq.com/s/eJ_ConoGHP6az3IKNe6L2g

### 如何在SpringBoot中异步请求和异步调用

#### 一、SpringBoot中异步请求的使用

##### 1、异步请求与同步请求

![图片](https://mmbiz.qpic.cn/mmbiz_png/sG1icpcmhbiaDNlwThtWwev3dFv9N4ufqichzGT5OqoUQHdvKicoLygQgz9J1TOrk2SBU76QIkJpWRJcoFibGIZzvjQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

特点：

可以先释放容器分配给请求的线程与相关资源，减轻系统负担，释放了容器所分配线程的请求，其响应将被延后，可以在耗时处理完成（例如长时间的运算）时再对客户端进行响应。

一句话：增加了服务器对客户端请求的吞吐量（实际生产上我们用的比较少，如果并发请求量很大的情况下，我们会通过nginx把请求负载到集群服务的各个节点上来分摊请求压力，当然还可以通过消息队列来做请求的缓冲）。

##### 2、异步请求的实现

**方式一：Servlet方式实现异步请求**

```java
@RequestMapping(value = "/email/servletReq", method = GET)
    public void servletReq(HttpServletRequest request, HttpServletResponse response) {
        AsyncContext asyncContext = request.startAsync();      
        //设置监听器:可设置其开始、完成、异常、超时等事件的回调处理
        asyncContext.addListener(new AsyncListener() {
            @Override
            public void onTimeout(AsyncEvent event) throws IOException {
                System.out.println("超时了...");              
                //做一些超时后的相关操作...
            }

            @Override
            public void onStartAsync(AsyncEvent event) throws IOException {
                System.out.println("线程开始");
            }

            @Override
            public void onError(AsyncEvent event) throws IOException {
                System.out.println("发生错误：" + event.getThrowable());
            }

            @Override
            public void onComplete(AsyncEvent event) throws IOException {
                System.out.println("执行完成");              
                //这里可以做一些清理资源的操作...
            }
        });      
        //设置超时时间
        asyncContext.setTimeout(20000);
        asyncContext.start(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(10000);
                    System.out.println("内部线程：" + Thread.currentThread().getName());
                    asyncContext.getResponse().setCharacterEncoding("utf-8");
                    asyncContext.getResponse().setContentType("text/html;charset=UTF-8");
                    asyncContext.getResponse().getWriter().println("这是异步的请求返回");
                } catch (Exception e) {
                    System.out.println("异常：" + e);
                }              
                //异步请求完成通知              
                //此时整个请求才完成
                asyncContext.complete();
            }
        });      
        //此时之类 request的线程连接已经释放了
        System.out.println("主线程：" + Thread.currentThread().getName());
    }
```

**方式二：使用很简单，直接返回的参数包裹一层callable即可，可以继承WebMvcConfigurerAdapter类来设置默认线程池和超时处理**

```java
 @Configuration
    public class RequestAsyncPoolConfig extends WebMvcConfigurerAdapter {
        @Resource
        private ThreadPoolTaskExecutor myThreadPoolTaskExecutor;

        @Override
        public void configureAsyncSupport(final AsyncSupportConfigurer configurer) {
            //处理 callable超时
            /configurer.setDefaultTimeout(60 * 1000);
            configurer.setTaskExecutor(myThreadPoolTaskExecutor);
            configurer.registerCallableInterceptors(timeoutCallableProcessingInterceptor());
        }

        @Bean
        public TimeoutCallableProcessingInterceptor timeoutCallableProcessingInterceptor() {
            return new TimeoutCallableProcessingInterceptor();
        }
    }
```

**方式三：和方式二差不多，在Callable外包一层，给WebAsyncTask设置一个超时回调，即可实现超时处理**

```java
@RequestMapping(value = "/email/webAsyncReq", method = GET)
    @ResponseBody
    public WebAsyncTask<String> webAsyncReq() {
        System.out.println("外部线程：" + Thread.currentThread().getName());
        Callable<String> result = () -> {
            System.out.println("内部线程开始：" + Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(4);
            } catch (Exception e) {
                // TODO: handle exception
            }
            logger.info("副线程返回");
            System.out.println("内部线程返回：" + Thread.currentThread().getName());
            return "success";
        };
        
        WebAsyncTask<String> wat = new WebAsyncTask<String>(3000L, result);
        wat.onTimeout(new Callable<String>() {
            @Override
            public String call() throws Exception {
                // TODO Auto-generated method stub
                return "超时";
            }
        }); return wat;
    }
```

**方式四：DeferredResult可以处理一些相对复杂一些的业务逻辑，最主要还是可以在另一个线程里面进行业务处理及返回，即可在两个完全不相干的线程间的通信**。

```java
@RequestMapping(value = "/email/deferredResultReq", method = GET)
    @ResponseBody
    public DeferredResult<String> deferredResultReq() {
        System.out.println("外部线程：" + Thread.currentThread().getName());
        //设置超时时间
        DeferredResult<String> result = new DeferredResult<String>(60 * 1000L);
        //处理超时事件 采用委托机制
        result.onTimeout(new Runnable() {
            @Override
            public void run() {
                System.out.println("DeferredResult超时");
                result.setResult("超时了!");
            }
        });
        result.onCompletion(new Runnable() {
            @Override
            public void run() {
                //完成后
                System.out.println("调用完成");
            }
        });
        myThreadPoolTaskExecutor.execute(new Runnable() {
            @Override
            public void run() {
                //处理业务逻辑
                System.out.println("内部线程：" + Thread.currentThread().getName());
                //返回结果
                /result.setResult("DeferredResult!!");
            }
        }); return result;
    }
```

#### 二、SpringBoot中异步调用的使用

###### 1、介绍

异步请求的处理。除了异步请求，一般上我们用的比较多的应该是异步调用。通常在开发过程中，会遇到一个方法是和实际业务无关的，没有紧密性的。比如记录日志信息等业务。这个时候正常就是启一个新线程去做一些业务处理，让主线程异步的执行其他业务。

###### 2、使用方式（基于spring下）

需要在启动类加入@EnableAsync使异步调用@Async注解生效

在需要异步执行的方法上加入此注解即可@Async("threadPool"),threadPool为自定义线程池

代码略。。。就俩标签，自己试一把就可以了

###### 3、注意事项

在默认情况下，未设置TaskExecutor时，默认是使用SimpleAsyncTaskExecutor这个线程池，但此线程不是真正意义上的线程池，因为线程不重用，每次调用都会创建一个新的线程。可通过控制台日志输出可以看出，每次输出线程名都是递增的。所以最好我们来自定义一个线程池。

调用的异步方法，不能为同一个类的方法（包括同一个类的内部类），简单来说，因为Spring在启动扫描时会为其创建一个代理类，而同类调用时，还是调用本身的代理类的，所以和平常调用是一样的。

其他的注解如@Cache等也是一样的道理，说白了，就是Spring的代理机制造成的。所以在开发中，最好把异步服务单独抽出一个类来管理。下面会重点讲述。

###### 4、什么情况下会导致@Async异步方法会失效？

- a.调用同一个类下注有@Async异步方法：在spring中像@Async和@Transactional、cache等注解本质使用的是动态代理，其实Spring容器在初始化的时候Spring容器会将含有AOP注解的类对象“替换”为代理对象（简单这么理解），那么注解失效的原因就很明显了，就是因为调用方法的是对象本身而不是代理对象，因为没有经过Spring容器，那么解决方法也会沿着这个思路来解决。
- b.调用的是静态(static )方法
- c.调用(private)私有化方法

###### 5、解决4中问题1的方式（其它2,3两个问题自己注意下就可以了）

将要异步执行的方法单独抽取成一个类，原理就是当你把执行异步的方法单独抽取成一个类的时候，这个类肯定是被Spring管理的，其他Spring组件需要调用的时候肯定会注入进去，这时候实际上注入进去的就是代理类了。

其实我们的注入对象都是从Spring容器中给当前Spring组件进行成员变量的赋值，由于某些类使用了AOP注解，那么实际上在Spring容器中实际存在的是它的代理对象。那么我们就可以通过上下文获取自己的代理对象调用异步方法。

```java
 @Controller
    @RequestMapping("/app")
    public class EmailController {
        //获取ApplicationContext对象方式有多种,这种最简单,其它的大家自行了解一下
        @Autowired
        private ApplicationContext applicationContext;

        @RequestMapping(value = "/email/asyncCall", method = GET)
        @ResponseBody
        public Map<String, Object> asyncCall() {
            Map<String, Object> resMap = new HashMap<String, Object>();
            try {
                //这样调用同类下的异步方法是不起作用的
                this.testAsyncTask();            
                //通过上下文获取自己的代理对象调用异步方法
                EmailController emailController = (EmailController) applicationContext.getBean(EmailController.class);
                emailController.testAsyncTask();
                resMap.put("code", 200);
            } catch (Exception e) {
                resMap.put("code", 400);
                logger.error("error!", e);
            }
            return resMap;
        }

        //注意一定是public,且是非static方法
        @Async
        public void testAsyncTask() throws InterruptedException {
            Thread.sleep(10000);
            System.out.println("异步任务执行完成！");
        }
    }
```

6、开启cglib代理，手动获取Spring代理类,从而调用同类下的异步方法。

首先，在启动类上加上@EnableAspectJAutoProxy(exposeProxy = true)注解。

代码实现，如下：

```java
 @Service
    @Transactional(value = "transactionManager", readOnly = false, propagation = Propagation.REQUIRED, rollbackFor = Throwable.class)
    public class EmailService {
        @Autowired
        private ApplicationContext applicationContext;

        @Async
        public void testSyncTask() throws InterruptedException {
            Thread.sleep(10000);
            System.out.println("异步任务执行完成！");
        }

        public void asyncCallTwo() throws InterruptedException {
           // this.testSyncTask();
           // EmailService emailService = (EmailService) applicationContext.getBean(EmailService.class);
           // emailService.testSyncTask();
            boolean isAop = AopUtils.isAopProxy(EmailController.class);
            //是否是代理对象；
            boolean isCglib = AopUtils.isCglibProxy(EmailController.class);
            //是否是CGLIB方式的代理对象；
            boolean isJdk = AopUtils.isJdkDynamicProxy(EmailController.class);
            //是否是JDK动态代理方式的代理对象；
            // 以下才是重点!!!
            /EmailService emailService = (EmailService) applicationContext.getBean(EmailService.class);
            EmailService proxy = (EmailService) AopContext.currentProxy();
            System.out.println(emailService == proxy ? true : false);
            proxy.testSyncTask();
            System.out.println("end!!!");
        }
    }
```

#### 三、异步请求与异步调用的区别

两者的使用场景不同，异步请求用来解决并发请求对服务器造成的压力，从而提高对请求的吞吐量；而异步调用是用来做一些非主线流程且不需要实时计算和响应的任务，比如同步日志到kafka中做日志分析等。

异步请求是会一直等待response相应的，需要返回结果给客户端的；而异步调用我们往往会马上返回给客户端响应，完成这次整个的请求，至于异步调用的任务后台自己慢慢跑就行，客户端不会关心。
