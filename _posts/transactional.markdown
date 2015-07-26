#---
layout: post
title:  "The magic of @Transactional and its performance"
categories: spring, mysql, transactional, benchmark, spring data, jdbc
#---
I was planning to write small article about `rollbackFor` parameter for @Trnsactional annotation. But when I 
was preparing the code examples, I decided to do some benchmarks. Let's strat with the brief annotation description. 

## @Transactional

@Transactinal is annotation which allows you to work with databases' transactions in declarative way.

{% highlight java %}
    @Transactional
    public void operation() {
        ... //Code with whick works with database
    }
{% endhighlight %}

This is pretty convinient - you don't need to think about direct transactions management and exceptions handling.
Everything will be done automaticaly in proxy class. This image 
(from [spring documentaion](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/transaction.html)) show how class 
hierarchy looks like:

<p>
<img src="http://docs.spring.io/spring/docs/current/spring-framework-reference/html/images/tx.png" />
</p>

So, it looks pretty clear. [JavaDoc](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html) 
about this annotation is always there and gives enought information to get started. But there's one unclear thing with
exception handling - here's an example:

{% highlight java %}
    @Transactional
    public void operation() {
        entityManager.persist(new User("Dima"));
        throw new Exception();
    }
{% endhighlight %}

Will it be commited? - Yes! 

<p>
<img width="300" src="http://vignette2.wikia.nocookie.net/epicrapbattlesofhistory/images/5/59/Stare-What-GIF.gif/revision/latest?cb=20140928165911"/>
</p>

The documentation says:

>Although EJB container default behavior automatically rolls back the transaction on a system exception 
(usually a runtime exception), EJB CMT does not roll back the transaction automatically on anapplication 
exception (that is, a checked exception other than java.rmi.RemoteException). While the Spring default 
behavior for declarative transaction management follows EJB convention (roll back is automatic only on 
unchecked exceptions), it is often useful to customize this behavior.
>
> [Spring Documentation](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/transaction.html#transaction-declarative)

Okaaay...
So, if you expect that checked exception will be thrown in your code, you better use `rollbackFor` in this way:

{% highlight java %}
    @Transactional(rollbackFor = Exception.class)
    public void operation() {
        entityManager.persist(new User("Dima"));
        throw new Exception();
    }
{% endhighlight %}

## Benchmarking

Now it's clear. Let's check how much we pay for the AOP magic. 
