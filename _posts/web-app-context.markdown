Recently I got stuck with a problem of starting `WebApplicationContext` within stand-alone java application. This is not
so obvious as I expected.

Let me describe evolution of that application. It was a stand-alone java applicatoin without spring for a long time. We had embeded jetty, few pure java servlets and web services on top of cxf. And the main problem of that architecture was code complexity and a lot of ploblems with code testing. We decided to reduce code complexity using IoC and DI. Fot that time, most of my experience was related to spring framework. That's we picked it and rewrite most difficult parts using spring framework with xml configuration, annotation-based dependency injection, java-proxy-based AOP. And it was good.

Then we got a request to implement a web equevalent for the part of our swing interface. After research, it looked less time-consuming and more flexible way is develop "thin server".
