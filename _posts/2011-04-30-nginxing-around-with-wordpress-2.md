---
layout: post
status: publish
published: true
title: nginxing around with WordPress
categories:
- Webserver
- nginx
- Wordpress

---
Some of you may already have noticed, that this blog uses [Wordpress](http://www.wordpress.com) as blog software. Nothing special about that I guess. As I mentioned before in January/February I moved my website from my old provide to a new one making some significant changes to the blog and it's infrastructure.

I am in the web business since a couple of years and I already worked with a couple of servers. Like most of the others I worked most with the [Apache web server](http://httpd.apache.org). Which is still the work horse of the internet. But recently the [nginx](http://nginx.org/) web server which is a lean, mean web serving machine. To get a little more experience with this product I decided to use it for my blog. The [nginx](http://nginx.org/) web server already matured already 9 years and reached recently (12\. April 2011) the version 1.0.0 (congratulations on this by the way ;-) ). But there are still some major differences between the [Apache](http://httpd.apache.org) and the [nginx](http://nginx.org/) web servers.For me this started when I wanted to use [Wordpress](http://www.wordpress.com) which is build upon [PHP](http://www.php.net). [PHP](http://www.php.net) itself depends on the [common gateway interface](http://en.wikipedia.org/wiki/Common_Gateway_Interface) (a.k.a. [CGI](http://en.wikipedia.org/wiki/Common_Gateway_Interface)) which is a [long time component](http://httpd.apache.org/docs/2.0/mod/mod_cgi.html) of the Apache web server. Unfortunately the nginx server does not support [CGI](http://en.wikipedia.org/wiki/Common_Gateway_Interface) so you'll have cover some more basics.

nginx itself supports CGI using the [fast CGI](http://en.wikipedia.org/wiki/FastCGI) standard. Which does actually a little bit more the plan CGI. To serve fast CGI you may use the [nginx fast CGI module](http://wiki.nginx.org/HttpFcgiModule), so you still have access to all the scripting languages you like. But be aware that the nginx web server is known for it's fabulous capabilities serving static content.

After installing nginx, a fast CGI wrapper app to execute the PHP scripts and configuring the fast CGI module, the setup of the Wordpress blog was pretty basic. I'm using a standard [MySQL](http://www.mysql.com) database backing the Wordpress. The whole system runs on a [CentOS](http://www.centos.org/) linux. The choice of the linux distribution was also inspired by the preferred system I use at my job for almost two years. Some times it's good to stick to things you know ;-)

This was the status on which the whole system ran since February. But as a few - regarding to google analytics - of you may have noticed that the whole system was very slow. This is a direct result of my stinginess, because I don't want to pay more money for my root server ;-)

So today I invested some spare time to look for other optimizations and I would like to say, that I was successful. My first search lead to a Wordpress plugin to cache the pre-rendered Wordpress pages. I have not this much posting and commenting traffic, so pre-rendering the pages seemed to be a good idea. The plugin of my choice is [WP Super Cache](http://wordpress.org/extend/plugins/wp-super-cache/) which provides a very easy to configure caching of Wordpress pages. But I wanted to go one step further: The [WP Super Cache](http://wordpress.org/extend/plugins/wp-super-cache/) plugin offer the possibility to bypass Wordpress and PHP completely by using compressed pre-rendered pages via [mod_rewrite](http://httpd.apache.org/docs/2.0/mod/mod_rewrite.html). I guess most of you noticed the problem: [mod_rewrite](http://httpd.apache.org/docs/2.0/mod/mod_rewrite.html) is an Apache module, which does not work with the nginx server, but nginx provides a powerful [rewriting module](http://wiki.nginx.org/HttpRewriteModule) too. A very helpful [blog post](http://ocaoimh.ie/2009/11/23/wordpress-nginx-wp-super-cache/) from the WP Super Cache author helped to configure the web server. And now - at least for any anonymous user - the site is fast.

It do no longer take about two seconds to return the the first bytes to the user and the page itself is compressed only approximately five kilobytes large. To speed up the perceived loading time of the page, I am using the [Use Google Libraries](http://wordpress.org/extend/plugins/use-google-libraries/) plugin pass the delivery of standard Java Script libraries to google. So the libraries can participate on sharing between different pages and the long time expiry which is used by [google](http://www.google.com). The plugin [WP Minify](http://wordpress.org/extend/plugins/wp-minify/) joins and compresses the JS and CSS used by the theme and other plugins I use, so the browser does download only a few minimized files instead of dozens of CSS and JS files.

I can say, that the experiment was quite successful and after getting familiar with some peculiarities the setup process was quite easy.
