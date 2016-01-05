---
layout: post
title: Vagrant - An unexpected error occurred while loading the vagrant-login plugin
description: "**TL;DR**: If you've encountered this error in Vagrant, reinstall all your plugins."
---

**TL;DR** If you've encountered this error in Vagrant, reinstall all your plugins.

Today after updating my Arch Linux (which I wasn't doing for a long time) I had a problem with vagrant. After trying to `vagrant up` my machine I got following output:

{% highlight bash %}
An unexpected error occurred while loading the vagrant-login
plugin. Please contact support with the following
error code: '7'.
Vagrant failed to initialize at a very early stage:

The plugins failed to load properly. The error message given is
shown below.

exit
{% endhighlight %}


I didn't found solution for this particular problem, but for similar problems,
I've found recommendations to reinstall the plugins.
So I did that.

{% highlight bash %}
~% cat ~/.vagrant.d/plugins.json                                                                                                                                                17:00:24
{"version":"1","installed":{"vagrant-omnibus":{"ruby_version":"2.0.0","vagrant_version":"1.7.4","gem_version":"","require":"","sources":[]},"vagrant-vbguest":{"ruby_version":"2.0.0","vagrant_version":"1.7.4","gem_version":"","require":"","sources":[]},"vagrant-berkshelf":{"ruby_version":"2.0.0","vagrant_version":"1.6.5","gem_version":"3.0.1","require":"","sources":[]},"vagrant-lxc":{"ruby_version":"2.0.0","vagrant_version":"1.7.4","gem_version":"","require":"","sources":[]},"vagrant-share":{"ruby_version":"2.0.0","vagrant_version":"1.7.4","gem_version":"","require":"","sources":[]}}}
{% endhighlight %}

From that file I got information about which plugins I have installed. Then I've runned a series of commands:

{% highlight bash %}
vagrant plugin install vagrant-omnibus
vagrant plugin install vagrant-vbguest
...
{% endhighlight %}

And it worked.

I hope it will help you too.

FYI, during my Arch Linux upgrade I've made upgrades from 1.7.4 to 1.8.1.
