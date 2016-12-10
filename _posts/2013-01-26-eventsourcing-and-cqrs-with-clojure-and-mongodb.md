---
layout: post
status: publish
published: true
title: Eventsourcing and CQRS with Clojure and MongoDB
author:
  display_name: Niclas Meier
  login: niclas
  email: mail@niclas-meier.de
  url: 'http://www.niclas-meier.de'
author_login: niclas
author_email: mail@niclas-meier.de
author_url: 'http://www.niclas-meier.de'
excerpt: "It was about a year ago when I stumbled about the concepts of <a href=\"http:&#47;&#47;martinfowler.com&#47;eaaDev&#47;EventSourcing.html\">event sourcing<&#47;a> and <a href=\"http:&#47;&#47;martinfowler.com&#47;bliki&#47;CQRS.html\">command query responsibility segregation<&#47;a> in a talk from <a href=\"http:&#47;&#47;codebetter.com&#47;gregyoung&#47;\">Greg Young<&#47;a> at QCon 2011 in London which I watched on <a href=\"http:&#47;&#47;www.infoq.com&#47;presentations&#47;Events-Are-Not-Just-for-Notifications\">InfoQ<&#47;a>. This challenged some things I knew for sure regarding how to store and manage data. \r\n\r\nFortunately I joined <a href=\"http:&#47;&#47;herolabs.com\">HEROLABS<&#47;a> in December 2011 and due to the fact that we where starting a brand new product, I was able to play around with these concepts. Additionally we chose <a href=\"http:&#47;&#47;www.mongodb.org&#47;\">MongoDB<&#47;a> instead of a relational database in our <a href=\"http:&#47;&#47;itunes.apple.com&#47;de&#47;app&#47;em-hero-2012&#47;id530513702?l=de&ls=1&mt=8\">first project<&#47;a> as our main database management system. <a href=\"http:&#47;&#47;www.mongodb.org&#47;\">MongoDB<&#47;a> is attributed to the NoSQL databases and is in the first place a big document (<a href=\"http:&#47;&#47;json.org\">JSON<&#47;a>&#47;<a href=\"http:&#47;&#47;bsonspec.org\">BSON<&#47;a>) store. Additionally MongoDB offers facilities to index and search the collection of documents.\r\n\r\nIn conjunction with <a href=\"http:&#47;&#47;clojure.org\">Clojure<&#47;a> your are able to build a simple a quite powerful system to use event sourcing and CQRS."
wordpress_id: 184
wordpress_url: 'http://www.niclas-meier.de/?p=184'
date: '2013-01-26 23:25:24 +0100'
date_gmt: '2013-01-26 22:25:24 +0100'
categories:
  - Clojure
  - MongoDB
tags: []
comments: []
---

It was about a year ago when I stumbled about the concepts of [event sourcing](http://martinfowler.com/eaaDev/EventSourcing.html) and [command query responsibility segregation](http://martinfowler.com/bliki/CQRS.html) in a talk from [Greg Young](http://codebetter.com/gregyoung/) at QCon 2011 in London which I watched on [InfoQ](http://www.infoq.com/presentations/Events-Are-Not-Just-for-Notifications). This challenged some things I knew for sure regarding how to store and manage data.

[](http://www.infoq.com/presentations/Events-Are-Not-Just-for-Notifications)

[Fortunately I joined](http://www.infoq.com/presentations/Events-Are-Not-Just-for-Notifications) [HEROLABS](http://herolabs.com) in December 2011 and due to the fact that we where starting a brand new product, I was able to play around with these concepts. Additionally we chose [MongoDB](http://www.mongodb.org/) instead of a relational database in our [first project](http://itunes.apple.com/de/app/em-hero-2012/id530513702?l=de&ls=1&mt=8) as our main database management system. [MongoDB](http://www.mongodb.org/) is attributed to the NoSQL databases and is in the first place a big document ([JSON](http://json.org)/[BSON](http://bsonspec.org)) store. Additionally MongoDB offers facilities to index and search the collection of documents.

[](http://bsonspec.org)

[In conjunction with](http://bsonspec.org) [Clojure](http://clojure.org) your are able to build a simple a quite powerful system to use event sourcing and CQRS.[]()[]()

# Creating events

Okay, lets start with the events: How to model events in Clojure? Basically there are two good choices, the first one is the Clojure standard namely maps. The second choice are records with protocols (e.g. `defrecord` and`defprotocol) which are a little more turbocharged. So the protocol for our event may look like this:

{% highlight clojure %}
(defprotocol EventProtocol
  (id [this])
  (type [this])
  (created [this])
  (aggregate-id [this])
  (aggregate-version [this]))
{% endhighlight %}

We have:

- an ID to identify the event.
- an event type (e.g. `:user-created`).</li>`
- `a timestamp when the event was created/occured. Notice that the event does not have a modification timestamp, because we do not modify events.</li>`
- `an ID of the aggregate where the events is related to.</li>`
- `a version identifier which tells us which aggregate version/modification this event represents.



Lets have a quick view on the theoretical part. What is the goal? We want to represent the state the application or of an (aggregate) entity to represented as a stream of events, that is was [event sourcing](http://martinfowler.com/eaaDev/EventSourcing.html) is all about. The work on event sourcing and CQRS is often closely tied to the [domain driven design (DDD)](http://en.wikipedia.org/wiki/Domain-Driven_Design) ideas, so you often find references to [aggregates](http://en.wikipedia.org/wiki/Domain-Driven_Design) or [aggregate roots. Aggregates are one of the building blocks of DDD and are similar to entities.

So if you take a look at the defined protocol you will notice, that we have a minimal set of properties defined for a event. The relation to the aggregate is defined by the `aggregate-id` and we are able to order the events by the`aggregate-version.

To bring the events in Clojure to life we define a record like this:

{% highlight clojure %}
[(defrecord Event [_id _aid _t _c _v]
  EventProtocol
  (id [_] _id)
  (type [_] _t)
  (created [_] _c)
  (aggregate-id [_] _aid)
  (aggregate-version [_] _v))
{% endhighlight %}

The first question may be: Why these ugly names (like `_id`,`_aid`, `_t`,`_c` or `_v`)? I already mentioned MongoDB and this relates to my target implementation on MongoDB (I will explain the idea behind this a little later) and to be honest: This source code relates to my blog entry and I am not planning to create a generic event sourcing library for Clojure, so I decided to make my life a little easier here.`````

To create new events we may define a function like this:

{% highlight clojure %}
[(defn new-event
  ([type aggregate-id] (new-event type aggregate-id nil))
  ([type aggregate-id payload]
    (map->Event (assoc payload
                  :_aid aggregate-id
                  :_t type
                  :_c (now)))))
{% endhighlight %}

To create a new event like the creation of a new user we may call:

{% highlight clojure %}
[(clj-eventsourcing.event/new-event :user-created "5050480d3004e17b14ecb3fe" {:user-name "niclas"})
{% endhighlight %}

and receive this record:

{% highlight clojure %}
#clj_eventsourcing.event.Event{:_id nil, :_aid "5050480d3004e17b14ecb3fe", :_t :user-created, :_c #<datetime 2012-09-12T08:34:36.331Z>, :_v nil, :user-name "niclas"}
</datetime>
{% endhighlight %}

You see that the payload `{:user-name "niclas"}` is attached to the record even when the field was not defined in the`defrecord but if you check:

{% highlight clojure %}
[(instance? Event (clj-eventsourcing.event/new-event :user-created "5050480d3004e17b14ecb3fe" {:user-name "niclas"}))
{% endhighlight %}

As you can see, this evaluates to `true`, so the instance returned is still anEvent.

# Storing events

Creating events is nice, but if the events stay ephemeral they are of no use to your application. To store events we may define this preliminary protocol.

{% highlight clojure %}
[(defprotocol EventStore
  (version-from [this aggregate-id])
  (store-events-into [this version events])
  (load-events-from [this aggregate-id]))
{% endhighlight %}

and a very simplistic implementor for MongoDB

{% highlight clojure %}
[(deftype MongoStore [collection]
  clj_eventsourcing.store.EventStore
  (version-from [_ aggregate-id] (version collection aggregate-id))
  (store-events-into [_ version events] (store collection version events))
  (load-events-from [_ aggregate-id] (load collection aggregate-id))
  java.lang.Object
  (toString [this] (str "<#MongoStore: " collection ">")))
{% endhighlight %}

which will dispatch the calls to the protocol to dedicated functions merging his internal state (the MongoDB collection to use) with the call parameter.

Let's have a quick look at the simplest of the three functions which should be `version-from`, unfortunately this leads to some basics first.`

#### Versions and optimistic locking

One important part of storing an event is to assign a version to the event to create an ordering. This is very important when you want to project events into a data structure. If you do not know is a the value of a field was first 'A' then 'B' or vice versa is bad.

How do we now assign versions? Unfortunately this is not an easy task in a distributed environment with only a MongoDB. If you use a SQL database you may use a sequence, `SELECT … FOR UPDATE` clause or an auto-increment field, but the MongoDB does not provide a similar mechanism.`

For this example I choose a quite simple strategy. As version I used (positive) integers starting at one. When inserting events I will fetch the current version out of the database and assign new versions to the events by simply increasing the version (by one). Additionally I create a unique index for the events collection containing the aggregate ID (`_aid`) and the version (`:_v)` field. If e.g. another process is doing the same in parallel we will have overlapping versions and MongoDB will raise an error when the second one tries to insert the event(s).

Using this simple strategy we have an optimistic locking, where no node need to lock down the whole collection (or even worse database) to make sure that no one else updated the data while he was working on them. We can take this even one step further by supplying the version of the aggregate in use back to the `(store-events-into ...)` function of the protocol.

You can find a simple realization of the `version-from` function on my [github repository.](https://github.com/niclasmeier/clj-eventsourcing/blob/master/src/clj_eventsourcing/store/mongo.clj)

## Inserting into MongoDB

The insertion of the events is now very simple. Provided we have a function that adds the version to the events you will have something like:](<https://github.com/niclasmeier/clj-eventsourcing/blob/master/src/clj_eventsourcing/store/mongo.clj>)

{% highlight clojure %}
[(defn store [collection version events]
  (insert-batch collection (enrich-events version events))
{% endhighlight %}

The only problem is that you have no safety net on this one. You want to check some constraints like:

* If the version is only a number, you should make sure that all events refer to the same aggregate
* If the version is a map from aggregate ID to a number, you should make sure that you have versions for all aggregates referred by the events.
* If an Exception about violating the unique index on the MongoDB collection is thrown, it might be a good idea converting it into something more useful (e.g. a different exception type)



# Loading events

Loading events on the other hand side is very simple, now we known how the documents should look like:

{% highlight clojure %}
[(defn load [collection aggregate-id]
  (with-collection collection
    (where {:_aid aggregate-id})
    (sort {:_v 1})))
{% endhighlight %}

Simply fetching all event documents from the MongoDB ordered by the version.

# Putting it together

Creating, loading and storing events with Clojure and storing them into MongoDB is not that hard and requires very lithe source code. You may use Protocols and Records to get a little bit more structure into this, but the initial event sourcing system used at HEROLABS was relying on Clojure maps and works fine.

Unfortunately we are far from done yet. Working on event streams is harder than fetching data from a business "object" (they will be maps or Records in Clojure). So the next step will be to explore how to project the events into data structures or operations, so that

{% highlight clojure %}
[[{:_aid 1 :_v 1 :_t :set-name :first-name "Niclas" :last-name "Meier"}
{:_aid 1 :_v 2 :_t :add-blog :url "http://www.niclas-meier.de/"}]
{% endhighlight %}

will get

{% highlight clojure %}
{:_aid 1 :_v 2 :first-name "Niclas" :last-name "Meier" :url "http://www.niclas-meier.de/"}
{% endhighlight %}

or at least something like this, so stay tuned for part two. In the meantime you may check out some more source code at my [github repository](https://github.com/niclasmeier/clj-eventsourcing), but be aware that this is only an implementation sketch and not fully implemented.
