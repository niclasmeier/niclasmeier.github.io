---
layout: post
status: publish
published: true
title: The curl cheat sheet to myself
categories:
- Technology
- Software development
- REST
tags: []

---
It has been not even a full week building REST style interfaces for the [Playmaker Studio](http://www.playmakerstudio.com/) mobile platform and I am very annoyed of all the `curl` guides how to access your REST API with `curl`. So here is my cheat sheet on this topic. Be aware that the I will update this frequently.

## Common use cases

### GETting data from a URL

This is a very simple use case just use:

{% highlight bash %}
curl https://graph.facebook.com/740999649
{"id":"740999649","name":"Niclas Meier"}
{% endhighlight %}

`curl` will automatically use the HTTP method `GET`.

### POSTting updates

In the REST dictionary the HTTP method `POST` represents the verb 'update'. So updating data implies a `POST` request to an URL.

#### Basic

Let's have a look at the basics, I added some (revised / half the original) debugging output by supplying the `-v` option.

{% highlight bash %}
curl -v -X POST http://localhost:8080/1/stuff/23f96c6bd4c87910c.json -d 'name=Niclas'

> POST /1/stuff/23f96c6bd4c87910c.json HTTP/1.1
> Host: localhost:8080
> Accept: */*
> Content-Length: 11
> Content-Type: application/x-www-form-urlencoded
>
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=utf-8
< Content-Length: 25
<
Stuff: 23f96c6bd4c87910c

{% endhighlight %}

There are some options involved:

*   `-v` added the request/response header for debugging
*   `-X` specified the request method (`POST`) to use.
*   `-d` added some data in the request (`POST`) body.

So the `'name=Niclas'` was sent to the server using the request body, but `curl` does not display the request body using the `-v` option.

#### Using a JSON body

So this is nice, when you want to post key-value pairs to your server, but many applications (e.g. [CouchDB](http://couchdb.apache.org/) want to post JSON objects to the server. The initial guess

{% highlight bash %}
curl -v -X POST http://localhost:8080/1/stuff/23f96c6bd4c87910c.json -d '{"name":"Niclas"}'

Stuff: 23f96c6bd4c87910c, JSON body:

{% endhighlight %}

does not work. That's where the content type comes in. If you post the JSON whiteout supplying the proper content type `curl` and/or the server will try to use `application/x-www-form-urlencoded`[<sup>*</sup>](http://www.w3.org/TR/html4/interact/forms.html) content type. This is the same content type your browser uses when submitting a form to a web server. Doesn't sound bad, does it?

But it's bad because the JSON will be ignored/dropped/whatever and the server is unable to understand the request. To solve the problem use:

{% highlight bash %}
curl -H "Content-Type: application/json" -X POST  http://localhost:8080/1/stuff/23f96c6bd4c87910c.json -d '{"name":"Niclas"}'

Stuff: 23f96c6bd4c87910c, JSON body: {"name":"Niclas"}

{% endhighlight %}

So these options are involved

*   `-X` specified the request method (`POST`) to use.
*   `-H` added the header `Content-Type: application/json` to the request.
*   `-d` defined the data used for the request body (`{"name":"Niclas"}`).

#### A JSON file as body

So far so good. But if you have huge JSON requests, you don't want to type all the JSON into the console. You may use the `@` symbol in conjunction with the `-d` option so `curl` will read the request body contents from a file. This may look like this.

{% highlight bash %}
curl -H "Content-Type: application/json" -X POST  http://localhost:8080/1/stuff/23f96c6bd4c87910c.json -d @some-huge-request.json

Stuff: 23f96c6bd4c87910c, JSON body: {"name":"Niclas", ..., "foo":"Bar"}

{% endhighlight %}

A nice trick is, that if you use the `-d` with `@-`. `curl` will read the request body from the standard input. So you can do stuff like this:

{% highlight bash %}
cat some-huge-request.json | curl -H "Content-Type: application/json" -X POST  http://localhost:8080/1/stuff/23f96c6bd4c87910c.json -d @-

Stuff: 23f96c6bd4c87910c, JSON body: {"name":"Niclas", ..., "foo":"Bar"}

{% endhighlight %}

### PUTting stuff

Is much like `POST`, so if you can `POST` you can `PUT`. One nasty little difference is, that you don't have to supply a `Content-Type` HTTP header for `PUT`. `curl` does not use `application/x-www-form-urlencoded` as content type for `PUT`. Lost an hour (or so), because I did some `PUT` stuff before switching to `POST`.

## Debugging

#### Detailed information

The key to get almost every information by supplying the `-v` option to curl.

{% highlight bash %}
curl -v https://graph.facebook.com/740999649

* About to connect() to graph.facebook.com port 443 (#0)
*   Trying 66.220.147.38... connected
* Connected to graph.facebook.com (66.220.147.38) port 443 (#0)
* SSLv3, TLS handshake, Client hello (1):
* SSLv3, TLS handshake, Server hello (2):
* SSLv3, TLS handshake, CERT (11):
* SSLv3, TLS handshake, Server finished (14):
* SSLv3, TLS handshake, Client key exchange (16):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSL connection using RC4-MD5
* Server certificate:
* 	 subject: C=US; ST=California; L=Palo Alto; O=Facebook, Inc.; CN=*.facebook.com
* 	 start date: 2010-01-13 00:00:00 GMT
* 	 expire date: 2013-04-11 23:59:59 GMT
* 	 common name: *.facebook.com (matched)
* 	 issuer: C=US; O=DigiCert Inc; OU=www.digicert.com; CN=DigiCert High Assurance CA-3
* 	 SSL certificate verify ok.
> GET /740999649 HTTP/1.1
> User-Agent: curl/7.21.4 (universal-apple-darwin11.0) libcurl/7.21.4 OpenSSL/0.9.8r zlib/1.2.5
> Host: graph.facebook.com
> Accept: */*
>
< HTTP/1.1 200 OK
< Access-Control-Allow-Origin: *
< Cache-Control: private, no-cache, no-store, must-revalidate
< Content-Type: text/javascript; charset=UTF-8
< ETag: "f89bfe40c87e487ae641d427c96746b226a91f3c"
< Expires: Sat, 01 Jan 2000 00:00:00 GMT
< Pragma: no-cache
< X-FB-Rev: 482216
< X-FB-Server: 10.36.50.102
< X-Cnection: close
< Date: Wed, 07 Dec 2011 13:14:19 GMT
< Content-Length: 183
<
* Connection #0 to host graph.facebook.com left intact
* Closing connection #0
* SSLv3, TLS alert, Client hello (1):
{"id":"740999649","name":"Niclas Meier","first_name":"Niclas","last_name":"Meier","link":"http:\/\/www.facebook.com\/people\/Niclas-Meier\/740999649","gender":"male","locale":"de_DE"}

{% endhighlight %}

But for may occasions that is simply too much, so check out the `-i` option.

#### Header information

Often it's enough to get only the response headers for debugging. To get only these headers just use the `-i` option in `curl`.

{% highlight bash %}
curl -i https://graph.facebook.com/740999649
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Cache-Control: private, no-cache, no-store, must-revalidate
Content-Type: text/javascript; charset=UTF-8
ETag: "f89bfe40c87e487ae641d427c96746b226a91f3c"
Expires: Sat, 01 Jan 2000 00:00:00 GMT
Pragma: no-cache
X-FB-Rev: 482216
X-FB-Server: 10.42.67.29
X-Cnection: close
Date: Wed, 07 Dec 2011 13:18:24 GMT
Content-Length: 183

{"id":"740999649","name":"Niclas Meier","first_name":"Niclas","last_name":"Meier","link":"http:\/\/www.facebook.com\/people\/Niclas-Meier\/740999649","gender":"male","locale":"de_DE"}

{% endhighlight %}
