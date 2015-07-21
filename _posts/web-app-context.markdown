Recently I got stuck with a problem of starting `WebApplicationContext` within stand-alone java application. This is not
so obvious as I expected.

Let me describe evolution of that application. It was a stand-alone java applicatoin without spring for a long time. We had embeded jetty, few pure java servlets and web services on top of cxf. And the main problem of that architecture was code complexity and a lot of ploblems with code testing. We decided to reduce code complexity using IoC and DI. Fot that time, most of my experience was related to spring framework. That why we picked it and rewrite most difficult parts using spring framework with xml configuration, annotation-based dependency injection, java-proxy-based AOP. And it was good.

Then I got a request to implement a web equevalent for the part of our swing interface. After research, "thin server" looked less time-consuming and more flexible.

> **Thin server architecture**. A SPA moves logic from the server to the client. This results in the role of the web server evolving into a pure data API or web service. This architectural shift has, in some circles, been coined "Thin Server Architecture" to highlight that complexity has been moved from the server to the client, with the argument that this ultimately reduces overall complexity of the system.
>
>~[Wikipedia](https://en.wikipedia.org/wiki/Single-page_application#Thin_server_architecture)

Honestly, I implemented not pure 'Thin server architecture', I left some logic on the server. Anyway, I built API on top of spring mvc. The first try looked like:

{% highlight java %}
ServletHolder servlet = new ServletHolder(DispatcherServlet.class);
servlet.setInitParameter("contextConfigLocation", "[context-configuration-location].xml");
{% endhighlight %}

Of couruse, also we had something like this:

{% highlight java %}
new ClassPathXmlApplicationContext("[context-configuration-location].xml");
{% endhighlight %}

It's the original application context. I put all beans, including controllers, in one XML file. And it doesn't work. Aclually, it compiles and works unless you try to call some of the RESTful methods. When you call a RESTful method, an instance of **WebApplicationContext** will be created in `DispatcherServlet#initWebApplicationContext`. It'll set null as a parent context and all `ApplicationContextAware` will get new context as well as @Autowired fields in newly created beans. It means that even singletons will become non-singletons.

## WebApplicationContext

I went deeply into spring framework's sources and figured that I need to use `[context-configuration-location].xml` to initialize a parent context and create new one xml config with all web related beans. First application context will be ClassPathXmlApplicationContext and second one should one which implements `WebApplicationContenxt`. I stared to read documentation about `WebApplicationContenxt` interface. The JavaDoc says:

>This interface adds a getServletContext() method to the generic ApplicationContext interface, and defines a well-known application attribute name that the root context must be bound to in the bootstrap process.
>
>Like generic application contexts, web application contexts are hierarchical. There is a single root context per application, while each servlet in the application (including a dispatcher servlet in the MVC framework) has its own child context.
>
>~[JAVA DOC](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/WebApplicationContext.html)

The main idea of this context structure is adding extra scopes like: "application", "globalSession", "request", "session". The application in general should look like that:

<p>
<img src="http://m2.img.srcdd.com/farm5/d/2012/1120/15/8AB82451C1D7C3B8658DBB80A2E9177A_B500_900_500_355.PNG" class="img-responsive">
</p>

My case could be covered by `XmlWebApplicationContext`. I wrote it in this way:

{% highlight java %}
            ServletHandler springServletHandler = new ServletHandler();
            XmlWebApplicationContext context = new XmlWebApplicationContext();
            context.setConfigLocations("classpath:[web-context-configuration-location].xml");
            context.setParent(parent);

            DispatcherServlet dispatcherServlet = new DispatcherServlet(context);
            ServletHolder servlet = new ServletHolder(dispatcherServlet);
{% endhighlight %}

