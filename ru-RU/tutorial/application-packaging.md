# Упаковка приложения

Чтобы смягчить [проблемы](https://github.com/joyent/node/issues/6960) с длинными
именами под  Windows, немного ускорить `require` и скрыть ваши исходные коды, Вы
можете упаковать его в архив [asar][asar], немного поменяв исходный код.

## Генерация архива `asar`

Архив [asar][asar] - простой фомат похожий на tar, который собирает много файлов
в один. Electron может читать такой файл без распаковки.

Шаги для упавки вашего приложения архив `asar`:

### 1. Установите саму утилиту asar

```bash
$ npm install -g asar
```

### 2. Упакуйте с помощью `asar pack`

```bash
$ asar pack your-app app.asar
```

## Использование архивов `asar`

В Electron есть два вида API: API Node, которые устанавливаются с помощью Node.JS и
веб API, которые предоставляюся Chromium. Оба предоставляют возможность считывать из
архивов `asar`.

### Node API

Со специальными патчами в Electron, части Node API вроде `fs.readFile` и `require`
считают архивы `asar` виртуальными папками и файлы в них доступны как в обычных.

Например, у нас есть архив `example.asar` в `/path/to`:

```bash
$ asar list /path/to/example.asar
/app.js
/file.txt
/dir/module.js
/static/index.html
/static/main.css
/static/jquery.min.js
```

Прочитаеем файл в архиве `asar`:

```javascript
const fs = require('fs');
fs.readFileSync('/path/to/example.asar/file.txt');
```

Список всех файлов начиная от корня архива:

```javascript
const fs = require('fs');
fs.readdirSync('/path/to/example.asar');
```

Используем модуль из архива:

```javascript
require('/path/to/example.asar/dir/module.js');
```

Вы также можете показывать веб-страницы из архива `asar` через `BrowserWindow`:

```javascript
const BrowserWindow = require('electron').BrowserWindow;
var win = new BrowserWindow({width: 800, height: 600});
win.loadURL('file:///path/to/example.asar/static/index.html');
```

### Веб API

На веб-страницах файлы запрашиваются с помощью протокола `file:`. Как и в Node API
архивы `asar` считаются за директории.

Пример получения файла с помощью `$.get`:

```html
<script>
var $ = require('./jquery.min.js');
$.get('file:///path/to/example.asar/file.txt', function(data) {
  console.log(data);
});
</script>
```


### Использование архива `asar` в качестве обычного файла

Для случаев, когда Вам, например, нужно проверить хэш-сумму архива `asar`,
нужно использовать архив как файл. Для этой цели существует встроенный модуль
`original-fs`, который предоставляет доступ к `fs` без поддежки `asar`:

```javascript
var originalFs = require('original-fs');
originalFs.readFileSync('/path/to/example.asar');
```

Вы также можете выставить `process.noAsar` в `true`, чтобы выключить поддержку `asar`
в модуле `fs`:

```javascript
process.noAsar = true;
fs.readFileSync('/path/to/example.asar');
```

## Ограничения Node API

Хотя мы и старались как могли, чтобы сделать `asar` максимально похожим на папки,
всё ещё существуют некоторые ограничения из-за низкоуровневой архитектуры Node API.

### Архивы только для чтения

Архивы не могут быть изменены, так что все функции API Node, которые меняют файлы,
не буду работать с архивами `asar`.

### Нельзя установить рабочую директорию в архиве

Хотя архивы `asar` и считаются папками, они ими на самом деле не являются ими,
так что Вы не можете установить в них рабочую директорию. Передача
архивов `asar` в качестве аргумента `cwd` некоторым API также может вызывать ошибки.


### Распаковка для некоторых API

Большинство API `fs` могут читать файлы или получить сведения о них прямо из
архива, без распаковки, однако для некоторых, которые передают путь к системным
вызовам, Electron распакует нужный файл в временную папку и передаст путь к этому
файлу.

API которым нужна распаковка:

* `child_process.execFile`
* `child_process.execFileSync`
* `fs.open`
* `fs.openSync`
* `process.dlopen` - используется `require` на нативных модулях

### Подельная информация для `fs.stat`

Объект `Stats`, возвращаемый `fs.stat`, и его друзья для остальных файлов
в архиве `asar` специально генерируются, потому что на самом деле этих файлов
не существует в файловой системе. Вам не стоит доверять информации
из объектов `Stats`, кроме, разве что, размера и типа файлов.

### Запуск исполняемых файлов из архивов `asar`

Существуют некоторые API Node, которые исполняют файлы, например `child_process.exec`,
`child_process.spawn` и `child_process.execFile`, но только `execFile` может
исполнять файлы из архивов `asar`.

Так вышло потому, что `exec` и `spawn` принимают `команду`, а не `файл` как параметр,
а `команды` исполняются в оболочке. Нет никакой реальной возможности проверить,
реален ли файл или находится в архиве `asar`, и даже если мы смогли бы проверить,
то неясно, как земенить путь к файлу без побочных эффектов.

## Добавление распакованых файлов в архив `asar`

Как говорилось выше, некоторые API Node будут распаковывать файлы,
чтобы их использовать. Кроме увеличенного потребления ресурсов это также
может вызвать предупреждения от антивирусов.

Чтобы обойти это, Вы можете распаковать некоторые файлы, создавая архивы,
с помощью опции `--unpack`. Пример показывает распаковку нативных модулей:

```bash
$ asar pack app app.asar --unpack *.node
```

После запуска команды выше в вашей папке, кроме `app.asar`, появится
`app.asar.unpacked`, которая будет содержать распакованные файлы, эту
папку стоит копировать вместе с `app.asar` при распространении.

[asar]: https://github.com/atom/asar