---
layout: post
status: publish
published: true
title: Alternate Futures and Promises for Clojure
categories:
- Java
- Scala
- Clojure

---
At the moment I am experimenting with various techniques to speed up some update queries from our [client](http://www.emhero2012.com). One important element of this efforts is making things parallel. E.g. to compute a list update of deltas for the client we need to compare the version of the user on the client and in the database and we need the user data at some other places on the delta computation. So it would be a good idea to initiate the loading of the user data asynchronous right at the beginning of the request processing. While the user data is loaded we can load/compute some other stuff (e.g. we need some data about the [competition](http://de.uefa.com/uefaeuro/index.html) and the matches). When all the required information is present continue the rest of the the job (e.g. generate some XML or JSON).

Futures implementations like [`java.util.concurrent.Future`](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/Future.html), [`scala.concurrent.Future`](http://www.scala-lang.org/archives/downloads/distrib/files/nightly/docs/library/scala/concurrent/Future.html) or [`clojure.core/future`](http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/future) seem to be the weapon of choice for this.

Due to the fact that our backend is written in [Clojure](http://clojure.org/) I took a closer look at the Clojure Future implementation `clojure.core/future`. The Clojure implementation is pretty basic and very similar the the underlying Java implementation. This isn't bad, but rather unsatisfying for a language emphasizing on concurrency like Clojure.<a id="more"></a><a id="more-156"></a>The Clojure Future API consists of these functions in the `clojure.core` namespace:

* `(future …)` - Creates a future instance and executes the provided code.
* `(future-call …)`- Creates a future instance and executes a provided function. This function is used by the `(future …)` marco.
* `(future-cancel …)` - Cancels the further, if possible.
* `(future-cancelled? …)` - Returns `true` if future f is cancelled
* `(promise …)` - Returns a promise object that can be read with `deref`/`@`, and set, once only.
* `(deliver …)` - Delivers the supplied value to the promise, releasing any pending derefs. A subsequent call to deliver on a promise will throw an exception.

This is a small but sufficient API. The `promise`/`deliver` functions are relatively new and having a _"Alpha - subject to change"_ note.

So how does this work:

{% highlight clojure %}
(let [f (future (do (java.lang.Thread/sleep 42))]
	(println "value:" @f)
)
{% endhighlight %}

You simply wrap the code you want the be execute in parallel in the `(future …)` macro and you get a future datatype which reifies `clojure.lang.IDeref`, `clojure.lang.IBlockingDeref`, `clojure.lang.IPending` and `java.util.concurrent.Future`. This has some pretty nice effects. The future can be read with `@`/`deref` and you may use it like any other Java future.

Not so nice is that you have the `(future-cancel …)` function or the `cancel()` method in the `java.util.concurrent.Future` interface. So what is so bad about this? First: This is a mutating function on the reference, so if you pass your future running a important job to someone else, he may simply cancel this job and you can't prevent this. Second: What happens when you cancel the job? When your long running job has a database update at the end, will the update be executed? Will truncations be rolled back? Connections closed?

That you don't get noticed when the future processing is finished is somewhat sad too. Due to the fact the functions are first class citizens and closures are all around, it would be great if you could provide e.g. a function as callback when the future was successfully finished. The Scala scala.concurrent.Future implementation offers method like `onComplete(…)`, `onSuccess(…)` or `onFailure(…)` where you can supply these kind of handlers. What is also great on Scala Futures is that they offer methods like `flatMap`, etc. which are used for the Scala `for` comprehension. This enables you to do stuff like this:

{% highlight scala %}
for (a <- someExpensiveComputation()
     b <- hugeLoadFromDatabase())
  yield merge (a, b)
{% endhighlight %}

This would execute in parallel the `someExpensiveComputation` and `hugeLoadFromDatabase` and both results are available in the list comprehension.

The last flaw I see is, that you have no control about the execution. Clojure (1.3) uses the same `Executor` used for `(send-off …)` to execute the future but you cant't influence this and you have no control on the executor. In Java there is lot's of boilerplate, because you'll have to create an `Executor` and submit a `Runnable` manually, therefore you have the full control. Scala had also this flaw hiding the execution of a Future but with the rework of Futures and Promises in Scala ([SIP-14](http://docs.scala-lang.org/sips/pending/futures-promises.html) an `ExecutionContext` was introduced, which solves this issue.

So let's explore how we can build a Future / Promise library in Clojure where that does not have these issues.

* * *

### The goal

My goal would be to write something like this

{% highlight clojure %}
(future (load-data-from-somewhere ctx)
	(on-success [v] (initiate-process v))
	(on-failure [cause] (warn cause (format "Unable to load data, retrying in %s seconds" retry-threshold))))
{% endhighlight %}

it would be also nice to do something like

{% highlight clojure %}
(let [f (for [order-data (future (fetch-order oid))
              customer-data (future (fetch-customer cid))] (merge-data order-data customer-data))]
(on-success f [data] (send-email data)))
{% endhighlight %}

This would return a future that aggregates the result of both fetch requests. You would be able to register for the `:on-success` event and perform an operation if wanted.

Lets see how far we get ...

### Definition of Terms

First - as some kind of Homework - I want to define what a Future and a Promise is for me (and yes this is heavily inspired by other sources - e.g. some talks by [Viktor Klang](https://twitter.com/#!/viktorklang) or the [SIP-14](http://docs.scala-lang.org/sips/pending/futures-promises.html)).

* _Future_ - A Future is a read handle to a value that may be come available some times later. A Future may be read many times.
* _Promise_ - A Promise is a write handle for a value that will be available in the later process. You may pass the promise out to someone else to provice a reference to a value that you don't have yet. A promise may be written only one.

A Future and a Promise - at least in this example - will have this three states:
* `:pending` - The Future/Promise has not yet received a value.
* `:success` - The Promise/Future has received the a value and the previous computation has to considered successful.
* `:failure` - The Promise/Future has received the a value and the previous computation failed e.g. threw an Exception.

### Promise - the basics

So let's look at some code and start with some [code](https://github.com/niclasmeier/clj-future). You will find a somewhat working [demo project on github.com](https://github.com/niclasmeier/clj-future)

First I want to define some artifacts needed for a Promise.

{% highlight clojure %}
(defn promise [] …)
(defn status [promise] …)
{% endhighlight %}

I guess these are the first very functions needed for a promise. The first one `(promise …)` is a basic factory function for the handle and the `(status promise)` returns the status. The promise will be a `ref` data type, so the value may be obtained by using `deref`.To set the value we will define functions like:

{% highlight clojure %}
(defn success [promise value] …)
(defn failure [promise cause] …)
{% endhighlight %}

These two function will replace the `(declare …)` function of the Clojure original. The `(success …)` function will set the Promise value to the passed value and the state of the promise will change to `:success`. The `(failure …)` function will indicate that the computation that gave out the Promise failed, the value of the Promise is reflects the cause for the failure and the state of the promise will change to `:failed`. Both functions will throw an exception if someone tries to write to the Promise a second time.

How do we implement the handle. I defined a Clojure protocol to have an interface:

{% highlight clojure %}
(defprotocol Promise
  (future* [promise])
  (try-success* [promise x])
  (try-failure* [promise x])
  (status* [promise])
  )
{% endhighlight %}

Pretty simple I guess so here is how it works:

* `(future* [promise])` - This function will create a future for this promise. So someone create a promise and pass it to an asynchronous process while passing out a future to another process that will receive the value when the value was written to the promise. In fact in my variant it simply returns itself.
* `(try-success* [promise x])` - This function try to write a value to the promise and switch to the `:success` state
* `(try-failure* [promise x])` - This function try to write a value to the promise and switch to the `:success` state
* `(status* [promise])` - This function returns the current state of the promise.

Why the `*` at the end of the function name. I use this convention to indicate some private/delegate functions. E.g. the `(success …)` function will simply delegate to the `(try-success* [promise x] …)`. The main difference is that both `(try-…)` method returns a boolean value indicating a successful operation while `(success …)` returns the supply value and will throw an exception if an error occurs.

The next step is to create an instance:

{% highlight clojure %}
(defn- notify [x & listeners]
  (doseq [listener listeners :when listener] (listener x)))

(defn- try-complete-promise [^java.util.concurrent.CountDownLatch d v s x complete variable]
  (if (pos? (.getCount d))
    (if (compare-and-set! v nil [s x])
      (do
        (.countDown d)
        (apply notify x (concat @complete @variable))
        true)
      false)))

(defn promise []
  (let [d (java.util.concurrent.CountDownLatch. 1)
        v (atom nil)
        on-complete (create-list on-complete)
        on-success (create-list on-success)
        on-failure (create-list on-failure)]
    (reify
      Promise
      (future* [this] this)
      (try-success* [_ x] (try-complete-promise d v :success x on-complete on-success))
      (try-failure* [_ x] (try-complete-promise d v :failure x on-complete on-failure))
      (status* [this] (if (.isRealized this) (first @v) :pending ))
      clojure.lang.IDeref
      (deref [_] (.await d) (second @v))
      clojure.lang.IBlockingDeref
      (deref
        [_ timeout-ms timeout-val]
        (if (.await d timeout-ms java.util.concurrent.TimeUnit/MILLISECONDS)
          (second @v)
          timeout-val))
      clojure.lang.IPending
      (isRealized [this]
        (zero? (.getCount d)))
      )))

{% endhighlight %}

The previously announced factory function simply uses `reify` to create a new instance. The state is held as closure by the defined `let` bindings. Primarily important are `d` - a `CountDownLatch` to prevent multiple writes and handle synchronization - and `v` an atom holding the value. The value of `v` will be an array containing the status (the first element) and the supplied value (the second element).

To read the value of the Promise the reified instance also implements `clojure.lang.IDeref` and `clojure.lang.IBlockingDeref`. This offers the possibility to read the promise by using `deref` or the `@` reader macro. This works exactly like the original Clojure promise and to be honest the source is also very similar … ;-)

Additionally there is the possibility to pass some listener functions which will be invoked with the value (or cause) when the promise is written. This will be used to the Future realization later on.

If you are still curious have a look at the [next post](2012/06/from-promises-to-futures/).
