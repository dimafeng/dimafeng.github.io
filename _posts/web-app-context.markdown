Recently I got stuck with a problem of starting `WebApplicationContext` within stand-alone java application. This is not
so obvious as I expected.

Let me describe evolution of that application. It was a stand-alone java applicatoin without spring for a long time. We had embeded jetty, few pure java servlets and web services on top of cxf. And the main problem of that architecture was code complexity and a lot of ploblems with code testing. We decided to reduce code complexity using IoC and DI. Fot that time, most of my experience was related to spring framework. That's we picked it and rewrite most difficult parts using spring framework with xml configuration, annotation-based dependency injection, java-proxy-based AOP. And it was good.

Then we got a request to implement a web equevalent for the part of our swing interface. After research, "thin server" looked less time-consuming and more flexible.

> **Thin server architecture**. A SPA moves logic from the server to the client. This results in the role of the web server evolving into a pure data API or web service. This architectural shift has, in some circles, been coined "Thin Server Architecture" to highlight that complexity has been moved from the server to the client, with the argument that this ultimately reduces overall complexity of the system.
>
>~[Wikipedia](https://en.wikipedia.org/wiki/Single-page_application#Thin_server_architecture)

Honestly, we implemented not pure 'Thin server architecture', we left some login on the server. Anyway, we built API on top of spring mvc. The first try looked like:

{% highlight java %}
ServletHolder servlet = new ServletHolder(DispatcherServlet.class);
servlet.setInitParameter("contextConfigLocation", "[context-configuration-location].xml");
{% endhighlight %}





