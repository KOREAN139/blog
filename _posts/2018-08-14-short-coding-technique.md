---
layout: post
title: "Short coding technique"
description: "Some kind of tricks in C"

tags: [shortcoding, c]
---
## Read input
{% highlight C %}
while(~scanf("%d",&x)){/* do something */}
{% endhighlight %}
Since scanf returns `EOF` when *end-of-file* is reached while reading, this while-loop read one integer to `x` until it reaches *end-of-file*.