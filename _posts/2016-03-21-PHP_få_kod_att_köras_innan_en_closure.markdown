---
layout: post
title:  "PHP: Få kod att exekveras innan en closure"
date:   2016-03-21 16:14:22 +0100
---
Ganska dålig rubrik va? I vilket fall så stötte vi på ett tillfälle där vi ville köra kod innan en closure kördes. Vi vill alltså "prependa kod i en closure". Just i det här fallet var det att vi ville köra newrelic_name_transaction för att mappa requests till en viss route innan koden för routen exekverades, och utan att behöva lägga till koden i varje routes kod.

{% highlight php %}
<?php
/* Såhär skapar vi en route exempelvis */
$routes[] = new Route('GET /index/@var1', function($var1) {
  echo 'Hello world! '.$var1;
});

/**
 * Såhär löste vi det
 */
class Route
{
    /* .. */
    function __construct($request, closure $functions, $weight = NULL)
    {
        $this->request = $request;
        $this->weight = $weight;

        $this->function = function (...$vars) use ($functions) {
            if(function_exists('newrelic_name_transaction')) {
                newrelic_name_transaction($this->request);
            }

            return $functions(...$vars);
        };
    }
    /* .. */
}
{% endhighlight %}
