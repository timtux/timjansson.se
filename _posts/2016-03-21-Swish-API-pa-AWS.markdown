---
layout: post
title:  "Swish API, klientcertifikat & Amazon Linux (EBS) (NSS)"
date:   2016-03-21 15:12:42 +0100
---
På jobbet håller vi på att migrera till Amazons infrastruktur från de virtualiserade VMWare-instanserna med Debian vi kör idag. På Amazon har vi valt en uppsättning med Amazon Linux (PHP) som körs i "Elastic Beanstalk". Målet med migreringen är att få bort så mycket manuellt underhåll som möjligt, öka driftsäkerheten och få en miljö som skalar av sig själv.

Amazons Linuxversion bygger på CentOS / Redhat / RPM av pakethanteringssystem och struktur att döma. Deras curl är kompilerat mot Mozillas NSS istället för OpenSSL. I Swish e-handelslösning som vi använder så autentiserar man mot deras API med hjälp av klientcertifikat, något ganska ovanligt för betallösningar riktade mot e-handel.  

Fick en massa problem med att SSL-handskaken inte fungerade, curl spottade ut felmeddelande på felmeddelande. Här var PHP-koden som användes. Fungerar felfritt med curl & openssl:

{% highlight php %}
<?php
$options = [
   CURLOPT_CUSTOMREQUEST => 'POST',
   CURLOPT_RETURNTRANSFER => true,
   CURLOPT_HEADER => true,
   CURLOPT_SSL_VERIFYPEER => false,
   CURLOPT_SSL_VERIFYHOST => false,
   CURLOPT_URL => $url,
   CURLOPT_CAINFO => '/etc/ssl/certs/Swish server TLS certificate.pem',
   CURLOPT_SSLCERT => APP_DIR . 'config/Swish/Swish.pem',
   CURLOPT_POST => true,
   CURLOPT_FOLLOWLOCATION => true,
   CURLOPT_POSTFIELDS => $data_string,
   CURLOPT_HTTPHEADER => [
       'Content-Type: application/json'
   ]
];
{% endhighlight %}

Efter en massa felsökning så misstänkte jag att NSS var boven i dramat. Visade sig att man var tvungen att lägga till Swish certifikat i NSS certifkikatdb. Det fungerade alltså inte att skicka med certfikaten som ovan.

**a) Skapa en pksc12-fil från din private key och Swish-certet:**
{% highlight bash %}
openssl pkcs12 -export -out /tmp/server.pfx -inkey /tmp/www_partykungen_se.key -in /tmp/swish.pem -certfile /tmp/Swish_server_TLS_certificate.pem -password pass:
{% endhighlight %}

**b) Importera deras CA-cert.**
{% highlight bash %}
certutil -d sql:/etc/pki/nssdb/ -A -t "C,C,C" -n "Swish Root Cert" -i /tmp/Swish_server_TLS_certificate.pem
{% endhighlight %}

**c) Importera Swish-certet vi skapade i steg "a"**
{% highlight bash %}
pk12util -i /tmp/server.pfx -d sql:/etc/pki/nssdb/ -W "" -K ""
{% endhighlight %}

Nu får vi istället i CURL använda oss av certifikatets namn i nssdb. Det hittar vi genom att köra följande:

{% highlight bash %}
[ec2-user@ip-127-0-0-1 ~]$ certutil -d sql:/etc/pki/nssdb/ -L

Certificate Nickname                                         Trust Attributes
                                                             SSL,S/MIME,JAR/XPI

Swish Root Cert                                              C,C,C
12345678 - Swedbank AB (publ)                              u,u,u #<- Denna
Swedbank Customer CA1 v1 for Swish - Swedbank AB (publ)      ,,   
Swedbank Root CA v1 for Swish - Getswish AB                  ,,   
Swish Root CA v1 - Getswish AB                               ,,  
{% endhighlight %}

Så vi får anpassa PHP-koden:

{% highlight php %}
<?php
$options = [
   CURLOPT_CUSTOMREQUEST => 'POST',
   CURLOPT_RETURNTRANSFER => true,
   CURLOPT_HEADER => true,
   CURLOPT_URL => $url,
   CURLOPT_SSLCERT => '12345678 - Swedbank AB (publ)',
   CURLOPT_POST => true,
   CURLOPT_FOLLOWLOCATION => true,
   CURLOPT_POSTFIELDS => $data_string,
   CURLOPT_HTTPHEADER => [
       'Content-Type: application/json'
   ]
];
{% endhighlight %}

Sen kommer nästa problem: inget är persistent i en EBS-miljö. Miljön "nollas" ju ibland, så vi får se till att köra allt när en ny instans startas med hjälp av .ebextensions:

**.ebextensions/03-certificate-files.config**
Lägger till våra certifikat-filer.
{% highlight yaml %}
files:
  "/tmp/Swish_server_TLS_certificate.pem" :
    mode: "000755"
    owner: root
    group: root
    content: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
  "/tmp/swish.pem" :
    mode: "000755"
    owner: root
    group: root
    content: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
  "/tmp/www_partykungen_se.key" :
    mode: "000755"
    owner: root
    group: root
    content: |
      -----BEGIN PRIVATE KEY-----
      ...
      -----END PRIVATE KEY-----
{% endhighlight %}

**.ebextensions/04-generate-pkcs12-and-add-to-nssdb.config**
Genererar pkcs12-filen från certifikaten vi behöver och lägger till Swish CA-cert och vårt cert i nssdbn. Sen städar upp! Finns ingen anledning för dem att ligga kvar.
{% highlight yaml %}
commands:
  a_create_pfx:
    command: 'openssl pkcs12 -export -out /tmp/server.pfx -inkey /tmp/www_partykungen_se.key -in /tmp/swish.pem -certfile /tmp/Swish_server_TLS_certificate.pem -password pass:'
  b_import_swish_ca:
    command: 'certutil -d sql:/etc/pki/nssdb/ -A -t "C,C,C" -n "Swish Root Cert" -i /tmp/Swish_server_TLS_certificate.pem'
  c_import_swish_cert:
    command: 'pk12util -i /tmp/server.pfx -d sql:/etc/pki/nssdb/ -W "" -K ""'
  d_rm_files:
    command: 'rm /tmp/Swish_server_TLS_certificate.pem /tmp/server.pfx /tmp/swish.pem /tmp/www_partykungen_se.key'
{% endhighlight %}



[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
