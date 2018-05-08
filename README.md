# Passport: The Hidden Manual

The official [passport documentation](http://www.passportjs.org/docs/) has a long, example driven style.  Some people like that.  Some people, on the other hand, want to know "when I call this function, what's it going to do?"  This is for those people.

If you find inaccuracies, please feel free to open an issue or a PR.

## Passport class

When you `import 'passport'`, you get back an instance of the "Passport" class.  You can create new passport instances:

```js
import passport from 'passport';

const myPassport = new passport.Passport();
```

You'd want to do this if you want to use Passport in a library, and you don't want to polute the "global" passport with your authentication strategies.

### passport.initialize()

Returns a middleware which must be called at the start of connect or express based apps.  This sets `req._passport`, which passport uses all over the place.  Calling `app.use(passport.initialize())` for more than one passport instance will cause problems.

This will also set up `req.login()` and `req.logout()`.

### passport.session(\[options])

"If your application uses persistent login sessions, `passport.session()` middleware must also be used."  Should be after your session middleware.

This returns a middleware which will try to read a user out of the session; if one is there, it will store the user in `req.user`, if not, it will do nothing.  Behind the scenes, this:

```js
app.use(passport.session());
```

is actually the same as:

```js
app.use(passport.authenticate('session'));
```

which is using the [built-in "session strategy"](https://github.com/jaredhanson/passport/blob/2327a36e7c005ccc7134ad157b2f258b57aa0912/lib/strategies/session.js).  You can customize this behavior by registering your own session strategy.

`session()` does take an 'options' object.  You can set pass `{pauseStrem: true}` to turn on a hacky work-around for problems with really old node.js versions (pre-v0.10).  Never set this true.

### passport.authenticate(strategyName\[, options][, callback])

strategyName is the name of a strategy you've previously registered with `passport.use(name, ...)`.  This can be an array, in which case the first strategy to succeed, redirect, or error will halt the chain.  Auth failures will proceed through each strategy in series, failing if all fail.

This function returns a middleware which runs the strategies.  If one of the strategies succeeds, this will set `req.user`.  If you pass no options or callback, and all strategies fail, this will write a 401 to the response.  Note that some strategies may also cause a redirect (OAuth, for example).

Valid options:
* successRedirect - path to redirect to on a success.
* failureRedirect - path to redirect to on a failure (instead of 401).
* failureFlash - True to flash failure messages or a string to use as a flash message for failures (overrides any from the strategy itself).
* successFlash - True to flash success messages or a string to use as a flash message for success (overrides any from the strategy itself).
* successMessage - True to store success message in req.session.messages, or a string to use as override message for success.
* failureMessage - True to store failure message in req.session.messages, or a string to use as override message for failure.
* session - boolean, enables session support (default true)
* failWithError - On failure, call next() with an AuthenticationError instead of just writing a 401.

Note that the entire `options` object will be passed on to the strategy as well, so there may be extra options you can pass here defined by the strategy.  For example, you can pass a `callbackURL` along to an [oauth strategy](https://github.com/jaredhanson/passport-oauth1/blob/4f8e3404126ef56b3e564e1e1702f5ede1e9a661/lib/strategy.js#L232).

callback is an (err, user, info) function.  No req, res, or next, because you're supposed to get them from the closure.  If authentication fails, user will be false.  If authentication succeeds, your callback is called and *`req.user` is NOT set*.  You need to set it yourself, via `req.login()`:

```js
app.post('/login', function(req, res, next) {
    passport.authenticate('local', function(err, user, info) {
        if (err) { return next(err); }
        if (!user) { return res.redirect('/login'); }
	
	// NEED TO CALL req.login()!!!
        req.login(user, next);
    })(req, res, next);
});
```

Don't just set `req.user = user`, since this won't update your session.
	
### passport.authorize(strategyName\[, options], callback)

Like `passport.authenticate()`, but this doesn't set req.user or change the session.  It sets 'req.account' instead.

### passport.use(\[strategyName,] strategy)

Configure a strategy.  Strategies have a "default name" assigned to them, so you don't have to give them a name.

### passport.serializeUser(fn(user, done) | fn(req, user, done))

Passport will call this to serialize the user to the session.  Should call done(null, user.id).

Undocumented: fn() can be a `fn(req, user, done)`.  If multiple serializers are registered, they are called in order.  Can return 'pass' as err to skip to next serialize.

### passport.deserializeUser(fn(serializedUser, done) | fn(req, serializedUser, done))

Passport will call this to deserialize the user from the session.  Should call done(null, user).

It can happen that a user is stored in the session, but that user is no longer in your database (maybe the user was deleted, or did something to invalidate their session).  In this case, the deserialize function should pass `null` or `false` for the user, not `undefined`.

Undocumented: fn() can be a `fn(req, id, done)`.  As with serializeUser, serializers are called in order.

## Strategies

### Writing custom strategies

Write a custom strategy by extending the `SessionStrategy` class from [passport-strategy](https://github.com/jaredhanson/passport-strategy).  You can unit test a strategy in isolation with [passport-strategy-runner](https://github.com/jwalton/passport-strategy-runner).

```js
import Strategy from 'passport-strategy';

export default class SessionStrategy extends Strategy {
    constructor() {
        super();

        // Set the default name of our strategy
        this.name = 'session';
    }

    /**
     * Authenticate a request.
     *
     * This function should call exactly one of `this.success(user, info)`, `this.fail(challenge, status)`,
     * `this.redirect(url, status)`, `this.pass()`, or `this.error(err)`.
     * See https://github.com/jaredhanson/passport-strategy#augmented-methods.
     *
     * @param {Object} req - Request.
     * @param {Object} options - The options object passed to `passport.authenticate()`.
     * @return {void}
     */
    authenticate(req, options) {
    	if(req.cookie.apikey === '6398d011-d80f-4db1-a36a-5dcee2e259d0') {
	    this.success({username: 'dave'});	  
	} else {
	    this.fail();
	}
    }
}
```

Note when calling `fail()`, the `challenge` should be a challenge as defined by [RFC 7235 S2.1](https://tools.ietf.org/html/rfc7235#section-2.1), suitable for including in a WWW-Authenticate header.

### Verify Callback

Passport strategies require a verify callback, which is generally a `(err, user, options?)` object.  `options.message` can be used to give a flash message.  `user` should be `false` if the user does not authenticate.  `err` is meant to indicate a server error, like when your DB is unavailable; you shouldn't set `err` if a user fails to authenticate.

## Functions added to the Request

### req.login(user, callback)

Log a user in (causes passport to serialize the user to the session).  On completion, req.user will be set.

### req.logout()

Removes req.user, and clears the session.

## Passport and Sessions

Passport creates a key in the session called `session.passport`.

When a request comes in to the `passport.session()` middleware, passport runs the [built-in 'session' strategy](https://github.com/jaredhanson/passport/blob/2327a36e7c005ccc7134ad157b2f258b57aa0912/lib/strategies/session.js) - this calls `deserializeUser(session.passport.user, done)` to read the user out of the session, and stores it in req.user.

You can override how passport deserializes a session by creating a new strategy called 'session' and registering it with `passport.use()`.

When you call `req.login()`, or when a strategy successfully authenticates a user, passport uses the [session manager](https://github.com/jaredhanson/passport/blob/2327a36e7c005ccc7134ad157b2f258b57aa0912/lib/sessionmanager.js#L12-L28), and essentially does:

```js
serializeUser(req.user, (err, user) => {
    if(err) {return done(err);}
    session.passport.user = user;
});
```

Although it's more verbose about it.  You can override the session manager by creating your own implementation and setting `passport._sm`, but this is not documented or supported, so use at your own risk.
