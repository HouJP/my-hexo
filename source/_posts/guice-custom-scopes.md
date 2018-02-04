---
title: Guice Custom Scopes
date: 2018-02-04 19:08:10
tags: [guice, java]

---

*Guice官方并不建议自定义scopes，它认为自带的scopes已经足够使用。*

创建custom scopes需要以下几步：

1. 定义一个scoping annotation。
2. 实现`Scope`接口。
3. 将scope annotation绑定到它的实现。
4. 触发scope的开始以及结束。

<!-- more -->

### Defining a scoping annotation

Scoping annotation用来标识自定义的scope，可以用它来注解Guice构造生成的类型、`@Provides`方法以及用在`in()`从句用于声明绑定。

```Java
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.ElementType.TYPE;
import java.lang.annotation.Retention;
import static java.lang.annotation.RetentionPolicy.RUNTIME;
import java.lang.annotation.Target;

@Target({ TYPE, METHOD }) @Retention(RUNTIME) @ScopeAnnotation
public @interface BatchScoped {}
```

### Implement Scope

Scope接口可以保证在每个scope实例中每种类型最多只有一个实例。官方提供了一个实现叫做`SimpleScope`，我们可以直接使用，也可以根据需要对`SimpleScope`进行调整生成自己的Scope实现。

```Java
import static com.google.common.base.Preconditions.checkState;
import com.google.common.collect.Maps;
import com.google.inject.Key;
import com.google.inject.OutOfScopeException;
import com.google.inject.Provider;
import com.google.inject.Scope;
import java.util.Map;

/**
 * Scopes a single execution of a block of code. Apply this scope with a
 * try/finally block: <pre><code>
 *
 *   scope.enter();
 *   try {
 *     // explicitly seed some seed objects...
 *     scope.seed(Key.get(SomeObject.class), someObject);
 *     // create and access scoped objects
 *   } finally {
 *     scope.exit();
 *   }
 * </code></pre>
 *
 * The scope can be initialized with one or more seed values by calling
 * <code>seed(key, value)</code> before the injector will be called upon to
 * provide for this key. A typical use is for a servlet filter to enter/exit the
 * scope, representing a Request Scope, and seed HttpServletRequest and
 * HttpServletResponse.  For each key inserted with seed(), you must include a
 * corresponding binding:
 *  <pre><code>
 *   bind(key)
 *       .toProvider(SimpleScope.&lt;KeyClass&gt;seededKeyProvider())
 *       .in(ScopeAnnotation.class);
 * </code></pre>
 *
 * @author Jesse Wilson
 * @author Fedor Karpelevitch
 */
public class SimpleScope implements Scope {

  private static final Provider<Object> SEEDED_KEY_PROVIDER =
      new Provider<Object>() {
        public Object get() {
          throw new IllegalStateException("If you got here then it means that" +
              " your code asked for scoped object which should have been" +
              " explicitly seeded in this scope by calling" +
              " SimpleScope.seed(), but was not.");
        }
      };
  private final ThreadLocal<Map<Key<?>, Object>> values
      = new ThreadLocal<Map<Key<?>, Object>>();

  public void enter() {
    checkState(values.get() == null, "A scoping block is already in progress");
    values.set(Maps.<Key<?>, Object>newHashMap());
  }

  public void exit() {
    checkState(values.get() != null, "No scoping block in progress");
    values.remove();
  }

  public <T> void seed(Key<T> key, T value) {
    Map<Key<?>, Object> scopedObjects = getScopedObjectMap(key);
    checkState(!scopedObjects.containsKey(key), "A value for the key %s was " +
        "already seeded in this scope. Old value: %s New value: %s", key,
        scopedObjects.get(key), value);
    scopedObjects.put(key, value);
  }

  public <T> void seed(Class<T> clazz, T value) {
    seed(Key.get(clazz), value);
  }

  public <T> Provider<T> scope(final Key<T> key, final Provider<T> unscoped) {
    return new Provider<T>() {
      public T get() {
        Map<Key<?>, Object> scopedObjects = getScopedObjectMap(key);

        @SuppressWarnings("unchecked")
        T current = (T) scopedObjects.get(key);
        if (current == null && !scopedObjects.containsKey(key)) {
          current = unscoped.get();

          // don't remember proxies; these exist only to serve circular dependencies
          if (Scopes.isCircularProxy(current)) {
            return current;
          }

          scopedObjects.put(key, current);
        }
        return current;
      }
    };
  }

  private <T> Map<Key<?>, Object> getScopedObjectMap(Key<T> key) {
    Map<Key<?>, Object> scopedObjects = values.get();
    if (scopedObjects == null) {
      throw new OutOfScopeException("Cannot access " + key
          + " outside of a scoping block");
    }
    return scopedObjects;
  }

  /**
   * Returns a provider that always throws exception complaining that the object
   * in question must be seeded before it can be injected.
   *
   * @return typed provider
   */
  @SuppressWarnings({"unchecked"})
  public static <T> Provider<T> seededKeyProvider() {
    return (Provider<T>) SEEDED_KEY_PROVIDER;
  }
}
```

### Binding the annotation to the implementation

我们必须将scoping annotation绑定到对应的scope implementation上，这个在module的`configure()`方法中配置。

```Java
public class BatchScopeModule extends AbstractModule {
  public void configure() {
    SimpleScope batchScope = new SimpleScope();

    // tell Guice about the scope
    bindScope(BatchScoped.class, batchScope);

    // make our scope instance injectable
    bind(SimpleScope.class)
        .annotatedWith(Names.named("batchScope"))
        .toInstance(batchScope);
  }
}
```

### Triggering the Scope

`SimpleScope`需要手动进入和退出。需要注意：调用`exit()`方法需要在`finally`从句中，否则如果抛出了异常，scope将一直处于open状态。

```Java
   @Inject @Named("batchScope") SimpleScope scope;

  /**
   * Runs {@code runnable} in batch scope.
   */
  public void scopeRunnable(Runnable runnable) {
    scope.enter();
    try {
      // explicitly seed some seed objects...
      scope.seed(Key.get(SomeObject.class), someObject);

      // create and access scoped objects
      runnable.run();

    } finally {
      scope.exit();
    }
  }
```

