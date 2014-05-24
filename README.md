FAQ по Derby 0.6 (на русском)
=====================

В этом репозитории я буду собирать часто задаваемые вопросы, а так же всевозможные тонкости [derbyjs](http://derbyjs.com)

### Как я могу добавить что-то в FAQ?

Напишите мне @zag2art сообщение через гитхаб или на почту (zag2art@gmail.com), а лучше сделайте форк, добавьте информацию и посылайте пулл-реквест.

### У меня есть вопрос, ответа я не знаю, но думаю всем бы было полезно

Оформляйте [issue](https://github.com/zag2art/derby-faq-ru/issues) в данном репозитории - постараюсь ответить, и если посчитаю вопрос достаточно интересным и общим - добавлю его в FAQ.

## Запросы

#### Как сделать реактивный запрос к количеству элементов в коллекции (сами элементы мне не нужны)?

```js
  var topicsCount = model.query('topics', {
    $count: true,
    $query: {
      active: true
    }
  });
  
  model.subscribe(tipicsCount, function(){
    topicsCount.refExtra('_page.topicsCount');
    
    // ...
  });
```

---
#### Как подписаться на определенные элементы коллекции (у меня уже есть реактивный список id, нужных элементов)?

В model.query обычно передаются 2 параметра, имя коллекции и объект с параметрами запроса, но есть еще один синтаксис:
```js
model.query(collection, path)
```

Здесь path - это путь к массиву id-шников документов, например, "_page.userIds", причем сам массив, вполне себе может быть реактивным.

Для чего это может быть нужно. Представьте себе чат, мы выводим страницу с одной из комнат чата. Туда постоянно входят новые люди, что-то там пишут. В сообщениях чата мы храним только id юзера, а остальная информация хранится в коллекции users. Естественно, для вывода сообщений имена юзеров нам нужны. Всех юзеров грузить на клиента смысла нет, нужны только те, сообщения которых есть в нашей комнате.

Зделаем так: 
 1. подпишемся на сообщения в комнате, 
 2. запустим реактивную функцию, которая будет собирать id всех юзеров, от которых в нашей комнате есть сообщения,
 3. подпишемся на коллекцию users, используя в качестве списка id, результат выполнения реактивной функции

```js
app.get('/chat/:room', function(page, model, params, next){
  var room = params.room;
  
  var messages = model.query('messages', {room: room});

  model.subscribe(messages, function(){
  
    messages.ref('_page.messages');
    
    // Запускаем реактивную функцию, она будет срабатывать при изменении messages
    // и записывать id-шки всех user-ов в _page.userIds
    
    model.start('_page.userIds', 'messages', 'pluckUserIds');
    
    var users = model.query('users', '_page.userIds');
    
    model.subscribe(users, function(){
      // ...
      page.render();
    });
  });
}

// Реактивные функции необходимо регистрировать после того, 
// как модель создана
app.on('model', function(model){
  model.fn('pluckUserIds', function (messages) {
    var ids = {};
    
    for (var key in messages) ids[messages[key].userId] = true;
    
    return Object.keys(ids);
  });
});
```

---
#### Как получить не сами элементы коллекции, а только их id?
```js
  var query = model.query('topics');
  
  model.subscribe(query, function(){
    query.refIds('_page.topicIds');
  });
```  
---
  Но необходимо учитывать, что сама коллекция topics в данном случае будет копироваться в браузер, чтобы этого избежать используйте проекции. В серверной части derby, в server.js определите проекцию для коллекции topics:
```  
  store.shareClient.backend.addProjection("topicIds", "topics", "json0", {id: true});
```  
  Далее с проекцией можно работать, как с обычной коллекцией.
```js  
  var query = model.query('topicsIds');
  
  model.subscribe(query, function(){
    query.refIds('_page.topicIds');
  });
```  
---
## Компоненты

#### Как в шаблонах компонент получить доступ к корневой области видимости модели?

Как известно, в компонентах своя, изолированная область видимости, поэтому для доступа к корню необходимо использовать префикс #root, например:
```html
<ul>
  {{each #root._page.topics as #topic}}
    <!-- ... -->
  {{/}}
<ul>
```
---
#### Как в коде компонент получить доступ к корневой области видимости модели?

Как известно, в компонентах своя, изолированная область видимости, поэтому, чтобы обратиться к корневой модели, вместо model, здесь необходимо использовать model.root. Например:
```js
function MyComponent() {}

MyComponent.prototype.init = function(model){
  // model.get('_page.topics') работать не будет
  var topics = model.root.get('_page.topics');
  // ...
  
}

MyComponent.prototype.onClick = function(event, element){
  var topics = this.model.root.get('_page.topics');
  // ...
  
}
```
---
## Модель

#### Мне не нужны все поля коллекции в браузере, как получать только определенные поля (проекцию коллекции)?

В серверной части derby-приложения прописываются все проекции:
```js
store.shareClient.backend.addProjection("topic_headers", "topics", "json0", {
  id: true, 
  header: true, 
  autor: true, 
  createAt: true
});

store.shareClient.backend.addProjection("users", "auth", "json0", {
  id: true, 
  username: true, 
  email: true
});
```
Далее с проекциями users и topic_headers в derby-приложении можно работать, как с обычными коллекциями.
```js
model.subscribe('users' function(){
  model.ref('_page.users', 'users');
  // ...
});
```
При создании проекций обратите внимание: поле id обязательно, пока поддерживается только белый список полей (перечисляем только поля, которые должны попасть в проекцию), так же поддерживается пока только возможность задавать поля первого уровня вложенности.

---
## База данных

#### Говорят появилась возможность обходится без redis-а, используя только mongodb, как это сделать?

В серверной части derby-приложения в момент создания объекта store необходимо прописать только mongodb-данные:
```js
  var store = derby.createStore({
    db: liveDbMongo(mongoUrl + '?auto_reconnect', {safe: true})
  });
```

Но стоит учесть то, что redis необходим, если вы планируете горизонтальное масштабирование (запускать несколько derby-серверов параллельно). 

---
## View'хи

#### Как вставить в шаблон неэкранированный html или текст?

Необходимо использовать unescaped модификатор, например:
```html
<header>
  {{topic.header}}
<header>

<!-- topic.unescapedTitle сделал только для примера, не знаю зачем такое может понадобиться -->
<article title="{{unescaped topic.unescapedTitle}}">
  {{unescaped topic.html}}
</article>
```

Учитывайте то, что это - потенциальная дыра в безопасности. Ваши данные должны быть полностью отчищены от опасных тегов, данные для атрибутов должны быть экранированы. В общем, прежде чем использовать такое, убедитесь, что вы понимаете, что делаете.

---
#### Как в шаблоне определенный блок сделать нереактивным (чтобы он не обновлялся сразу при изменении данных в модели)?

Во-первых, стоит сказать о том, что если нам в приложении вообще не нужна реактивность, то вместо подписки на данные нам стоит просто запрашивать их текущее состояние - вместо model.subscribe делать model.fetch.
```js
  // Так данные будут реактивно обновляться
  model.subscribe('topics', function(){
    // ...
  });

  // А так не будут
  model.fetch('topics', function(){
    // ...
  });
```  
Важно понимать, что здесь мы пока говорим только о модели и только об обновлениях, приходящих с сервера. Важно, что сделав fetch, если мы что-то добавили в коллекцию, использовав model.add - наши данные вполне себе попадут на сервер в б.д., с другой стороны, если данные в коллекцию добавил кто-то другой - они к нам не придут.
  
Теперь поговорим о реактивности html-шаблонов, по умолчанию, все привязки к данным там реактивные, то есть, как только поменялись данные в модели (не важно пришли ли обновления с сервера, или модель изменена кодом), изменения сразу же отразятся в html-шаблоне, но этим можно управлять.

Для управления руактивностью в шаблонах, используются такие зарезервированные слова, как bound и unbound, их можно использовать как в блочной нотации, так и в виде модификатора для выражений. Продемонстрирую:

```html
<p>
  <!-- по умолчанию привязка рекативная-->
  {{_page.text}}
</p>
<p>
  <!-- явно заданная реактивная привязка-->
  {{bound _page.text}}
</p>

<!-- Внутри этого блока все привязки по-молчанию реактивные -->
{{bound}}
  <p>
    <!-- прявязка реактивная, так как лежит внутри bound-блока-->
    {{_page.text}}
  </p>
  <p>
    <!-- прявязка не реактивная, так как это указано явно-->
    {{unbound _page.text2}}
  </p>
{{/}}

{{unbound}}
  <p>
    <!-- прявязка не реактивная, так как лежит внутри unbound-блока-->
    {{_page.text}}
  </p>
  
  <p>
    <!-- прявязка реактивная, так как это указано явно-->
    {{bound _page.text}}
  </p>
{{/}}

```

Естественно, для удобства, блоки могут быть вложены один в другой.

---
#### Как в шаблоне сделать так, чтобы в определенный момент обновился нереактивный блок?

Для того, чтобы в определенный момент перерисовать unbound-блок, нужно использовать ключевое слово on, примерно так:

```html
{{on _page.trigger}}
  {{unbound}}
    <!-- нереактивный html -->
  {{/}}
{{/}}

<!-- кнопка, по которой будем все это обновлять -->
<a href="#" on-click="refresh()">Refresh</a>
```
Нажатии на кнопку изменяем _page.trigger:
```js
app.proto.refresh = function(){
  app.model.set('_page.trigger', !app.model.get('_page.trigger'))
}
```
---
#### Как привязать реактивную переменную к элементу select?

---
#### Как привязать реактивную переменную к элементу input type=radio?

Никаких событий ловить не понадобится, derby все делает за нас:

```html
<label>
  <input type="radio" name="opt" value="one"   checked="{{_page.radioVal === 'one'  }}">One  
</label>
<label>
  <input type="radio" name="opt" value="two"   checked="{{_page.radioVal === 'two'  }}">Two  
</label>
<label>
  <input type="radio" name="opt" value="three" checked="{{_page.radioVal === 'three'}}">Three
</label>
```

В результате выбора в _page.radioVal будет либо 'one', либо 'two', либо 'three', в зависимости от того, что выберет пользователь.

При реактивной привязке к input-у c типом radio, derby предолагает, что встретит в атрибуте checked выражение проверки на равенство (сами равенства нужны для первоначальной установки занчения). Пердполагается, что левым параметром проверки на равенство будет путь, который обновится значением из атрибута value соответствующего input-а, при его выборе пользователем.
---
#### Как привязать реактивную переменную к элементу textarea?

Все очень просто:
```html
 <textarea value="{{@newSection.post}}"></textarea>
```

---
## Модули

#### Как к derby подключить шаблонизатор jade?
Для подключения jade к дерби необходимо использовать модуль [derby-jade](https://github.com/cray0000/derby-jade).

Устанавливаем его:
```bash
npm install derby-jade
```
В derby-приложении до использования app.loadViews() необходимо подключить этот модуль вот таким образом:
```js
app.serverUse(module, 'derby-jade');
```
Убедитесь, что у вас derby версии не младше 0.6.0-alpha7

---
#### Какой модуль использовать для авторизации в derby?

Используйте, надавно созданный специально под 0.6 версию derby, модуль [derby-login](https://github.com/vmakhaev/derby-login).

Доступна регистрация/авторизация через соц. сети, ведь внутри этот модуль использует [passportjs](passportjs.org).

---
#### Как подключать клиентские скрипты к derby-приложение, например, jquery?

Если скрипт находится на где-то на cdn-сервере, можно подключить его, просто вставив в html-шаблон тег script, например:

```html
<Body:>
  <!-- ... -->
  <script src="//code.jquery.com/jquery-1.11.0.min.js"></script>
```
Можно, конечно, сюда засунуть и сам скрипт, но лучше воспользоваться для этого предоставляемым browserify интерфейсом. Этот способ хорош еще и тем, что скрипт попадает в "бандл", который к странице подтягивается вторым запросом (после самого html-я страницы со встроенными стилями), что дает высокую скорость отрисовки.

Итак, рекомендуемый способ:

```js
// на серверной стороне derby? в файле server.js

store.on('bundle', function(browserify){
  // ваш локальный путь до файла скрипта
  browserify.add("../js/minified/jquery-1.11.0.min.js");
});
```

У себя для подключения клиентских скриптов, мы бычно используем bower, далее в grunt-e у нас настроены задачи по минификации и конкатенации всех вендорских скриптов в один. Итоговый vendor.min.js мы подключаем, используя вышеизложенный метод.

---
## Написание модулей

#### Как проверить, что код в derby-приложении выполняется на клиенте/на сервере?

В derby есть модуль с утилитами, в нем присутствуют нужные нам флаги:
```js
var derby = require('derby');

if (derby.util.isServer) {
  // код, который должен выполняться, 
  // только если мы находимся на сервере
}

if (derby.util.isClient) {
  // код, который должен выполняться, 
  // только если мы находимся на клиенте
}
```

---
#### Как сделать require модуля в derby-приложении, код которого будет выполняться только на сервере?

Объясню сначала в чем вообще особенность require модулей, которые нужны нам только для сервера. Дело в том, что все модули derby-приложения, которые подключаются к нему через классическое CommonJS-ное require полюбому попадают в "бандл", а следовательно будут копироваться в браузер к клинету. Нам же не особо хочется, чтобы в браузер попадали лишние данные, тем более, если модуль заведомо нужен только для работы на сервере. Поэтому вместо require используем serverRequire из набора утилит derby:

```js
var derby = require('derby');

if (derby.util.isServer) {
  // Пакет точно не попадет к клиентам в браузер
  var myPackage = derby.util.serverRequire('mypackage');
  // ...
}
```
