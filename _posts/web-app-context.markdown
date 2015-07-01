Recently I got stuck with a problem of starting `WebApplicationContext` within stand-alone java application. This is not
so obvious as I expected.

Let me describe evolution of that application. It was a stand-alone java applicatoin without spring for more than 8 years.
We had embeded jetty, few pure java servlets and web services on top of cxf. It was 
