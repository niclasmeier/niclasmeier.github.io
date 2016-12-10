---
layout: post
status: publish
published: true
title: Suddenly somewhat famous
categories:
- Privat
- Technology
---
Thanks to a former colleague, I received word that my post about my first [100 days of working with Clojure](/2012/04/first-hundred-days-of-clojure/) scored Number 3\. on the [Hacker News](http://news.ycombinator.com/).

I just wanted to thank all of you for the interest and I really hope my english is not that worse. Thanks to some attentive readers, I where able to fix some glitches in some of the examples or the spelling.

Due to fortunate circumstances I had some time to setup [my blog on a new platform](/2011/04/nginxing-around-with-wordpress-2/) almost exactly a year ago. Now using [Nginx](http://wiki.nginx.org) with [Wordpress](http://wordpress.org/). Witch performed remarkably well.<a id="more"></a><a id="more-152"></a>If your are interested here are some stats:

![Localhost nginx status day](/assets/localhost-nginx_status-day.png)

Starting with about 200 concurrent connections to the my Nging web server, which are 66 times the average over the last year. That's nice because I saw some setups which would have died due to the load. Thanks to [the guy who introduced me into Nginx](https://www.xing.com/profile/Lukas_Loesche2). We were using it at [scoyo](www.scoyo.com) and [gamigo](http://www.gamigo.com) and this proved that it was a good decision - if I had any doubts left.

As a matter of fact, that worked so well, that it didn't even had an impact on the CPU load: ![Localhost load day](/assets/localhost-load-day.png) ![Localhost load month](/assets/localhost-load-month1.png)

Or even the memory usage:

![Localhost memory day](/assets/localhost-memory-day.png)

I guess making all the content static using [WP SuperCache](http://wordpress.org/extend/plugins/wp-super-cache/) and the [HTTP rewrite module](http://wiki.nginx.org/HttpRewriteModule) was a good idea.
