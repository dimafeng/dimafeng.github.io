---
layout: post
title:  "Dynamic bean definition for automatic FilterRegistrationBeanFactory unregistration"
categories: spring framework, spring boot, BeanFactoryPostProcessor
---
Sometimes you need to define beans dynamically e.g. you need to generate many bean those definition depends on
external conditions, so you can't declare them via xml/java/annotation config. For these situations, there is ability
to generate bean definitions dynamically.  

## Dynamic bean creation

Typically, the bean creation workflow looks as follows:

<p>
<img src="https://docs.google.com/drawings/d/1LXzmwbu4mxcTCp3smQXNe9KL2vOsQHdNNUvM0Vk1Xtc/pub?w=960&amp;h=720">
</p>

To declare new beans, there are 2 places to add custom logic:

* (1) `BeanFactoryPostProcessor`: we can modify `BeanFactory` so it'll know about new beans before context initialization.
* (2) After `ApplicationContext` creation: we can `BeanFactory` after context initialization, obviously these beans won't be autowired to other beans.

In both approaches, we need to create a `BeanDefinition`. The preferred implementation of the `BeanDefinition` interface is `GenericBeanDefinition`. Also, I found out that there is a useful class `BeanDefinitionBuilder` that can help with bean definition creation. Let's try to use it.

Here's a simple example of the 2nd approach:

{% highlight java %}
public class Example1 {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Config.class);

        try {
            System.out.println(applicationContext.getBean("testBean"));
        } catch (NoSuchBeanDefinitionException e) {
            System.out.println("Bean not found");
        }

        BeanDefinitionRegistry beanFactory = (BeanDefinitionRegistry) applicationContext.getBeanFactory();

        beanFactory.registerBeanDefinition("testBean",
                BeanDefinitionBuilder.genericBeanDefinition(String.class)
                        .addConstructorArgValue("test")
                        .getBeanDefinition()
        );

        System.out.println(applicationContext.getBean("testBean"));
    }

    @Configuration
    public static class Config {

    }
}
{% endhighlight %}

If you launch it, you'll see that initially we didn't have `testBean` inside the context, but then it becomes available.

Let's look at a more complex example which will demonstrate the 1st approach.

{% highlight java %}
public class Example2 {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Config.class);

        System.out.println(applicationContext.getBean("testBean"));
    }

    @Configuration
    public static class Config {

        @Bean
        public String myTestStringBean() {
            return "My test String Bean";
        }

        @Bean
        public BeanFactoryPostProcessor beanFactoryPostProcessor() {
            return bf -> {
                BeanDefinitionRegistry beanFactory = (BeanDefinitionRegistry) bf;

                beanFactory.registerBeanDefinition("testBean",
                        BeanDefinitionBuilder.genericBeanDefinition(TestBean.class)
                                .addConstructorArgReference("myTestStringBean")
                                .getBeanDefinition()
                );
            };
        }
    }

    public static class TestBean {
        private String value;

        public TestBean(String value) {
            this.value = value;
        }

        @Override
        public String toString() {
            return "Test bean with value: " + value;
        }
    }
}
{% endhighlight %}

Here we define a bean `beanFactoryPostProcessor` that is responsible for `BeanFactory` modification. We added a bean definition with the following specification:

* It'll be a bean of class TestBean. By default, it'll be a singleton bean instantiated using its constructor.
* The required field `value` will be passed to the constructor by spring during the bean creation. It's specified by the method `addConstructorArgReference`.

If you launch this example, it'll work as expected - the command line will print:

<pre>
Test bean with value: My test String Bean
</pre>

## Real world example: FilterRegistrationBeanFactory unregistration

Now we can talk about real examples. I've been using spring very active for about 3 years, but I used this technique only once.

Here's a spring boot app.

{% highlight java %}
@SpringBootApplication
public class ApplicationExample3 {
    public static void main(String[] args) throws Exception {
        SpringApplication.run(ApplicationExample3.class, args);
    }
}
{% endhighlight %}

Yeah, it's one class only. If you launch it you'll see:

<pre>
2015-11-27 20:10:03.066  INFO 4527 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'characterEncodingFilter' to: [/*]
2015-11-27 20:10:03.067  INFO 4527 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
2015-11-27 20:10:03.067  INFO 4527 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'springSecurityFilterChain' to: [/*]
</pre>

It says that spring registered some filters by itself and mapped to `/*`. I didn't ask it to do that. What if I don't need them? In my specific case, I'm configuring specific rules by filter chain, so I need to disable all default filter mappings. We go to [spring boot issue tracker](https://github.com/spring-projects/spring-boot/issues/2173) and
[stackoverflow thread](http://stackoverflow.com/questions/28421966/prevent-spring-boot-from-registering-a-servlet-filter/28428154#28428154)

They suggest to do it in this way:

{% highlight java %}
@Bean
public FilterRegistrationBean registration(PreAuthenticationFilter filter) {
    FilterRegistrationBean registration = new FilterRegistrationBean(filter);
    registration.setEnabled(false);
    return registration;
}
{% endhighlight %}

What if I have tens of filters? This approach is way too complex for maintaince. Let's generate `FilterRegistrationBean`s for all filters dynamically. Here's my `BeanFactoryPostProcessor`:

{% highlight java %}
public class DefaultFiltersBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf)
            throws BeansException {
        DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) bf;

        Arrays.stream(beanFactory.getBeanNamesForType(javax.servlet.Filter.class))
                .forEach(name -> {

                    BeanDefinition definition = BeanDefinitionBuilder
                            .genericBeanDefinition(FilterRegistrationBean.class)
                            .setScope(BeanDefinition.SCOPE_SINGLETON)
                            .addConstructorArgReference(name)
                            .addConstructorArgValue(new ServletRegistrationBean[]{})
                            .addPropertyValue("enabled", false)
                            .getBeanDefinition();

                    beanFactory.registerBeanDefinition(name + "FilterRegistrationBean",
                            definition);
                });
    }
}
{% endhighlight %}

As you see, the BeanDefinition reflects suggested bean definition. And now all default filters are
disabled.


<pre>
2015-11-27 21:09:24.745  INFO 5119 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'springSecurityFilterChain' to: [/*]
2015-11-27 21:09:24.746  INFO 5119 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Filter orderedHiddenHttpMethodFilter was not registered (disabled)
2015-11-27 21:09:24.746  INFO 5119 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Filter filterChainProxy was not registered (disabled)
2015-11-27 21:09:24.746  INFO 5119 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Filter orderedCharacterEncodingFilter was not registered (disabled)
</pre>

All examples from this blog post you can find [here in my github profile](https://github.com/dimafeng/dimafeng-examples/tree/master/dynamic-bean-def).

**UPD**: Right after I posted the blog post I found out that there's the interface `BeanDefinitionRegistryPostProcessor` exactly for this type of BeanFactory modification since Spring 3.0.1. It seems it's a preferable interface to use and code snippet described in the post can be used as it is with this interface.
