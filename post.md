I recently entered an application in to the Static Showdown, a 48 hour, frontend hacakathon. For the competition I built [Deckify](http://deckify.willrax.com). It's a small app where you can upload a pdf of your slide deck and it creates a shareable page to show it off and takes advantage of query params to create a URL for each slide in the deck. What I wanted to focus on in this article is how I added GitHub authentication using [Firebase](https://firebase.com).

##### Provider Set Up

Firebase have authentication methods for a few different OAuth providers. You can use Twitter, Facebook, GitHub etc, so pick your provider and add the tokens to your Firebase apps auth page (https://your-app.firebaseio.com/?page=Auth).

Lucky for us, Firebase has an Ember library that's ready to go with ember-cli. So you can just run `npm install emberfire --save-dev`. This will also install an ember-data adapter, but you don't have to use that if you're using your own api/backend. In this post, we'll just be using the authentication parts of the library.

##### Logging In

You'll want to create a button for your users to click which kicks off the login action.

```markup
<p class="lead button">
  <button {{action "login"}} class="btn btn-default">
    {{fa-icon "github"}}
      Sign in with GitHub
  </button>
</p>
```

We've created a button and tied the click event to trigger the `login` action in the controller. Let's look at the controller.

```javascript
actions: {
  login: function() {
    var controller = this;
    controller.get("session").login().then(function(user) {
      // Persist your users details.
    }, function() {
      // User rejected authentication request
    });
  }
}
```

You might notice above that I'm accessing a property on the controller called session. This is an initializer that gets added to routes, controllers, and templates when the app boots. The aim of `session` is to provide an interface to Firebase and give us easy access to functions like `isAuthenticated` in our views, etc.

```javascript
import Ember from "ember";

var session = Ember.Object.extend({
  ref: new Firebase("https://your-app.firebaseio.com"),

  addFirebaseCallback: function() {
    var session = this;

    this.get("ref").onAuth(function(authData) {
      if (authData) {
        session.set("isAuthenticated", true);
      } else {
        session.set("isAuthenticated", false);
      }
    });
  }.on("init"),

  login: function() {
    return new Promise((resolve, reject) => {
      this.get("ref").authWithOAuthPopup("github", function(error, user) {
        if (user) {
          resolve(user);
        } else {
          reject(error);
        }
      });
    });
  },

  currentUser: function() {
    return this.get("ref").getAuth();
  }.property("isAuthenticated")
});

export default {
  name: "Session",

  initialize: function(container, app) {
    app.register("session:main", session);
    app.inject("controller", "session", "session:main");
    app.inject("route", "session", "session:main");
  }
};

```

The session is just a plain old Ember Object with a few functions. The first function we create is used to setup a callback for Firebase. `onAuth` is fired whenever the auth details change for the sesssion. Since Firebase manages its own internal session, we can use these functions to create a nice wrapper. When the auth status changes we check to see if the user is authenticated. We then update our own property `isAuthenticated` accordingly which the views use to show different states depending on whether the user is logged in or out.

The next function is `login`. This is the function we used previously in the login action within our controller. We wrap it in an Ember promise and kick off the authentication flow. I've used GitHub here, so make sure to sub in whatever provider you're using. Firebase will take care of the popup and getting the OAuth token for us using their backend servers. Firebase will return with either an error or a user object, depending on whether the user successfully authenticated. We can just check for the presence of a user then either resolve or reject the promise. If you look back at our controller, you can now either persist the user details to your own database or continue with your application's next step.

Because Firebase manages its own session details, the next function just provides a nicer way to access the an object representing the currently authenticated user's details. We set this up as a property that will watch for `isAuthenticated` changes.

#### Templates
Because we injected the session object in to our controllers, we can use it in any template to show both authenticated and unauthenticated states. The current user's details is also available to help personalize the page.

```markup
{{#if session.isAuthenticated}}
  <div class="collapse navbar-collapse">
    <ul class="nav navbar-nav">
      <li> {{#link-to "dashboard"}} All {{/link-to}} </li>
    </ul>
    
    <p class="navbar-text navbar-right avatar"><img src={{session.currentUser.github.cachedUserProfile.avatar_url}} class="img-circle" /></p>
  </div>
{{/if}}
```

The emberfire plug in makes it very easy to build ember projects with a solid backend. I'd highly recommend taking a look and seeing if it would be a good fit for your next project, or even something you could integrate in to a current project.