---
layout: post
title:  "Callbacks aren't that bad"
date:   2016-07-10 23:28:30 -0400
categories: jekyll update
---

I hear about callback hell all the time in javascript-land. The javascript
callback pattern is one of those things that I once complained about, but later
learned to appreciate. In organizing my code to be used in callbacks, I end up
defining semantic names with very clear arguments. Because this "flattens"
the code, it also reduces dependence on closures, which helps with grokking the
current function's state.

Check out what I mean with this extremely contrived code!

First, the "callback hell" version:

{% highlight javascript %}
function getMedia(id, callback) {
  $.get('/media', { id: id }, function(data) {
    if (data.success) {
      $('body').addClass('success');
      $.get('/validate', { payload: data.payload }, function(validateResp) {
        if (validateResp.success) {
          $('html,body').animate({ scrollTop: 100 }, function() {
            callback("I guess this was successful.");
          });
        } else {
          $('body').addClass('error');
          $('html,body').animate({ scrollTop: 400 }, function() {
            console.log("Whoops, this was an error.");
          });
        }
      });
    } else {
      $('body').addClass('error');
      $('html,body').animate({ scrollTop: 400 }, function() {
        callback("Whoops, this was an error.");
      });
    }
  });
}
{% endhighlight %}

That thing is pretty nasty; I can understand why people want to avoid it. But
if I refactor it into named functions, it gets reasonable pretty quickly.

{% highlight javascript %}
function getMedia(id, callback) {
  $.get('/media', { id: id }, function(data) {
    if (data.success) {
      handleFetchSuccess(data, callback);
    } else {
      handleMediaError(data, callback);
    }
  });
}

function handleFetchSuccess(data, callback) {
  validatePayload(data.payload, function() {
    handleMediaSuccess(callback);
  }, function() {
    handleMediaError(callback);
  }
}

function handleMediaSuccess(callback) {
  $('html,body').animate({ scrollTop: 100 }, function() {
    callback("I guess this was successful.");
  });
}

function handleMediaError(callback) {
  $('body').addClass('error');
  $('html,body').animate({ scrollTop: 400 }, function() {
    callback("Whoops, this was an error.");
  });
}

function validatePayload(payload, successCallback, errorCallback) {
  $.get('/validate', { payload: payload }, function(validateResp) {
    if (validateResp.success) {
      successCallback(validateResp);
    } else {
      errorCallback(validateResp);
    }
  });
}
{% endhighlight %}

This version is longer, and I've probably done a disservice by being very
abstract, but, at least in my opinion, the code is much easier to read and much
easier to modify in discreet ways. The trouble it's left us with is in naming
our functions, but if we scope properly, it's not a big deal.

Now I'm wondering, "what does this look like with promises?"

{% highlight javascript %}
function getMedia(id) {
  var promise = new Promise();
  $.get('/media', { id: id }).then(function(data) {
    $.get('/validate', { payload: data.payload }).then(function(validateResp) {
      if (validateResp.success) {
        handleMediaSuccess().then(function() {
          promise.resolve();
        });
      } else {
        handleMediaError().then(function() {
          promise.reject();
        });
      }
    });
  }).catch(function(reason) {
    handleMediaError().then(function() {
      promise.reject();
    });
  });
}

function handleMediaSuccess() {
  var promise = new Promise();
  $('html,body').animate({ scrollTop: 100 }, function() {
    promise.resolve("I guess this was successful.");
  });
  return promise;
}

function handleMediaError(callback) {
  var promise = new Promise();
  $('body').addClass('error');
  $('html,body').animate({ scrollTop: 400 }, function() {
    promise.resolve("Whoops, this was an error.");
  });
  return promise;
}
{% endhighlight %}

So in this world, instead of needing to pass around a callback function, we
need to create and manage promises for each asynchronous function. The nice
thing about promises, in my view, is the `catch` function. This is an advantage
over the pure callback method, though again, if you've broken down your
callback functions appropriately, then it's very clear how errors are handled
in each function, and you can disregard closures.

### So what does it matter?

I still like promises, since they can be refactored similarly to callbacks as
above, and give you the added benefit of built-in success/failure callbacks,
which is basically syntactic sugar. But the point of this post is that callback
hell is only bad if you let it be bad, and that the javascript world still
hasn't really moved on from it.
