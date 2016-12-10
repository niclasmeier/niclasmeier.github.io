---
layout: post
status: publish
published: true
title: From Promises to Futures
categories:
- Java
- Clojure
tags: []
---
This is the second post on an alternate implementation/approach to Futures and Promises on the Clojure programming language. In my [first post](/2012/06/alternate-futures-and-promises-for-clojure/) I talked a little bit about strength and weaknesses of the the actual implementation of Futures and Promises in Clojure and I defined how my Future and Clojure implementation should work.

At the end of the [first post](/2012/06/alternate-futures-and-promises-for-clojure/) I described how to implement a Promise in Clojure. A Promise in the context of my implementation should be a write handle to e.g. pass later on computed values to other processes.

In this blog post I want to:

*   show you how to get from the Promise implementation to a simple Future realization
*   talk about different execution strategies and why it's good do have influence on the future execution

So let's get started ...

Why it's good to have the control about the execution of your futures? With futures you are delegating the execution of your code to another thread (if we stick to the JVM) and getting the results back in your thread. So where is this other thread coming from?

This is very important question, because on the one hand creating a thread is a quite costly operation and on the other hand having very many threads is also not very good for your performance on the JVM, due to the heady scheduling required. E.g. if you have a quad core server you may execute for things in parallel, if you have 400 thread in you JVM the virtual machine schedules the execution of these 400 threads, wanting to to execute something, on these four cores. This can add a very huge amount of internal processing load to your server. Additionally every Thread has to allocate some memory for its call stack (see also [-XX:ThreadStackSize=512](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)) which also increases the memory consumption.

So it's not very clever to create a new Thread every time you want to execute a future and its also not a good idea to have hundreds of threads hanging around and waiting for work. Prior to Java 5 you had to create and manage threads in pools manually. But nowadays the [`java.util.concurrent.Executor`](http://docs.oracle.com/javase/1.5.0/docs/api/java/util/concurrent/Executor.html) and [`java.util.concurrent.ExecutorService`](http://docs.oracle.com/javase/1.5.0/docs/api/java/util/concurrent/ExecutorService.html) are the weapons of choice. You can obtain instances from [`java.util.concurrent.Executors`](http://docs.oracle.com/javase/1.5.0/docs/api/java/util/concurrent/Executors.html) and if you take a look at the `java.util.concurrent.Executors` you will find several different methods to create `ExecutorService`s with different execution strategies. If you have a look at Java 7 you'll find event more implementations like the [`java.util.concurrent.ForkJoinPool`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ForkJoinPool.html). Why do need all this different implementations? Unfortunately there is no solution that fits for any problem domain. Let's have two examples:

In the first scenario let's image that you have a legacy service which is optimized for batch processing and has really limited online query capabilities. So if you are connecting to this system via futures you maybe want to make sure that you cannot flood the service accidentially. So you would use a e.g. a fixed thread pool using [`newFixedThreadPool`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executors.html#newFixedThreadPool(int)) to create a pool with a fixed number of threads, so if you are flooding the system the waiting will occur on your thread pool and not messing up your legacy system.

Another scenario would be e.g. a service quering Facebook for user data. Assumed that Facebook has unlimited scalability ;-) there is no sense in limiting the access to Facebook on your side but you have to consider that some request may take very long due to the internet. So maybe [`newCachedThreadPool`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executors.html#newCachedThreadPool()) would be a good choice or you would consider the [`ForkJoinPool`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ForkJoinPool.html) which optimizes the throughput with a very small number of threads. How the choice of the right strategy can influence the throughput of your system was measured and documented by the [Akka](http://akka.io/) team in a nice [blog post](http://letitcrash.com/post/17607272336/scalability-of-fork-join-pool). When I am not totally wrong the results of the Akka guys also influenced [some updates on the `ForkJoinPool`](http://cs.oswego.edu/pipermail/concurrency-interest/2012-January/008987.html) which are announced for Java 8.

So to summarize this: It's a good idea to be in control of your execution strategy. If you have a look at the simple solution Future implementation Clojure even I would say, it meets 90% of the requirements even with no big performance penalty. And if you want to have a special solution you can easily create your special `java.util.concurrent.Executor` and submit your code there. If you remember the your function is a runnable a `(.submit executor (fn [] (do-some-heavy-loading this and that)))` should do the job.

Enough on the theory stuff, so how to we implement the alternate Futures now in the [demo project](https://github.com/niclasmeier/clj-future)?

Lets think about what we need for implementing the Future:

*   Another thread that will do the work for us
*   A reference to get the computed value later

We talked about the thread already and I will demonstrate how you may use it in a moment, so lets take a look at this reference thingy first. The reference need to be readable from my current thread and I need to get the reference in the very moment the I call the Future to execute my code. The thread which performs the execution must be able to write the result value of my computation to this thing. In the [previous post](/2012/06/alternate-futures-and-promises-for-clojure/) I showed how to implement a promise. This is a read handle to some value that will be available sometime and it is writable (once) to the current thread. So vóila there we are, we can use the promise to pass the value from the worker thread back to the original thread.

So how does this look like in Clojure code: Let's define a default executor at the beginning. An executor will be a simple function that will execute a passed function on another thread

{% highlight clojure %}
(def ^:private default-executor* (atom nil))

(defn default-executor [] (or @default-executor*
  (swap! default-executor* (fn [_] (let [s (Executors/newCachedThreadPool)]
                                     (fn [^Runnable r] (.submit s r)))))))

{% endhighlight %}

Very simple construction. The `(default-executor …)` function will return the default executor function and the private var will store an atom the contains a previous initialized value. In this example I simple choose a cached thread pool, which is the same that Clojure uses internally for its futures.

Next step is to create an execution function like `(future-call …)`:

{% highlight clojure %}
(defn execute-future [:as params}]
  (let [p (promise :on-complete on-complete :on-success on-success :on-failure on-failure)
        runnable (fn [] (try
                     (try-success p (body))
                     (catch java.lang.Exception e
                       (try-failure p e))))]
    (executor runnable)
    p))
{% endhighlight %}

Jepp, that is it ;-) So how does it work? You may pass a body function that will be executed asynchronously and the trinity of` :on-complete`,` :on-success` and` :on-failure` hanlders. Additionally you may pass an executor. If you don't pass an executor the previously defined default will be used.

The first thing that happens is, that we create a Promise instance and then wrap the body function to a runnable Function which will be passed to the executor function. The runnable function is nothing more like a simple `try-catch` expression which will trigger the `try-success` or the `try-failure` functions of the promise. The only things left is to call the executor function with the fresh runnable function and return the promise to the caller.

Nice, isn't it? Almost, because your code will look like:

{% highlight clojure %}
(execute-future
  :body (fn [] (java.lang.Thread/sleep 1000) 42)
  :on-success (fn [v] (println "Received:" v))
  :on-failure (fn [e] (println "Exception:" e)))
{% endhighlight %}

Kind of clumsy, so let's enhance it. But Clojure has the possibility to use macros to make it good looking.

{% highlight clojure %}
(defmacro future [body & callbacks]
  (let [params (reduce (fn [l e] (concat l e)) [:body `(fn [] ~body)]
    (select-keys
      (reduce (fn [m form] (let [[k & r] form
                                 fn (cons 'fn r)] (assoc m (keyword k) fn))) nil callbacks) [:on-complete :on-success :on-failure ]))]
    `(execute-future ~@params)))
{% endhighlight %}

This is small but a little bit nasty, so I simply tell you how beautiful the code will look instead of talking about the macro.

{% highlight clojure %}
(future (do (java.lang.Thread/sleep) 42)
   (on-success [v] (println "Received:" v))
   (on-failure [e] (println "Exception:" e)))
{% endhighlight %}

So that's more elegant - at least from my point of view - and I meet the goal that I set to myself. Looking exactly like my idea in the [previous post](/2012/06/alternate-futures-and-promises-for-clojure/).

But how do I influence the execution when using the macro. That's the part I swept under the rug a little bit. In the [demo project](https://github.com/niclasmeier/clj-future) you will find a macro for something like `(with-executor my-special-fork-join-executor …)` which will use `(binding ...)` to override the default for a certain code block.

### So what's next?

The next open issue would be to create composeable futures to be able to do stuff like

{% highlight clojure %}
(let [f (for [order-data (future (fetch-order oid))
              customer-data (future (fetch-customer cid))] (merge-data order-data customer-data))]
(on-success f [data] (send-email data)))
{% endhighlight %}

That would be the next challenging thing and I am not sure if I will achieve this. If you keep a look at the demo project you will see my progress and at the moment you will see, that I haven't started yet.

I will try to keep you informed, so stay tuned ...
