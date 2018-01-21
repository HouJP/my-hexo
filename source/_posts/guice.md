---
title: Guice Notes
date: 2018-01-18 23:26:22
tags: [guice, java]
---

### Overview

Guice是一个轻量级的依赖注入(DI)框架。它减轻了对`factories`以及`new`的使用，我们可以把Guice视为另一种`new`。在某些时候依然需要写`factories`，但是代码不需要直接依赖它们。使用Guice有以下好处：

1. 易于改变
2. 易于单元测试
3. 易于复用

<!-- more -->

### Motivation

#### Dependency Injection

DI是一种设计模型，核心原则是*separate behaviour from dependency resolution*。比如说，`RealBillingService`不负责查找对`TransactionLog`和`CredictCardProcessor`的依赖，这些依赖通过构造函数被传入。这样：

1. 依赖就在*API signature*这一层面上被暴露出来了。
2. 当依赖增加或者减少时，编译器会告诉我们哪些tests需要被修复。

接下来，依赖`BillingService`的类需要在他们的构造函数中接收`BillingService`。对于最高层的类来说，最好能有一个框架来帮助解决依赖关系，否则就需要递归的在构造函数中传递依赖。这也就是**Guice**存在的意义。

#### Dependency Injection with Guice

DI模式可以让代码模块化和易于测试，而Guice让这一切变得容易实现。使用Guice需要以下几个步骤：

1. 告诉Guice从interfaces到implementations的映射关系。这个配置是在Guide的module中完成的，也就是某个实现`Module`接口的Java类。

   ```java
   public class BillingModule extends AbstractModule {
     @Override
     protected void configure {
       bind(TransactionLog.class).to(DatabaseTransactionLog.class);
       bind(CredictCardProcessor.class).to(PaypalCreditCardProcessor.class);
       bind(BillingService.class).to(RealBillingService.class);
     }
   }
   ```

2. 在`RealBillingService`的构造函数中添加`@Inject`注解。Guice会检查含有注解的构造函数，并为每个参数查找对应值。

   ```java
   public class RealBillingService implements BillingService {
     private final CreditCardProcessor processor;
     private final TransactionLog transactionLog;
     
     @Inject
     public RealBillingService(CredictCardProcessor processor, TransactionLog transactionLog) {
       this.processor = processor;
       this.transactionLog = transactionLog;
     }
   }
   ```

3. 最后，通过`Injector`来获取经过绑定的类的实例。

   ```java
   public static void main(String[] args) {
     Injector injector = Guice.createInjector(new BillingModule());
     BillingService billinService = injector.getInstance(BillingService.class);
     ...
   }
   ```

这样，我们通过Guice构建了一个小的有向无环图，这个图中包含了billing service和它的依赖。

### Bindings

Injector的任务就是将对象组装成一个有向无环图。当请求一个对象的实例时，Injector会去查找和解决所需要的依赖。为了说明依赖的对象如何被创建，我们需要使用bindings来配置Injector。

#### Creating Bindings

为了创建bindings，需要继承`AbstractModule`并重载`configure`方法，并在函数体中调用`bind()`方法来指明binding。创建好modules之后，将它作为参数传递给`Guice.createInjector()`来创建一个injector。

#### Linked Bindings

Linked bindings将一个type映射为它的implementation。如下：

```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
  }
}
```

这样，当我们调用`injector.getInstance(TransactionLog.class)`，或者当injector遇到对`TransactionLog`的依赖时，它将会使用`DatabaseTransactionLog`。将type映射到任何一个它的subtypes（比如一个implementing class或者一个extending class）。你甚至可以将一个具体的`DatabaseTransactionLog`类映射到它的子类上去：

```java
bind(DatabaseTransactionLog.class).to(MySqlDatabaseTransactionLog.class);
```

Linked bindings也可以被串成一条链：

```java
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
    bind(DatabaseTransactionLog.class).to(MySqlDatabaseTransactionLog.class);
  }
}
```

这样，当需要`TransactionLog`时，injector将会提供`MySqlDatabaseTransactionLog`。       

#### Binding Annotations

有时候你想要对某个type实现多个绑定，比如，你希望同时存在一个PayPal credit card processor和一个Google Checkout processor。Guice支持可选的*binding annotation*。*annotation*和*type*共同决定了一个唯一的binding。这个pair称为key。

*binding annotation*可以定义在单独的`.java`文件中，可以在它所注解的type定义中：

```java
package example.pizza;

import com.google.inject.BindingAnnotation;
import java.lang.annotation.Target;
import java.lang.annotation.Retention;
import static java.lang.annotation.RetentionPolicy.RUNTIME;
import static java.lang.annotation.ElementType.PARAMETER;
import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.METHOD;

@BindingAnnotation @Target({ FIELD, PARAMETER, METHOD }) @Retention(RUNTIME)
public @interface PayPal {}
```

* `@BindingAnnotation`告诉Guice这是一个binding annotation。如果一个成员有多个binding annotations，Guice将会报错。
* `@Target({FIELD, PARAMETER, METHOD})`是对使用者的礼貌性声明。它可以阻止`@PayPal`被意外使用在它不希望被使用的地方。
* `@Retention(RUNTIME)`使得annotation在runtime时可被使用。

当被注入参数需要annotated binding时，需要进行申明：

```java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@PayPal CreditCardProcessor processor,
      TransactionLog transactionLog) {
    ...
  }
```

最后，在binding时，使用`bind()`方法中可选的`annotatedWith`从句：

```java
    bind(CreditCardProcessor.class)
        .annotatedWith(PayPal.class)
        .to(PayPalCreditCardProcessor.class);
```

##### @Named

Guice提供了一个build-in binding annotation `@Named`，它需要一个string的输入：

```java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@Named("Checkout") CreditCardProcessor processor,
      TransactionLog transactionLog) {
    ...
  }
```

对应的在`build()`方法中，使用`Names.named()`创建一个实例并传给`annotatedWith`从句：

```java
bind(CreditCardProcessor.class)
        .annotatedWith(Names.named("Checkout"))
        .to(CheckoutCreditCardProcessor.class);
```

但是，compiler不会检查这个string，所以尽量少的使用`@Named`。定义自己的purpose-build annotations更符合type-safety。

#### Instance Bindings

我们可以将type绑定为一个特定的实例。这通常用在内部没有其他依赖的对象中，比如value objects:

```java
    bind(String.class)
        .annotatedWith(Names.named("JDBC URL"))
        .toInstance("jdbc:mysql://localhost/pizza");
    bind(Integer.class)
        .annotatedWith(Names.named("login timeout seconds"))
        .toInstance(10);
```

避免对构建复杂的objects使用`.toInstance`，这将会拖累应用的启动。这种情况应该使用`@Provides`。

#### @Provides Methods

当需要代码来创建一个object的时候，使用`@Provides`方法。它需要在module中定义，并且含有`@Provides`注解。方法的返回类型是被绑定的类型。当injector需要这个类型时，就会调用这个方法：

```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    ...
  }

  @Provides
  TransactionLog provideTransactionLog() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setJdbcUrl("jdbc:mysql://localhost/pizza");
    transactionLog.setThreadPoolSize(30);
    return transactionLog;
  }
}
```

也可以在`@Provides`方法中使用像`@PayPal`或者`@Named(""Checkout)`之类的binding annotation。

```java
 @Provides @PayPal
  CreditCardProcessor providePayPalCreditCardProcessor(
      @Named("PayPal API key") String apiKey) {
    PayPalCreditCardProcessor processor = new PayPalCreditCardProcessor();
    processor.setApiKey(apiKey);
    return processor;
  }
```

##### Throwing Exceptions

Guice不允许Providers中抛出异常。`@Provides`中抛出的异常会被包装成`ProvisionException`。如果一定要抛出异常，请使用`@CheckedProvides`方法。

#### Provider Bindings

当`@Provides`方法变得复杂的时候，可以考虑为其生成一个单独的类。这个provider类实现了Guice的`Provider`接口：

```java
public interface Provider<T> {
  T get();
}
```

以下实现的provider类有它自己的依赖，并通过`@Inject`-annotated constructor来接收：

```java
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  private final Connection connection;

  @Inject
  public DatabaseTransactionLogProvider(Connection connection) {
    this.connection = connection;
  }

  public TransactionLog get() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setConnection(connection);
    return transactionLog;
  }
}
```

在绑定时使用`.toProvider`从句：

```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class)
        .toProvider(DatabaseTransactionLogProvider.class);
  }
```

当providers的逻辑复杂时，需要对其进行测试。

##### Throwing Exceptions

Guice不允许providers抛出异常。RuntimeExceptions可能被包装成`ProvisionException`或者`CreationException`。如果一定要抛出异常，请使用ThrowingProviders extension。

#### Untargeted Bindings

我们可以创建不指定目标的bindings。当具体的类或者类型通过`@ImplementedBy`或者`@ProvidedBy`被注解时，使用这种绑定。

> <font color="red">TODO</font> 什么情况下/为什么用untargeted bindings

Untargeted bindings没有`to`从句：

```java
    bind(MyConcreteClass.class);
    bind(AnotherConcreteClass.class).in(Singleton.class);
```

当要指明binding annotations时，需要在`bind()`方法中添加一个相同的绑定目标：

```java
    bind(MyConcreteClass.class)
        .annotatedWith(Names.named("foo"))
        .to(MyConcreteClass.class);
    bind(AnotherConcreteClass.class)
        .annotatedWith(Names.named("foo"))
        .to(AnotherConcreteClass.class)
        .in(Singleton.class);
```

#### Constructor Bindings

有时候需要绑定一个type到任意一个constructor。`@Inject`注解在这种情况无法使用：1)这个构造函数可能属于一个第三方的类，2)这个类有多个构造函数，DI不知道该使用哪个。`@Provides`方法通过显示调用构造函数可以很好的解决这个问题。但是这种方法有局限性：手动构造实例不参与`AOP`。

> <font color="red">TODO</font> AOP是什么

为了解决这个问题，Guice提供了`toConstructor()`绑定。它要求我们选择需要的构造函数并处理constructor找不到时的异常：

```Java
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {
    try {
      bind(TransactionLog.class).toConstructor(
          DatabaseTransactionLog.class.getConstructor(DatabaseConnection.class));
    } catch (NoSuchMethodException e) {
      addError(e);
    }
  }
}
```

在这个例子中，`DatabaseTransactionLog`必须含有一个接受单个`DatabaseConnection`参数的构造器。这个构造器不需要`@Inject`注解。Guice会调用满足绑定条件的constructor。

每个`toConstructor()`绑定是范围独立的。如果你创建了多个目标构造器相同的singleton绑定，每个绑定会产生它自己的实例。

#### Built-in Bindings

> <font color="red">TODO</font> More bindings we can use

#### Just-In-Time Bindings

当injector需要某个类型的实例时，就需要对应的绑定。Modules中的绑定成为*explicit bindings*，当它们available的时候injector就会使用它们。如果需要某种类型但是explicit bindings中没有时，injector会尝试创建*Just-In-Time binding*，这也被称为*JIT bindings*和*implicit bindings*。

##### Eligible Constructors

Guice可以为具体类型创建bindings，通过使用该类型的*injectable constructor*。这个constructor可以是non-private, no-arguments constructor，也可以是带有`@Inject`注解的constructor。

```Java
public class PayPalCreditCardProcessor implements CreditCardProcessor {
  private final String apiKey;

  @Inject
  public PayPalCreditCardProcessor(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
```

> <font color="red">TODO</font> Guice如何为*nested class*和*inner classes*创建JIT bindings?

##### @ImplementedBy

经过注解的类型告诉injector它们的默认实现类型是什么。`@ImplementedBy`注解类似于*linked binding*，指明了构建类型时使用的子类。

```Java
@ImplementedBy(PayPalCreditCardProcessor.class)
public interface CreditCardProcessor {
  ChargeResult charge(String amount, CreditCard creditCard)
      throws UnreachableException;
}
```

以上例子与下面使用`bind()`的语句等价：

```Java
    bind(CreditCardProcessor.class).to(PayPalCreditCardProcessor.class);
```

如果一个类型同时使用了`bind()`和`@ImplementedBy`注解，最终生效的是`bind()`语句。注解的方式提供了一个*default implement*，它可以被binding覆盖。需要小心使用`@ImplementedBy`，它添加了一个从接口到其实现的compile-time dependency。

##### @ProvidedBy

`ProvidedBy`告诉injector这个`Provider`类来生成实例：

```Java
@ProvidedBy(DatabaseTransactionLogProvider.class)
public interface TransactionLog {
  void logConnectException(UnreachableException e);
  void logChargeResult(ChargeResult result);
}
```

它与`toProvider()`绑定等价：

```Java
    bind(TransactionLog.class)
        .toProvider(DatabaseTransactionLogProvider.class);
```

如果在使用该注解的同时使用了`bind()`语句，最终生效的是`bind()`语句。

### Scopes

在默认情况下，Guice会返回一个新的实例。这个行为可以通过*scopes*进行配置。Scopes允许我们复用生成的实例：

* application lifetime (`@Singleton`)
* session lifetime (`@SessionScoped`)
* request lifetime (`@RequestScoped`)

Guice包含一套servlet extension，它是为web apps定义的scopes。也可以为其他应用自定义scopes (custom scopes)。

####Applying Scopes

Guices使用annotations来定义scopes。通过对实现类使用scope annotation来指定类型的scope。

`@Singleton`要求类是线程安全的：

```Java
@Singleton
public class InMemoryTransactionLog implements TransactionLog {
  /* everything here should be threadsafe! */
}
```

Scopes可以在`bind`语句中配置：

```Java
  @Provides @Singleton
  TransactionLog provideTransactionLog() {
    ...
  }

```

如果类型中的scope和`bind()`语句中的scope冲突，则`bind()`语句中的scope会被使用。如果type中注解了一个你不希望使用的scope，则可以通过在`bind`绑定`Scopes.NO_SCOPE`来覆盖它。

在*linked bindings*中，scopes在binding source上生效，而不是binding target。举例，加入我们有一个类`Applebees`实现了`Bar`和`Grill`接口。则以下的绑定方法会允许该类型的两个实例，一个给`Bar`s，另一个给`Grill`s：

```Java
  bind(Bar.class).to(Applebees.class).in(Singleton.class);
  bind(Grill.class).to(Applebees.class).in(Singleton.class);
```

如果想只生成一个实例，则需要为这个类添加`@Singleton`注解，或者添加如下的绑定：

```Java
  bind(Applebees.class).in(Singleton.class);
```

`in()`从句接收两种参数：

1. a scoping annotation，如`RequestScoped.class`

2. `Scope` instances 比如`ServletScopes.REQUEST`：

   ```Java
     bind(UserPreferences.class)
         .toProvider(UserPreferencesProvider.class)
         .in(ServletScopes.REQUEST);
   ```

### Injections

#### Injections

**The dependency injectoin pattern separated behaviour from dependency resolution.** 

这种模式要求依赖是传入的，而不是在类中查找依赖或者通过工厂方法导入。为对象设置依赖的过程称为*injection*。

##### Constructor Injection

Constructor injection将实例化与injection相结合。需要给constructor添加`@Inject`注解，并在参数中接收类的依赖。大多数constructor会将这些参数赋值给final fields：

```Java
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processorProvider;
  private final TransactionLog transactionLogProvider;

  @Inject
  public RealBillingService(CreditCardProcessor processorProvider,
      TransactionLog transactionLogProvider) {
    this.processorProvider = processorProvider;
    this.transactionLogProvider = transactionLogProvider;
  }

```

如果类中没有经过`@Inject`注解的constructor，Guice将会使用一个public, no-arguments constructor (if exists)。最好使用注解，这样可以显示申明这个类型参与了DI。

Constructor injection对于unit testing非常友好：

1. 如果类通过单个constructor接收它的所有依赖，那么我们不会忘记去设置依赖。
2. 如果新增加了依赖，那么涉及到的测试代码会break掉，很直接的提示我们去fix这些compile errors。

##### Method Injection

Guice也可以将依赖注入到方法中，如果方法含有`@Inject`注解：

```Java
public class PayPalCreditCardProcessor implements CreditCardProcessor {
  
  private static final String DEFAULT_API_KEY = "development-use-only";
  
  private String apiKey = DEFAULT_API_KEY;

  @Inject
  public void setApiKey(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
```

##### Field Injection

Guice会注入含有`@Inject`注解的fields。很简洁，但是对test很不友好：

```Java
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  @Inject Connection connection;

  public TransactionLog get() {
    return new DatabaseTransactionLog(connection);
  }
}
```

避免对`final` fields使用*field injection* (因为 *weak semantics*)。

> <font color="red">TODO</font> 什么是weak semantics?

##### Optional Injections

Method injections和fields injections可以设置为可选的，这样使得当dependencies不存在的时候Guice会silently ignore这些依赖：

```java
public class PayPalCreditCardProcessor implements CreditCardProcessor {
  private static final String SANDBOX_API_KEY = "development-use-only";

  private String apiKey = SANDBOX_API_KEY;

  @Inject(optional=true)
  public void setApiKey(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
```

Optional injection和just-in-time bindings混用时需要格外留心。例如，以下情况在`Date`没有显式绑定时意外会被注入。因为`Date`有一个public no-arguments constructor：

```java
@Inject(optional=true) Date launchDate;
```

#### On-demand Injection

Method injection和field injection可以被用来初始化一个已经存在的实例。你可以通过`Injector.injectMembers` API来实现这一目的：

```Java
  public static void main(String[] args) {
    Injector injector = Guice.createInjector(...);
    
    CreditCardProcessor creditCardProcessor = new PayPalCreditCardProcessor();
    injector.injectMembers(creditCardProcessor);
```

#### Static Injections

> <font color="red">TODO</font> 不是很明白

#### Automatic Injection

Guice会自动注入下列这些：

* `bind`语句中传递给`toInstance()`的实例。
* `bind`语句中传递给`toProvider()`的provider instances。

#### Injecting Providers

当你想要为依赖的类型生成多个实例时，可以使用Guice的provider。Providers在每次调用`get()`方法时产生一个实例：

```java
public interface Provider<T> {
  T get();
}
```

##### Provider for multiple instances

当同种类型需要产生多个实例时，使用providers:

```java
public class LogFileTransactionLog implements TransactionLog {

  private final Provider<LogFileEntry> logFileProvider;

  @Inject
  public LogFileTransactionLog(Provider<LogFileEntry> logFileProvider) {
    this.logFileProvider = logFileProvider;
  }

  public void logChargeResult(ChargeResult result) {
    LogFileEntry summaryEntry = logFileProvider.get();
    summaryEntry.setText("Charge " + (result.wasSuccessful() ? "success" : "failure"));
    summaryEntry.save();

    if (!result.wasSuccessful()) {
      LogFileEntry detailEntry = logFileProvider.get();
      detailEntry.setText("Failure result: " + result);
      detailEntry.save();
    }
  }
```

##### Providers for lazy loading

当某种依赖生成实例的代价非常昂贵时，可以使用provider来推迟实例的生成。当不经常使用该依赖时，使用providers会非常有效：

```Java
public class DatabaseTransactionLog implements TransactionLog {
  
  private final Provider<Connection> connectionProvider;

  @Inject
  public DatabaseTransactionLog(Provider<Connection> connectionProvider) {
    this.connectionProvider = connectionProvider;
  }

  public void logChargeResult(ChargeResult result) {
    /* only write failed charges to the database */
    if (!result.wasSuccessful()) {
      Connection connection = connectionProvider.get();
    }
  }
```

##### Providers for Mixing Scopes

依赖于一个narrower scope的对象是错误的行为。假设有一个singleton transanction log需要request-scoped current user。直接注入user是错误的，因为不同的request对应的user是不同的。这时候应该使用providers来通过调用`get()`方法来产生实例，保证mix scopes safely:

```
@Singleton
public class ConsoleTransactionLog implements TransactionLog {
  
  private final AtomicInteger failureCount = new AtomicInteger();
  private final Provider<User> userProvider;

  @Inject
  public ConsoleTransactionLog(Provider<User> userProvider) {
    this.userProvider = userProvider;
  }

  public void logConnectException(UnreachableException e) {
    failureCount.incrementAndGet();
    User user = userProvider.get();
    System.out.println("Connection failed for " + user + ": " + e.getMessage());
    System.out.println("Failure count: " + failureCount.incrementAndGet());
  }
```

### AOP

#### Aspect Oriented Programming

Guice支持*method interception*。它使得当一个*matching*方法被调用时执行我们额外补充的代码。因为interceptors将问题切分成了aspects，而不是对象，所以称为Aspect Oriented Programming (AOP)。

Guice官网提供了一个**Forbidding method calls on weekends**的[示例](https://github.com/google/guice/wiki/AOP#example-forbidding-method-calls-on-weekends)。

### References

\[1\] [google/guice](https://github.com/google/guice)

\[2\] [Interface, Abstract Class和Concrete Class的区别](http://ot-note.logdown.com/posts/208733/interface-and-abstract-class-different-from-the-concrete-class)

### More Questions

\[1\] Java中`extends`和`implements`的区别

[2] Java中`nested classes`和`inner classes`的区别

\[3\] Java中的`threadsafe`