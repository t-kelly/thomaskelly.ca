---
layout: post
title: 5 Ways to Make Your Mixpanel Tracking Code More Maintainable
date: 2015-03-28 13:30:00
author: Thomas Kelly
categories: blog
summary: "Stop polluting your application codebase with bad tracking code! Implement these patterns to make your tracking code more maintainable, more efficient, and less error-prone."
published: true
sideImage: "/images/IMAG0391.jpg"
image: "/images/IMAG0398.jpg"
---

Recently, I’ve had the opportunity to work with [Mixpanel](https://mixpanel.com/), an event-driven, real-time web and application tracking service. Mixpanel focuses on retrieving metrics with greater substance and utility by tracking application events - cutting away from more common [bullshit metrics](https://mixpanel.com/blog/2012/12/17/bs-metrics/).

In their [JavaScript API documentation](https://mixpanel.com/help/reference/javascript), Mixpanel provides brief examples of how to integrate their tracking into your application. Following these docs, it's extremely easy to get things up and running. Before you know it, Mixpanel is pumping out application events with all sorts of juicy details!

Unfortunately, their is an unsightly side-effect to your new-found tracking goodness. Your code blocks have now doubled in size and are littered with `mixpanel.track` events. Not only do these tracking events look bad in your code, but they are incredibly hard to maintain (especially on a large scale) and are terribly inefficient if you consider the amount of repeated code.

``` js
// An example of the 'Tracking Pollution' effect

// BEFORE - Original Code
var form = {};
form.submit = function (data) {
    return service.sendData(data)
        .catch(function (err) {
            console.log(err);
        })
        .then(function (response) {
            return response.Value;
        });

}

// AFTER - Code with tracking
var form = {};
form.submit = function (data) {
    return service.sendData(data)
        .catch(function (err) {
             console.log(error);
             mixpanel.track("Submit Form", {
                  "Category": "Submit",
                  "Type": "Form",
                  "Status": "Error",
                  "Value": error
             });
        })
        .then(function (response) {
             mixpanel.track("Submit Form", {
                  Category: "Submit",
                  Type: "Form",
                  Status: "Success",
                  Value: response.Value
             });
             return response.Value;
        });
}
```

After realizing what a mess I was producing, I set out on a mission to eliminate tracking pollution and implement a more maintainable, less error-prone solution.


## 1. Use super properties and extend objects

One of the first problems is the amount of repeated properties and values found within groups of Mixpanel events. This can be solved in two ways:

1. **Mixpanel Super Properties** - If you discover you are including a property in all of your events, then it should probably be set as a Mixpanel [super property](https://mixpanel.com/help/reference/javascript#super-properties).

2. **Extend Objects** - Group repeated event properties in separate objects and extend these objects with event specific properties. Here I’m using [lodash](https://lodash.com/docs#assign) to extend my objects but there are plenty of other ways of doing this.

```js

var form = {};

// Repeated properties found in both events go here
var common = {
   Category: "Submit",
   Type: "Form",
};

form.submit = function (data) {
   return service.sendData(data)
        .catch (function (err) {
             console.log(error);

             // We extend the common object with our event specific properties
             mixpanel.track("Submit Form", _.assign(common, {
                  Status: "Error",
                  Value: error
             }));
        }
        .then(function (response) {
             console.log("Submit success");
             mixpanel.track("Submit Form", _.assign(common, {
                  Status: "Success",
                  Value: response.Value
             }));
             return response;
        }
}   
```


## 2. Combine repeated events into a single function

As a best practice, Mixpanel recommends you [keep your Mixpanel events as general as possible](https://mixpanel.com/blog/2012/04/12/selecting-events-and-properties-with-extensibility-and-targeted-reporting-in-mind/). For example, instead of having `'Click Submit Button'` and `'Click Close Button'` as events, use a single `'Click'` event with a property `'Type: Submit'` and `'Type: Close'`. However, a side effect is that you end up with lots of repeated functions. Let's take care of that by creating a new function with the repeated code.

```js

var form = {};
var mix = {};

var common = {
     Category: "Submit",
     Type: "Form",
};

// Common function for all 'Submit Form' events
mix.submitForm = function (properties) {
     mixpanel.track("Submit Form", _.assign(common, properties));
};

form.submit = function (data) {
     return service.sendData(data)
          .catch(function (err) {
               console.log(error);
               mix.submitForm({
                    Status: "Error",
                    Value: error
               });
          })
          .then(function (response) {
               console.log("Submit success");
               mix.submitForm({
                    Status: "Success",
                    Value: response.Value
               });
               return response;
          });
}   
```

## 3. Use constants instead of strings

One of the biggest pains I discovered after my first time implementing Mixpanel on a large scale is the risk of mistyping a property or value string. These typo's are hard to detect and most of the time are only found when it's too late and your data is split between the correctly spelled value and the typo. Using constants allows your IDE to help type by using intelligent code completion and your browser will throw an error if the constant is referenced incorrectly.

```js

var form = {};
var mix = {};

// Mixpanel property strings
mix.properties = var p = {
    CATEGORY: "Category",
    TYPE: "Type",
    STATUS: "Status",
    VALUE: "Value"
};

// Mixpanel value strings
mix.values = var v = {
     CATEGORY_SUBMIT: "Submit",
     TYPE_FORM: "Form",
     STATUS_SUCCESS: "Success",
     STATUS_ERROR: "Error"
};

// Mixpanel event strings
mix.events = var e = {
    SUBMIT_FORM: "Submit Form"
};

var common = {
     p.CATEGORY: v.CATEGORY_SUBMIT:,
     p.TYPE: v.TYPE_FORM,
};

mix.submitForm = function (properties) {
     mixpanel.track(e.SUBMIT_FORM, _.assign(common, properties));
};

form.submit = function (data) {
     return service.sendData(data)
          .catch(function (err) {
               console.log(error);
               mix.submitForm({
                    p.STATUS: v.STATUS_ERROR,
                    p.VALUE : error
               });
          })
          .then(function (response) {
               mix.submitForm({
                    p.STATUS: v.STATUS_SUCCESS,
                    p.VALUE: response.Value
               });
               return response;
          });
}   
```

## 4. Isolate Mixpanel code from application code

Having all of your Mixpanel code to single file makes it a lot easier to maintain then having it scattered across your application codebase. Unfortunately, even with all the above improvements we still have some lingering, Mixpanel specific code... So how can we still make this better?

```js
// form.js

var form = {}
var p = mix.properties;
var v = mix.values;

form.submit = function (data) {
     return service.sendData(data)
          .catch(function (err) {
               console.log(error);
               mix.submitForm({
                    p.STATUS: v.STATUS_ERROR,
                    p.VALUE : error
               });
          })
          .then(function (response) {
               mix.submitForm({
                    p.STATUS: v.STATUS_SUCCESS,
                    p.VALUE: response.Value
               });
               return response;
          });
}
    
```


## 5. Use a central event dispatcher

Using a central event dispatcher, AKA [mediator](https://carldanley.com/js-mediator-pattern/), has a lot of benefits. One of these benifits is allowing modules to communicate with each-other in a [loosely coupled](http://en.wikipedia.org/wiki/Loose_coupling) manor.
 Application architectures like [Facebook’s Flux](https://facebook.github.io/flux/) rely heavily on this design pattern to link application logic and views. I discovered this pattern also excels at loosely coupling Mixpanel event tracking and application code!

When a [module](http://addyosmani.com/writing-modular-js/) triggers an event, the dispatcher emits that event through a single point to all other subscribed modules. These events can include a payload which includes all the data needed by others modules to react accordingly.

Here's where Mixpanel fits in.

Ideally, all events which you need to track in your application should have an event being fired through a central dispatcher. In addition, all of these events should include in their payload all the data needed for tracking. By treating your now isolated Mixpanel code like any other module, you can listen to all the events you wish to track through the dispatcher and completely remove all Mixpanel specific code from your modules.

```js

// form.js

var form = {}

form.events = var e = {
     SUBMIT_FORM_ERROR: "Submit Form Error",
     SUBMIT_FORM_SUCCESS: "Submit Form Success"
}

form.submit = function (data) {
     return service.sendData(data)
          .catch(function (error) {
               dispatcher.trigger(e.SUBMIT_FORM_ERROR, error);
          })
          .then(function (response) {
               dispatcher.trigger(e.SUBMIT_FORM_SUCCESS, response);
               return response.Value;
          });
}
```


```js
// tracking.js

var mix = {};
mix.events = {};

// Step 3 - Use constants instead of strings
mix.properties = var p = {
    CATEGORY: "Category",
    TYPE: "Type",
    STATUS: "Status",
    VALUE: "Value"
};
mix.values = var v = {
     CATEGORY_SUBMIT: "Submit",
     TYPE_FORM: "Form",
     STATUS_SUCCESS: "Success",
     STATUS_ERROR: "Error"
};
mix.events = var e = {
    SUBMIT_FORM: "Submit Form"
};

// Step 2 - Combine repeated events into a single function
mix.submitForm = function (customProps) {
     mixpanel.track(e.SUBMIT_FORM, _.assign(submitTracking, customProps));
};

// Wrapper to enclose grouped events
mix.listeners = _.assign(mix.listeners, (function () {
     var e = form.events;

     // Step 1 - Use super properties and extend objects
     var common = {
          p.CATEGORY: v.CATEGORY_SUBMIT:,
          p.TYPE: v.TYPE_FORM,
     };

     return {
          e.SUBMIT_FORM_ERROR : function (e, payload) {
               mix.submitForm(_.assign(common, {
                    p.STATUS: v.STATUS_ERROR,
                    p.VALUE : payload
               }));
          },

          e.SUBMIT_FORM_SUCCESS : function (e, payload) {
               mix.submitForm(_.assign(common, {
                    p.STATUS: v.STATUS_SUCCESS,
                    p.VALUE : payload.Value
               }));
          }
     }
}()));

// ...
// Repeat above enclosure for other groups
// ...

// Assign callback to each event in mix.events
_.each(mix.events, function(fn, event) {
     dispatcher.on(event, fn);
});
    
```

Awwhh yeah! Your application is back to being clean and tidy and all of your Mixpanel code is isolated in a single file. It's now easier to maintain because of being loosly coupled to modules, less error prone thanks to constants, and more efficient thanks to reducing the amount of repeated code.

Have any other tips or tricks you've discovered while implementing Mixpanel? Leave me a comment!