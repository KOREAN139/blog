---
layout: post
title: "Short coding technique"
description: "Some kind of tricks in C"

tags: [shortcoding, c]
---
## Read integer

{% highlight C %}
while(~scanf("%d",&x)){/* do something */}
{% endhighlight %}

Since scanf() returns `EOF` when *end-of-file* is reached while reading, 
this while-loop read one integer to `x` until it reaches *end-of-file*.

`EOF` usally defined as -1, and `~(-1) == 0`.

So when scanf() reads `EOF`, *which indicates end-of-file*, that while-loop terminates.

## Discard first line

{% highlight C %}
gets(x); /* x must be pointer */
{% endhighlight %}

## Loop [n-1, 0]

{% highlight C %}
while(n--){/* do something */}
for(;n--;){/* do something */}
{% endhighlight %}
