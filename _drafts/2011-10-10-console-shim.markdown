---
layout: post
title:  "Console shim for crippled browsers"
date:   2014-10-10 11:15:00
categories: javascript oldbrowsers
---
I've gotten too used to having the Chrome Console API around, to the point that when doing testing
with older, crippled browsers, such as Internet Explorer 8. These come in occasionally and
for the sake of graceful degradation, keep older browsers chugging along for at least a bit
more.

Instead of the usual global console override, I'm using a case-by-case polyfill for undefined
methods. Internet Explorer 9 is one case that has logging functionality, but not e.g. grouping.

{% highlight javascript %}
var
  noop = function () {},
  args = [
    'log', 'info', 'warn', 'error', 'debug', 'trace', 'dir', 'group', 'groupCollapsed',
    'groupEnd', 'time', 'timeEnd', 'profile', 'profileEnd', 'dirxml', 'assert',
    'count', 'markTimeline', 'timeStamp', 'clear'
  ];

if (!('console' in window)) {
  window.console = {};
}

_.each(args, function (arg) {
  if (window.console[arg] == null) {
    window.console[arg] = noop;
  }
});
{% endhighlight %}
