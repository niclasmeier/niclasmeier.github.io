---
layout: post
status: publish
published: true
title: Bootstrapping EC2 servers with Puppet
excerpt: >-
  At <a href="http://herolabs.com">HEROLABS</a> we are doing sports / gaming apps
  and one inherent characteristic is, that you have load peaks around the games.
  I guess having load peaks is characteristic to for most gaming apps. If we take
  a look at your EMHERO2012 app we had most server traffic during the time the
  German national Football team played and almost no traffic in between.



  What we wanted to do: Scale up our server capacity around periods of increased
  load and scale down the rest of the time. To achieve this, we chose the Amazon
  (AWS) cloud as deployment environment and an automated server deployment
  process. To shed some light on the result already, after some a hard work and
  steep learning curve, it works ;-)
wordpress_id: 185
wordpress_url: 'http://www.niclas-meier.de/?p=185'
date: '2013-02-10 20:28:26 +0100'
date_gmt: '2013-02-10 19:28:26 +0100'
categories:
  - Technology
  - Clojure
  - Amazon EC2
  - Amazon AWS
  - Puppet
---

At HEROLABS we are doing sports / gaming apps and one inherent characteristic is, that you have load peaks around the games. I guess having load peaks is characteristic to for most gaming apps. If we take a look at your EMHERO2012 app we had most server traffic during the time the German national Football team played and almost no traffic in between. What we wanted to do: Scale up our server capacity around periods of increased load and scale down the rest of the time. To achieve this, we chose the Amazon (AWS) cloud as deployment environment and an automated server deployment process. To shed some light on the result already, after some a hard work and steep learning curve, it works ;-)

Okay here is how we did it: The first thing you need is a (metaphorically) unlimited amount of servers to rent, which should be available in a very short time (minutes). This is the job of the EC2 cloud in our scenario. That is the easy part, if you have a credit card. Now you have rented an almost blank server, except of the operating system. We chose Ubuntu (12.04 LTS - Precise Pangolin) as server operating system which is quite well documented and maintained for use in the EC2 cloud. The hard parts are now:

- automate the installation process of the software you need
- add the right configuration files to your server
- add the system to your cluster

At the moment there are several products/projects for deployment automation / configuration management. The most known are [Chef](http://www.opscode.com/chef) and [Puppet]:<https://puppetlabs.com/what-is-puppet> from [Puppetlabs](https://puppetlabs.com/). Due to the fact that we are using [Clojure](http://clojure.org) we first started with [Pallet](http://palletops.com), but we changed to Puppet this year and it started to work out well for us.

In general you can divide these systems in two groups. One group is like a SSH shell on steroids: You script what needs to be done on the server and the tools does it in parallel on all the hosts you want. The other group defines the state you want the servers in and the the transition steps you need to do. And every node tries to to get into the state you defined. Chef and Pallet are more fitting into the first group and Puppet into the second.

The first step for me was to define the state and transitions for our backend server installation. To run our backend server we need Java, Leiningen and our Software package (cloned from [GitHub](https://github.com)).

A nice about thing on Puppet is, that you don't need any special programming skills. You define classes in so called manifest files which define the dependencies and actions you want to take. If you understand how Puppet works you get really fast into it. E.g. if you want Java installed you simply say:

{% highlight puppet %} package {'openjdk-7-jdk': ensure => present, } {% endhighlight %}

and Puppet will install the JDK on the next run.

Maybe a quick word how Puppet works. Puppet has two major part: A Puppetmaster, which is a central server with all state information, and a Puppet Agent which runs on each node. The Puppet Agent runs in a fixed interval and compares the state from the Puppetmaster with the local state. If these state are not identical it will try to adjust the local state to match the required state.

To create the Manifests was not very hard. We had some special issues with some SSH keys, GitHub and the local non-privileged user. But this was more about my lack of knowledge in these things. But I learned a lot (e.g. to write [Upstart](http://upstart.ubuntu.com) scripts) about creating a server environment. I guess it's always hard when you know what you should do, but you don't known how to get there. But let me assure you that I took me only 2-3 days to create the automatic setup of our Clojure backend server (including setup of a Puppetmaster, etc.), starting by near zero.

One major obstacle was how Puppet defines the state of a server. Normally you find a manifest file on the Puppetmaster (e.g. `site.pp`) which defines the classes a server should contain. This looks like:

{% highlight puppet %}

# /etc/puppetlabs/puppet/manifests/site.pp

node 'www1.example.com' { include common include apache include squid

} {% endhighlight %}

Nice for a small website, not so good in a cloud, when you don't really know the names of your server. But Puppet has a great mechanism to solve our problems. It's called [External Node Classifiers (ENC)](http://docs.puppetlabs.com/guides/external_nodes.html). How does this work? Instead of using the manifest data Puppet called an external program which returns the node classification as YAML. Great and simple solution.

After some googling I stumbled across a nice [blog post](http://engineering.gowalla.com/2011/10/21/puppet/) by [Kevin Lord](https://github.com/lordkev) on the Gowalla engineering blog, which helped a lot. So I decided to write my own ENC which you can find [here](https://github.com/niclasmeier/puppet-ec2-enc). It's basically the same solution, but my version takes more than one security group into account and also merges the EC2 tags into the `parameter` section of the result. Which comes handy e.g. when you want to activate special services only on one server.

So we are now almost done on the automatic installation part. So when we start Puppet on a new host the Puppetmaster will figure out this is a productive backend server and install all the software we need on the on the server. Unfortunately there a no standard AMIs with Puppet pre-installed and I sure don't want any standard AMI with our Puppetmaster address too. A first idea was to create an AMI with Puppet and our Puppetmaster pre-installed, but for an initial version we wanted to be able to use standard AMI.

The solution was a so-called [`user-data` script](http://alestic.com/2009/06/ec2-user-data-scripts). This is a mechanism that works on the EC2 Ubuntu images. The `user-data` script is a script which executed on instance startup and can be passed as a parameter on the Amazon EC2 API call.

With a suitable `user-data` script starting new instances looks like this:

{% highlight bash %} ec2-run-instances -k my-ec2-key -n 1 --user-data-file [install-puppet.sh](https://github.com/niclasmeier/puppet-ec2-enc/blob/master/install-puppet.sh) -g backend-productive -t m1.small ami-960f03e2 {% endhighlight %}

Our script does nothing more than adding the Puppetlabs `apt` repository, installs the newest version of Puppet and writes a very basic `puppet.conf` file. Fortunately Puppet is also managed by Puppet, so the `puppet.conf` will be adjusted later. The `-g backend-productive` selects the security group of the server and determines what to install.

As a matter of fact we don't use the command line to start servers. Our admin tool as a button to start servers and uses the Amazon EC2 API directly to spawn new instances if needed.

Having a server installed is great, but worth nothing if you don't get any traffic on the system. For our new product we use the Amazon [elastic load balancer](http://aws.amazon.com/en/elasticloadbalancing/) (ELB). This makes the task somewhat easy.

The solution I chose was to create a [small script](https://github.com/niclasmeier/puppet-ec2-enc/blob/master/register_at_elb.rb) that registers the instance to the ELB (using the ELB web service interface).

This script is part of the installation manifest for the backend server and a simple `exec`:

{% highlight puppet %} exec {'register_at_api_elb': command => "${api::params::scripts_dir}/register_at_elb.rb ${ec2_instance_id} ${loadbalancer_name} && touch /var/run/register_at_api_elb.lock", creates => '/var/run/register_at_api_elb.lock', logoutput => on_failure, require => Class['api::install'], } {% endhighlight %}

To prevent the multiple registration the script is succeeded by a touch on a lock file. The `exec` will only performed by Puppet only if the lock file does not exists.

Some closing words: I love DevOps. Programming on your infrastructure is nice and opens a lot of opportunities. In my former jobs I worked with some great SysAdmins. This on hone hand helps very much, but can be a burden sometimes. Especially when you don't known enough about the tools you want/have to use.

To give you a better perspective. All our servers use Puppet right now, but some of them (e.g. the databases) are not able to autoscale (yet ;-) ).
