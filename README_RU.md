# Passport: Скрытое руководство

* [Вступление](#вступление)
* [Класс Passport](#класс-passport)
  * [passport.initialize()](#passportinitialize)
  * [passport.session(options)](#passportsessionoptions)
  * [passport.authenticate(strategyName, options, callback)](#passportauthenticatestrategyname-options-callback)
  * [passport.authorize(strategyName, options, callback)](#passportauthorizestrategyname-options-callback)
  * [passport.use(strategyName, strategy)](#passportusestrategyname-strategy)
  * [passport.serializeUser(fn(user, done) | fn(req, user, done))](#passportserializeuserfnuser-done--fnreq-user-done)
  * [passport.deserializeUser(fn(serializedUser, done) | fn(req, serializedUser, done))](#passportdeserializeuserfnserializeduser-done--fnreq-serializeduser-done)
* [Стратегии](#стратегии)
  * [Написание кастомных стратегий](#написание-кастомных-стратегий)
  * [Подтверждающий коллбек](#подтверждающий-коллбек)
* [Функции добавленные в Request](#функции-добавленные-в-request)
  * [req.login(user, callback)](#reqloginuser-callback)
  * [req.logout()](#reqlogout)
* [Passport и Сессии](#passport-и-сессии)

## Вступление

Официальная [документация passport](http://www.passportjs.org/docs/) имеет длинный стиль, сопровождаемый примерами. Некоторым людям это нравится, а другие хотят узнать "когда я вызываю эту функцию, что произойдет?". Это руководство именно для таких людей.

Если вы обнаружите неточности, пожалуйста не стеснятесь открыть ишью или пулл реквест.

## Класс Passport

Когда вы пишите `import 'passport'`, вы получаете экземпляр класса "Passport". Вы можете создать новый экземпляр следующим образом:

```js
import passport from 'passport';

const myPassport = new passport.Passport();
```

Вы бы хотели так сделать, если вам нужно использовать Passport в библиотеке, и вы не хотите загрязнять "глобальный" passport своими стратегиями аутентификации.

### passport.initialize()

Возвращает middleware которая должная быть вызвана при старте приложения основанного на connect или express. Она устанавливает `req._passport`, который используетcя passport повсюду. Вызов `app.use(passport.initialize())` для более чем одного экземпляра passport будет иметь проблемы.

### passport.session(\[options])

"Если ваше приложение использует сессии, `passport.session()` так же должно быть установлено." Оно должно следовать после вашего промежуточного ПО для сессий.

Это возвращает middleware, которая будет пытаться прочитать пользователя из сессии; если он есть, то будет сохранен в `req.user`, иначе ничего не будет. Это выглядит так:

```js
app.use(passport.session());
```

фактически это то же самое что:

```js
app.use(passport.authenticate('session'));
```

которое использует [встроенную "сессионную стратегию"](https://github.com/jaredhanson/passport/blob/2327a36e7c005ccc7134ad157b2f258b57aa0912/lib/strategies/session.js). Вы можете изменить данное поведение зарегистрировав собственную сессионную стратегию.

`session()` принимает объект 'options'. Вы можете установить `{pauseStrem: true}`, чтобы включить обход проблем с действительно старыми версиями node.js (pre-v0.10). Никогда не устанавливайте это.

### passport.authenticate(strategyName\[, options][, callback])

strategyName - это имя стратегии, которую вы зарегистрировали ранее с помощью `passport.use(name, ...)`. Это может быть массив и в таком случае сперва идет стратегия успеха, затем редирект или обработчик ошибки. Неудачная аутентификация будет проходить через каждую стратегию в цепочке до последнего обработчика, если все они проваляться.

Данная функция возвращает middleware которая запускает стратегии. Если одна из стратегий пройдет успешно, будет установлен `req.user`. Если вы не передали options или callback, и все стратегии провалились, то будет записан 401 статус в ответе. Заметим что некоторые стратегии могут так же вызывать редирект (например OAuth). Это так же установит `req.login()`, `req.logout()` и `req.isAuthenticated()`.

Допустимые options:

* successRedirect - путь редиректа в случае успеха.
* failureRedirect - путь редиректа в случае неудачи (вместо 401).
* failureFlash - `true` для flash сообщения с ошибкой или `string` для использования в качестве сообщения об ошибке (переопределяет любые сообщения из самой стратегии).
* successFlash - `true` для flash сообщения успеха или `string` для использования в качестве сообщения успеха (переопределяет любые сообщения из самой стратегии).
* successMessage - `true` для сохранения сообщения успеха в `req.session.messages`, или `string` для перезаписи сообщения успеха.
* failureMessage - `true` для сохранения сообщения ошибки в `req.session.messages`, или `string` для перезаписи сообщения ошибки.
* session - `boolean`, разрешает поддержку сессий (по умолчанию `true`)
* failWithError - в случае ошибки вызывает `next()` с `AuthenticationError` вместо простой записи 401.

Обратите внимание, что весь объект `options` также будет передан в стратегию, поэтому могут быть дополнительные опции определенные стратегией. Например, вы можете передать `callbackURL` вместе с [oauth стратегией](https://github.com/jaredhanson/passport-oauth1/blob/4f8e3404126ef56b3e564e1e1702f5ede1e9a661/lib/strategy.js#L232).

callback - это функция с параметрами (err, user, info). Без req, res, or next, потому что вы должны получить их из замыкания. Если аутентификация провалилась, то `user` будет `false`. Если аутентификация завершилась успешно, ваш колбек будет вызван и *`req.user` не будет установлен*. Вам нужно установить его самому при помощи `req.login()`:

```js
app.post('/login', function(req, res, next) {
    passport.authenticate('local', function(err, user, info) {
        if (err) { return next(err); }
        if (!user) { return res.redirect('/login'); }

    // НУЖНО ВЫЗВАТЬ req.login()!!!
        req.login(user, next);
    })(req, res, next);
});
```

Не устанавливайте просто `req.user = user`, так как это не обновит вашу сессию.

### passport.authorize(strategyName\[, options], callback)

Этот метод не очень хорошо назван, потому что не имеет ничего общего с авторизацией. Эта функция точно такая же как `passport.authenticate()`, но вместо параметра `req.user`, она устанавливает `req.account` и не вносит никаких изменений в сессию.

Если вы хотите сделать что-то вроде "Связать мою учетную запись с 'Каким-то Сервисом'", у вас может быть пользователь который уже залогинен и вы используете Passport чтобы извлечь его данные из 'Какого-то Сервиса'(Google, Twitter и т.п.). Полезно для связки с социальными меда платформами и т.п.

### passport.use(\[strategyName,] strategy)

Настройка стратегии. Стратегии имеют назначенное имя, так что не нужно придумывать им имя.

### passport.serializeUser(fn(user, done) | fn(req, user, done))

Passport будет вызвывать это для сериализации пользователя в сессию. Нужно вызвать done(null, user.id).

Незадокументировано: fn() может быть `fn(req, user, done)`. Если зарегистрировано несколько сериализаторов, они вызываются по порядку. Может вернуть ошибку, чтобы перейти к следующей сериализации.

### passport.deserializeUser(fn(serializedUser, done) | fn(req, serializedUser, done))

Passport будет вызывать это для десериализации пользователя из сессии. Нужно вызывать done(null, user).

Может случиться так, что пользователь будет сохранен в сессии, но этот пользователь больше не находится в вашей базе данных (может он был удален или  нужно сделать что-то чтобы аннулировать его сеанс). В данном случае функция десериализации должна передать `null` или `false` в качестве user, но не `undefined`.

Незадокументировано: fn() может быть `fn(req, id, done)`. Так же как в случае с serializeUser, сериалайзеры вызываются по порядку.

## Стратегии

### Написание кастомных стратегий

Кастомные стратегии пишутся путем расширения класса `SessionStrategy` из [passport-strategy](https://github.com/jaredhanson/passport-strategy). Вы можете изолированно модульно протестировать стратегию при помощи [passport-strategy-runner](https://github.com/jwalton/passport-strategy-runner).

```js
import Strategy from 'passport-strategy';

export default class SessionStrategy extends Strategy {
    constructor() {
        super();

        // Установка дефолтного имени для нашей стратегии
        this.name = 'session';
    }

    /**
     * Аутентификация запроса.
     *
     * Данная функция должна вызвать ровдо один из методов `this.success(user, info)`, `this.fail(challenge, status)`,
     * `this.redirect(url, status)`, `this.pass()`, или `this.error(err)`.
     * Смотри https://github.com/jaredhanson/passport-strategy#augmented-methods.
     *
     * @param {Object} req - Request.
     * @param {Object} options - Объект опций передаваемый в `passport.authenticate()`.
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

Заметим что когда вызывается `fail()`, `challenge` должен быть либо строкой как определено в [RFC 7235 S2.1](https://tools.ietf.org/html/rfc7235#section-2.1), подходит для включения в заголовок WWW-Authenticate, либо объектом `{message, type}`, где `message` используется для флэш сообщений и `type` является типом флэш сообщения (по умолчанию `type`).

### Подтверждающий коллбек

Паспорт стратегии требуют подтверждающего коллбэка, который обычно принимает параметры `(err, user, options?)`. `options.message` может использоваться для флэш сообщений. `user` должен быть `false` если пользователь не аутентифицирован. `err` означает ошибку сервера, например когда ваша база данных недоступна; вам не нужно устанавливать `err` если пользователь не прошел аутентификацию.

## Функции добавленные в Request

### req.login(user, callback)

Авторизует пользователя (заставляет passport сериализовать пользователя в сессию). По завершению req.user будет установлен.

### req.logout()

Удаляет req.user и очищает сессию.

## Passport и Сессии

Passport создает ключ в сессии называемый `session.passport`.

Когда запрос приходит в middleware `passport.session()`, passport запускает [встроеную стратегию 'session'](https://github.com/jaredhanson/passport/blob/2327a36e7c005ccc7134ad157b2f258b57aa0912/lib/strategies/session.js) - она вызывает `deserializeUser(session.passport.user, done)` чтобы прочитать пользователя из сессии и сохранить в req.user.

Вы можете переписать то как passport десериализует сессию путем создания новой стратегии 'session' и зарегистрировав ее с помощью `passport.use()`.

Когда вы вызываете `req.login()` или когда стратегия успешно аутентифицирует пользователя, passport использует [session manager](https://github.com/jaredhanson/passport/blob/2327a36e7c005ccc7134ad157b2f258b57aa0912/lib/sessionmanager.js#L12-L28) и по сути делает:

```js
serializeUser(req.user, (err, user) => {
    if(err) {return done(err);}
    session.passport.user = user;
});
```

Вы можете перезаписать менеджер сессий путем создания своей собственной реализации и установки `passport._sm`, но это не задокументировано, поэтому используйте на свой страх и риск.
