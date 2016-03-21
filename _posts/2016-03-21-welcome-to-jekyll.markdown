---
layout: post
title:  "Sätta tidszon i Amazon Linux EBS"
date:   2016-03-21 15:12:42 +0100
categories: amazon aws linux
---
För att timezonen ska sparas även om vi skapar nya environments och vid auto-skalning osv så skapar vi en ny "ebextension".

**.ebextensions/00-set-timezone.config**
{% highlight yaml %}
commands:
  set_time_zone:
    command: 'ln -f -s /usr/share/zoneinfo/Europe/Stockholm /etc/localtime'
{% endhighlight %}
