## Spring MVC 异步处理

在 Servlet 3.0 的时候增加了对异步的支持，在此之前 Servlet 采用 Thread Per Request 的方式处理请求。即如果一个请求如果进行耗时的操作例如 IO，数据库或者调用第三方的接口等等，其所对应的的线程将同步的等待响应操作的完成，所以此时线程并不能即使的释放线程放回线程池以供后续使用，在并发量大的情况下，这会带来性能问题。在 Servlet 3.0 之后引入异步操作就可以避免这样的事情发生，增加异步操作之后的图如下所示：

![异步处理](http://img.programya.com/Snipaste_2019-12-01_08-43-37.png)

一个简单地异步的 Servlet 如下所示：
```java
@WebServlet(value = "/async", asyncSupported = true)
public class MyAsyncServlet extends HttpServlet {
    Log log = LogFactory.getLog(MyAsyncServlet.class);

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        AsyncContext asyncContext = req.startAsync();
        log.info("Request Start");
        StringBuilder stringBuilder = new StringBuilder("Request Start");
        asyncContext.start(() -> {
            try {
                log.info("Sub Thread Start");
                Thread.sleep(3000);
                AsyncContext asyncContextInner = req.getAsyncContext();
                ServletResponse response = asyncContextInner.getResponse();
                response.getWriter().write("Async Request End");
                log.info("Sub Thread End");
                asyncContext.complete();
            } catch (InterruptedException | IOException e) {
                e.printStackTrace();
            }
        });
        stringBuilder.append("Request End");
        log.info("Request End");
        resp.getWriter().write(stringBuilder.toString());
    }
}
```

其日志打印如下：

```log
01-Dec-2019 23:40:16.664 信息 [http-nio-8080-exec-44] com.xinyue.myspring4.servlet.MyAsyncServlet.doGet Request Start
01-Dec-2019 23:40:16.665 信息 [http-nio-8080-exec-44] com.xinyue.myspring4.servlet.MyAsyncServlet.doGet Request End
01-Dec-2019 23:40:16.666 信息 [http-nio-8080-exec-45] com.xinyue.myspring4.servlet.MyAsyncServlet.lambda$doGet$0 Sub Thread Start
01-Dec-2019 23:40:19.667 信息 [http-nio-8080-exec-45] com.xinyue.myspring4.servlet.MyAsyncServlet.lambda$doGet$0 Sub Thread End
```

当然 Spring MVC 也提供了对于异步的支持，主要有 `Callable` 和 `DeferredResult` 两种方式，首先看一下 `Callable` 的方式：

首先在 Web 容器的 Config 中添加上对于异步的支持：

```java
@Override
public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(30);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("ronnie-task-");
    executor.initialize();
    configurer.setTaskExecutor(executor);
}
```

然后可以设置相应的 Controller 如下：

```java
@Controller
public class MyAsyncController {
    Log log = LogFactory.getLog(MyAsyncController.class);

    @ResponseBody
    @RequestMapping(value = "/mvcasync1")
    public Callable<String> myAsync() {
        log.info("Main Start");
        Callable<String> callable = () -> {
            log.info("Sub Start");
            Thread.sleep(3000);
            log.info("Sub End");
            return "Demo String";
        };
        log.info("Main End");
        return callable;
    }
}
```

打印日志如下：

```log
01-Dec-2019 23:45:22.950 信息 [http-nio-8080-exec-49] com.xinyue.myspring4.controller.MyAsyncController.myAsync Main Start
01-Dec-2019 23:45:22.952 信息 [http-nio-8080-exec-49] com.xinyue.myspring4.controller.MyAsyncController.myAsync Main End
01-Dec-2019 23:45:22.958 信息 [ronnie-task-1] com.xinyue.myspring4.controller.MyAsyncController.lambda$myAsync$0 Sub Start
01-Dec-2019 23:45:25.973 信息 [ronnie-task-1] com.xinyue.myspring4.controller.MyAsyncController.lambda$myAsync$0 Sub End
```

大致的原理是 如果 Controller 返回 Callable；那 Spring MVC 将 Callable 提交到 `TaskExecutor` 使用一个隔离的线程进行执行；`DispatcherServlet` 和 所有的 `Filter` 退出 Web 容器线程，但 `Response ` 仍然保持打开状态； Callable 返回结果 Spring 将结果重新派发到 Web 容器，在回复之前处理；根据 Callable 返回结果 Spring MVC 进行视图渲染流程等。普通的拦截器并不能拦截异步请求的过程，如果要拦截则需要使用显示 `AsyncHandlerInterceptor` 接口的拦截器。

下面看一下 `DeferredResult` 的方式实现异步，同样首先要设置 MVC 支持异步，然后一个简单的 Controller 如下：

```java
@Controller
public class MyAsyncController {
    @ResponseBody
    @RequestMapping(value = "/mvcasync2")
    public DeferredResult<Object> myAsync2() {
        DeferredResult<Object> deferredResult = new DeferredResult<>(10000L, "121231");
        DeferredResultQueue.save(deferredResult);
        return deferredResult;
    }

    @ResponseBody
    @RequestMapping("/mvcasync3")
    public String myAsync3() {
        DeferredResult<Object> objectDeferredResult = DeferredResultQueue.get();
        UUID uuid = UUID.randomUUID();
        objectDeferredResult.setResult(uuid.toString());
        return "My Async3:::" + uuid;
    }
}
```

使用的一个中间类如下：

```java
public class DeferredResultQueue {
    private static Queue<DeferredResult<Object>> queue = new ConcurrentLinkedDeque<>();

    public static void save(DeferredResult<Object> result) {
        queue.add(result);
    }

    public static DeferredResult<Object> get() {
        return queue.poll();
    }
}
```

打开 `/mvcasync2` 页面之后会默认等待 `/mvcasync3` 之后返回结果，其实也就是在 `DeferredResult` 得到值之后返回结果。









