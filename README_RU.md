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
* [Стратегии](#strategies)
  * [Написание кастомных стратегий](#writing-custom-strategies)
  * [Подтверждающий коллбек](#verify-callback)
* [Функции добавленные в Request](#functions-added-to-the-request)
  * [req.login(user, callback)](#reqloginuser-callback)
  * [req.logout()](#reqlogout)
* [Passport и Сессии](#passport-and-sessions)

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

Это так же установит `req.login()` и `req.logout()`.

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

Данная функция возвращает middleware которая запускает стратегии. Если одна из стратегий пройдет успешно, будет установлен `req.user`. Если вы не передали options или callback, и все стратегии провалились, то будет записан 401 статус в ответе. Заметим что некоторые стратегии могут так же вызывать редирект (например OAuth).

Допустимые options:
* successRedirect - путь редиректа в случае успеха.
* failureRedirect - путь редиректа в случае неудачи (вместо 401).
* failureFlash - `true` для flash сообщения с ошибкой или `string` для использования в качестве сообщения об ошибке (переопределяет любые сообщения из самой стратегии).
* successFlash - `true` для flash сообщения успеха или `string` для использования в качестве сообщения успеха (переопределяет любые сообщения из самой стратегии).
* successMessage - `true` для сохранения сообщения успеха в `req.session.messages`, или `string` для перезаписи сообщения успеха.
* failureMessage - `true` для сохранения сообщения ошибки в `req.session.messages`, или `string` для перезаписи сообщения ошибки.
* session - `boolean`, разрешает поддержку сессий (по умолчанию `true`)
* failWithError - в случае ошибки вызывает `next()` с `AuthenticationError` вместо простой записи 401.

