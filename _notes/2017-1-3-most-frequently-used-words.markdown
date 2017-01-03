---
layout: note
title:  "Easy way to collect most frequently used words"
---
If you learning english, you probably know the technique of learning most common words in some certain scope.
Here's a simple bash one-liner that does all the magic:

{% highlight bash %}
cat [file] | tr ' ' '\n' | grep '[a-zA-Z]\{4,\}' | sort | uniq -c | sort -nr | head -200
{% endhighlight %}
