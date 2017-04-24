# Мобилизация. Школа разработки интерфейсов. 
### Задание3

Начнём распутывать фатальную цепочку событий с конца.

Начнём с того, что ServiceWorker перестал обрабатывать запросы за ресурсами приложения: HTML-страницей, скриптами и стилями из каталогов vendor 
и assets.

Сначала я запустил `gifs.html`, ввёл несколько запросов, попытался добавить какие-то из найденных гифок в избранное, увидел, что они не 
кэшируются, зашёл в консоль, увидел там `«[ServiceWorkerContainer] ServiceWorker is registered! blocks.js:471»`, перешёл на 471 строчку кода в 
`blocks.js`. Дальше я зашёл в документацию по Service Worker 
(https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API/Using_Service_Workers) и в пункте «Регистрация воркеров» прочитал про 
параметр 'scope' — область видимости. В 469 строке кода ServiceWorker регистрируется в ./assets/, поэтому корневая папка находится вне его 
зоны видимости. Далее я вернул `service-worker.js` в корневую папку и заменил в 469 строке 

```javascript .register('./assets/service-worker.js')```
на ```javascript .register('./service-worker.js')``` 

Также в 9 строке `service-worker.js` я заменил 

```javascript importScripts('../vendor/kv-keeper.js-1.0.4/kv-keeper.js');``` 
на ```javascript importScripts('./vendor/kv-keeper.js-1.0.4/kv-keeper.js');```

Стало лучше, но ненамного. Гифки всё равно не кэшируются. Далее я вспомнил про проблемы с обновлением статики из директорий vendor и assets и 
решил начать копать оттуда. Волшебный Ctrl+F по запросу 'vendor' почти сразу показал мне функцию

```javascript function needStoreForOffline(cacheKey) {
    return cacheKey.includes('vendor/') ||
        cacheKey.includes('assets/') ||
        cacheKey.endsWith('jquery.min.js');
        }
```


В ней видно, что кэшируются только файлы в папкак assets и vendor, но никак не gifs.html. Добавил его в функцию.

Теперь мы вернулись к первой проблеме: после рефакторинга под названием «Более надёжное кеширование на этапе fetch» у клиентов HTML-страница 
стала браться из кеша не только в офлайн-режиме, а всегда. 
Я перешёл к обработчику события 'fetch' и начал его изучать. Увидел, что в проверке

```javascript if (needStoreForOffline(cacheKey)) {
        response = caches.match(cacheKey)
            .then(cacheResponse => cacheResponse || fetchAndPutToCache(cacheKey, event.request));
    } else {
        response = fetchWithFallbackToCache(event.request);
    }
```
нарушена логика и файлы после кэширования всегда берутся из кэша. Заменил этот фрагмент на следующий:

```javascript if (needStoreForOffline(cacheKey)) {
    response = fetchAndPutToCache(cacheKey, event.request);
} else {
    response = fetchWithFallbackToCache(event.request);
}
```

Ну и напоследок я переименовал константу `'CACHE_VERSION'`   с ```javascript const CACHE_VERSION = '1.0.0-broken';``` на ```javascript const CACHE_VERSION = '1.0.1';```

***
### Ответы на вопросы.

***// Вопрос №1: зачем нужен этот вызов?***
      ```javascript  .then(() => self.skipWaiting())```

**Ответ:** Этот вызов позволет обновлённому ServiceWorker`у вызвать событие активации. Подробнее во втором ответе.

***// Вопрос №2: зачем нужен этот вызов?***
           ```javascript self.clients.claim();```

**Ответ:** Делает ServiceWorker активным для всех страниц, которые он контролирует. Иначе после обновления ServiceWorker пришлось бы закрывать все страницы, которые им контролируется, так как страницы контролировались бы устаревшим ServiceWorker (см. https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API/Using_Service_Workers →
Базовая архитектура → пункт 7.)

***// Вопрос №3: для всех ли случаев подойдёт такое построение ключа?***
    ```javascript const cacheKey = url.origin + url.pathname;```

**Ответ:** Нет. В этом ключе не будут учтены параметры GET-запроса (url.search) или, например, якоря (url.hash).

***// Вопрос №4: зачем нужна эта цепочка вызовов?***
```javascript return Promise.all(
                names.filter(name => name !== CACHE_VERSION)
                    .map(name => {
                        console.log('[ServiceWorker] Deleting obsolete cache:', name);
                        return caches.delete(name);
                    })        
```
**Ответ:** Тут удаляется весь кэш, чья версия отличается от текущей версии хэша (равной константе 'CACHE_VERSION').


***// Вопрос №5: для чего нужно клонирование?***
                  ```javascript  cache.put(cacheKey, response.clone());```


**Ответ:** Поток можно обработать только один раз. Поэтому тут создаётся клон. Один request отправляется браузеру, а второй кэшируется.
