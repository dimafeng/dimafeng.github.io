---
layout: post
title:  "Dynamic bean definition for automatic FilterRegistrationBeanFactory unregistration"
categories: spring framework, spring boot, BeanFactoryPostProcessor
---
Sometimes you need to define beans dynamically (e.g. you need to generate many beans because the bean definitions depend on
external conditions) since you cannot declare them via xml/java/annotation config. For these situations, it is possible
to generate bean definitions dynamically.  

## Dynamic bean creation

Typically, the bean creation workflow happens as follows:

<p>
<img src="https://docs.google.com/drawings/d/1LXzmwbu4mxcTCp3smQXNe9KL2vOsQHdNNUvM0Vk1Xtc/pub?w=960&amp;h=720">
</p>

To declare new beans, there are 2 places to add custom logic:

* (1) `BeanFactoryPostProcessor`: we can modify the `BeanFactory` so it knows about new beans before the context initialization phase.
* (2) After the `ApplicationContext` creation phase, we can reconfigure the `BeanFactory`, keeping in mind that these beans won't be autowired to other beans.

In both approaches, we need to create a `BeanDefinition`. The preferred implementation of the `BeanDefinition` interface is `GenericBeanDefinition`. Additionally, there is a useful class called `BeanDefinitionBuilder` that can help with bean definition creation. Let's try to use it!

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

If you launch the above example, you'll see that initially we don't have `testBean` inside the context, but then it becomes available.

Let's now look at a more complex example that demonstrates the 1st approach:

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

Above we defined a bean named `beanFactoryPostProcessor` that is responsible for modifying the `BeanFactory` and added a bean definition with the following specification:

* It's a bean of the class `TestBean`. By default, it's a singleton bean instantiated using its constructor.
* The required field `value` will be passed to the constructor by Spring during the bean creation and specified by the method `addConstructorArgReference`.

If you launch this example, it will work as expected and the console will print the following:

<pre>
Test bean with value: My test String Bean
</pre>

## In practice: FilterRegistrationBeanFactory unregistration

Now we can talk about a few real world examples. I've been using Spring very actively for about 3 years but I used this technique only once.

Here's a spring boot app:

{% highlight java %}
@SpringBootApplication
public class ApplicationExample3 {
    public static void main(String[] args) throws Exception {
        SpringApplication.run(ApplicationExample3.class, args);
    }
}
{% endhighlight %}

Yes, it is only one class. If you launch it, you will see this:

<pre>
2015-11-27 20:10:03.066  INFO 4527 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'characterEncodingFilter' to: [/*]
2015-11-27 20:10:03.067  INFO 4527 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
2015-11-27 20:10:03.067  INFO 4527 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'springSecurityFilterChain' to: [/*]
</pre>

It says that spring registered some filters by itself and mapped each one to `/*`. I didn't ask it to do that. What if I don't need them? In my specific case, I'm configuring specific rules by filter chain, so I need to disable all default filter mappings. I found out that other people were having the same problem ([spring boot issue tracker](https://github.com/spring-projects/spring-boot/issues/2173) and
[stackoverflow thread](http://stackoverflow.com/questions/28421966/prevent-spring-boot-from-registering-a-servlet-filter/28428154#28428154)).

The aforementioned solutions suggest the following solution:

{% highlight java %}
@Bean
public FilterRegistrationBean registration(PreAuthenticationFilter filter) {
    FilterRegistrationBean registration = new FilterRegistrationBean(filter);
    registration.setEnabled(false);
    return registration;
}
{% endhighlight %}

What if I have a large number of filters? This approach becomes unwieldy. Let's generate multiple `FilterRegistrationBean`s for all filters dynamically. Here's my `BeanFactoryPostProcessor`:

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

As you can see, the BeanDefinition reflects a suggested bean definition and now all the default filters are
disabled.


<pre>
2015-11-27 21:09:24.745  INFO 5119 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'springSecurityFilterChain' to: [/*]
2015-11-27 21:09:24.746  INFO 5119 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Filter orderedHiddenHttpMethodFilter was not registered (disabled)
2015-11-27 21:09:24.746  INFO 5119 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Filter filterChainProxy was not registered (disabled)
2015-11-27 21:09:24.746  INFO 5119 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Filter orderedCharacterEncodingFilter was not registered (disabled)
</pre>

All code examples for this post can be found at [my github profile](https://github.com/dimafeng/dimafeng-examples/tree/master/dynamic-bean-def).

**UPDATE**: Right after I posted this blog, I learned that there's an interface called `BeanDefinitionRegistryPostProcessor` that is meant exactly for this type of BeanFactory modifications since Spring 3.0.1. It seems it's a preferable interface to use and the code snippet described in this post can be used as-is with this interface.
