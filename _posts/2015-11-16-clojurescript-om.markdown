---
layout: hipster
title:  "ClojureScript: Real world app"
categories: clojurescript, om, react.js
---
About 2 months ago I started learning clojure. Not too active - mostly on weekends. Recently, I wrote a [post](/2015/10/25/clojure-hello-world/) where I showed how easy is to build a small application connected to mongo db. Today I'm going to show you how to create a real world client-side application using clojurescript.

Clojurescript is a subset of clojure that compiles to javascript. Obviously, you can't use JVM classes and clojure libraries which were written for JVM, however it's still very powerful tool. These days, the most progressive approach for building client side applications is react.js. That's why when somebody is talking about clojurescrit they will definitely mention a framework which has react.js under the hood. The most popular react.js wrapper in clojurescript world is [om](https://github.com/omcljs/om), so, I'm going to use it.

<div class="disclaimer">
<h1>Disclaimer</h1>
I've been using clojurescript for 2 weeks. I don't have much experience with it. Everything below is my first experiment with clojurescrit. I doubt that it can be used in production without rethinking.
</div>


## What we're building

Today I'm going to create an admin panel for my blog. It'll be a modern admin panel with realtime markdown preview like [ghost blog engine](https://ghost.org/).

For this app, I created simple backend api which should emulate real application. The RESTful api looks as follows:

| Url        | Description          |
| ------------- |-------------|
| GET /articles      | Retrieves all articles |
| GET /articles/{id}      | Retrieves a full Article object      |
| PUT /articles | Saves a given article |

<p></p>

The web application will have following component placement:

<p>
<img src="/assets/cljs/layout.png" />
</p>

## Client side

First of all, we need to generate a project. There is a lein template called 'chestnut'.

>Chestnut is a Leiningen template for a Clojure/ClojureScript app based on Om, featuring a great dev setup, and easy deployment.

{% highlight bash %}
$ lein new chestnut <name>
{% endhighlight %}

Now, you are able to start the app.

{% highlight bash %}
$ lein repl
{% endhighlight %}

Within the repl session, you need to execute `(run)` form.

### Api

For communication with the backend, we're going to use [cljs-ajax](https://github.com/JulianBirch/cljs-ajax), so, add
`[cljs-ajax "0.3.14"]` to `:dependencies` in project.clj.

Let's create **api** namespace `api.cljs`.

{% highlight clojure %}
(ns admin.api
  (:require [ajax.core :refer [GET POST PUT]]
            [cljs.core.async :refer [put! chan <!]]))

(def base-url "http://localhost:8080")

(defn call
  ([method url]
   (call method url {}))
  ([method url body]
   (let [ch (chan)]
     (method (str base-url url) (merge {:format          :json
                                        :response-format :json
                                        :keywords?       true
                                        :handler         #(put! ch %)}
                                       body))
     ch)))

(defn get-all-articles
  []
  (call GET "/admin2/articles"))

(defn get-article
  [id]
  (call GET (str "/admin2/articles/" id)))

(defn save-article
  [article]
  (call PUT (str "/admin2/articles/" (:id article)) {:params article}))
{% endhighlight %}

The heart of all functions is the function `call`. This function returns channel (core.async) as a result - it should help in future when application becomes more complex.

### Routing and component bindings

Before we go to routing configuration, let's create app layout. If you open `resources/index.html` you'll see default html page. To make it work as I described we need to add 2 blocks:

{% highlight html %}
...
<div id="app-menu"></div>
...
<div id="app-content"></div>
...
{% endhighlight %}

You can place these blocks wherever you want. I used Twitter Bootstrap and created 2-column design (I'm not covering these steps in this blog post).

Now we can write the routing logic and component bindings. The om documentation says:

>Does Om provide routing?
>Om does not ship with a router and is unlikely to. However ClojureScript routing libraries exist that handle this problem quite well:...

I picked [Secretary](https://github.com/gf3/secretary) as it has more stars and forks than competitors. You need to add `[secretary "1.2.3"]` to your dependencies.

Here's my `core.cljs`:

{% highlight clojure %}
(ns admin.core
  (:require [goog.events :as events]
            [om.core :as om :include-macros true]
            [secretary.core :as secretary :refer-macros [defroute]]
            [admin.dashboard :as dsh]
            [admin.article :as art]
            [admin.articles :as arts]
            [admin.menu :as mn])
  (:import goog.History
           goog.history.EventType))

; Allows using print/println for browser console logging
(enable-console-print!)

(def history (History.))

(def menu-state
  (atom [{:name "Dashboard" :path "/"}
         {:name "Articles" :path "/articles"}]))

; State of the page with all articles
(def articles-state (atom {}))
; State of the page with specific article
(def article-state (atom {}))

(defroute "/" []
          (om/root dsh/dashboard navigation-state
                   {:target (. js/document (getElementById "app-content"))}))

(defroute "/articles" []
            (om/root arts/articles articles-state
                     {:target (. js/document (getElementById "app-content"))}))

(defroute "/article/:id" {:as params}
            ;Reset article page state
            (reset! article-state {:id (:id params)})

            (om/root art/article article-state
                     {:target (. js/document (getElementById "app-content"))}))

(doto history
  (goog.events/listen EventType.NAVIGATE #(secretary/dispatch! (.-token %)))
  (.setEnabled true))

(defn main []
  (om/root
    mn/menu
    menu-state
    {:target (. js/document (getElementById "app-menu"))}))

{% endhighlight %}

In this namespace I configured 3 routes. Once **Secretary** triggers event, appropriate route will be executed. The code

{% highlight clojure %}
(om/root arts/articles articles-state
         {:target (. js/document (getElementById "app-content"))})
{% endhighlight %}

binds the react.js component `arts/articles` with state `articles-state` to an html element with id "app-content".

So, on this step, we bound menu statically (we won't change this block) and dynamic part will be bound to block depending on an url.

As you see, we require `admin.*` namespaces in this one, it means it won't work until we create them - let's do it.

### Menu component

Let's create menu component. To write less code I found 2 helpful libraries:

* [om-tools](https://github.com/Prismatic/om-tools) reduces boilerplate of component creation.
* [sablono](https://github.com/r0man/sablono) helps to write html-like structures without hustle.

You need to add `[sablono "0.2.22"]` and `[prismatic/om-tools "0.3.12"]` to your dependencies.

{% highlight clojure %}
(ns admin.menu
  (:require [om.core :as om :include-macros true]
            [sablono.core :as html :refer-macros [html]]
            [om-tools.core :refer-macros [defcomponent]]))

(defcomponent menu-item [{:keys [path name]} owner]
              (render [_]
                      (html
                        [:li
                         [:a {:href (str "#" path)} name]])))

(defcomponent menu [app owner]
              (render [_]
                      (html
                        [:div [:ul.nav.menu (om/build-all menu-item app)]])))
{% endhighlight %}

I think, it's pretty clear and doesn't require any comments.

### Articles component

The article component will retrieve all articles and show them as list of links.

{% highlight clojure %}  
(ns admin.articles
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [om.core :as om :include-macros true]
            [sablono.core :as html :refer-macros [html]]
            [om-tools.core :refer-macros [defcomponent]]
            [usnbrs-admin-client.api :as api]
            [cljs.core.async :refer [<!]]))

(defcomponent article-link [{:keys [data item]} owner]
  (render [_]
    (html [:div [:a {:href (str "#/article/" (:id item))} (:name item)]])))

(defcomponent articles [data owner]
  (will-mount [_]
    (go (let [articles (<! (api/get-all-articles))]
          (om/update! data :articles-list articles))))
  (render [_]
    (html
      [:div.row
       [:div.col-lg-12
       [:div.panel.panel-default
        [:div.panel-heading "Articles"]
        [:div.panel-body (map #(om/build article-link {:data data :item %}) (:articles-list data))]
        ]]])))
{% endhighlight %}

### Article page component

This part is most complex one. At the beginning we need to do a preparation step. As I mentioned in description I want to build an article editor with real-time preview. So, we need a library which can process markdown on the client side. I found a few written in clojure/clojurescript, but I didn't manage to make them work in my app. That's why I found JS lib called [marked](https://github.com/chjj/marked).

I downloaded this lib and added to the bottom of the page as regular javascript:

{% highlight html %}
<script src="marked.min.js" type="text/javascript"></script>
{% endhighlight %}

Once we add it, **marked** can be executed as follows `(js/marked "my markdown")`. And it's pretty cool.

{% highlight clojure %}
(ns admin.article
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [goog.events :as events]
            [om.core :as om :include-macros true]
            [sablono.core :as html :refer-macros [html]]
            [om-tools.core :refer-macros [defcomponent]]
            [cljs.core.async :refer [<! chan put!]]
            [admin.api :as api]))

(def update-chan (chan))
(def save-article-chan (chan))

(defn get-text [data]
  (:text (:article @data)))

(defcomponent edit-form [data _]
  (render [_]
    (html [:div.edit-area
           [:textarea {:onChange      (fn [e] (let [val (.. e -target -value)]
                                                (put! update-chan val)
                                                (om/transact! data :article #(merge % {:text val}))))
                       :default-value (get-text data)}]])))

(defcomponent preview [data _]
  (will-mount [_]
    (go (loop []
          (let [changed-data (<! update-chan)
                marked (js/marked changed-data)]
            (om/update! data :preview marked)
            (recur)))))
  (render-state [_ _]
    (html
      [:div.preview-area {:dangerouslySetInnerHTML {:__html (:preview @data)}}])))

(defcomponent control-buttons [data _]
  (render [_]
    (html [:div [:button.btn.btn-primary
                 {:onClick #(put! save-article-chan (:article data))}
                 "Save"]])))

(defcomponent article [data _]
  (will-mount [_]
    ; Init state
    (go (let [art (<! (api/get-article (:id @data)))]
          (om/update! data :article art)
          (put! update-chan (get-text data))))
    ; Save handler
    (go (loop []
          (let [art (<! save-article-chan)]
            (prn art)
            (go (<! (api/save-article art))))
          (recur))))
  (render [_]
    (html
      [:div
       [:div.row [:div.col-lg-12 [:h1 (:name (:article @data))]]]
       (if (contains? @data :article)
         [:div.row
          [:div.col-md-6
           [:div.panel.panel-default
            [:div.panel-heading "Article"]
            [:div.panel-body (om/build edit-form data)]]]
          [:div.col-md-6
           [:div.panel.panel-default
            [:div.panel-heading "Preview"]
            [:div.panel-body (om/build preview data)]]]
          [:div.col-md-12
           [:div.panel.panel-default
            [:div.panel-body (om/build control-buttons data)]]]
          ]
         [:div.row [:div "Loading..."]])]
      )))
{% endhighlight %}

As you see, here we have more code than in previous parts. The entry point of this namespace is `article` component. It loads current article on the component init and start listening to events from the `save-article-chan`.
Once it obtain article object, it's ready to render article editor (block with textarea) and preview.

`edit-form` is simple. We have textarea and it triggers event when we type some text. After each entered symbol we update component state and put current text to the channel. At the same time, `preview` component reads from the channel and re-renders preview.

## Wrap up

I'll skip a dashboard component and show what I've got.

<p>
<img src="/assets/cljs/demo.gif" />
</p>

## Conclusion

As I said before, it's my first acquaintance with clojurescript. I spent 2 weekends to make it work. Here are some thoughts about it.

**Pros**

* I can say I like how my app looks and how code is structured. It's good that code is really small and expressive. If I got used to the clojurescript syntax and remembered all necessary forms I would be much more productive than in javascript;
* Immutable approach and pure functions make code more clear. But it's not obvious how to keep functions always pure.


**Cons**

* Weird messages and errors. Most errors are not clear, sometimes browser points to javascript sources of **goog** and it's really difficult to understand what is going on there;
* Documentation for most libraries is insufficient, especially for **om**. Hope it'll be solved.
