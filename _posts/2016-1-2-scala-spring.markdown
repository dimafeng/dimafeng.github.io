---
layout: post
title:  "Scala with a human face"
categories: scala, spring framework, spring data, mongo
image: scala-spring.jpg
---
I've been interested in scala since it has been mentioned during the course of functional programming in my university (thank you [@alexey_r][7] for the introduction to FP), but I haven't had a chance to try it (especially in production). Two weeks ago we started a new project using scala. We decided to give a try in the brand new project and then we'll be able to migrate good practices to other projects. We had following impulses to try scala:

1. to have immutability out of the box;
2. to have less verbose syntax;
3. to have more declarative approach;

at the same time, the project structure and code itself had to be clear for the rest of the team. That's why we picked production proven tech-stack: gradle, spring boot, spring data, mongo and started to write it in scala.

In this blog post, I'm going to describe all challenges and problem which we got and which we've solved so far. Here I use my demo project. It's pretty simple, but I tried to show as many aspects of scala development as I can. So, let's dive in.

<div class="disclaimer">
<h1>Disclaimer</h1>
I've been writing in scala for 2 weeks. This example can hurt the feelings of functional programming nerds. Moreover, we're writing code that is clear for the teammates who less familiar with scala i.e. we use this approach intentionally.
</div>

Anyway, if you know how to improve any parts of our approach you're welcome to comment.

## Project Structure

As I mentioned above we've picked gradle, so, this project has not to many differences from standard java/gradle layout.

{% highlight bash %}
➜  scala_spring (master) ✗ tree
.
├── build.gradle
├── scala_spring.iml
├── settings.gradle
└── src
    └── main
        ├── resources
        │   ├── application.properties
        │   └── views
        │       ├── base.html
        │       ├── blog.html
        │       ├── category.html
        │       └── index.html
        └── scala
            └── com
                └── dimafeng
                    └── examples
                        └── scala_spring
                            ├── Application.scala
                            ├── controller.scala
                            ├── dao.scala
                            ├── model.scala
                            ├── service.scala
                            └── util.scala
{% endhighlight %}

Only a few things should be added to `build.gradle` to make this project the scala one.

* Add scala plugin: `apply plugin: 'scala'`
* Add scala runtime to dependencies: `compile("org.scala-lang:scala-library:2.11.7")`

Now we can write scala classes. Due to the fact that scala code not verbose, we can put multiple classes and traits into one file. As you can see in the project tree, we have scala files which start with lowercase, these files correspond to appropriate package and contain all classes belong to a logical unit.

## Basics

The entry point to the application is the `Application` singleton and class annotated with spring configuration annotations.

{% highlight scala %}
object Application extends App with LazyLogging {
  logger.info("App is being started with {}", args)
  SpringApplication.run(classOf[Application], args: _*)
}

@SpringBootApplication
class Application {

  @Bean
  def viewResolver(): ViewResolver = {
    new JtwigViewResolver() {
      setPrefix("classpath:/views/")
      setSuffix(".html")
    }
  }

  ...
}
{% endhighlight %}

This looks almost as a java version of spring boot app. As well as in java you can use any spring annotations, define `@Autowired`, extend spring basic classes and interface, etc.

### Logging

For logging, you don't have to write ugly log definition in each class anymore

{% highlight java %}
private static final Logger log = Logger.getLogger(ClassName.class.getName());
{% endhighlight %}

Instead, you're able to use logging mixin. You need just to add dependency `com.typesafe.scala-logging:scala-logging-slf4j_2.11:$scalaLoggingVersion` and extend the trait `LazyLogging`. More info about logging is [here][1].

### Don'ts

There should have been more rules, but I'll describe two main ones:

* **Never use `scala.collection.JavaConversions`**. If your DAO layer or some libraries return or takes java collections you should use `JavaConverters` explicitly. Otherwise, the code will be very fragile. And it'll be difficult to test it.

* **Don't use method delegation for public methods in this way:
`def myMethod = service.method _`**. It looks good and minimalistic but doesn't help in a further perspective. Almost all libraries which based on types won't work with these methods correctly (e.g. mockito won't mock it). Also, we found out that it's very useful to define a return type for public methods which return collections to define more basic type explicitly (e.g. `Seq[_]` instead of `mutable.Buffer` that was inferred by scala compiler).

## Controller and View

A simple controller looks exactly the same as java equivalent.

### Rest Controllers

{% highlight scala %}
@RestController
@RequestMapping(Array("/users"))
class UserRestController @Autowired()(userService: UserService) {

  @RequestMapping(value = Array(""), method = Array(POST))
  def add(@RequestBody user: User): User = userService.add(user)

  @RequestMapping(value = Array(""), method = Array(GET))
  def all(): Seq[User] = userService.findAll()

  ...
}
{% endhighlight %}

Pretty clear, right?

We like the way we can autowire beans via constructor, we use it over the whole project. It's convinient to write and convinient to use in unit tests. The only thing with this autowiring appoach - it doesn't work with cyclic bean dependencies.

In the example above, I'm using scala collections and case classes as the structures which will be serialized into json (or whatever format that you configured) by spring converters. Out the box, spring converters don't work properly with these structures. To let spring understand these structures, you need to add following dependency `com.fasterxml.jackson.module:jackson-module-scala_2.11:${jacksonScalaVersion}` and define a bean for custom mapper:

{% highlight scala %}
@Configuration
@EnableWebMvc
class WebConfig extends WebMvcConfigurerAdapter {
  override def configureMessageConverters(converters: util.List[HttpMessageConverter[_]]): Unit =
    converters.add(jackson2HttpMessageConverter())

  @Bean
  def jackson2HttpMessageConverter(): MappingJackson2HttpMessageConverter =
    new MappingJackson2HttpMessageConverter(objectMapper())

  @Bean
  def objectMapper(): ObjectMapper =
    new ObjectMapper() {
      setVisibility(PropertyAccessor.FIELD, Visibility.ANY)
      registerModule(DefaultScalaModule)
    }

  ...
}
{% endhighlight %}

Now you can return and pass to controller methods all scala collections, case classes and even case classes with inherited fields.

### Classic Controllers with View

We decided to use modern view template engine - [Jtwig][2]. It's simple and powerful at the same time. The only thing, it doesn't work with scala collections, but it's easy to solve. We came up with the `ModelAndView` wrapper that looks like this:

{% highlight scala %}
class MV(val view: String, val attributes: Map[String, _]) extends ModelAndView {
  setViewName(view)
  addAllObjects(toJava(attributes).asInstanceOf[java.util.Map[String, _]])
}
{% endhighlight %}

`toJava` function we found [here][3] and now this wrapper may be used as follows:

{% highlight scala %}
@RequestMapping(Array(""))
def mainPage() =
  MV("index", Map(
    "blogs" -> blogPostService.linksToAllBlogPosts()
  ))
{% endhighlight %}

## Service Layer

A service layer is the most expressive part where scala works very well. Here we almost have no limits related to java conversions.
Typical service will look like:

{% highlight scala %}
@Service
class BlogPostService @Autowired()(blogPostRepository: BlogPostRepository) {

  def blogPostsByCategoryId(categoryId: String): Seq[Link] =
    blogPostRepository.findByCategoryId(categoryId).map(createLink)

  def linksToAllBlogPosts(): Seq[Link] = blogPostRepository.findAll()
    .asScala
    .toSeq
    .map(createLink)

  ...
}

case class Link(title: String, url: String)

object Link {
  def createLink(entity: Model) = entity match {
    case b: BlogPost => Link(b.name, urlForEntity(b))
  }
}
{% endhighlight %}

I believe there is nothing to say here anything else. Just use scala with all its power.

## DAO Layer

In our project, we're using mongo with spring data. Here I'll show you how we adapted it to scala enviroment.

### Model

Mongo driver doesn't prevent you from using of case classes and this is very good because this approach has a lot of useful and nice applications (especially pattern matching).

I like the way when all common parts go to basic class, so my abstract model looks like this:

{% highlight scala %}
@Document
class Model(@Id val id: String, @(Version@field) @(JsonIgnore@field) val version: Int)
{% endhighlight %}

`@Version` annotation is related to optimistic locking (see **Optimistic Locking** bellow). And we don't want to expose this field via rest api, that's why we have `@JsonIgnore`.

All model classes extend this main one.

{% highlight scala %}
@TypeAlias(BLOG_COLLECTION_NAME)
@Document(collection = BLOG_COLLECTION_NAME)
case class BlogPost(name: String,
                    body: String,
                    categoryIds: Array[String],
                    override val id: String = null,
                    override val version: Int = 0) extends Model(id, version)
{% endhighlight %}

It's wise to specify `@TypeAlias` and collection name in `@Document`, in this case, you won't have any difficulties with refactoring, and also will be able to use these constants in queries (e.g. when you're querying dbref fields).

### Repositories

Repositories can be defined using scala traits like so:

{% highlight scala %}
trait BlogPostRepository extends PagingAndSortingRepository[BlogPost, String] with BlogPostRepositoryCustom {

  @Query("{'categoryIds': ?0 }")
  def findByCategoryId(categoryId: String): Array[BlogPost]
}
{% endhighlight %}

When you expect a collection as a return type it's convenient to use scala array because it can be treated as java array but you don't loose all function features like filter, map, etc. Moreover, it's immutable structure and that's good. In the new version of Spring Data Mongo repositories will also support scala option as a return type.

### Custom Queries

To make queries more readable we're using [casbah] that adds some syntax sugar. Also we come up with a bunch of utility conversions:

{% highlight scala %}
object Mongo {
  val MO = MongoDBObject

  implicit def dbObjectToQueryGenerator[T <: DBObject](value: T): QueryGenerator =
    new QueryGenerator(value)

  class QueryGenerator(val dbObject: DBObject) {
    def toQuery: BasicQuery = new BasicQuery(dbObject)
    def toUpdate: BasicUpdate = new BasicUpdate(dbObject)
  }
}
{% endhighlight %}

This code allows you write queries in this way:

{% highlight scala %}
mongoTemplate.findAndModify(
      MO("_id" -> new ObjectId(blogPostId)).toQuery,
      $addToSet("categoryIds" -> categoryId).toUpdate,
      classOf[BlogPost]
)
{% endhighlight %}

### Optimistic Locking

I don't know the reason, but the official spring data documentation doesn't say anything about optimistic locking. I was told how it works by my co-worker. The good thing about it is that you just need to add a special field to your model and annotate it with `@Version` (this annotation is from `org.springframework.data.annotation`, not from `javax.persistence`). As I mentioned before, my model looks as follows:

{% highlight scala %}
@Document
class Model(@Id val id: String, @(Version@field) @(JsonIgnore@field) val version: Int)
{% endhighlight %}

Now, if 2 threads read the same document then one of them changes some fields and saves it, another one won't be able to same the object in this way, it should re-read it. This works out of the box with all build-in Repositories methods and methods described by dsl. If you're going to write your custom queries, you should add another method to `QueryGenerator`. It may look like this:

{% highlight scala %}
  class QueryGenerator(val dbObject: DBObject) {
    def toUpdateWithLock: BasicUpdate = {
      new BasicUpdate($inc("version" -> 1) ++ dbObject)
    }
    ...
  }
{% endhighlight %}

Queries should be written in this way:

{% highlight scala %}
mongoTemplate.findAndModify(
      MO("_id" -> new ObjectId(blogPostId)).toQuery,
      $addToSet("categoryIds" -> categoryId).toUpdateWithLock,
      classOf[BlogPost])
{% endhighlight %}


## Unsolved Problems

During the development, we met problems which we haven't managed to solve.

### Custom annotations

Sometimes, custom annotations are very needed. In scala, this task isn't so trivial as I expected. You're able to create own annotations in scala but they won't be available in runtime (actually, scala documentations says that [there's a way to do it][5], but it looks much more unreadable, more boilerplate code required and it's marked as the **EXPERIMENTAL** feature).

**Workaround**: We decided to define annotations in java.

### Spring Aspects

The standard way to create aspects in spring doesn't work in all cases. For example, we haven't managed to create an aspect for a controller, but it works for services, though.

**Workaround**: Use [`HandlerInterceptor`][6].
**UPD**: We've solved this issue, in fact it's not even an issue. Please make sure, you use CGLIB proxies instead of java-proxies when your controller extends some traits.

### @DBRef annotation

We spent a lot of time to make these annotations work in both modes lazy and not-lazy. Our goal was to declare lazy DBRef's to a collection of model objects (as a collection implementation, we wanted to use scala immutable List). The logic that makes conversions from proxy a collection/array to a collection of fetched models is hardcoded to work with java collections.

**Workaround**: For non-lazy references, you can use scala array without any additional changes. For lazy references, use Array/List/Seq of models' ids/DBRef's rather than `@DBRef` to a collection of models.

## Conclusion

Scala in Spring Framework environment has good and bad sides. We're not happy about the fact that spring framework knows nothing about scala collections and structures like `Option`. Another bad thing is that Intellij Idea's spring support doesn't work with scala - you won't be able navigate through the project efficiently and also there won't be smart, spring specific autocompletion. On the other hand, you'll get more declarative code, immutable structures, pattern matching, less verbose code. Is it worth? It depends on how deep you want to integrate your code with spring.

I didn't cover unit tests in scope of this article. I'll write another article that will be describing usage of scala tests with examples from this article.

Overall, I feel like it's good start for diving in FP. And I considered it as a lighweigh start of using of FP languages in production. In next projects, I would go with scala frameworks like [playframework][8] or [scalatra][9].

The working examples of all code snippets mentioned in this article, you can find [here in my GitHub][10].

[1]: https://github.com/typesafehub/scala-logging
[2]: http://jtwig.org/
[3]: http://stackoverflow.com/a/32380463/315287
[4]: https://mongodb.github.io/casbah/
[5]: http://docs.scala-lang.org/overviews/reflection/annotations-names-scopes.html
[6]: http://docs.spring.io/spring-framework/docs/3.2.4.RELEASE/javadoc-api/org/springframework/web/servlet/HandlerInterceptor.html
[7]: https://twitter.com/alexey_r
[8]: https://www.playframework.com
[9]: http://www.scalatra.org/
[10]: https://github.com/dimafeng/dimafeng-examples/tree/master/scala_spring
