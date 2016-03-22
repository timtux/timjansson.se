---
layout: post
title:  "Sätta tidszon i Amazon Linux EBS"
date:   2016-03-21 15:12:42 +0100
---
För att timezonen ska sparas även när nya EBS-instanser skapas (ex. vid skalning) skapar vi en ny fil i  ".ebextensions".

**.ebextensions/00-set-timezone.config**
{% highlight yaml %}
commands:
  set_time_zone:
    command: 'ln -f -s /usr/share/zoneinfo/Europe/Stockholm /etc/localtime'
{% endhighlight %}
