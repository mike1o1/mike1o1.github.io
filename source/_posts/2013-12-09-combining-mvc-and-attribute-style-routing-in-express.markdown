---
layout: post
title: "Combining MVC and Attribute Style Routing in Express"
date: 2013-12-09 00:18:03 -0800
comments: false
published: true
categories:
---

### Background

I've been writing a decent sized application using Node.JS and Express, and got frustrated with the routing situation. Every time I would add a new route method, I had to remember to also modify my Express configuration to add the route, and to remember what, if any, middleware I also wanted associated with that route.

Having these definitions in two different places made it difficult for maintenance once you grow to a non-trivial number of routes, as well as multiple middleware for different routes.

I'm a big fan of [Attribute Routing](http://attributerouting.net/ "Attribute Routing") and wanted something similar for Node and Express. Fortunately, with the flexibility of JavaScript this isn't very difficult to implement.

<!-- more -->

### Attribute Based Routing

Attribute Routing in the MVC world is very simple, and allows you to define your routes and HTTP verbs as an attribute of an individual method, or on the controller itself. From the example documentation, the below route will match on `/posts/index`.

``` c# Attribute Based Routing Example
[GET("Posts/Index")]
public ActionResult Index() { /* ... */ }
```

### Typical ASP.NET MVC Routing

ASP.NET MVC has a routing concept where you can define your controller and your action, and then build the URL based off of that.  For example, the below class will generate a URL for `/blogs/index`, as long as the default URL route of `{controller}/{action}/{id}` is setup.

``` c# MVC Style Routing
public class BlogController
{
  public ActionResult Index() { /* ... */ }
}
```

### Combining The Two in JavaScript

I wanted to be able to combine the two concepts in JavaScript, where by default a URL matching '{controllerName}/{actionName}' would create a URL matching `/controllerName/methodName/`.

I wanted some additional functionality as well:

- Define single or multiple controller level middleware method to run on each route
- Define single or multiple action level middleware to run for this specific route
- Override the controller name prefix, and cascade to each controller action as well
- Override the action name of an individual action
- Override the complete matching URL path of an individual action
- Default to GET, but be able to override the HTTP verb

### Get to the code already!

To do this, I created a simple npm module called [express-conventional-routing](https://npmjs.org/package/express-conventional-routing) to assist with this.

To use, simply install the module
``` bash
$ npm install express-conventional-routing --save
```

and then include the module in your code. You will need to pass in an instance of express, as well as the full path to where your controllers are located. The code below

``` js
var errors = require('../errorHandling'),
    express = require('express'),
    path = require('path'),
    routes = require('express-conventional-routing');

function() {

    var server = express(),
        controllersPath = path.join(__dirname, './server/controllers');


    // Setup any other express options here, such as views
    // and any public folders

    // setup our routes
    routes.setup(server, controllersPath);

    // right now all partial views are public
    server.get('/views/*', function(req, res) {
        res.render('partials/' + req.params[0], {}, function(err, html) {
            if (err) {
                res.render('404', { error: req.originalUrl, title: 'Something went wrong...' });
            } else {
                res.end(html);
            }
        });
    });

    // anything else is a 404
    server.get('/public/*', errors.error404);

    // anything else is a 404
    server.get('/api/*', errors.error404);
};
```

#### Example Controller

Example controllers are a simple JavaScript object with some controller level options, such as controller name, whether the controller is an API controller, and any middleware methods (either a single function, or an array of functions).

Within each controller are a list of action objects, which specify a single http request for your application. You must specify at least the method for Express to use, and optionally any overrides. Overrides include:

- Action name
- Path
- Http verb

If you specify a middleware action, note that it will be appended and called after the controller level middlware. If needed, I might add a way of overriding that in the future.

``` js
var sampleController = {

    controllerName: 'test',
    controllerMiddleware: function(req, res, next) {
        // will be first middleware called for every route below
        // this can also be an array of functions
        console.log('middleware action here');
        return next();
    },


    // sample route, will be setup as HTTP GET '/test/action1'
    action1: {
        method: function(req, res, next) {
            req.called = true;

            return next();
        },
        verb: 'GET'
    },

    // sample route, will be setup as HTTP GET '/test/action'
    action2: {
        method: function(req, res, next) {
            req.called = true;
            return next();
        },
        verb: 'GET',
        action: 'action'
    },

    // sample route, will be setup as HTTP GET '/test/action3' with middleware
    action3: {
        method: function(req, res, next) {
            req.called = true;
            return next();
        },
        middleware: function(req, res, next) {
            // will be called after controller level middleware
            // can also be an array of functions
            req.middleware = true;
            return next();
        }
    },

    // sample route, will be setup as HTTP GET '/action4'
    action4: {
        method: function(req, res, next) {
            req.called = true;
            return next();
        },
        path: '/action4'
    },

    action5: {
        method: function(req, res, next) {
            req.called = true;
            return next();
        },
        verb: 'POST'
    }

};

module.exports = sampleController;
```

With this method, we get easy automatic routing, as well as the flexibility of overriding any specifics we might need. For example, if we have a users controller, we can create a login action, and override the path to simply be `'/login'` if needed.




