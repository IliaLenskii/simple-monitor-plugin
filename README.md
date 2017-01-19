# Пример плагина для монитора

* [Описание](#Описание)
* [package.json](#packagejson)
* [Маршрутизация](#Маршрутизация)
* [API](#api)
* [Примеры](#Примеры)
* [Ссылки](#Ссылки)

## Описание (актуально для монитора версии 6.4.1.47)

Плагин должен быть полноценным модулем в формате npm, самостоятельно реализующим свои зависимости от других модулей.

Каждый плагин запускается в отдельном процессе.
При этом монитор создаёт:

* рабочий каталог, в котором плагин может работать с файлами
* каталог для логов
* сокет, на который будут переправляться запросы, предназначенные плагину. Если плагин должен обрабатывать http запросы, переадресуемые ему монитором, его экземпляр `http.Server` должен слушать этот сокет

После запуска процесса с плагином монитор инициализирует в глобальной области видимости объект [`global.KServerApi`](#api).

## **package.json**

В `package.json` добавлены следующие значимые для монитора поля:

- `route` \<String\> строка URI, определяющая запросы, переадресуемые плагину
- `required_licenses` \<Array\> массив номеров лицензий кодекса, которые должны присутствовать в рег.файле для запуска плагина

В `package.json` плагина должен быть указан список лицензий рег. файла кодекса, которые необходимы для его запуска: 
```
"required_licenses": [4360]
```
Лицензия 4360 выбрана для примера. Перед запуском плагина монитор проверяет наличие всех перечисленных лицензий, и если чего-то не хватает, плагин не будет запущен.

## **Маршрутизация**

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

В [репозитории monitor-plugin-sim](https://github.com/monitor-plugin-sim/monitor-plugin-sim) представлен симулятор подсистемы плагинов монитора, отвечающий за маршрутизацию запросов-ответов между монитором и плагином.

## **Лицензирование и права доступа**

Каждый плагин должен иметь хотя бы одну лицензию (*см. package.json: required_licenses*). Кроме того, может возникнуть потребность лицензировать какой-либо отдельный функционал плагина.

 *Например: система имеет в своём составе модули (подсистемы) создания и редактирования документов и генерации отчетов и хочется количество рабочих мест для этих функционалов регулировать отдельно друг от друга и от всей системы в целом, (количество рабочих мест на систему в целом __50__, на создание и редактирование документов __12__, на генерацию отчетов __3__), а также имеется функциональность по управлению компонентами системы доступ к которой должен быть просто ограничен (администратором, без ограничения на количество рабочих мест)*.

Для каждого функционала плагина, который должен быть подвергнут лицензированию и/или разграничению прав доступа, необходимо получить номер лицензии *(у разработчиков kserver'а)*.

Перед тем как разрешить доступ к функционалу для конкретного пользователя код плагина должен проверить разрешен ли пользователю такой доступ с помощью метода `KServerApi.CheckAccess`, передав ему id сессии кодекс-сервера или объекта `request` (IncomingMessage) *(см. `pickKServerInfo`)*

*Рекомендации разработчикам плагинов:<br>
Выполнять проверку `KServerApi.CheckAccess` для каждого запроса может оказаться накладно с точки зрения используемых ресурсов сервера. Что бы снизить нагрузку на систему "кэшировать" ответы `KServerApi.CheckAccess` используя, например, токен с коротким (1 минута) временем жизни совместно с механизмом сессий.*


## **API**

### Вспомогательные методы

#### **pickKServerInfo(request)**

* `request` \<IncomingMessage\>

Собирает относящуюся к KServer'у информацию о запросе и помещает её в свойство `KServer` объекта `request` (IncomingMessage):

* `session` идентификатор сессии кодекс-сервера

Пример:

```
app.use((req, res, next) => {
  if (pickKServerInfo)
    pickKServerInfo(req);
  next();
});
//...
app.get('<some_url>', (req, res) => {
  some_api_func(req.KServer.session);
});

```

### Содержимое объекта `global.KServerApi`:

* `Name` уникальное в пределах монитора имя плагина
* `Path` полный путь к плагину
* `StoragePath` полный путь к каталогу, в котором предполагается хранение файлов плагинов
* `LogsPath` полный путь к каталогу логов плагина
* `SocketPath` имя сокета, который должен слушать плагин
* `Info` `package.json` плагина
* `UserInfo` метод для получения информации о пользователе
* `CheckAccess` метод для проверки доступа к функционалу
* `KodeksDocInfo` метод для получения информации о документе ИС "Кодекс/Техэксперт"
***

#### **UserInfo(session)**

* `session` идентификатор сессии пользователя (см. `pickKServerInfo`) или объект `request` (IncomingMessage)
* Returns: \<Promise\>.

Обработчику resolve (в случае успеха) передается объект с информацией о пользователе:

* `authenticated` булево значение, флаг аутентифицирован ли пользователь

// если не аутентифицирован
* `ip` ip адрес пользователя (IPv4 или IPv6)

// если аутентифицирован
* `login` логин
* `groups` массив объектов вида {id: <id>, name: <name>}, содержащих идентификаторы и названиям групп, в которые пользователь входит
* `name` имя пользователя
* `email`
* `department` подразделение
* `position` должность
* `disabled` булево значение, содержащее статус блокировки учётной записи
* `expired` булево значение, содержащее информацию об истечении срока действия учётной записи

Обработчику reject (в случае неудачи) передается объект `Error` c описанием ошибки.

Пример:
```
/**
 * @param {string} session
 * @returns {Promise}
 */
global.KServerApi.UserInfo(session)
.then(userinfo => {})
.catch(error => {})
```
***

#### **CheckAccess(featureId, session)**

* `featureId` идентификатор функционала, который должен быть проверен (см. *Лицензирование*)
* `session` идентификатор сессии пользователя (см. *`pickKServerInfo`*) или объект `request` (IncomingMessage)
* Returns: \<Promise\>.

Обработчику resolve (в случае успеха) передается объект:
```
{
  granted: <Bolean>, // флаг: доступ разрешен/запрещён
  reason: <String>   // причина отказа (в случае отказа)
}
```
Обработчику reject (в случае неудачи) передается объект `Error` c описанием ошибки.

Пример:

```
//...
app.get('<some_url>', (req, res, next) => {
  KServerApi.CheckAccess(555000, req)
  .then(access => {
    if (!access.granted) {
      accessDenyHandler(req, res, access.reason);
      return;
    }
    next();
  })
  .catch(error => {
    accessDenyHandler(req, res, error);
  })
});

```
***

#### **KodeksDocInfo(docNum[, session])**

* `docNum` номер документа, информацию о котором требуется получить.
* `session` идентификатор сессии пользователя (см. `pickKServerInfo`), или объект `request` (IncomingMessage), или `null`
* Returns: \<Promise\>.

Обработчику resolve (в случае успеха) передается объект:
```
{
  status: <String>,  // статус документа
  tooltip: <String>  // наименование документа с дополнительной информацией
}
```
Свойство `status` может принимать следующие значения:
   *'active', 'inactive', 'card_active', 'card_inactive', 'card_undefined', 'project', 'project inactive', 'situation', 'situation_inactive', 'themes', 'technicalDocument', 'technicalDocumentNew', 'book', 'bookmark', 'imp_news'*

Обработчику reject (в случае неудачи) передается объект `Error` c описанием ошибки.

## **Примеры**

- [simple](https://github.com/simple-monitor-plugin/tree/master/simple) - пример простого плагина
- [express](https://github.com/simple-monitor-plugin/tree/master/express) - пример плагина, использующего [express.js](http://expressjs.com/)

## Ссылки

- [симулятор подсистемы плагинов монитора](https://github.com/dev-kodeks/monitor-plugin-sim)
