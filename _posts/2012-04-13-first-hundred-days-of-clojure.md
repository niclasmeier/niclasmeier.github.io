---
layout: post
status: publish
published: true
title: First hundred days of Clojure
categories:
- Clojure
---
In politics you have a one hundred days period of grace. In December I started to work with [Clojure](http://www.clojure.org), so I guess it's time to have a clojure look.

In the last couple of years I encountered a couple of different languages. At [scoyo](http://www.scoyo.de) it was [ActionScript](http://www.adobe.com/devnet/actionscript.html) and [Flex](http://flex.org/) from Adobe. I had the honor to lead a team of great engineers, but with good people you don't get close enough to the real stuff. At [gamigo](http://www.gamigo.de) the new technology was [PHP](http://www.php.net). But for the last thirteen years [Java](http://www.java.com/) was always a big part of my career... At my new [job](http://www.herolabs.com) the new kid on the block is [Clojure](http://www.clojure.org) - a LISP dialect based on Java VM. Clojure was created by [Rich Hickey](http://twitter.com/#!/richhickey) and first appeared in 2007.

When we started to work on our new product I said to my boss and my colleagues: In Java I know how a state of the art application has to look like, but I have no clue how to build a Clojure application. Fortunately a colleague had some Common LISP experience and already a closer look on the stack we should use. At the moment our application uses this tools and libraries. I guess I will come back to some of them later on.

*   [Leiningen](https://github.com/technomancy/leiningen) (aka [lein](https://github.com/technomancy/leiningen)) - The build and dependency managmement tool for Clojure.
*   [Noir - A nice library to build websites/-services in Clojure.](http://webnoir.org/)
*   [CongoMongo](https://github.com/aboekhoff/congomongo) and [mongoDB](http://www.mongodb.org/) - The database driver and the database we use.

**My first days with Clojure** When you are coming from a language like Java you are used to some things like: _Structure everything_ - Static typing is your friend, so smack every data you have into a class. I guess most Java Enterprise application use a significant part of their computing time with copying references from one bean to another. When you use Clojure you simply don't do this and sometimes is still hard for me not to worry about it. But this gives you a whole new level of freedom, you may use e.g. `(merge …)` to simply combine two of you entities into one larger one.

_Lots of ceremony_ - When you work in Java you create a huge amount of classes. The longer you work with Java the more inner and anonymous classes you tend to use. From these classes ninety percent are stupid bean classes with some attributes plus a setter and a getter per attribute and if you work with experts a `hashCode` and `equals` method. The remaining ten (or so) percent of you classes are services or stuff like that, who transform beans in different beans or XML, JSON, String, stuff like that. In clojure you put your structured data in maps - that's it. Due to the fact that all of closures datatypes are immutable you build some functions who do the same like the services and you are done. But instead of shuffling data from one bean to another you may use `(assoc …)`, `(dissoc …)`, `(select-keys …)` or `(merge …)` to efficiently create modified or new entities for the business logic or output generation.

But I can say, you'll get used to this really fast and so I had programmed my first JSON web service very fast. But as always stuff get a little more complex …

The first hurdle I had to climb where collections. Unlike the good old `for (…;…;…) {…}` loop in Java the `(for …)` list comprehension in Clojure is a beast. The first thing to do, is to realize that despite the name these two guys are totally different. Staring with the fact that the Clojure `(for …)` will be evaluated lazily, so if you want a similar evaluation behavior that the Java version you should try `(doseq …)`. A standard pattern in Java is to define some nice result variables and then iterate with `for` through the collection changing your variables. After the loop your result is in the variable ready for further action. But you cannot do this in Clojure because you don't have variables. You have `let` bindings which look somewhat similar but they are not mutable like your datatypes. So the `(for …)` list comprehension works a little bit different. It takes a collection and returns a collection of same or shorter length. In the function body you can compute the new value of the new collection `for` returns. You even may skip values with the `:when <val></val>`option. A very similar function is the `(map …)` function which also maps a value to a new value and returns a lazy collection of these new values.

**The first pitfalls** Don't get me wrong, the Clojure collection library is great, but you need some time to get used to it. A nasty side effect was, that shortly after I though: "Yes, I am king of all collections!" you stumble over oddities like the different behavior in the `(conj …)` function. In Clojure you have these functions `(cons …)` which adds an element in front of a sequence and `(conj …)` which append an element at the end of a sequence. At least this is what I though. In fact it does append an element if the sequence is a vector, if it is a list it behaves like `(cons …)`. You can spend some hours before you find this in the doc.

Another nice one - showing the power and danger - was related to Facebook and Mongo DB. We query some data from Facebook and store them into the a Mongo DB. The process only does some restructuring of the data but doesn't touch the values before we insert them into the database. To optimize the Facebook querying we switched from a hundered calls of the Facebook [graph API](http://developers.facebook.com/docs/reference/api/) to three [FQL](http://developers.facebook.com/docs/reference/fql/) queries. This worked really well. But Facebook has a minor difference in the results. In the graph API the IDs are strings in the FQL result set IDs are numbers. The whole process worked percieved well for a while but the I noticed that the queries on the Mongo DB delivered not enough results. What happened? The [Clojure JSON library](https://github.com/mmcgrana/clj-json) we used started to deliver numbers instead of strings and we where storing them into the Mongo DB. This worked without changing a line of code, which is nice if you want it. But the queries on the Mongo DB where still looking for strings …

**Thinking more, coding less** I guess this says it all. The first weeks of Clojure where unsatisfying. When coded in Java I had lots of code: Classes, XML files, etc. But in Clojure the result of the day were sometimes two functions and a macro. First that occurred very strange to me but I realized that I had to code less to get the same result. And these two functions and the macro where very lean and elegant and solved problems worth a couple of Java classes. This was the first time I really realized the power of Clojure and the different approach.

**Homoiconic or code is data** If you start with Clojure you soon stumble about this whole code is data stuff. This is was homoiconic in the end means, that the syntax of your language can be written in data structures of your language. If you look at:

{% highlight clojure %}
(for [i (range 1 100)] (println i))
{% endhighlight %}

You can read it as list with the first element `for` as symbol, a vector of further forms as second element and another list as third elements. The term for this is forms.

So doing this all LISPs have an extremely small syntax. At least that is what they are telling you. But I believe that the LISP/Clojure gurus (e.g. [Gerald Jay Sussman](http://groups.csail.mit.edu/mac/users/gjs/) or [Rich Hickey](http://twitter.com/#!/richhickey)) are cheating us a little bit.

I guess I have to agree that the languages it self has very little syntax, but the language is shifting the complexity into the libraries. If you have a look at the `(for …)` comprehension. You saw the simple variant a few lines ago but you can also du this:

{% highlight clojure %}
(for [i (range 1 100) :let [f (format "%03d" i)] :when (odd? i)] (println f))
{% endhighlight %}

This nice little feller prints all odd numbers from 1 to 100 formatted with leading zeros. But instead of using `(let …)` or filtering the range collection I used some not so well documented options of the binding vector of the `(for …)`. So how does it work? First `for` works internally different because it is a Clojure special form, but there are several macros - yes and I also wrote some - who do the same. If you are interested you may look at the `(defpage …)` macro, which does also some black magic. You usually use macros to do this type of things but you also may use functions (to a certain degree). So what there macros is to analyze the vector on the first position of the parameter list. In the `for` case you would iterate through the list and if you encounter a keyword (the things with `:` like `:let`) instead of a symbol you do some extra special stuff.

Usually you use this to define your own [domain specific language (DSL)](http://en.wikipedia.org/wiki/Domain-specific_language) in Clojure. This is what makes the solution of your problems very elegant because you make your problem related language where you can model and solve the problem very easily.

But also the Clojure standard functions are using this heavily and are so hinding a lot of implicit syntax form the user who depends on documentation and examples to discover how this works. For me this was sometimes very frustrating because very helpful options (e.g. `:when <val></val>`) are sometimes hard to find.

**Java interoperability or it's safe as long as you are in object land** Clojure lives (at least in the version we use) on the Java VM - there is a .NET variant and also with [ClojureScript](https://github.com/clojure/clojurescript) a variant which runs as Java-Script. So if you are on the JVM you want to use some the great libraries already exist in the Java ecosystem. If you e.g. look at [clj-json](https://github.com/mmcgrana/clj-json) a very fast JSON library for Clojure. It depends on the [Jackson](http://jackson.codehaus.org/) library and provides only a thin layer upon it.

To use Java objects and libraries in Clojure is very simple, the messy part only begins if you want to use primitive types. These are supported but its no fun anymore. A word of warning on working with Java objects. You are leaving the zone of immutability when you work with java objects. If you get yourself a `java.util.Date` you can mutate the date like you can do it in Java and mess up big time.

A very nice interop feature are `(deftype …)`, `(proxy …)` or `(reify …)`. You can use this macros to create or extend java Objects. Have a look at this:

{% highlight clojure %}
(proxy [com.rabbitmq.client.DefaultConsumer] [(channel connection)]
       (handleDelivery [tag envelope prop body] (println body))
       )
{% endhighlight %}

This tiny piece of code extends the `DefaultConsumer` and overrides the `handleDelivery` method. You can hand over the received object to a channel as a consumer and integrate directly into the Rabbit MQ Java driver.

**Some words to the ecosystem** Finally I want to have some word about some nice tools, library and stuff on Clojure ecosystem.

_Leiningen_ - A very popular build tool which also offers dependency resolution. My experience with Leiningen is, that it works out of the box in most cases. Unfortunately it tends to hang once or twice a day, I guess it tries to figure out if it must update some dependencies when it's gone.

_[Midje](https://github.com/marick/Midje)_ - A very powerful Clojure testing framework which includes facilities to easily mock every function. This makes writing even complex tests very easy.

_[avout](http://avout.io/)_ - This one is nice. It integrates tools like Clojure atoms and refs into Zookeeper (or some other coordination engines). It also offers distributed locks. Very easy to use and to build distributed cluster coordination.

_[pallet](http://palletops.com/)_ - A library who offers infrastructure management (e.g. for Amazon EC2) in clojure. You also may remotely install stuff or start and stop services. Great for dev-ops, but the documentation could be better and at the moment only Clojure 1.2 is supported. It took me some effort to get it working but when you know how to use it very powerful.

**What suc**s?** In this post I told you various thing about the power and beauty of Clojure but where are some things that I don't like:

_No informations on data structures_ - I already told you that in Java you write a lot of classes to structure you data. On one hand side this is a lot of overhead, but on the other hand side its also a kind of always correct and available documentation. If you have to get into some foreign code that fetches data from a database it's very hard to discover what is the right data structure and which type are the right ones. This also makes changes very difficult.

_Dynamic invocation_ - Functions are compiled and invoked when whey are invoked. You don't have any chance of previous checking if the function call is possible or not. This is extremely annoying when you change the signature of a function e.g. you remove a parameter. If you miss a spot where the function is called you get an error. This gets even worse if you use restructuring and variable parameter lists. You can do something like this:

{% highlight clojure %}
(defn fetch [database & {:keys [:where]}] …)
{% endhighlight %}

You can call this function with or without the `:where` parameter. This looks like this:

{% highlight clojure %}
(let [all (fetch my-database)
      some (fetch my-database :where {:height {:$gt 42}})]
	…
)
{% endhighlight %}

This is every powerful when you are creating flexible functions for DSLs but can cause very nasty side effects when you refactor these functions and change the signature.

_No bootstrapping_ - In the Java world you have dependency injection frameworks like [Google Guice](http://code.google.com/p/google-guice/) or the [Spring framework](http://www.springsource.org/). These frameworks are very helpful when you want to invert the control flow. The services you write do not have to be aware how to find the suitable services and resources. You are centralizing the bootstrapping process in one location (e.g. classes in Guice or XML in Spring). Clojure does not offer this facilities - you don't have objects to store state, not even configuration state. The solution we choose is like a typical [pattern](http://java.sun.com/blueprints/corej2eepatterns/Patterns/ServiceLocator.html). One namespace provides functions to access the resources (e.g. database connections, etc.) and there functions can be called from every where. But this has also the same drawbacks than the classical service locator. You don't have control about timing and the class is like a spider in the web called from every where.

_Clojure 1.2_ - The current stable release is Clojure 1.3\. Unfortunately some libraries we wanted to use where still based on Clojure 1.2\. In general this could work but be had some strange effects on some libraries which took us a while to discover that it was a Cojure 1.2/1.3 problem. A clear strategy how to deal with this isn't there yet, so all you can do is try and error.

**Loose ends** I didn't talk about stuff like STM (software transactional memory), Agents, Refs or Atoms. Mostly because they worked in the first place for me, or never even tried them. There are also a couple of nice library in the Clojure eco system which make your life as developer very easy. I recommend, that you have closer look yourself.

**… and in the end** To be honest, I had a tough couple of days in the last days. We are are in a stage where our prototype application grows into a real world application and we want to establish structures which makes maintenance easier. Unfortunately this is not so easy as we expected.

So my conclusions right now:

*   Clojure is fun - I really like it to write two line functions that kick ass.
*   Clojure does not need to hide - The eco system is growing fast and has some interesting libraries
*   The lack of tooling is sometimes a pain. There are times where `clojure.tools.logging/spy` or `println` are not enough. The absence of a debugger is sometimes a real productivity black hole.
*   Due to the dynamic nature you'll have to write a lot of tests to be able to perform refactorings. Only with a lot of tests you can make sure that the code will work afterwards.
*   Document your data structures to make sure that other developers have a chance to work in your code. I made some good experiences with a namespace that containes accessor functions. You can validate there functions with unit tests and the other guys in your team can easily see which values the can expect.

Nobody can say right now, how our little adventure with Clojure will end. But for the first release of out product our backend services will be based on Clojure. The time will tell how the maintainability and the performance of the backend will develop.

Right now I am not that confident that the solution will be good enough to be in service for a long time. But maybe Clojure surprises me … again …
