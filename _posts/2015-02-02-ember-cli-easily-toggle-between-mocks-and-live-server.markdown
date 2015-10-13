---
layout: inner
title: "Ember CLI - Easily toggle between mocks and live server"
date: 2015-02-02 21:41:17 -0800
comments: true
published: true
categories: [ember, ember-cli]
---

One of the great things about [Ember](http://www.emberjs.com) is how productive I feel, and how it allows me to focus solely on the front end. [Ember CLI](http://www.ember-cli.com/) makes this even faster, as it makes it super easy to create http-mocks of the backend, which allows me to continue working on the front-end while the Web API portion of the application is still being developed or finalized. If the same team member is working on both, sometimes an approach of starting with the UI also helps vet out the requirements for the Web API as well.

However, I was recently looking for an easy way to toggle between using my mocks and using a live development server to validate everything. In addition, I sometimes work completely offline and want to solely rely on my http mocks. Fortunately, with some tweaks to the generated code from Ember CLI, this was very easy to do.

<!-- more -->

Creating an http-mock with Ember CLI is very easy to do.  All it takes is a call to

{% highlight bash %}
$ ember generate http-mock user
{% endhighlight %}

This will create a few things the first time it's run. The first, is it creates a `server` folder under your application, an `index.js` file under the `server` folder which sets up the server, as well as a file for the mock API you wish to stub out.

However, one of the problems with the Ember CLI generated code is that out of the box, it wires up both http-proxy's as well as http-mocks.

{% highlight js %}
module.exports = function(app) {
  var globSync   = require('glob').sync;
  var mocks      = globSync('./mocks/**/*.js', { cwd: __dirname }).map(require);
  var proxies    = globSync('./proxies/**/*.js', { cwd: __dirname }).map(require);

  // Log proxy requests
  var morgan  = require('morgan');
  app.use(morgan('dev'));

  mocks.forEach(function(route) { route(app); });
  proxies.forEach(function(route) { route(app); });
};
{% endhighlight %}

This can be problematic if you want to use alternate back and forth between a mock and your dev server, or if you want to exclusively use mocks (for example if you don't have access to your dev server), or exclusively use your proxy without blowing away your mocks.

### Diving into the source

An easy, but hidden, way of doing this is to modify the generated and get access to the options which are passed in from the `ember server` command.

If you dive into the ember-cli source, specifically into `express-server.js`, you'll see that the `processAppMiddlewares` function is where the server is being initiated, and that two properties are passed in: `app` and `options`. The `options` object is what contains a reference to the environment the server is started with.


{% highlight js %}
   processAppMiddlewares: function(options) {
     if (this.project.has(this.serverRoot)) {
       var server = this.project.require(this.serverRoot);
       if (typeof server !== 'function') {
         throw new TypeError('ember-cli expected ./server/index.js to be the entry for your mock or proxy server');
       }
       server(this.app, options);
     }
   }
{% endhighlight %}

When the `ember server` commandline task is called, you can pass in any type of arguments you want for the environment, and now you can access that environment on the `options.environment` within your `server/index.js` method! This allows you to easily toggle back and forth between using mocks or proxies. This can also be really helpful if you want to do development against a live server, but only hit a mock when running `ember test`.

See the example below, where if I want to use mocks when `environment=offline`, otherwise use a proxy to my dev server.

All I then have to do is call `ember server --environment=offline` and I can use my mocks.

{% highlight js %}
module.exports = function(app, options) {
	var globSync = require('glob').sync;
	var mocks, proxies;

	if (options.environment === 'offline') {
		mocks = globSync('./mocks/**/*.js', {
			cwd: __dirname
		}).map(require);
	} else {
		proxies = globSync('./proxies/**/*.js', {
			cwd: __dirname
		}).map(require);
	}

	// Log proxy requests
	var morgan = require('morgan');
	app.use(morgan('dev'));

	if (mocks) {
		mocks.forEach(function(route) {
			route(app);
		});
	}

	if (proxies) {
		proxies.forEach(function(route) {
			route(app);
		});
	}
};
{% endhighlight %}

Hopefully this helps those of you who might do development against a server not running on your machine, but still want offline access, or simply want a way to easily toggle between your mocks and a live server.
