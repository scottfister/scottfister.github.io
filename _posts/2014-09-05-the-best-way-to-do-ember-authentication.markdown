---
layout: post
title:  The Best Way To Do Ember Authentication
date:   2014-09-05 17:19:45
categories: ember authentication tutorial
comments: true
---
When I began researching how to implement user authentication in my Ember application, I found there are three options available:

1. [Ember Simple Auth][simple-auth]
2. [Ember Auth][ember-auth]
3. Roll your own

Typically, I've always been a proponent of the "don't reinvent the wheel" philosophy; in other words, using existing tried and true frameworks to achieve my goals.

However, after spending 2 days attempting to integrate both Ember Simple Auth and Ember Auth unsuccessfully, I decided to try my hand at rolling my own. Not only is it *surprisingly simple*, but I am enjoying the freedom from dependancy on a module that - at any time - could become unsupported or broken with the ([very frequent][ember-updates]) updates to Ember.

I will be writing this guide for [Ember CLI][ember-cli] - which you really should be using for your project. If you cannot for whatever reason, and have trouble translating it to vanilla Ember, comment below and I'll help. I originally built this in non-Ember CLI.

**A quick breakdown of what we'll be creating:**

1. Login and Logout controllers to authenticate with your backend
2. A "Session" object that handles state and persistance
3. An Adapter to include the token in request headers
4. A [mixin][ember-mixins] that can be included in controllers that require a valid session

# Authenticating with the backend server
The general consensus on authentication seems to be including a token in request headers. This token is retrieved by the client supplying a valid username/password combination to the server.

A basic ([Bootstrap][get-bootstrap]) `login.hbs` template may look like this:

{% highlight html %}
{% raw %}
<form {{action signIn on="submit"}}>
{% endraw %}
  <div class="form-group">
    <label class="control-label">Email</label>
    {% raw %}{{input value=email class="form-control"}}{% endraw %}
  </div>

  <div class="form-group">
    <label class="control-label">Password</label>
    {% raw %}{{input value=password type="password" class="form-control"}}{% endraw %}
  </div>

  <div class="form-group">
    <button type="submit" class="btn btn-success">Login</button>
  </div>
</form>
{% endhighlight %}

> You may notice the action is in the form with an "on" event of "submit" as opposed to having the action directly on the button. This is best practice and supports behaviour like the user pressing enter to have the form submit.

Create a controller named `login.js` with a `signIn` action:

{% highlight javascript %}
import Ember from "ember";

export default Ember.Route.extend({
  actions: {
    signIn: function() {
      var that = this;

      return Ember.$.post("http://0.0.0.0:5000/v1/users/sign_in",
      {
        user: {
          email: that.get('email'),
          password: that.get('password')
        }
      }).then(function(data) {
        console.log('Success', data);
      }).fail(function(error) {
        console.log('Error', error);
      });
    }
  }
});
{% endhighlight %}

Replace the login path with your server, if it's on the same domain (such as if Ember is part of a Rails app) then you can omit the base url.

Now if you enter credentials into the login form, you should see success and error messages as appropriate. This is great, but now we need a way to persist the token and have Ember made aware that the user is now authenticated. Enter the Session object.

# The Session object
Create an [initializer][ember-initializer] in `initializers/session.js`. We will create our Session object here and initialize it using any stored values.

> Note that I've chosen to use the [Basil.js][basil-js] library which describes itself as "The missing Javascript smart persistence layer. Unified localstorage, cookie and session storage JavaScript API". You could opt to use regular [HTML5 localstorage][local-storage] or cookies on their own if you'd prefer.

Install Basil.js using Bower with `bower install basil.js --save-dev`

{% highlight javascript %}
import Ember from "ember";

export default {
  name: 'session',

  initialize: function(container, application) {

    var Session = Ember.Object.extend({
      init: function() {

        var userId = basil.get('user_id');
        this.set('userId', userId);
        this.set('authToken', basil.get('auth_token'));
        // We use run once to run this in the next loop, this prevents issues
        // where the object has not been created fully but property bindings
        // are fired and cause an error
        Ember.run.once(this, 'userIdChanged');
      },

      authTokenChanged: function() {
        basil.set('auth_token', this.get('authToken'));
      }.observes('authToken'),

      userIdChanged: function() {
        var userId = this.get('userId');
        basil.set('user_id', userId);
        if (!Ember.isEmpty(userId)) {
          this.set('user', container.lookup('store:main').find('user', userId));
        }
      }.observes('userId'),

      signedIn: function() {
        return !!this.get('authToken');
      }.property('authToken')

    });

    application.register('session:main', Session);

    application.inject('controller', 'session', 'session:main');
    application.inject('route', 'session', 'session:main');
    application.inject('adapter', 'session', 'session:main');
  }
};
{% endhighlight %}

Yeah a lot happened there, and don't be intimidated if a lot of that looks foreign to you. Lets break it down.

In the `init` method we call `this_super()` as is recommended for Object creation in Ember. Then set the properties `userId` and `authToken` to the values that Basil reads (if any). The [Ember.run.once][ember-run-once] is explained in the comment above it.

> Note that a userId may be irrelevant to your application; all you need to make auth work is a token. However for completeness I'll be showing how to have this working with a User model also.

`authTokenChanged` is an [observer][ember-observer] that simply persists the `token` value any time it changes on the Session object.

`userIdChanged` does the same for `user_id`, however it also does a store lookup on the ID and sets the `user` property.

`signedIn` simply returns a boolean of whether the `token` exists or not - we can use this in views and controllers to check if authenticated.

This last part is one of the most exciting parts I learnt whilst building this: we register our Session object as what Ember calls a `factory` which enables us to [inject][ember-inject] it into our controllers, routes, or adapters. This means that instead of having to set our Session object as a global variable like `App.Session` (which is not allowed in Ember CLI anyway), we can instead access it via `this.get('session')` in adapters, controllers and routes. This will be shown more below, but suffice to say it's really powerful, and you can go so far as to only inject it into specific places.

#### Making libraries accessible globally
Whilst this needn't be explicitly covered in this guide, I tripped up on it myself, so I want to cover it briefly. In Ember CLI in order to [access libraries globally][ember-dependencies] like we have above (you'll notice Basil has no import statement), you need to declare it within your `Brocfile.js` like this:

{% highlight javascript %}
// Brocfile.js
var EmberApp = require('ember-cli/lib/broccoli/ember-app');
...
app.import('vendor/basil.js/build/basil.js');

module.exports = app.toTree();
{% endhighlight %}

# Persisting the data
Let's go back and modify our `login.js` to persist the data using the Session object:

{% highlight javascript %}
import Ember from "ember";

export default Ember.Route.extend({
  actions: {
    signIn: function() {
      var that = this;

      return Ember.$.post("http://0.0.0.0/sign_in",
      {
        user: {
          email: that.get('email'),
          password: that.get('password')
        }
      }).then(function(data) {
        console.log('Success', data);
        that.set('session.authToken', data.user_token);
        that.set('session.userId', data.user_id);
      }).fail(function(error) {
        console.log('Error', error);
      });
    }
  }
});
{% endhighlight %}

Of course, your server may return different JSON so modify as appropriate. Now if you try logging in, you can use the Chrome developer tools under "Resources" and "Local Storage" to see the data persisted from your server!

# Sending token with headers
It's straight-forward to send the token allow with any requests we make to the server now:

{% highlight javascript %}
// adapters/application.js
import DS from "ember-data";

var Adapter = DS.ActiveModelAdapter.extend();

Adapter.reopen({
  headers: function() {
    return {
      'Authorization': 'Token token="%@"'.fmt(this.get('session.authToken'))
    };
  }.property("session.authToken"),

  // host: 'http://0.0.0.0:5000/v1' this is optional, if your API is on a different domain
});

export default Adapter;
{% endhighlight %}

Accessing the Session object is made extremely simple thanks to the fact the Session object is injected into our adapters.

# Protected Routes Mixin
This [mixin][ember-mixins] allows you to protect certain routes within your application by redirecting if they have not authenticated.

Create a mixin named `mixins/protected-route.js` and paste the following:

{% highlight javascript %}
// mixins/protected-route.js
import Ember from "ember";

export default Ember.Mixin.create({
  beforeModel: function(transition) {
    if (transition.targetName === 'login') {
      return;
    }

    if (!this.get('session.signedIn')) {
      this.redirectToLogin(transition);
    }
  },

  redirectToLogin: function(transition) {
    this.set('session.attemptedTransition', transition);
    this.transitionTo('login');
  }
});
{% endhighlight %}

We implement the Ember [beforeModel hook][ember-before-model] to first check if the route they are attempting to access is the `login` route, and if so do nothing (prevents infinite redirect). We then check if the user is not signed in, and if so call `redirectToLogin` passing in the current route they are attempting to access.

The `redirectToLogin` method sets aforementioned route as a property on our Session object for later use, and transitions them to the `login` route.

Now simply include this mixin in whichever route you'd like protected. For me, my entire application should be so I placed it within `routes/application.js` which cascades to all routes:

{% highlight javascript %}
// routes/application.js
import Ember from "ember";
import ProtectedRoutesMixin from "../mixins/protected-routes";

export default Ember.Route.extend(ProtectedRoutesMixin, {
  ...
});
{% endhighlight %}

> You'll notice the path to the mixin is relative, this is [recommended][ember-modules] as best practice by Ember CLI.

Try navigating to a route other than `login` and you should be redirected - if not ensure there is no `token` stored by Basil, or continue on to implement the logout functionality.

# Logout
The logout controller is fairly self explanatory:

{% highlight javascript %}
// controllers/logout.js
import Ember from "ember";

export default Ember.Route.extend({
  beforeModel: function() {
    var self = this;
    return Ember.$.ajax({
      url: 'http://0.0.0.0:5000/v1/users/sign_out',
      type: 'DELETE',
      headers: {
        'Authorization': 'Token token="%@"'.fmt(self.get('session.authToken'))
      }
    }).always(function(response) {
      self.set('session.authToken', '');
      self.set('session.userId', '');
      self.set('session.user', null);
      self.transitionTo('login');
    });
  }
});
{% endhighlight %}

You'll notice we're chaining `always` instead of `then`, this is because we want the user to be logged out regardless of whether it's confirmed on the backend or not. This is just good UX.

We're setting the `token`, `userId` and `user` to null or empty strings so our observes fire and Basil sets them thus.

# Use within templates
Now we'd like to show a login link to unauthenticated users and logout for authenticated. Again, this is made extremely simple thanks to the fact the Session object is injected into our controllers:

{% highlight html %}
// templates/application.hbs
<ul class="nav nav-pills">
{% raw %}{{#if session.signedIn}}
  <li>{{#link-to "login"}}Login{{/link-to}}</li>
{{else}}
  <li>{{#link-to "logout"}}Logout{{/link-to}}</li>
{{/if}}{% endraw %}
</ul>
{% endhighlight %}

# Good UX: Taking the user to the route they attempted
If you've implemented the mixin as I have above, when a user attempts to access a protected route it will redirect them to the login page instead and store the `attemptedTransition` on the Session object. After a successful sign-in we can use this to redirect them to the page they were attempting. Add the following snippet in your login controller below where we persist the data:

{% highlight javascript %}
...
that.set('session.authToken', data.user_token);
that.set('session.userId', data.user_id);

var attemptedTransition = that.get('session.attemptedTransition');
if (attemptedTransition) {
  attemptedTransition.retry();
  that.set('session.attemptedTransition', null);
} else {
  that.transitionTo('index');
}
...
{% endhighlight %}

You can replace `that.transitionTo('index')` with whatever you think the default landing route should be.

# Optional: Error handling on login
If you'd like to display relevant errors to your users attemping login, modify the login controller by adding `that.set('error', error.responseJSON.error);` within the `fail` block. Then in your `login.hbs` template you can display them:

{% highlight html %}
{% raw %}
{{#if error}}
  <div class="alert alert-danger">{{error}}</div>
{{/if}}
{% endraw %}
{% endhighlight %}

# Finishing up
Whilst there is certainly more code written out compared to using an external library, this is using the latest (at time of writing) Ember conventions, and you have complete control without needing to hack anything.

Please leave your feedback in the comments below; I'm more than happy to improve upon this guide and I intend to write more in the future.

[simple-auth]:        https://github.com/simplabs/ember-simple-auth
[ember-auth]:         http://ember-auth.herokuapp.com/docs
[ember-cli]:          http://www.ember-cli.com/
[ember-updates]:      http://emberjs.com/blog/2013/09/06/new-ember-release-process.html
[ember-mixins]:       http://emberjs.com/api/classes/Ember.Mixin.html
[get-bootstrap]:      http://getbootstrap.com/
[basil-js]:           http://wisembly.github.io/basil.js/
[local-storage]:      https://developer.mozilla.org/en-US/docs/Web/Guide/API/DOM/Storage#localStorage
[ember-initializer]:  http://emberjs.com/api/classes/Ember.Application.html#toc_initializers
[ember-observer]:     http://emberjs.com/guides/object-model/observers/
[ember-before-model]: http://emberjs.com/api/classes/Ember.Route.html#method_beforeModel
[ember-modules]:      http://www.ember-cli.com/#using-modules
[ember-run-once]:     http://emberjs.com/api/classes/Ember.run.html#method_once
[ember-dependencies]: http://www.ember-cli.com/#managing-dependencies
[ember-inject]:       http://emberjs.com/api/classes/Ember.Application.html#method_inject
