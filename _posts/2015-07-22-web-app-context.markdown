---
layout: post
title:  "WebApplicationContext in the stand-alone application"
categories: java, spring mvc, spring framework, context 
---
Recently I got stuck with a problem of starting `WebApplicationContext` within a stand-alone java application. This is not so obvious as I expected. This article will be useful in a theoretical aspect - it shows how spring contexts work under the hood. Most likely you don't need to use such approach (more details in the conclusion) 

Let me describe the evolution of that application. It was a stand-alone java application without spring for a long time. We had embedded jetty, few pure java servlets and web services on top of CXF. And the main problem of that architecture was code complexity and a lot of problems with code testing. We decided to reduce code complexity using IoC and DI. For that time, most of my experience was related to spring framework. That why we picked it and rewrite most difficult parts using spring framework with XML configuration, annotation-based dependency injection, java-proxy-based AOP. And it was good.

Then I got a request to implement a web equivalent for the part of our swing interface. After research, "thin server" looked less time-consuming and more flexible.

> **Thin server architecture**. A SPA moves logic from the server to the client. This results in the role of the web server evolving into a pure data API or web service. This architectural shift has, in some circles, been coined "Thin Server Architecture" to highlight that complexity has been moved from the server to the client, with the argument that this ultimately reduces overall complexity of the system.
>
>~[Wikipedia](https://en.wikipedia.org/wiki/Single-page_application#Thin_server_architecture)

Honestly, I implemented not pure 'Thin server architecture', I left some logic on the server. Anyway, I built API on top of spring mvc. The first try looked like:

{% highlight java %}
ServletHolder servlet = new ServletHolder(DispatcherServlet.class);
servlet.setInitParameter("contextConfigLocation", "[context-configuration-location].xml");
{% endhighlight %}

Of course, also we had something like this:

{% highlight java %}
new ClassPathXmlApplicationContext("[context-configuration-location].xml");
{% endhighlight %}

It's the original application context. I put all beans, including controllers, in one XML file. And it doesn't work. Actually, it compiles and works unless you try to call some of the RESTful methods. When you call a RESTful method, an instance of **WebApplicationContext** will be created in `DispatcherServlet#initWebApplicationContext`. It'll set null as a parent context and all `ApplicationContextAware` will get new context as well as @Autowired fields in newly created beans. It means that even singletons will become non-singletons.

## WebApplicationContext

I went deeply into spring framework's sources and figured that I need to use `[context-configuration-location].xml` to initialize a parent context and create new one XML config with all web related beans. **ApplicationContext** is a root context for an entire application, it means that application should have only one root application context (application can have another **ApplicationContext**'s but they should have root one as a parent).

In my case, root application context will be represented by ClassPathXmlApplicationContext and child one should one which implements `WebApplicationContenxt`. I started to read documentation about `WebApplicationContenxt` interface. The JavaDoc says:

>This interface adds a getServletContext() method to the generic ApplicationContext interface, and defines a well-known application attribute name that the root context must be bound to in the bootstrap process.
>
>Like generic application contexts, web application contexts are hierarchical. There is a single root context per application, while each servlet in the application (including a dispatcher servlet in the MVC framework) has its own child context.
>
>~[JAVA DOC](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/WebApplicationContext.html)

The main idea of this context structure is binding each bean from web context to the appropriate servlet context. Therefore, it gives extra scopes like: "globalSession", "request", "session"; which aren't available in **ApplicationContext**.

<p>
<img src="{{ site.url }}/assets/webappcontext.png" class="img-responsive">
</p>

My case can be covered by `XmlWebApplicationContext`. I wrote it in this way:

{% highlight java %}            
ServletHandler springServletHandler = new ServletHandler();
XmlWebApplicationContext context = new XmlWebApplicationContext();
context.setConfigLocations("classpath:[web-context-configuration-location].xml");
context.setParent(parent);

DispatcherServlet dispatcherServlet = new DispatcherServlet(context);
ServletHolder servlet = new ServletHolder(dispatcherServlet);
{% endhighlight %}

Now it's working as expected.

## Conclusion

In this article I described a not standard approach for contexts' initialization; it was required by the application architecture. In most cases, you should use initialization via **web.xml/WebApplicationInitializer** for web application, or regular **ApplicationContext** for stand-alone applications. Moreover, since [Spring Boot](http://projects.spring.io/spring-boot/) is released, you can get rid of all context initialization problems.
