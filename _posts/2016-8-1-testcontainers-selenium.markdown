---
layout: post
title:  "Integration testing using docker"
categories: docker, selenium, scala
image: testcontainers-scala.png
---

A few weeks ago I published a post on Reddit where I [tried to advertise][2] my lib - [testcontainers-scala][1].
This failed, I got no feedback at all. This prompted me to write this post and show how awesome integration testing might be.

Here I'm going to show a simple example of selenium tests. Let's get it started.

## Testcontainers

[Testcontainers-scala][1] is a wrapper for [testcontainers-java][4] that allows using docker containers for functional/integration testing.

>TestContainers is a Java 8 library that supports JUnit tests, providing lightweight, throwaway instances of common databases, Selenium web browsers, or anything else that can run in a Docker container.

## Assumptions

* You know how to use SBT
* You use scala 2.11
* You run all of these on java >8

If all of those points are true then we're ready to write some code.

## Build script

I'm not going to make it complex, so here's the simplest minimalistic `build.sbt` that we need to start. It's enough for showing the power of **testcontainers**.

{% highlight scala %}
name := "testcontainers-example"

version := "1.0"

scalaVersion := "2.11.8"

libraryDependencies += "com.dimafeng" % "testcontainers-scala" % "0.2.0" % "test" //testcontainers itself
libraryDependencies += "org.scalatest" % "scalatest_2.11" % "2.2.6" % "test"
libraryDependencies += "org.testcontainers" % "selenium" % "1.1.0" % "test" // testcontainer-java module for selenium
libraryDependencies += "org.seleniumhq.selenium" % "selenium-java" % "2.53.0" % "test"
libraryDependencies += "org.slf4j" % "slf4j-simple" % "1.7.21" // logger implementation
{% endhighlight %}

This is pretty standard build script, so there's no problem just to copy and paste this into `build.sbt` of your existing project.

<div class="disclaimer">
<h1>Important note</h1>
The version of <strong>selenium-java</strong> links to a version of <strong>selenium/standalone-chrome-debug</strong> container. If you are using a version different from 2.53.0 please check a version of the container manually. See <a href="https://hub.docker.com/r/selenium/standalone-chrome-debug/tags/">the official docker hub page</a>
</div>

## Specs

Now we can write some specs. Since I have no prepared backend, I'm going to write specs in terms of [scalatest][3] which test reddit.com.

{% highlight scala %}
package com.dimafeng.scala.example.testcontainers

import java.io.File

import com.dimafeng.testcontainers.SeleniumTestContainerSuite
import org.openqa.selenium.remote.DesiredCapabilities
import org.scalatest.FlatSpec
import org.scalatest.selenium.WebBrowser
import org.testcontainers.containers.BrowserWebDriverContainer.VncRecordingMode


class RedditSpec extends FlatSpec with SeleniumTestContainerSuite with WebBrowser {
  override def desiredCapabilities = DesiredCapabilities.chrome() // (1)
  override def recordingMode = (VncRecordingMode.RECORD_FAILING, new File("./")) // (2)

  // (3)
  "Reddit" should "show non-zero number of links on the main page" in {
    go to "https://www.reddit.com/"

    assert(cssSelector("#siteTable .thing").findAllElements.nonEmpty)
  }

  // (4)
  "Reddit" should "redirect to https" in {
    go to "http://www.reddit.com/"

    assert(currentUrl startsWith "https://www.reddit.com")
  }

  // (5)
  "Reddit" should "stay reddit after click on a random link" in {
    go to "https://www.reddit.com/"

    click on cssSelector(".entry a.title").findAllElements.filter(_.isDisplayed).next()

    println(currentUrl)
    assert(currentUrl startsWith "https://www.reddit.com")
  }
}
{% endhighlight %}

This is all you need to test your GUI inside the docker. Now let's break it down. We have `SeleniumTestContainerSuite` trait that does all the magic under the hood. It starts a clean new container on each test case, so you don't need to worry about the state - no data shared between test cases. To make it work in a basic scenario you just need to configure `DesiredCapabilities` (1) - here it's chrome, but you can use firefox as well.

There are three test cases (3-5). The first two (3-4) are regular selenium tests wrapped with nice **scalatest** DSL.
And the last one is a test that sometimes fails. But you can say that there is no way to figure out what is going on inside the container and this may be true, but not in our case. As you see in (3) we ask **testcontainers** to record all failed tests and produce `.flv` file. This is how it looks like:

<p>
<img src="/assets/testcontainers-selenium/testcontainers-selenium.gif" />
</p>

The tests case checks that after clicking on a first link in the list a user won't be redirected to a foreign domain, in fact, he will.

## Conclusion

I had a lot of problems with old fashioned approaches like Chrome/Firefox/Remote/etc. drivers running them locally and on CI servers, and I had a tough time with modern approaches like **phantom.js**. This seems much more reliable and convenient because:

* **Test are reproducible:** if it fails on CI server it fails on your dev environment and visa verse.
* **Each test case is stateless:** you don't need to worry about cleaning cookies, local storage, etc. Each test works with its own clean browser.
* **Easy to start in parallel:** since there's no state, you can run it in parallel.
* **Simple troubleshooting:** if a test fails you can watch recorded video or fallback to local Chrome/Firefox drivers.

<p style="text-align: center;">
<img style="width: 300px;" src="/assets/testcontainers-selenium/Shut-up-and-take-my-money.jpg" />
</p>

It's free and open-sourced - [github project with all instruction][1]. Your contributions are very welcome.

[1]: https://github.com/dimafeng/testcontainers-scala
[2]: https://www.reddit.com/r/scala/comments/4s695y/use_docker_containers_for_testing_in_scala/
[3]: http://www.scalatest.org/
[4]: https://github.com/testcontainers/testcontainers-java
