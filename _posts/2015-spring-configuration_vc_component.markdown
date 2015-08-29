---
layout: post
title:  "Spring @Configuration vs @Component"
categories: spring framework, spring configuration
---
In [previous post](/2015/08/16/cglib/) I said that you can use `@Component` as an alternative for `@Configuration`. 
It's official [suggestion from spring team](https://jira.spring.io/browse/SPR-12833).

>That said, there is a 'lite' mode of @Bean processing where we don't apply any CGLIB processing: simply declare your @Bean methods on classes not annotated with @Configuration (but typically with another Spring stereotype instead, e.g. @Component). As long as you don't do programmatic calls between your @Bean methods, this is going to work just as fine.

Simple saying, contexts configured with these two condfigurations are acting totally differently:

{% highlight java %}
    @Configuration
    public static class Config {

        @Bean
        public SimpleBean simpleBean()
        {
            return new SimpleBean();
        }

        @Bean
        public SimpleBeanConsumer simpleBeanConsumer()
        {
            return new SimpleBeanConsumer(simpleBean());
        }
    }
{% endhighlight %}

{% highlight java %}
    @Component
    public static class Config {

        @Bean
        public SimpleBean simpleBean()
        {
            return new SimpleBean();
        }

        @Bean
        public SimpleBeanConsumer simpleBeanConsumer()
        {
            return new SimpleBeanConsumer(simpleBean());
        }
    }
{% endhighlight %}

The first piece of code works fine, and as expected, `SimpleBeanConsumer` will get a link to singleton `SimpleBean`.
But unfortunately, it [doesn't work in signed enviroment](/2015/08/16/cglib/).

The second configuration is totally incorrect: spring will create singleton of `SimpleBean`, but `SimpleBeanConsumer` 
will obtain another instance of `SimpleBean` which is out of the spring context control.

The reason of this behaviour is pretty obvious. If you use `@Configuration` all methods marked as `@Bean` will be wrapped 
into CGLIB wrapper which works as follows: if it's the first call of this method, then the original method's body will be executed and resulted object will be 
registered in spring context. All further calls just return the bean retrieved from the context. 
The second piece of code, `new SimpleBeanConsumer(simpleBean())`, just calls a pure java method.

To solve the second example, we can use something like that:

{% highlight java %}
    @Component
    public static class Config {

        @Autowired
        SimpleBean simpleBean;

        @Bean
        public SimpleBean simpleBean()
        {
            return new SimpleBean();
        }

        @Bean
        public SimpleBeanConsumer simpleBeanConsumer()
        {
            return new SimpleBeanConsumer(simpleBean);
        }
    }
{% endhighlight %}

All code examples for this post can be found in [my GitHub profile](https://github.com/dimafeng/dimafeng-examples/tree/master/spring-config).
