---
layout: post
title:  "Spring @Configuration vs @Component"
categories: spring framework, spring configuration
---
[In the previous post](/2015/08/16/cglib/), I said that you can use `@Component` as an alternative for `@Configuration`.
In fact, this is the official [suggestion from the spring team](https://jira.spring.io/browse/SPR-12833).

>That said, there is a 'lite' mode of @Bean processing where we don't apply any CGLIB processing: simply declare your @Bean methods on classes not annotated with @Configuration (but typically with another Spring stereotype instead, e.g. @Component). As long as you don't do programmatic calls between your @Bean methods, this is going to work just as fine.

Simply put, each application context configuration shown below will act in a totally different way:

{% highlight java %}
@Configuration
public static class Config {

    @Bean
    public SimpleBean simpleBean() {
        return new SimpleBean();
    }

    @Bean
    public SimpleBeanConsumer simpleBeanConsumer() {
        return new SimpleBeanConsumer(simpleBean());
    }
}
{% endhighlight %}

{% highlight java %}
@Component
public static class Config {

    @Bean
    public SimpleBean simpleBean() {
        return new SimpleBean();
    }

    @Bean
    public SimpleBeanConsumer simpleBeanConsumer() {
        return new SimpleBeanConsumer(simpleBean());
    }
}
{% endhighlight %}

The first piece of code works fine, and as expected, `SimpleBeanConsumer` will get a link to singleton `SimpleBean`.
But unfortunately, it [doesn't work in a signed enviroment](/2015/08/16/cglib/).

The second configuration is totally incorrect because spring will create a singleton bean of `SimpleBean`, but `SimpleBeanConsumer`
will obtain another instance of `SimpleBean` which is out of the spring context control.

The reason for this behaviour can be explained as follows:

If you use `@Configuration`, all methods marked as `@Bean` will be wrapped
into a CGLIB wrapper which works as if it's the first call of this method, then the original method's body will be executed and the resulting object will be
registered in the spring context. All further calls just return the bean retrieved from the context.

In the second code block above, `new SimpleBeanConsumer(simpleBean())` just calls a pure java method. To correct the second code block, we can do something like this:

{% highlight java %}
@Component
public static class Config {
    @Autowired
    SimpleBean simpleBean;

    @Bean
    public SimpleBean simpleBean() {
        return new SimpleBean();
    }

    @Bean
    public SimpleBeanConsumer simpleBeanConsumer() {
        return new SimpleBeanConsumer(simpleBean);
    }
}
{% endhighlight %}

All code examples for this post can be found at [my GitHub profile](https://github.com/dimafeng/dimafeng-examples/tree/master/spring-config).
