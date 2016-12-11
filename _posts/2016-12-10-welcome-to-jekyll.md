---
layout: post
status: publish
published: true
title:  "Welcome to Jekyll!"
date:   2016-12-10 10:06:50 +0100
categories:
  - Private
---
Look like the Jekyll default post, but it's a little more I guess. After more than three years I found some time update me digitial infrastructure to a more serverless way.

Two weeks ago Amazon had their [re:Invent](https://reinvent.awsevents.com) conference in Las Vegas and I was watching the keynote and some sessions with quite some interest. After launching/anncouning [AWS Lambda](https://aws.amazon.com/de/lambda/) in 2014 serverless computing is getting more and more a thing and this years re:Invent made it more and more clear to me. So more about time to get rid of my old vServer and to move forward to a serverless solution ...

Okay, let's be honest: My email account has long moved to a public email provider and is no longer hosted on my own server. As a matter of fact it never was, the sole purpose of Postfix was to provide [email greylisting](https://en.wikipedia.org/wiki/Greylisting) for improved SPAM protection. It worked quite well for a while, but my email provider can handle it very well own his own. The secondary reason to build an email facade was, that it was fun and I wanted to try it ...

Leaving me and my [Wordpress](https://wordpress.org) blog on the vServer. I already looked at some offers like [Blogger](https://www.blogger.com/) or [Wordpress.com](https://www.wordpress.com), but I wanted my domain to come along. But I also didn't want to migrate a MySQL database, web server and Wordpress to some different provider.

Then, last week I tried [Jekyll](https://jekyllrb.com) and was convinced. No more need for a database and stuff, just generate the pages. I am using Wikis for quite some year now and [github.com](http://github.com), I already have some stuff there, so free hosting!

The most complex part was migrating the data out of the Wordpress MySQL, seems I misplaced the root password for the instance running the Wordpress :-), but after migration the dump on my local compute the [jekyll-import](https://import.jekyllrb.com/docs/wordpress/) work reasonably well and getting all the ugly HTML to [Markdown](https://en.wikipedia.org/wiki/Markdown) was an easy job.

So, let's see how this works out, and ... "Welcome to Jekyll!"
