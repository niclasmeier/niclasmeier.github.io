---
layout: post
status: publish
published: true
title: Underestimated Amazon S3
author:
  display_name: Niclas Meier
  login: niclas
  email: mail@niclas-meier.de
  url: http://www.niclas-meier.de
author_login: niclas
author_email: mail@niclas-meier.de
author_url: http://www.niclas-meier.de
excerpt: "The <a href=\"http://aws.amazon.com/de/s3/\">Amazon
  Simple Storage Service (S3)</a> is the big data work horse of Amazon and one
  of the corner stones of their cloud infrastructure. For our recent <a href=\"http://itunes.apple.com/de/app/em-hero-2012/id530513702?l=de&amp;ls=1&amp;mt=8\">project</a>
  we received a considerable amount of game data from our data provider. At the moment
  these file are stored in the <a href=\"http://www.mongodb.org/display/DOCS/GridFS\">GridFS</a>
  of our Mongo DB installation, but once the data is processed we don't really need
  these files any more, except for development or archiving purposes.\r\n\r\nAt the
  moment we are using the <a href=\"http://aws.amazon.com/en/ec2/\">Amazon
  Elastic Compute Cloud (EC2)</a> to run our infrastructure, so it was a logical
  choice to evaluate the <a href=\"http://aws.amazon.com/en/s3/\">S3</a>
  to store the received game data and I have to confess I was really underestimating
  this service as simple file system."
wordpress_id: 177
wordpress_url: http://www.niclas-meier.de/?p=177
date: '2012-07-10 21:02:49 +0200'
date_gmt: '2012-07-10 20:02:49 +0200'
categories:
- Clojure
- Amazon S3
tags: []
comments: []
---
The [Amazon Simple Storage Service (S3)](http://aws.amazon.com/de/s3/) is the big data work horse of Amazon and one of the corner stones of their cloud infrastructure. For our recent [project](http://itunes.apple.com/de/app/em-hero-2012/id530513702?l=de&ls=1&mt=8) we received a considerable amount of game data from our data provider. At the moment these file are stored in the [GridFS](http://www.mongodb.org/display/DOCS/GridFS) of our Mongo DB installation, but once the data is processed we don't really need these files any more, except for development or archiving purposes.

At the moment we are using the [Amazon Elastic Compute Cloud (EC2)](http://aws.amazon.com/en/ec2/) to run our infrastructure, so it was a logical choice to evaluate the [S3](http://aws.amazon.com/en/s3/) to store the received game data and I have to confess I was really underestimating this service as simple file system.<a id="more"></a><a id="more-177"></a>

### Reliability

First of all, I wasn't really aware that S3 has this high reliability of 99.999999999 percent according to the Amazon website. Which is more than enough for our needs. An guaranteed uptime of 99.99 percent is equivalent to max 52.56 minutes of downtime per year, but I am sure that we can't / don't want to build a system which offers equal values. You can also lower the cost of you S3 usage by choosing the reduced redundancy service which reduces the reliability of 99.99 percent, which is good enough e.g. for [elastic map reduce](http://aws.amazon.com/en/elasticmapreduce/) usage.

To ensure a safe transmission of the data you can also specify a MD5 sum of the content you are storing into S3 to make sure that you have no errors while transferring the data.

### Expiration

A very nice feature of S3 is that you can set expiration dates or rules for objects. E.g. we receive commentary data from out sports data provider to enrich our content. This content is nice but also automatically generated so we don't really need to store it. In S3 we have now the possibility to delete these files automatically after 90 days (or something like this).

### Meta-data

You may associate meta data as key value pairs to any S3 object you store. These meta data is also transferred as HTTP header with `x-amz-meta-` prefix. So there is no need for additional `.md5` and `.meta` files. We will use this to store some game and data transfer related meta data to the files. The data transfer meta data is a map of several keys and values. As a lithe trick I am storing this additional data as JSON object.

### Versioning

This is a big one. You can configure the S3 buckets - which are a kind of root file system on S3 - to store the files versioned. So you are able to get any previous version of an object like on [git](http://git-scm.com/) or [Subversion](http://subversion.apache.org/).

This is perfect for our needs. We receive for data files like `commentary-3-2011-426847-de.xml` which contain all comments for a football (european not american) game. We are receiving a file with this name a couple of times during the game containing the complete data (updates and old data).

Using GridFS and [MongoDB](http://www.mongodb.org) we are already keeping multiple versions of the file and it's great to see, that we are able to store multiple versions in S3 too without auxiliary arrangements.

### Java SDK and Clojure

At the moment we are using the Java SDK from Amazon with a slim Clojure wrapper. This works very well and you can easily wrap the S3 calls into a couple of Clojure functions like this:

{% highlight clojure %}
(defn create-folder [client bucket-name key ]
  "Creates an S3 folder"
  (when (and client bucket-name key)
    (let [req (PutObjectRequest. bucket-name (if (.endsWith key folder-extension) key (str key folder-extension)) empty-content empty-content-meta)
          req (if acl (.withCannedAcl req acl) req)
          req (if listener (.withProgressListener req listener) req)]
      (.putObject client req))))
{% endhighlight %}

Just create request instance (e.g. `PutObjectRequest`) and chain the continuation style `.with` Methods. The named arguments are a big help here. A little more work is to convert the result objects of the S3 API into Clojure style datatypes.

{% highlight clojure %}
(defn list-objects [client bucket-name ]
  "Lists the objects in a bucket."
  (when (and client bucket-name)
    (let [r (.listObjects client (ListObjectsRequest. bucket-name prefix marker delimiter max-keys))]
      (letfn [(summaries [s] (lazy-seq (if-let [n (first s)] (cons (coerce-s3-object-summary n) (summaries (rest s))) nil)))]
        (-&gt; (transient {})
          (assoc-if! :bucket-name (.getBucketName r))
          (assoc-if! :prefix (.getPrefix r))
          (assoc-if! :marker (.getMarker r))
          (assoc-if! :max-keys (.getMaxKeys r))
          (assoc-if! :delimiter (.getDelimiter r))
          (assoc-if! :truncated (.isTruncated r))
          (assoc-if! :common-prefixes (when-let [p (.getCommonPrefixes r)] (when-not (empty? p) p)))
          (assoc-if! :next-marker (.getNextMarker r))
          (assoc-if!&nbsp;:object-summaries (when-let [s (.getObjectSummaries r)] (summaries s)))
          persistent!)))))
{% endhighlight %}

But most of it is just repetitive work, but in the your Clojure code this looks really nice.

{% highlight clojure %}
(list-objcts c "my-fancy-bucket" :prefix "foo")
{% endhighlight %}

### Conculsions

I didn't talk about some matters of course like access control, multiple APIs based on REST or SOAP available in Java, .NET or Ruby. Or other nice functions like uploading data as anonymous user on pre-signed URLs. But from my perspective this package is more than enough to keep a close look on the [Simple Storage Service](http://aws.amazon.com/de/s3/). I am pretty sure that the [equivalent Google Service](https://developers.google.com/drive/) has similar capabilities, so both services should be considered if you want to store some of your data in the cloud.

Another important question, many of the european/german users will ask: "Where will my data be stored?" - Amazon will guarantee that, if you use the EU-Ireland zone the data will remain in the EU. This sounds great, even from the data protection point of view, but remember the fact that the european Amazon (Amazon EU S.à.r.l) is a owned to hundred percent by the US Amazon. So if the Department of Homeland Security want's a DVD with all of you data, I am pretty confident that they will get it.

In the end, I am more than ever convinced that S3 is a very useful tool and I am exited to continue to work with it.
