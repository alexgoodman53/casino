# API Платформы

`API.URI` - уточняйте в тех.поддержке.

## Запросы

1. Все запросы к АПИ выполняются в формате `POST(HTTP)`.
2. `URI` запроса формируется по схеме `API.URI + '/' + serviceName + '.' + methodName`.
3. Все параметры передаются в `application/x-www-form-urlencoded`.
4. Все запросы должны быть подписаны с использованием Секретного ключа(если не указано обратное для конкретного метода).

- В подписи не участвуют параметры запроса имена которых начинаются с `partner.`
- Допустимо передавать в параметрах различную мета информацию (с неопределенной целью). Для того, чтобы эти параметры не участвовали в подписи их имена должны начинаться с `meta..`
- Пример: рекомендуется, но не обязательно передавать параметр со случайной величиной, который в дальнейшем может ускорить поиск нужного запроса в логах( например: `meta.id=random` ).

## Ответы

1. Все ответы Платформа отдает в формате `JSON`.

- `Http Status Code` - не является частью апи и во всех ответах должен быть `200` (в случае успешных и неуспешных вызовах). `Http Status Code` отличный от `200` говорит либо о серьезных неполадках на сервере, либо полностью ошибочном запросе. Сверьтесь с документацией или обратитесь в тех.поддержку.
- Общая схема положительного ответа : Все положительные ответы должны в теле ответа содержать `"status": 200`

```json
{
  "status": 200,
  "method": "service.method",
  "response": ...response entity as json...
}
```

- Общая схема отрицательного ответа:

```json
{
  "status": "Цифровой код отличный от 200",
  "method": "service.method",
  "message": "Строковое описание ошибки. В целях безопасности, описание не обязательно четко описывает, что произошло на сервере, но в некоторых случаях может служить подсказкой. В остальных случаях обратитесь в тех.поддержку."
}
```

## Перечень параметров, которые передаются для всех методов

- `@partner.alias` - Идентификатор партнера
- `@partner.sign` - Подпись сформированная с использованием параметров запроса и Секретного Ключа.

Во всех примерах ниже для формирования подписи использованы следующие параметры:

- `Идентификатор Партнера = test`
- `Секретный ключ = testsecret`

## Формирование Подписи

- Подпись в запросе передается как именованный параметр `partner.sign`
- В формировании подписи участвуют ВСЕ параметры, которые передаются в теле `POST` запроса (`GET` параметры не учитываются), кроме тех, чьи имена начинаются с `partner.` и `meta.`
- Формирование:

1. Формируем набор параметров любым удобным способом.
2. Берем список имен всех параметров, которые участвуют в запросе.
3. Фильтруем список имен. Оставляем только те имена, которые НЕ начинаются с `partner.` и `meta.`
4. Сортируем в лексикографическом порядке
5. Полный алгоритм по формированию подписи(pseudo code):

```scala
val params = Map[Name,Value]

val joined = names
  .filter( name => name.startsWith("partner.") == false && name.startsWith("meta.") == false)
  .sorted
  .map( name => name + "=" + params(name))
  .join("&")

// Соединительная частица "&" между 'joined' и 'serviceName' проставляется в любом
// случае, даже при отсутствии параметров.
val sign = md5(joined + "&" + serviceName + "." + methodName + "&" + "Идентификатор Партнера" + "&" + "Секретный Ключ")
```

## Полный пример формирования подписи

```text
METHOD: /games.list
PARAMS:
paramA = paramValueA
paramC = paramValueC
paramZ = paramValueZ
paramB = paramValueB
partner.alias = test
meta.param1 = someValue
meta.paramEtc = etcValue

SIGN:
// URL-encode - НЕ применяется
joined = "paramA=paramValueA&paramB=paramValueB&paramC=paramValueC&paramZ=paramValueZ"

stringToSign = joined + "&games.list&test&testsecret"
//stringToSign = "paramA=paramValueA&paramB=paramValueB&paramC=paramValueC&paramZ=paramValueZ&games.list&test&testsecret"

sign = md5(stringToSign)
//sign = "8cb94a439f507c1a6f9cede4982380a1"

FULL REQUEST BODY:
paramA=paramValueA&paramB=paramValueB&paramC=paramValueC&paramZ=paramValueZ&games.list&test&testsecret&partner.alias=test&partner.sign=8cb94a439f507c1a6f9cede4982380a1&meta.param1=someValue&meta.paramEtc=etcValue
```

## Список методов

### `/games.list` - Получение списка игр доступных для партнера

- Параметры - нет
- Подпись - `Не требуется`
- Результат - массив обьектов Игра:

```json
{
  "id": 114,
  "alias": "Aloha",
  "group_alias": "slots",
  "title": "Aloha",
  "provider": "netent",
  "is_enabled": true,
  "desktop_enabled": true,
  "mobile_enabled": true
}
```

Пример запроса/ответа в формате `CURL`

```bash
curl -X POST -d "partner.alias=test&partner.sign=c0b8489e2655c6b21c2b9cb4d239634b" http://api.netent.local/games.list
```

```json
{
  "method": "games.list",
  "status": 200,
  "response": [
    {
      "id": 114,
      "alias": "Aloha",
      "group_alias": "slots",
      "title": "Aloha",
      "provider": "netent",
      "is_enabled": true,
      "desktop_enabled": true,
      "mobile_enabled": true
    },
    ...
  ]
}
```

### `/netent.freeroundsInfo` - Получение краткой информации о состоянии фрираунда

- Параметры
  - `@partner.alias` - `String`
  - `@freerounds_id` - `String`
- Подпись - `Не требуется`
- Результат - обьект:

```json
{
  "max_step": "Int",
  "steps": "Int",
  "steps_last_win": "Long",
  "total_win": "Long"
}
```
