Recently I strated learning clojure. It seems it's interesting language with functional concepts. I don't have expertise
in it and I would like to share my clojure-hello-world.

It's going to be a simple app that does a few simple functions:

* Exposes an http api with 2 methods: add item and read all items
* Stores data in mongodb

## Database layer

To work with mongo, I'll use [**Clojure Monger**](http://clojuremongodb.info/) library. If you look at its documentation
you'll see that it's really simple and clear. 

My data layer looks as follows:

{% highlight clojure %} 
(ns test.db
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

## Web portion

{% highlight clojure %} 
(ns test.core
   (:use ring.adapter.jetty)
   (:use ring.middleware.reload)
   (:require [compojure.core :refer :all]
             [compojure.route :as route]
             [compojure.handler :as handler]
             [ring.middleware.json :as middleware]
             [cheshire.core :refer :all]
             [cheshire.generate :refer [add-encoder]]
             [test.db :as db]))

(add-encoder org.bson.types.ObjectId
(fn [c jsonGenerator]
    (.writeString jsonGenerator (str c))))

(defroutes app-routes
    (GET "/" [] 
         (db/read-all))
    (GET "/:value" [value]
         (db/add-item value)
         (str value " added!"))   
    (route/not-found "<h1>Page not found</h1>"))

(def app
    (-> (handler/site app-routes)
              (middleware/wrap-json-body {:keywords? true})
                    middleware/wrap-json-response))

(defn boot []
    (run-jetty #'app {:port 8080}))
{% endhighlight %}
