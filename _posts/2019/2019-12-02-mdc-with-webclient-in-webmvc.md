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
This is useful while performing WebClient's operation chain at execution time. 
However, this hook is not enough to make MDC work properly with `WebClient` in servlet environment.
There still needs to pass the original MDC values to the WebClient's operation chain.

To do this, we can create a `ExchangeFilterFunction` which grabs MDC values from current thread(where it processes http request) and pass them to WebClient's reactive operation chain.

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

Now, the MDC values are available to WebClient's execution chain. The only thing left is to use `Scheduler.onScheduleHook` to decorate the execution by the scheduler.

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
[Sergei(@bsideup) pointed me `Schedulers.addExecutorServiceDecorator`.](https://twitter.com/bsideup/status/1201779871803944960)  
Since that is less intrusive to operators, I'll update the code with that version soon.)_

We can use `Hooks.onEachOperator` to pass around the MDC values via subscriber context.

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

      MDC.clear();
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
