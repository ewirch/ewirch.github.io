---
layout: article
title: "A Executor Is Not A Thread - or: Correct ThreadPoolExecutor Error Handling"
categories: articles
date: 2013-12-16
modified: 2013-12-16
tags: [java, error handling, ExecutorService, pitfalls]
image:
  feature: 
  teaser: /2013/12/executor.jpg
  thumb: 
ads: false
---

Java 1.5 introduced the `Executor` framework. In summery: if you have some tasks and you want them to be processed in parallel by a couple of threads, then a `Executor` is the solution you want to look for. But there are some surprising caveats which may lead to problems. And they are hard to diagnose.


## Introduction
`Executor` is a interface. The `ExecutorService` interface extends `Executor`. One of the standard implementations of `ExecutorService` is `ThreadPoolExecutor`. This one manages some threads in a pool and executes the tasks you give it. You do this in form of a list of `Runnable` instances. A typical `Runnable` implementation looks like this:

{% highlight java %}
public class MyWorker implements Runnable {
    private final Object data;

    public MyWorker(final Object data) {
        this.data = data;
    }

    @Override
    public void run() {
        process();
    }

    private void process() {
        // process data
    }
}
{% endhighlight %}

You initialize a `ThreadPoolExecutor` like this:

{% highlight java %}
int nThreads = 8;
Executor executor = new ThreadPoolExecutor(nThread, nThreads, 0, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
{% endhighlight %}

And since this is so cumbersome, there is a helper class for that:

{% highlight java %}
int nThreads = 8;
Executor executor = Executors.newFixedThreadPool(nThreads);
{% endhighlight %}

This is how to use a Executor wrapped in a method:

{% highlight java %}
public void executeRunnables(final List<Runnable> runnables) {
    int nThreads = 8;
    Executor executor = Executors.newFixedThreadPool(nThreads);
    for (final Runnable command : runnables) {
        executor.execute(command);
    }
}
{% endhighlight %}

This method would return to the caller immediately after creating the executor. But usually you want to wait until all the tasks have been processed before returning to your caller. For this purpose `ExecutorService` defines the methods `shutdown()` and `awaitTermination()`:

{% highlight java %}
public void executeRunnables(final List<Runnable> runnables) throws InterruptedException {
    final int nThreads = 8;
    final ExecutorService executor = Executors.newFixedThreadPool(nThreads);

    for (final Runnable command : runnables) {
        executor.execute(command);
    }

    executor.shutdown();

    executor.awaitTermination(2, TimeUnit.SECONDS);
}
{% endhighlight %}

(notice the change of executor type from `Executor` to `ExecutorService`) `shutdown()` puts the executor in "finish your work" mode. In this mode the executor will not accept new tasks. `awaitTermination()` waits until all threads processed all tasks. The time out takes care that your program doesn't wait forever if something goes wrong.

## Error handling
So what if something goes wrong? What if one of your tasks throws an exception? How do you handle that? How do you even know something gone wrong? One possible approach is to catch and collect all exception inside the `Runnable` implementation:

{% highlight java %}
public class MyRunnable implements Runnable {
    private final Object            data;
    private final List<Exception>   exceptions;

    public MyRunnable(final Object data, final List<Throwable> exceptions) {
        this.data = data;
        this.exceptions = exceptions;
    }

    @Override
    public void run() {
        try {
            process();
        } catch (final Exception ex) {
            exceptions.add(ex);
        }
    }

    private void process() {
        // process data
    }
}
{% endhighlight %}

But this approach has two major drawbacks: First, it would also catch the `InterruptedException` which may be used to control the thread under normal conditions. And second, the responsibility of error handling is moved to each `Runnable` implementation. You are going to implement a lot of `Runnables` and it's easy to forget something. `Executor` error handling calls for a generic solution.

Java 1.5 adds a method to register exception handlers for exactly this purpose: `Thread.setUncaughtExceptionHandler()`. `Exceptions` which are not handled by the `run()` method of the `Thread` implementation, will be forwarded to this handler. Let's modify the above example to use a exception handler:

{% highlight java %}
/**
 * Implementation of a UncaughtExceptionHandler, which stores all exceptions in a List.
 */
private static class ExceptionCollector implements UncaughtExceptionHandler {
    final List<Throwable> exceptions = Collections.synchronizedList(new LinkedList<Throwable>());

    @Override
    public void uncaughtException(final Thread t, final Throwable e) {
        exceptions.add(e);
    }
}

/**
 * A ThreadFactory, which registers a UncaughtExceptionHandler.
 */
private static class ThreadWithUncaughtExHandlerFactory implements ThreadFactory {
    private final UncaughtExceptionHandler    exHandler;

    public ThreadWithUncaughtExHandlerFactory(final UncaughtExceptionHandler exHandler) {
        this.exHandler = exHandler;
    }

    @Override
    public Thread newThread(final Runnable r) {
        final Thread t = new Thread(r);
        t.setUncaughtExceptionHandler(exHandler);
        return t;
    }
}

public void executeRunnables(final List<Runnable> runnables) throws InterruptedException {
    final int nThreads = 8;
    // UnhandledExceptionHandler which will collect Exceptions:
    final ExceptionCollector exHandler = new ExceptionCollector();
    // create a executor with a custom Thread factory:
    final ExecutorService executor = Executors.newFixedThreadPool(nThreads,
                                       new ThreadWithUncaughtExHandlerFactory(exHandler));

    for (final Runnable command : runnables) {
        executor.execute(command);
    }

    executor.shutdown();

    executor.awaitTermination(2, TimeUnit.SECONDS);

    if (exHandler.exceptions.size() > 0) {
        // rethrow first of the collected exceptions
        throw new RuntimeException(exHandler.exceptions.size() +
                    " exceptions occured. First exception:", exHandler.exceptions.get(0));
    }
}
{% endhighlight %}

Now this looks good! If a exception is not handled by the `Runnable` implementation the `Thread` will get it and it will pass it on to the uncaught exception handler. The handler will store it in the list for later usage. `executeRunnables()` waits until all `Threads` are done with work and checks the exception list then. If there is a entry it will be wrapped in a `RuntimeException` and rethrown. Instead of just passing the first exception, it's also possible to append the whole exception list to the thrown exception. This way the caller will be notified of all errors.

Or won't it?

Well, this article would probably not exist if everything would be that simple, would it? The truth is: it doesn't work. At least not always. Which makes it even worse. Sometimes it works and sometimes it doesn't.

## Problem analysis
A `Thread` is capable of processing only one `Runable` in general. When the `Thread.run()` method exits the `Thread` dies. The `ThreadPoolExecutor` implements a trick to make a `Thread` process multiple `Runnables`: it uses a own `Runnable` implementation. The threads are being started with a `Runnable` implementation which fetches other `Runanbles` (your `Runnables`) from the `ExecutorService` and executes them: `ThreadPoolExecutor -> Thread -> Worker -> YourRunnable`. When a uncaught exception occurs in your `Runnable` implementation it ends up in the finally block of `Worker.run()`. In this finally block the `Worker` class tells the `ThreadPoolExecutor` that it "finished" the work. The exception did not yet arrive at the `Thread` class but `ThreadPoolExecutor` already registered the worker as idle.

And here's where the fun begins. The `awaitTermination()` method will be invoked when all `Runnables` have been passed to the `Executor`. This happens very quickly so that probably not one of the `Runnables` finished their work. A `Worker` will switch to "idle" if a exception occurs, before the `Exception` reaches the `Thread` class. If the situation is similar for the other threads (or if they finished their work), all `Workers` signal "idle" and `awaitTermination()` returns. The main thread reaches the code line where it checks the size of the collected exception list. And this may happen before any (or some) of the `Threads` had the chance to call the `UncaughtExceptionHandler`. It depends on the order of execution if or how many exceptions will be added to the list of uncaught exceptions, before the main thread reads it.

A very unexpected behaviour. But I won't leave you without a working solution. So let's make it work.

## Correct solution
We are lucky that the `ThreadPoolExecutor` class was designed for extendibility. There is a empty protected method `afterExecute(Runnable r, Throwable t)`. This will be invoked directly after the `run()` method of our `Runnable` before the worker signals that it finished the work. The correct solution is to extend the `ThreadPoolExecutor` to handle uncaught exceptions:

{% highlight java %}
public class ExceptionAwareThreadPoolExecutor extends ThreadPoolExecutor {
    private final List<Throwable> uncaughtExceptions = 
                    Collections.synchronizedList(new LinkedList<Throwable>());

    @Override
    protected void afterExecute(final Runnable r, final Throwable t) {
        if (t != null) uncaughtExceptions.add(t);
    }

    public List<Throwable> getUncaughtExceptions() {
        return Collections.unmodifiableList(uncaughtExceptions);
    }
}
{% endhighlight %}
