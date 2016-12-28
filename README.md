# Пример плагина для монитора

* [Описание](#Описание)
* [package.json](#packagejson)
* [Маршрутизация](#Маршрутизация)
* [API](#api)
* [Примеры](#Примеры)
* [Ссылки](#Ссылки)

## Описание (актуально для монитора версии 6.4.1.43)

Плагин должен быть полноценным модулем в формате npm, самостоятельно реализующим свои зависимости от других модулей.

Каждый плагин запускается в отдельном процессе.
При этом монитор создаёт:

* рабочий каталог, в котором плагин может работать с файлами
* каталог для логов
* сокет, на который будут переправляться запросы, предназначенные плагину. Если плагин должен обрабатывать http запросы, переадресуемые ему монитором, его экземпляр `http.Server` должен слушать этот сокет

После запуска процесса с плагином монитор инициализирует в глобальной области видимости объект `global.KodeksApi`((#api)).

## package.json

В `package.json` добавлены следующие значимые для монитора поля:

- `route (string)` строка URI, определяющая запросы, переадресуемые плагину
- `required_licenses (array)` массив номеров лицензий кодекса, коотрые должны присутствовать в рег.файле для запуска плагина

В `package.json` плагина может быть указан список лицензий рег. файла кодекса, которые необходимы для его запуска: 
```
"required_licenses": [4360]
```
Лицензия 4360 выбрана для примера. Перед запуском плагина монитор проверяет наличие всех перечисленных лицензий, и если чего-то не хватает, плагин не будет запущен.

## Маршрутизация

Если плагин должен обрабатывать http запросы, в его `package.json` должен присутствовать параметр 
```
"route": "<ROUTE_TEMPLATE>"
```
В этом случае все запросы, подходящие под шаблон, монитор будет переадресовывать плагину.

**Правила задания шаблонов роутинга соответствуют описанным в модуле [router](https://www.npmjs.com/package/router)**

Например, маршрут плагина задан маской
```
"route": "arm"
```
Монитор получил запрос
```
GET /arm/status?name=test
```
Плагину перенаправляется запрос:
```
GET /arm/status?name=test
```

В [репозитории monitor-plugin-sim](https://github.com/Murzilka/monitor-plugin-sim) представлен симулятор подсистемы плагинов монитора, отвечающий за маршрутизацию запросов-ответов между монитором и плагином.

## API

Содержимое объекта `global.KodeksApi`:

* `Name` уникальное в пределах монитора имя плагина
* `Path` полный путь к плагину
* `StoragePath` полный путь к каталогу, в котором предполагается хранение файлов плагинов
* `LogsPath` полный путь к каталогу логов плагина
* `SocketPath` имя сокета, который должен слушать плагин
* `Info` `package.json` плагина
* `UserInfo` метод для получения информации о пользователе

Если пользователь прошёл аутентификацию в кодексе, ему записывается кука `Auth`, хранящая строку вида `userId:pswd` в base64 кодировке.
В этом случае можно получить информацию о пользователе путём вызова метода `UserInfo` с передачей ему `userId` в качестве параметра.
Метод возвращает `Promise`.

```
/**
 * @param {string} userId
 * @returns {Promise}
 */
global.KodeksApi.UserInfo(userId)
.then(userinfo => {})
.catch(error => {})
```

Состав объекта, возвращаемого в случае успешного вызова UserInfo:

* `id` строка с id пользователя
* `login` логин
* `groups` массив строк, соответствующих названиям групп, в которые пользователь входит
* `name` имя пользователя
* `email`
* `department` подразделение
* `position` должность
* `disabled` булево значение, содержащее статус блокировки учётной записи
* `expired` булево значение, содержащее информацию об истечении срока действия учётной записи

## Примеры

- [simple](https://github.com/Murzilka/simple-monitor-plugin/tree/master/simple) - пример простого плагина
- [express](https://github.com/Murzilka/simple-monitor-plugin/tree/master/express) - пример плагина, использующего [express.js](http://expressjs.com/)

## Ссылки

- [симулятор подсистемы плагинов монитора](https://github.com/Murzilka/monitor-plugin-sim)
