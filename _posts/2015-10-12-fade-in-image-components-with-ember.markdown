---
layout: inner
title: "Fade in Image Components With Ember"
date: 2015-10-12T23:42:46-07:00
comments: true
categories: [ember, ember-cli]
---

One nice effect when you have a number of images, especially in a gallery, is to setup placeholder images while your images load. This makes a much nicer experience, especially for those who are on slower connections, as they don't have the images slowly load. This sounds like it would be fairly easy to do using CSS, and there are certainly a number of solutions to do this using CSS. However, in cases where you want to use the `background-image` property, it's challenging to get a nice fade-in effect. This is because the `background-image` is not a valid animateable property, and also because we have no way of knowing when the browser has downloaded the `background-image`, so if you just set the image, the user will see the image load which is a bad user experience.

We'll walk through how to create a placeholder image like in the animation below.

![animatedplaceholder](https://cloud.githubusercontent.com/assets/837213/10447332/7df155a0-7138-11e5-9bdd-dab476b6af5c.gif)

Fortunately, we can create an Ember component in addition to some CSS to do all the hard work and heavy lifting for us. In this article, we'll look at creating a `{{fade-in-image}}` Ember component using Ember CLI. With this component, we'll have a default placeholder image, which can be overridden, and we'll pass in a full URL path for the actual image we want to display. The component will display the placeholder image, and then once the browser has loaded the source image, we'll have the placeholder fade out and the source image fade in. The one caveat with this technique, is that since we're using `background-image` for our div, we need to make sure we specify the dimensions so there is actual content to cover!

First step is to use Ember CLI to create our component for us.

```bash
ember g component fade-in-image
```

This creates a template as well as a .js file for us to place our logic. Let's start with our fade-in-image.js file, and add some initial setup.

We'll specify our class name, and setup some initial properties.

```js title:"fade-in-image.js"
export default Ember.Component.extend({
    classNames: ['fade-in-image'],
    placeholderSrc: '',
    sourceImage: '',
    fadeInClass: 'fade-in'
});
```

Now, let's add some initial CSS rules for our image component. In a full app, we'd organize this better, for this example we'll just add it to our `app.css` file.

```css title:"app.css"
.fade-in-image {
  height: 250px;
  width: 250px;
}
```

Our first step is to get our component up and working, with our placeholder image. To do this, we'll need to modify our template to add the placeholder content. The idea with our placeholder is that we will have the main div with our component, which will get the `background-image` applied to it, but until then, we'll also have a child placeholder div, set to expand to the parent div, which will also have a `background-image` property set to it. Once the source image is loaded, we will add a class to the placeholder, which will have an `opacity` and `transition` rule set to it, forcing the placeholder image to fade out.

So, let's edit our template to add the placeholder.

```html title:"fade-in-image.hbs"
<div id="fade-in-image-placeholder"></div>
```

Let's add some CSS rules for our placeholder so it will expand the content, and give it a placeholder `background-image`. In our app.css, we'll add the following rules:

```css title:"app.css"
.fade-in-image {
    position: relative;
    overflow: hidden;
    height: 250px;
    width: 250px;
}

.fade-in-image #fade-in-image-placeholder {
    opacity: 1;
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
    background-color: #efefef;
}
```

Now if we load our component on the page, we should see a grey box. Cool! Well, not really, but now we have our foundation and we just need to display our loading image. For this example I'm going to use a placeholder image from [placehold.it](http://placehold.it), but I really should use a base64 encoded image instead if I were to be using this in production. It's a good idea to use a base64 encoded image, so that the browser doesn't need to make an additional http request just for our loading image. It kind of defeats the purpose of a placeholder image if the browser needs to fetch it!

```css
.fade-in-image #fade-in-image-placeholder {
    opacity: 1;
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
    background-color: #efefef;
    background-image: url('http://placehold.it/250?text=Placeholder');
}
```

Now if we refresh our component, we should see our placeholder image, which in my case are just some simple text.



Now that we have our placeholder image loaded, how can we load our source image? Now we'll need to write some JavaScript to do this, but fortunately we don't have to write too much.

As mentioned earlier, we don't have a way of knowing when the `background-image` url has loaded, as we don't have access to an event for that. However, the `<img>` tag does fire an `onload` event, but we don't want to use an `img` tag, as CSS background-images give us much better control over the scaling, sizing and positioning of the images.

However, we can still create an `Image` in JavaScript, subscribe to its onload event, and then set the source of our new `Image` to our source url, and then we'll know when the browser has the image loaded! Then all we'll need to do is set that source image to be our `background-image` of our component, and browser caching should handle everything for us.
	We'll create a method to do all this for us, and call it within our `didInsertElement` hook of our component.

```js title:fade-in-image.js
didInsertElement() {
  this._super(...arguments);

  // For demonstration purposes only, otherwise just call this._setupImage();
  Ember.run.later(this, this._setupImage, 1500);
  //this._setupImage();
},

_setupImage() {
  var img = new Image();

  img.onload = () => {
    // Image is loaded, so let's set our background-image and fade class
    this.$('#fade-in-image-placeholder').addClass(this.get('fadeInClass'));
    this.$().css('background-image', `url(${this.get('sourceImage')})`);

  };

  img.src = this.get('sourceImage');


}
```

Last thing we want to do is to update our application template to specify our source image. I'll be using another placeholder, this time from [lorempixel.com](http://lorempixel.com).

```html title:"application.hbs"
{{fade-in-image sourceImage="http://lorempixel.com/250/250/"}}
```

Now if we refresh we should see our placeholder image load first, followed by a smooth fade-in animation to our actual content.

![animatedplaceholder](https://cloud.githubusercontent.com/assets/837213/10447332/7df155a0-7138-11e5-9bdd-dab476b6af5c.gif)

If we wanted to take this component a step further, we can enhance it by adding the following features:

- Specify a placeholder class (which should have a css rule for a base64 encoded image)
- Configure the class to add after image has been loaded
- Fallback for cases where the image fails to load by adding a listener to [onerror](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/Attribute/onerror)

For a full example of the application, please see the example on [GitHub](https://github.com/mike1o1/fade-in-image-example).
