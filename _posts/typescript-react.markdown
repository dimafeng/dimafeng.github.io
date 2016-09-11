---
layout: post
title:  "Modern client side: typescript + react"
categories: typescript, react.js, es6, babel, javascript
image: testcontainers-scala.png
---
One year ago I wrote a [post][1] about **om** (clojurescrit library based on react.js). Now it's time to bring react to real life project
in "enterprice" world.

A few months ago, my team faced a necessity to build a new single page application. We used to love and hate (sometimes) **Angular 1** and have been actively
using it for two years, but **Angular 1** is dying and there's no point to start a new project with it. The first alternative to **Angular 1** is **Angular 2**.
Google made a lot of changes to it, too many changes, they've basically rewritten it using typescript. Brief look at **Angular 2** didn't impress us because of a few things: first, its api isn't finalized (it's broken every RC), second and the most important part, it became a monster, it's too difficult to start (let's recall how quickly you could start playing with **Angular 1**).

The next most obvious option is **react.js**. But **react.js** is not a framework but just a rendering library.
It means we have to build the app from small self-sufficient blocks by ourselves what is a little bit tricky. Moreover, in this project, we decided to try **typescript**, that should help us to write less tests and
catch typing errors at the compile time. And also we got adequate autocomplete in the IDE.

It this article I'm going to summarize my experience by building a simple application that will do the following things:

* Show a list of news
* Show content of the particular news

## Project structure

Let's start with a quick overview of the project structure.

{% highlight bash %}
.
├── package.json
├── src
│   ├── App.tsx
│   ├── Config.ts
│   ├── commons
│   │   └── Rest.ts
│   ├── index.html
│   ├── index.tsx
│   └── styles.css
├── tsconfig.json
├── typings.json
├── webpack.commons.js
├── webpack.config.js
└── webpack.production.config.js
{% endhighlight %}

In the following part of the post, we'll go through all files shown here.

## Libraries

To make all the steps more smooth, let's start with `package.json` file and define all dependencies.

{% highlight json %}
{
  ...

  "dependencies": {
    "es6-shim": "0.35.1",
    "lodash": "4.14.1",
    "react": "15.2.1",
    "react-autosuggest": "3.7.3",
    "react-bootstrap": "0.30.0",
    "react-dom": "15.2.1",
    "react-router": "2.6.0",
    "rest": "1.3.2",
    "rx": "4.1.0"
  },
  "devDependencies": {
    "babel-core": "6.10.4",
    "babel-loader": "6.2.4",
    "babel-preset-es2015": "6.9.0",
    "babel-preset-react": "6.11.1",
    "copy-webpack-plugin": "3.0.1",
    "css-loader": "0.23.1",
    "es6-shim": "0.35.1",
    "extract-text-webpack-plugin": "1.0.1",
    "file-loader": "0.9.0",
    "react-hot-loader": "1.3.0",
    "style-loader": "0.13.1",
    "ts-loader": "0.8.2",
    "tslint": "3.15.1",
    "tslint-loader": "2.1.5",
    "tslint-react": "1.0.0",
    "typescript": "1.8.10",
    "typescript-require": "0.2.9",
    "typings": "1.3.2",
    "url-loader": "0.5.7",
    "webpack": "1.13.2",
    "webpack-dev-server": "1.14.1",
    "json-loader": "0.5.4",
    "json5-loader": "0.6.0",
    "jsx-loader": "0.13.2"
  }
}
{% endhighlight %}

And you need to install additional global modules:

{% highlight bash %}
npm install webpack typescript typings -g
{% endhighlight %}


## Webpack

To build the project, I'm going to use [webpack][2]. Webpack simply collects all js/ts/tsx code that we use and puts it in a bundle. Besides it, webpack provides a dev server what is very useful tool.

We'll have two configs: development and production. Development config starts a dev server that monitors changes to source files and automatically rebuilds and refreshes js in the browser.
The production one builds final js and css files which could be distributed to end users.

Since webpack is just a node.js module, config file is a js files where you can `require` other modules. Both configs have common parts, so it makes sense to put common code to a separate file `webpack.commons.js`.

## Conslusion



[1]: /2015/11/16/clojurescript-om/
[2]: https://webpack.github.io/
