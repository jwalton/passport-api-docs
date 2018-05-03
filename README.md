# Passport: The Hidden Manual

When you `import 'passport'`, you get back an instance of the "Passport" class.  You can create new passport instances:

```js
import passport from 'passport';

const myPassport = new passport.Passport();
```

You'd want to do this if you want to use Passport in a library, and you don't want to polute the "global" passport with your authentication strategies.

These are the various functions on a Passport object:

## passport.initialize()

Returns a middleware which must be called at the start of connect or express based apps.  This sets `req._passport`, which passport uses all over the place.  Calling `app.use(passport.initialize())` for more than one passport instance will cause problems.

This will also set up `req.login()` and `req.logout()`.

## passport.session(\[options])

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

## passport.authenticate(strategyName, \[options | callback])

strategyName is the name of a strategy you've previously registered with `passport.use(name, ...)`.  This can be an array, in which case the first strategy to succeed, redirect, or error will halt the chain.  Auth failures will proceed through each strategy in series, failing if all fail.

This function returns a middleware which runs the strategies.  If one of the strategies succeeds, this will set `req.user`.  If you pass no options or callback, and all strategies fail, this will return a 401 to the client.

Valid options:
* successRedirect - path to redirect to on a success.
* failureRedirect - path to redirect to on a failure (instead of 401).
* failureFlash - True to flash failure messages or a string to use as a flash message for failures (overrides any from the strategy itself).
* successFlash - True to flash success messages or a string to use as a flash message for success (overrides any from the strategy itself).
* successMessage - True to store success message in req.session.messages, or a string to use as override message for success.
* failureMessage - True to store failure message in req.session.messages, or a string to use as override message for failure.
* session - boolean, enables session support (default true)
* failWithError - On failure, call next() with an AuthenticationError instead of just writing a 401.
* assignProperty - If provided, then the user will be assigned to `req[assignProperty]` (and to req.user?)

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
	
## passport.authorize(strategyName\[, options], callback)

Like `passport.authenticate()`, but this doesn't set req.user or change the session.  It sets 'req.account' instead.

## passport.use(\[strategyName,] strategy)

Configure a strategy.  Strategies have a "default name" assigned to them, so you don't have to give them a name.

## passport.serializeUser(fn(user, done))

Passport will call this to serialize the user to the session.  Should call done(null, user.id).

Undocumented: fn() can be a `fn(req, user, done)`.  If multiple serializers are registered, they are called in order.  Can return 'pass' as err to skip to next serialize.

## passport.deserializeUser(fn(id, done))

Passport will call this to deserialize the user from the session.  Should call done(null, user).

Undocumented: fn() can be a `fn(req, user, done)`.  As with serializeUser, serializers are called in order.

## verify callback

Passport strategies require a verify callback, which is generally a `(err, user, options?)` object.  `options.message` can be used to give a flash message.  `user` should be `false` if the user does not authenticate.  `err` is meant to indicate a server error, like when your DB is unavailable; you shouldn't set `err` if a user fails to authenticate.

## req.login(user, callback)

Log a user in (causes passport to serialize the user to the session).  On completion, req.user will be set.

## req.logout()

Removes req.user, and clears the session.

# Passport and Sessions

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
