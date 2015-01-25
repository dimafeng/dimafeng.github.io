---
layout: post
title:  "Craigslist ads monitoring with groovy"
categories: groovy, gradle
---
Recently I was looking for a new apartment in New York. Well known that first place where you need to start looking for something is craigslist. I didn't find any functionality which can send new ads by my criteria. I decided to write small script which could solve this problem. I've had some experience with groovy and wanted to refresh the memory.

##Let's get it started

What we need? Not so long ago I began to choose gradle for each project. This's not an exception too. 

{% highlight groovy %}
apply plugin: 'groovy'
apply plugin: 'application'

mainClassName = "com.dimafeng.craigslist.Main"

sourceCompatibility = 1.5
version = '1.0'

repositories {
    mavenCentral()
}

dependencies {
    compile 'org.codehaus.groovy:groovy-all:2.3.9'
    compile 'org.jsoup:jsoup:1.8.1'
    compile 'javax.mail:mail:1.4.7'
    testCompile group: 'junit', name: 'junit', version: '4.11'
}

task uberjar(type: Jar, dependsOn: build) {
    from files(sourceSets.main.output.classesDir)
    from {configurations.compile.collect {zipTree(it)}}

    manifest {
        attributes 'Main-Class': 'com.dimafeng.craigslist.Main'
    }
}
{% endhighlight %}

This build script is pretty simple. I added *application* plugin just for testing, you don't really need it. 
As for dependencies - we need: 

* *groovy* is needed for project compilation
* *jsoup* is good lib for html DOM manipulations in JQuery style
* *javax.mail* is simplest way to email sending

Let's write some groovy code.

{% highlight groovy %}
public static void main(String[] args) {

        println "App started " + new Date()

        //Reading file with old ads
        def old = new ArrayList()
        def file = new File('old')
        def firstStart = false
        if (!file.exists()) {
            file.createNewFile();
            firstStart = true;
        } else {
            old.addAll(file.readLines())
        }

        //Config parsing
        def config = new Properties()
        config.load(new File('app.properties').newDataInputStream())

        def targetUrl = config.getProperty("app.url")
        def baseUrl = targetUrl.replaceAll("(http://(.*?).craigslist.org)(.*)", '$1');

        Document doc = Jsoup.connect(targetUrl).get();
        Elements ads = doc.select("p.row");

        def result = new StringBuilder();
        int i = 0;

        ads.forEach({ e ->
            def id = e.attr("data-pid").toString()
            def url = e.getElementsByTag("a").first().attr("href")

            if (!old.contains(id)) {
                old.add(id)
                try {
                    i++;

                    if (i < 6) {
                        def link = baseUrl + url
                        Document doc2 = Jsoup.connect(link).get();
                        Elements details = doc2.select("section.body")

                        result.append("<a href=\"" + link + "\"><h2>Visit</h2></a>")
                        result.append(details.toString());
                        result.append("<br><br><br>");
                    }
                } catch (Exception ex) {
                    ex.printStackTrace()
                }
            }
        })

        // Send e-mails
        if (result.size() > 0 && !firstStart) {
            def emails = config.getProperty("app.recipients").split(",");
            emails.each {
                simpleMail(config.getProperty("app.gmail.email"),
                    config.getProperty("app.gmail.password"),
                    it,
                    "${i} ads found",
                    result.toString()
                );
            }
        }

        // Append new adds to file
        def writer = new PrintWriter(file)
        old.each { id -> writer.println(id) }
        writer.close()
    }
{% endhighlight %}

This code is pretty simple. We have file *old* for accumulating ads which have been sent. After that we parse current results and check if we have it in list of sent ads. The implementation of *simpleMail* was grabbed from [here][1] without any changes - didn't want to write it by myself.
That's it! Now we can start using it. 

## Running the app

First of all we have to build our app. It's easy enough. We need to execute only one gradle task.

{% highlight bash %}
$ gradle uberjar
{% endhighlight %}

It produces *craigslist-1.0.jar*. Now we need to copy this file to folder where the app will be executed. Also you need to create *app.properties* with following content:

{% highlight properties %}
app.url=[craigslist search url (e.g. http://newyork.craigslist.org/search/mnh/nfa...)]
app.gmail.email=[gmail email which will be used as a sender]
app.gmail.password=[password for gmail account]
app.recipients=[email addresses of recipients separated by comma]
{% endhighlight %}

For statring the app I use small bashscript and start it by *cron* every 5 minutes.

{% highlight bash %}
cd /root/craigslist
java -jar /root/apts/craigslist-1.0.jar >> /root/craigslist/log
{% endhighlight %}

## Conclusion

Developement with groovy is pretty fast and effective. I spended ~1 hour for full development and configuration. I didn't get any troubles during development and building jar file and I got what I wanted.

<p>
<img src="{{ site.url }}/assets/craiglist.png" class="img-responsive">
</p>

All sources you can find in my [github profile][2] 

[1]: http://www.jedox.com/en/send-email-using-javamail-groovy-script/
[2]: https://github.com/dimafeng/craigslist
