---
layout: post
title:  "Scala testing with a human face"
categories: scala, spring framework, spring data, mongo, scala test
image: scala-spring.jpg
---
In the [previous post][1] I described the way we use scala and spring. Today I'm going to show you how we test it. Actually, you can inherit all practices from your java project, but here we decided to try [ScalaTest][2]. So, this blog post is about application ScalaTest for testing spring application. Let's get it started.

## Gradle configuration

First of all, you need to add ScalaTest as a dependency into your build config. Plus we need to add spring-test and json-path for controller tests. In my gradle script it looks as follows:

{% highlight groovy %}
testCompile("org.springframework.boot:spring-boot-starter-test:${springBootVersion}",
        "org.scalatest:scalatest_2.11:$scalaTestVersion",
        "com.jayway.jsonpath:json-path:$jsonPathVersion")
{% endhighlight %}

I'd been using the latest versions of these libraries at the time of writing of this post.

## ScalaTest key concepts

Before we start let's answer the following questions:

### How to start the tests?

Now we need to understand to start tests. If you write a simple test and execute `gradlew test` the test won't be found, gradle knows nothing about ScalaTest. Here we've got two options:

* Use [ScalaTest plugin for gradle][4]
* Run tests as jUnit tests

The first option looks not so solid due to the number of stars and forks on github. The second one is better and widely suggested by the community.
Moreover, the second option allows you to use gradle junit features like parallel execution and reports. To run scala tests as jUnit one you just need to annotate them with `@RunWith(classOf[JUnitRunner])` - pretty simple.

### How to organize the code?

ScalaTest offers you a wide range of different approaches and concepts. You either can go with the totally different world of scala or with the approach that is very similar to junit. Which one is better is a good question. We picked the median of both.

If you go to [ScalaTest user guide][3] you'll find out that this library has a lot of different tests formats.
The documentation says:

>If you would rather be told which approach to take rather than pick one yourself, we recommend you use FlatSpec for unit and integration testing and FeatureSpec for acceptance testing.

Let's get this advice.

### What mock library to use?

The next question is how to mock our code. ScalaTest supports ScalaMock, EasyMock, JMock, Mockito. We can skip EasyMock and JMock because they provide almost the same functionality as Mockito, but Mockito is more popular these days.
Now, let's compare Mockito and ScalaMock. ScalaMock is written in scala and supports all scala specific features. Sounds good, but we're limited by spring framework in the main part of the project and we can't use scala at its full potential (maybe it's good) and you'll have to learn a new library/concepts. As for mockito, there is nothing bad with it, we have experience with it and we're totally satisfied.

## Simple unit test

We’re all setup and ready to go. Let's write some code/tests.

{% highlight scala %}
@RunWith(classOf[JUnitRunner])
class LinkTest extends FlatSpec {
  behavior of classOf[Link].getSimpleName

  it should "create a proper link for an instance of BlogPost" in {
    val link = Link.createLink(new BlogPost().copy(name = "test name", id = "42"))

    assert(link.title == "test name")
    assert(link.url == "/blog/42")
  }
}
{% endhighlight %}

This very simple unit test. `behavior of classOf[Link].getSimpleName` defines the value that will replace `it` and the test will be shown in log/test-runner as *Link should create a proper link for an instance of BlogPost*. The good thing about it is that this test is self-descriptive. You understand what this test checks without deep understanding of the test's code.

Another good thing in this test is assertions. They are very minimalistic and readable. And it shows pretty output in case of fail (not boolean value as you could assume):

<pre>
org.scalatest.exceptions.TestFailedException: "[test name]" did not equal "[name test]"
</pre>

If you want to go deeper you can try Matchers. You need to extend `Matchers` trait and then you'll be able to rewrite `assert(link.title == "test name")` as follows:

{% highlight scala %}
link.title should be === "test name"
{% endhighlight %}

See [this documentation][5] to get more information about matchers.

## Service test

Tests for service layer are simple as previous. Let's start with an abstract class that will specify common functionality.

{% highlight scala %}
@RunWith(classOf[JUnitRunner])
abstract class ServiceSpec(cls: Class[_]) extends FlatSpec with Matchers with MockitoSugar with BeforeAndAfterEach {
  behavior of cls.getSimpleName // (1)

  override def beforeEach() = MockitoAnnotations.initMocks(this) // (2)
}
{% endhighlight %}

* (1) as I described in previously, `behavior of` allows you to specify what we're going to test. In this case, we'll obtain the target name from the class that will be passed to the constructor.
* (2) this initializes mocks defined as a class field with annotation @Mock

Now, we can write a test.

{% highlight scala %}
class BlogPostServiceSpec extends ServiceSpec(classOf[BlogPostService]) {

  @Mock var blogPostRepository: BlogPostRepository = _

  it should "generate proper collection of links for given categoryId" in {
    val blogPostService = new BlogPostService(blogPostRepository)

    when(blogPostRepository.findByCategoryId("123"))
      .thenReturn(Array(new BlogPost().copy(id = "321",name = "test blog post")))

    val result = blogPostService.blogPostsByCategoryId("123")

    assert(result == Seq(Link("test blog post", "/blog/321")))
  }
}
{% endhighlight %}

If you want to define mock that will be used only in specific test you may use ScalaTest's sugar. It may look like:

{% highlight scala %}
val blogPostRepository = mock[BlogPostRepository]
val blogPostService = new BlogPostService(blogPostRepository)
...
{% endhighlight %}

## Controller test

Controller tests are more tricky. There is a standard way to test controllers - `MockMvc`. And it can work in 2 modes:

* Isolated (`MockMvcBuilders.standaloneSetup`) - builds just a controller class without the application context.
* Full context (`MockMvcBuilders.webAppContextSetup`) - starts an application context and maps all controllers from this context.

If this project was a java project we would able go with isolated in most cases, but for scala one it doens't work. The main reason is that we use custom mappers for json to let spring process case classes and scala collections correctly. So, we have no other option - let's code!

First of all we need a abstract class for all controller tests:

{% highlight scala %}
@RunWith(classOf[JUnitRunner])
abstract class ControllerSpec(controllerClass: Class[_]) extends FlatSpec with Matchers with MockitoSugar with BeforeAndAfter {
  behavior of controllerClass.getSimpleName

  var mvc: MockMvc = _
  @Autowired
  val webApplicationContext: WebApplicationContext = null // (1)

  before {
    new TestContextManager(this.getClass).prepareTestInstance(this)
    mvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build()
  }
}
{% endhighlight %}

Here we autowire a `webApplicationContext` (1) that we will create for each test separately.

{% highlight scala %}
@ContextConfiguration(classes = Array(classOf[Config], classOf[WebConfig])) // (1)
@WebAppConfiguration
class BlogPostRestControllerSpec extends ControllerSpec(classOf[BlogPostRestController]) with BeforeAndAfterEach {

  it should "assign categories to a blog post on /blogPosts/1234/addCategory" in {
    mvc.perform(post("/blogPosts/1234/addCategory")
      .contentType(MediaType.APPLICATION_JSON)
      .content("{\"categoryId\": [\"111\", \"222\"] }"))
      .andExpect(status().isOk())
    verify(blogPostService).addCategoryToBlog("1234", Array("111", "222"))
  }

  override def beforeEach() = {
    super.beforeEach()
    reset(blogPostService) // (5)
  }
}

object ReportApiControllerSpec {
  val blogPostService = Mockito.mock(classOf[BlogPostService]) // (2)

  @Configuration
  class Config {
    @Bean def blogPostServiceBean = blogPostService // (3)
    @Bean def controller = new BlogPostRestController(blogPostService) // (4)
  }
}
{% endhighlight %}

First what you need to look at (1). This annotation defines all the configurations which will be used for application context creation. As you see, there two configurations `Config` and `WebConfig`. `WebConfig` is a config of application, where we have beans for proper json mapping. `Config` is a config defined for this test. Here we need a controller that we're going to test (4) and all dependencies for the controller (3). All dependencies will be defined as mocks (2) and should be reset after each test (5).

## Conclusion

I've been writing tests using ScalaTest for about 3 weeks and probably didn't get all the beauty of this approach. At this point, it's difficult to assess, so I can summarize what I liked and what I didn't like.

### What I like about it:

* Self-descriptive test cases, they are readable and maintainable. I think this is the main point that should be taken into consideration.
<p>
<img src="/assets/scalatest-gradle.png" />
</p>

* Assertions and some matchers. I like writing `assert(value == 2)` and `assert(result.size > 10)`, it's concise and produce nice output as a result.
* `FeatureSpec`, I haven't covered this subject in the post, but it looks promising. I believe it can play good game at integration testing (this approach is pointless in other cases as almost all components are stateless, at least, I'm trying to write and keep them stateless)

### What I don't like:

* Verbosity in the specification. While I'm using just one annotation `@Test` in jUnit test, here I have to add one annotation, a bunch of traits and register name of the subject - I don't want to remember it and don't want to copy and paste it.
* Complex matchers with DSL. I'm not big fan of DSLs especially whey they make my code look like a poem and I don't want to learn a new language inside a language. This point is very subjective, so you can ignore it if you like DSL.
* Property-based tests look a bit weird for me. Let's look at example:

{% highlight scala %}
@RunWith(classOf[JUnitRunner])
class ControllerPropertyTest extends PropSpec with TableDrivenPropertyChecks with Matchers {

  val entities = Table(
    "value",
    Int.box(1),
    BlogPost("test", "test", Array(), id = "1"),
    Category("test category", id = "2"),
    Int.box(2)
  )

  property("controller should generate non-empty links for all valid entities") {
    forAll(entities) { e =>
      Controller.urlForEntity(e) should not be empty
    }
  }
}
{% endhighlight %}

I assumed that this code generates 5 separate tests, but in fact, it doesn't. So, my function `urlForEntity` expects String, BlogPost, and Category, and returns empty string otherwise. Not, look at the test's output:

<pre>
TestFailedException was thrown during property evaluation. (ControllerPropertyTest.scala:22)
  Message: "" was empty
  Location: (ControllerPropertyTest.scala:23)
  Occurred at table row 0 (zero based, not counting headings), which had values (
    value = 1
  )
</pre>
As for me, there are 2 problems here: 1) message is too verbose 2) it failed on `1` and didn't go ahead to find that `2` is incorrect as well.

[1]: /2016/01/02/scala-spring/
[2]: http://www.scalatest.org/
[3]: http://www.scalatest.org/user_guide/selecting_a_style
[4]: https://github.com/maiflai/gradle-scalatest
[5]: http://www.scalatest.org/user_guide/using_matchers
