---
layout: post
title: "MDC with WebClient in WebMVC"
comments: true
date: 2019-12-02
---

Since `RestTemplate` is in maintenance mode, it is common to use `WebClient` even in servlet environment.
However, when it comes down to use MDC in logging, there is a challenge to make it work properly.

MDC uses thread bound values. Since `WebClient` uses reactor-netty under the hood, it runs on different threads. Therefore, a plumbing work is needed to properly use MDC in `WebClient`.

## Spring Boot 2.2 (reactor 3.3)

Reactor 3.3 introduced a new API, `Schedulers.onScheduleHook`, which we can use to pass around the MDC values between schedulers. 
This is useful while performing WebClient's operator chain at execution time. 
However, this hook is not enough to make MDC work properly with `WebClient` in servlet environment.
There still needs to pass the original MDC values to the WebClient's operator chain.

To do this, we can create a `ExchangeFilterFunction` which grabs MDC values from current thread(where it processes http request) and pass them to WebClient's reactive operator chain.

```java
ExchangeFilterFunction function = (request, next) -> {
  // here runs on main(request's) thread
  Map<String, String> map = MDC.getCopyOfContextMap();
  return next.exchange(request)
          .doOnRequest(value -> {
            // here runs on reactor's thread
            if (map != null) {
              MDC.setContextMap(map);
            }
          });
};

WebClient webClient = WebClient.builder().filter(function).build();
```

Now, the MDC values are available in WebClient's execution chain. The only thing left is to use `Scheduler.onScheduleHook` to decorate the execution by the scheduler.

```java
Schedulers.onScheduleHook("mdc", runnable -> {
  Map<String, String> map = MDC.getCopyOfContextMap();
  return () -> {
      if (map != null) {
        MDC.setContextMap(map);
      }
      try {
        runnable.run();
      } finally {
        MDC.clear();
      }
  };
});
```


## Spring Boot 2.1 (reactor 3.2)

_(Update 2019-12-03:  
[Sergei(@bsideup) pointed me `Schedulers.addExecutorServiceDecorator` API](https://twitter.com/bsideup/status/1201779871803944960).
Added `SchedulerMdcDecorator` and `SchedulerMdcProxyDecorator` implementation.)_

We can use `Schedulers.addExecutorServiceDecorator` to return wrapped `ScheduledExecutorService` that propagates MDC values to the new thread.

Again, this decoration happens only when switching schedulers on WebClient's operator chains. So, here still needs `ExchangeFilterFunction` mentioned above to be added to `WebClient`.

```java
/**
 * Propagate MDC values by decorating {@link ScheduledExecutorService}.
 *
 * @author Tadaya Tsuyukubo
 */
public class SchedulerMdcDecorator implements BiFunction<Scheduler, ScheduledExecutorService, ScheduledExecutorService> {

  @Override
  public ScheduledExecutorService apply(Scheduler scheduler, ScheduledExecutorService scheduledExecutorService) {
    // decorate ScheduledExecutorService
    return new MdcScheduledExecutorService(scheduledExecutorService);
  }

  static final class MdcScheduledExecutorService implements ScheduledExecutorService {
    private final ScheduledExecutorService delegate;

    public MdcScheduledExecutorService(ScheduledExecutorService delegate) {
      this.delegate = delegate;
    }

    private Runnable wrap(Runnable runnable) {
      Map<String, String> map = MDC.getCopyOfContextMap();
      return () -> {
        if (map != null) {
          MDC.setContextMap(map);
        }
        try {
          runnable.run();
        } finally {
          MDC.clear();
        }
      };
    }

    // or just delegate the logic by calling one for Callable
//    private Runnable wrap(Runnable runnable) {
//      return () -> wrap(() -> {
//        runnable.run();
//        return null;
//      });
//    }

    private <V> Callable<V> wrap(Callable<V> callable) {
      Map<String, String> map = MDC.getCopyOfContextMap();
      return () -> {
        if (map != null) {
          MDC.setContextMap(map);
        }
        try {
          return callable.call();
        } finally {
          MDC.clear();
        }
      };
    }

    private <T> Collection<? extends Callable<T>> wrap(Collection<? extends Callable<T>> callables) {
      return callables.stream()
              .map(this::wrap)
              .collect(toList());
    }

    @Override
    public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
      return this.delegate.schedule(wrap(command), delay, unit);
    }

    @Override
    public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
      return this.delegate.schedule(wrap(callable), delay, unit);
    }

    @Override
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
      return this.delegate.scheduleAtFixedRate(wrap(command), initialDelay, period, unit);
    }

    @Override
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
      return this.delegate.scheduleWithFixedDelay(wrap(command), initialDelay, delay, unit);
    }

    @Override
    public void shutdown() {
      this.delegate.shutdown();
    }

    @Override
    public List<Runnable> shutdownNow() {
      return this.delegate.shutdownNow();
    }

    @Override
    public boolean isShutdown() {
      return this.delegate.isShutdown();
    }

    @Override
    public boolean isTerminated() {
      return this.delegate.isTerminated();
    }

    @Override
    public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
      return this.delegate.awaitTermination(timeout, unit);
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
      return this.delegate.submit(wrap(task));
    }

    @Override
    public <T> Future<T> submit(Runnable task, T result) {
      return this.delegate.submit(wrap(task), result);
    }

    @Override
    public Future<?> submit(Runnable task) {
      return this.delegate.submit(wrap(task));
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException {
      return this.delegate.invokeAll(wrap(tasks));
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException {
      return this.delegate.invokeAll(wrap(tasks), timeout, unit);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException {
      return this.delegate.invokeAny(wrap(tasks));
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
      return this.delegate.invokeAny(wrap(tasks), timeout, unit);
    }

    @Override
    public void execute(Runnable command) {
      this.delegate.execute(wrap(command));
    }
  }
}
```

Or, with using dynamic proxy:

```java
/**
 * Propagate MDC values by decorating {@link ScheduledExecutorService} using JDK dynamic proxy.
 *
 * @author Tadaya Tsuyukubo
 */
public class SchedulerMdcProxyDecorator implements BiFunction<Scheduler, ScheduledExecutorService, ScheduledExecutorService> {

  @Override
  public ScheduledExecutorService apply(Scheduler scheduler, ScheduledExecutorService scheduledExecutorService) {
    return (ScheduledExecutorService) Proxy.newProxyInstance(SchedulerMdcProxyDecorator.class.getClassLoader(),
          new Class[]{ScheduledExecutorService.class},
          new MdcDecoratingInvocationHandler(scheduledExecutorService));
  }

  static final class MdcDecoratingInvocationHandler implements InvocationHandler {
    private final ScheduledExecutorService delegate;

    public MdcDecoratingInvocationHandler(ScheduledExecutorService delegate) {
      this.delegate = delegate;
    }

    @Override
    @SuppressWarnings("unchecked")
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      Class<?>[] paramTypes = method.getParameterTypes();
      if (paramTypes.length == 0) {
        return method.invoke(this.delegate, args); // no replace, simply proceed
      }

      Class<?> firstParamType = paramTypes[0];

      // swap Runnable/Callable/Collection<? extends Callable<?>)
      Object swapped;
      if (firstParamType.isAssignableFrom(Runnable.class)) {
        swapped = wrap((Runnable) args[0]);
      } else if (firstParamType.isAssignableFrom(Callable.class)) {
        swapped = wrap((Callable<?>) args[0]);
      } else if (firstParamType.isAssignableFrom(Collection.class)) { // see the ExecutorService API
        swapped = ((Collection<? extends Callable<?>>) args[0]).stream()
                .map(this::wrap)
                .collect(toList());
      } else {
        return method.invoke(this.delegate, args); // bail out, no replace needed
      }
      args[0] = swapped;  // swap

      return method.invoke(this.delegate, args);
    }


    private Runnable wrap(Runnable runnable) {
      Map<String, String> map = MDC.getCopyOfContextMap();
      return () -> {
        if (map != null) {
          MDC.setContextMap(map);
        }
        try {
          runnable.run();
        } finally {
          MDC.clear();
        }
      };
    }

    private Callable<?> wrap(Callable<?> callable) {
      Map<String, String> map = MDC.getCopyOfContextMap();
      return () -> {
        if (map != null) {
          MDC.setContextMap(map);
        }
        try {
          return callable.call();
        } finally {
          MDC.clear();
        }
      };
    }
  }
}
```

To register above decorators:
```java
Schedulers.addExecutorServiceDecorator("mdc", new SchedulerMdcDecorator());
// or
Schedulers.addExecutorServiceDecorator("mdc", new SchedulerMdcProxyDecorator());
```

Alternatively, `Hooks.onEachOperator` can be used to pass around the MDC values via subscriber context.
This is more intrusive to operation chain.
Since `Hooks.onEachOperator` injects the logic to all operators, you don't need to create `ExchangeFilterFunction` to `WebClient`.

Here is a sample implementation:
```java
/**
 * Propagate MDC to each reactor operation.
 *
 * The propagated values are at the time that the operation is subscribed.
 *
 * @author Tadaya Tsuyukubo
 */
public class ReactorMdcSupport {

  // only lift when MDC value exists
  private static final Function<? super Publisher<Object>, ? extends Publisher<Object>> lifter =
    Operators.liftPublisher(
      publisher -> {
        if (MDC.getCopyOfContextMap() == null) {
          return false;
        }
        // #empty, #just, #error
        if (publisher instanceof Fuseable.ScalarCallable) {
          return false;
        }
        return true;
      },
      (publisher, coreSubscriber) -> new MdcPropagatingSubscriber<>(coreSubscriber)
    );


  public static void register() {
    Hooks.onEachOperator("mdc", lifter);
  }

  public static void unregister() {
    Hooks.resetOnEachOperator("mdc");
  }

  static class MdcPropagatingSubscriber<T> implements CoreSubscriber<T> {
    // Key of reactor context that holds MDC key-values
    static final String MDC_CONTEXT_KEY = "mdc-context";

    private final CoreSubscriber<T> delegate;
    private final Context context;

    public MdcPropagatingSubscriber(CoreSubscriber<T> delegate) {
      this.delegate = delegate;
      Context currentContext = this.delegate.currentContext();
      Context context;
      if (currentContext.hasKey(MDC_CONTEXT_KEY)) {
        context = currentContext;
      } else {
        // MDC.getCopyOfContextMap() never returns null since it is prechecked by lifter
        Map<String, String> map = new HashMap<>(MDC.getCopyOfContextMap());
        context = currentContext.put(MDC_CONTEXT_KEY, map);
      }
      this.context = context;
    }

    @Override
    public Context currentContext() {
      return this.context;
    }

    @Override
    public void onSubscribe(Subscription s) {
      this.delegate.onSubscribe(s);
    }

    @Override
    public void onNext(T t) {
      Map<String, String> map = this.context.get(MDC_CONTEXT_KEY);
      MDC.setContextMap(map);

      this.delegate.onNext(t);

      // Do not clear MDC values here, in order to keep the MDC values on the thread
      // that has subscribed the publisher (original thread).
    }

    @Override
    public void onError(Throwable t) {
      this.delegate.onError(t);
    }

    @Override
    public void onComplete() {
      this.delegate.onComplete();
    }
  }
}
```
