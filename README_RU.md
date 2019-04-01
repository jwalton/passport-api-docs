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

Официальная [документация passport](http://www.passportjs.org/docs/) имеет длинный стиль, сопровождаемый примерами. Некоторые людям это нравится, а другие хотят узнать "когда я вызываю эту функцию, что произойдет?". Это руководство именно для таких людей.

Если вы обнаружите неточности, пожалуйста не стеснятесь открыть ишью или пулл реквест.

## Класс Passport

