---
layout: post
title:  "UniFi® Security Gateway Pro rebootar och är seg"
date:   2016-03-21 16:14:22 +0100
---
Har installerat en UniFi® Security Gateway Pro. Tyvärr så verkar det som att när många enheter ansluter mot samma host så flippar den ur. Den rebootar med jämna intervall på några timmar och det tar lång tid att initiera nya anslutningar mot hosten. Den temporära lösningen som fungerade var att slå av hårdvaru TCP/IP offloading. För att göra detta lär man SSHa in på Gatewayen efter varje omstart eller konfigurationsändring.

{% highlight bash %}
configure
set system offload ipv4 pppoe disable
set system offload ipv4 vlan disable
set system offload ipv4 forwarding disable
set system offload ipv6 vlan disable
set system offload ipv6 forwarding disable
commit;save;exit
{% endhighlight %}

Får se om det kommer något nytt firmware som fixar detta eller om det blir s/UniFi® Security Gateway Pro/EdgerRouter PRO/g istället.
