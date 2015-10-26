---
layout: post
title:  "Clojure Hello World"
categories: clojure
---
Recently I strated learning clojure. It seems it's interesting language with functional concepts and deep integration with JVM. I've read the book ['The Joy of Clojure'](https://www.manning.com/books/the-joy-of-clojure-second-editio://www.manning.com/books/the-joy-of-clojure-second-edition) and watched several screencasts and video records from conferences.

You can start with this one:

<iframe width="620" height="415" src="https://www.youtube.com/embed/VVd4ow-ZcX0" frameborder="0" allowfullscreen></iframe>

## Clojure

Here are the core concepts of clojure:

* Clojure is functional;
* It's LISP, so syntax looks wierd first time due to the big amount of parentheses, but then you'll realize it's very convenient for coding; 
* It has deep integration with JVM - it's run on JVM. It means all JVM libraries will work out the box;
* There is REPL and it's amazing;
* More info is [here](http://clojure.org/).

Here is the [Awesome Clojure](https://github.com/razum2um/awesome-clojure) page. And you should check this out.

## My Hello World

First of all, you need to install **leiningen**. You can find how to do it [here](http://leiningen.org/). And also you need to 
pick which IDE/editor you'll use for coding. Currently, I'm using vim and I wrote a [post about basic vim configuration](/2015/09/13/vim1/).

I don't have deep expertise in clojure yet and I just would like to share my clojure-hello-world application.

It's going to be a simple app that does a few simple functions: 

* Exposes an http api with 2 methods: add item and read all items
* Stores data in mongodb

To start new project, you need to execute one simple command:

{% highlight bash %} 
$ lein new hello-world
{% endhighlight %}

It'll create basic clojure project structure.

## Database layer
 
I poke around and found out that most common lib for mongo is [**Clojure Monger**](http://clojuremongodb.info/). If you look at its documentation
you'll see that it's really simple and clear. Now we need to add it to dependencies.

Go to `project.clj` in the root of the project and modify `:dependencies` as follows:

{% highlight clojure %} 
:dependencies [[org.clojure/clojure "1.7.0"]
               [com.novemberain/monger "3.0.0-rc2"]]
{% endhighlight %}

Then we're ready to create namespace where all functions related to db will go. Create file `src/hello-world/db.clj` and write this code:

{% highlight clojure %} 
(ns hello-world.db
  (:require [monger.core :as mg]
            [monger.collection :as mc])
  (:import [com.mongodb MongoOptions ServerAddress]))

(def conn (mg/connect))
(def db   (mg/get-db conn "test-database"))

(def test-coll "test")

(defn add-item [value]
  (mc/insert-and-return db test-coll {:value value}))

(defn read-all []
  (mc/find-maps db test-coll))
{% endhighlight %}

It connects to mongo using default port and works with database named `test-database` and collection `test`.
Once you have written this code and saved it you can check this out in REPL. Type `(use 'hello-world.db)` in repl and you're all set to
test these functions.

<p>
<img src="/assets/clojure/db-check.gif" />
</p>

## Web portion

In clojure world, the most common approach for web development is [ring framework](https://github.com/ring-clojure/ring).

>Ring is a Clojure web applications library inspired by Python's WSGI and Ruby's Rack. By abstracting the details of HTTP into a simple, unified API, Ring allows web applications to be constructed of modular components that can be shared among a variety of applications, web servers, and web frameworks.

Let's add it to our dependencies:

{% highlight clojure %} 
:dependencies [[org.clojure/clojure "1.7.0"]
               [com.novemberain/monger "3.0.0-rc2"]
               [ring "1.4.0"]
               [ring/ring-json "0.4.0"]
               [compojure "1.4.0"]])
{% endhighlight %}

I added two more libraries `ring/ring-json` and `compajure`. The first one is a middleware to convert clojure maps to json in requests and 
responces. The second one is for very convinient routing.

{% highlight clojure %} 
(ns hello-world.core
   (:use ring.adapter.jetty)
   (:use ring.middleware.reload)
   (:require [compojure.core :refer :all]
             [compojure.route :as route]
             [compojure.handler :as handler]
             [ring.middleware.json :as middleware]
             [cheshire.core :refer :all]
             [cheshire.generate :refer [add-encoder]]
             [hello-world.db :as db]))

; Custom converter for ObjectId objects 
(add-encoder org.bson.types.ObjectId
(fn [c jsonGenerator]
    (.writeString jsonGenerator (str c))))

(defroutes app-routes
    (GET "/" [] 
        (db/read-all))
    (POST "/" request
        (let [value (:value (:body request))]
            (db/add-item value)
            (println (str value " is added")))
        "ok")   
    (route/not-found "Not found"))

(def app
    (-> (handler/site app-routes)
              ; Middlewares for json support
              (middleware/wrap-json-body {:keywords? true})
                    middleware/wrap-json-response))

(defn boot []
    (run-jetty #'app {:port 8080}))
{% endhighlight %}

That's it. All you need is to start the app. Go to the repl and type `(use 'hello-world.core)` and then `(boot)`. And you can test it:

{% highlight bash %} 
$ curl -X POST -H "Content-Type: application/json" -d '{"value":"my value1"}' http://localhost:8080/
{% endhighlight %}

and

{% highlight bash %}
$ curl -X GET http://localhost:8080/
{% endhighlight %}

## Conculsion

This is a very simple clojure example. And I like it. It's really small and expressive code. I bet the same demo in java would require 10 times more lines of code. It's worth noting that the big advantage of development in clojure is repl interaction - it's amazing. I know this small app can be dramatically improved and can be written canonically. But it's my first try and I know where I should dig into.
