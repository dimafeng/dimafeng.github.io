---
layout: post
title:  "Not obvious scala features for java developers"
categories: scala
image: scala-logo.png
---

A week ago I finished reading of [Scala for the Impatient][2] by Cay Horstmann. Here are some lessons which I learned from this book.

## 1. Method parentheses

For warming up, here's the first tip - it's well known that you can omit using of parentheses for methods without parameters. But there is a rule in scala world: you can omit parentheses on methods which don't mutate a state.

{% highlight scala %}
class State {
  private var value = 0

  def change(): Unit = {
    value = Random.nextInt()
  }

  def current = value
}

...

s.change() // With parentheses
s.current // Without parentheses
{% endhighlight %}

## 2. Traits vs abstract classes

Java 8 brings a new experience to interface usage - interface methods can have a default implementation.
But scala eliminates the distinction even more (moreover, it works on JVM < 8): scala traits can have a default method implementation, abstract and defined values and variables. That's why, in scala world, there is a rule: if you don't know that to use traits or abstract classes - use traits, you'll always be able to change them to abstract classes.

{% highlight scala %}
trait Car {
  val color: String

  def print() = {
    println(s"Car is ${color}")
  }
}
...
new Car {
  override val color: String = "red"
}.print()
{% endhighlight %}

### Why is it good

* You can use traits as [mixins][1]. You even can use it during object creation:

{% highlight scala %}
abstract class Car {
  def specs() = {
    print(s"Car with ${wheels} wheels")
  }

  def wheels: Int
}

trait FourWheel {
  def wheels = 4
}
...
(new Car with FourWheel).specs() // Output: Car with 4 wheels
{% endhighlight %}

### Why is it bad

* You can't define constructor
* Most features aren't supported in java. So, you loose java interoperability
* This approach has lesser performance

## 3. ProcessBuilder DSL

There is a nice feature ti execute external one or more external processes from scala code using very convinient DSL:

{% highlight scala %}
object CMDExample extends App {
  println("ls" !) //(1)
  println("ls" !!) //(2)
  println("ls" #| "grep gradle " !!) //(3)
}
{% endhighlight %}

1. Executes `ls` within working directory and prints to stdout, then returns exit code.
2. The save as 1. but returns command output as a string
3. Equivalent of `ls | grep gradle`

## 4. Regex

Scala provides an easy way to work with java RegEx api. There's a class and companion object which hold all this convenience - `scala.util.matching.Regex`.

To create an instance of `Regex`, you can use method `r` mixed to a string. It may looks like this: `"\\d+".r`. Then, you can work with it as follows:

### Find matches
{% highlight scala %}
val regex = s"""<a\\s+href=\"([^\"]*?)\"\\s*>([^<]*?)</a>""".r
val regex(href, name) = "<a href=\"test.html\">test</a>"
println(href + ":" + name) // Output: test.html:test
{% endhighlight %}

For multiple matchings:

{% highlight scala %}
val text =
    s"""
       |<a href="index.html">index</a>
       |<a href="secondary.html">secondary</a>
   """.stripMargin

for(regex(href,_) <- regex.findAllIn(text)) {
    println(href)
}
{% endhighlight %}

### Replace

{% highlight scala %}
println(regex.replaceAllIn(text, { m => m.group(2) }))
{% endhighlight %}

The output of this line will be a string where all `a` tags replaced with a content of those tags.

## 5. Structural type

Structural type is very interesting feature. It allows to define method argument of any type but with requirement of presence of specific methods. This is how it looks like:

{% highlight scala %}
def printInfo(obj: {def info: String}) = println(obj.info)

class A {
  def info: String = "A"
}

class B {
  def info: String = "B"
}

printInfo(new A)
printInfo(new B)
{% endhighlight %}



[1]: https://en.wikipedia.org/wiki/Mixin
[2]: http://www.amazon.com/Scala-Impatient-Cay-S-Horstmann/dp/0321774094
