# API Slotegrator

> Документ составлен **только** по PDF `GIS-API_Slotegrator 1.4.4.pdf`.
>
> Если в PDF нет достаточных данных для точного вывода, это **прямо помечено** в тексте как `Недостаточно данных в PDF`.

## Оглавление

- [1. О документе и источнике](#1-о-документе-и-источнике)
- [2. Общая модель API](#2-общая-модель-api)
- [3. Сводная таблица методов и точек входа](#3-сводная-таблица-методов-и-точек-входа)
- [4. Базовые правила API](#4-базовые-правила-api)
- [5. Безопасность и подпись запросов](#5-безопасность-и-подпись-запросов)
- [6. Общая схема взаимодействия](#6-общая-схема-взаимодействия)
- [7. Каталог игр и запуск игры](#7-каталог-игр-и-запуск-игры)
- [8. Callback API: как агрегатор ходит в backend казино](#8-callback-api-как-агрегатор-ходит-в-backend-казино)
- [9. Дополнительные методы агрегатора](#9-дополнительные-методы-агрегатора)
- [10. Freespins API](#10-freespins-api)
- [11. Freevouchers API](#11-freevouchers-api)
- [12. Self validation](#12-self-validation)
- [13. Схема прохождения транзакций](#13-схема-прохождения-транзакций)
- [14. Что обязательно учесть при реализации](#14-что-обязательно-учесть-при-реализации)
- [15. Что в PDF не раскрыто полностью](#15-что-в-pdf-не-раскрыто-полностью)
- [16. Примеры из PDF](#16-примеры-из-pdf)

---

## 1. О документе и источнике

Этот файл описывает API Slotegrator на русском языке по документу:

- `GIS-API_Slotegrator 1.4.4.pdf`

Версия документа:

- `1.4.4`
- дата в changelog: `2026-02-26`

По PDF видно, что API делится на две большие части:

1. **Game Aggregator API**
   Это методы, в которые ходит интегратор казино.
2. **Integrator callbacks**
   Это вызовы, в которые уже агрегатор ходит в backend казино во время игры.

Также в PDF есть дополнительные блоки:

- лимиты;
- jackpots;
- freespins;
- freevouchers;
- self validation.

---

## 2. Общая модель API

Slotegrator API работает по такой схеме:

1. Казино получает каталог игр.
2. Если игре нужен lobby, казино отдельно получает данные lobby.
3. Казино инициализирует игровую сессию.
4. Игрока редиректят в игру.
5. Во время игры агрегатор ходит в callback endpoint казино за:
   - балансом;
   - списанием ставки;
   - начислением выигрыша;
   - refund;
   - rollback.

То есть это **seamless wallet** схема:

- игра живет у агрегатора;
- деньги игрока живут у казино;
- агрегатор обращается к казино при денежных событиях.

Основание в PDF:

- launch flow: стр. `5-6`
- `/games`, `/games/lobby`, `/games/init`, `/games/init-demo`: стр. `8-14`
- callbacks `balance`, `bet`, `win`, `refund`, `rollback`: стр. `14-21`

---

## 3. Сводная таблица методов и точек входа

Ниже собраны все методы и точки входа, которые **прямо видны в PDF**.

### 3.1. Game Aggregator API

| Метод | Тип | Краткое описание по PDF |
| --- | --- | --- |
| `GET /games` | Метод агрегатора | Получение списка игр, доступных для Merchant ID. |
| `GET /game-tags` | Метод агрегатора | Получение списка игровых тегов. |
| `GET /games/lobby` | Метод агрегатора | Получение данных lobby для игр, где lobby предусмотрен. |
| `POST /games/init` | Метод агрегатора | Инициализация игровой сессии и получение URL запуска. |
| `POST /games/init-demo` | Метод агрегатора | Инициализация demo-сессии и получение URL запуска. |

### 3.2. Callback actions в сторону казино

| Callback / action | Тип | Краткое описание по PDF |
| --- | --- | --- |
| `action=balance` | Callback агрегатора | Запрос текущего баланса игрока у казино. |
| `action=bet` | Callback агрегатора | Запрос на списание ставки с баланса игрока. |
| `action=win` | Callback агрегатора | Запрос на зачисление выигрыша игроку. |
| `action=refund` | Callback агрегатора | Возврат средств по игровой операции. |
| `action=rollback` | Callback агрегатора | Откат ранее проведенных транзакций. |

### 3.3. Дополнительные методы агрегатора

| Метод | Тип | Краткое описание по PDF |
| --- | --- | --- |
| `GET /limits` | Метод агрегатора | Получение информации о лимитах по валютам и провайдерам. |
| `GET /limits/freespin` | Метод агрегатора | Получение freespin-лимитов по валютам и провайдерам. |
| `GET /jackpots` | Legacy-метод агрегатора | Получение jackpot-значений. В PDF помечен как `LEGACY`. |
| `POST /balance/notify` | Legacy-метод агрегатора | Уведомление об изменении баланса. В PDF помечен как `LEGACY`. |

### 3.4. Freespins API

| Метод | Тип | Краткое описание по PDF |
| --- | --- | --- |
| `GET /freespins/bets` | Метод freespins API | Получение допустимых ставок для freespin campaign. |
| `POST /freespins/set` | Метод freespins API | Создание freespin campaign для игрока. |
| `GET /freespins/get` | Метод freespins API | Получение состояния freespin campaign. |
| `POST /freespins/cancel` | Метод freespins API | Отмена freespin campaign. |

### 3.5. Freevouchers API

| Метод | Тип | Краткое описание по PDF |
| --- | --- | --- |
| `POST /freevouchers/set` | Метод freevouchers API | Создание freevoucher campaign. |
| `GET /freevouchers/get` | Метод freevouchers API | Получение состояния freevoucher campaign. |
| `POST /freevouchers/cancel` | Метод freevouchers API | Отмена freevoucher campaign. |

### 3.6. Self validation

| Метод | Тип | Краткое описание по PDF |
| --- | --- | --- |
| `POST /self-validate` | Метод self validation | Запуск self-validation проверки интеграции. |

### 3.7. Что важно про эту таблицу

- таблица включает только методы и точки входа, которые явно перечислены в PDF;
- `action=...` callbacks не являются REST-методами агрегатора, а являются входящими callback-действиями в backend казино;
- `/jackpots` и `/balance/notify` в PDF прямо помечены как `LEGACY`;
- по `/games/init-demo` и `/self-validate` PDF не раскрывает все детали payload, но сами методы как точки входа описаны явно.

---

## 4. Базовые правила API

### 4.1. Транспорт

Для запросов интегратора к агрегатору в PDF указано:

- HTTP/1.1
- параметры передаются как `application/x-www-form-urlencoded`

Основание:

- общий раздел API, стр. `2-3`

### 4.2. Формат ответа

По умолчанию ответы агрегатора — JSON:

- `Content-Type: application/json`

Основание:

- стр. `3`

### 4.3. HTTP-коды

В PDF перечислены стандартные HTTP-коды:

- `200`
- `201`
- `204`
- `304`
- `400`
- `401`
- `403`
- `404`
- `405`
- `415`
- `422`
- `429`
- `430`

Основание:

- стр. `3`

### 4.4. Пагинация

Для коллекций используются стандартные заголовки пагинации:

- `X-Pagination-Total-Count`
- `X-Pagination-Page-Count`
- `X-Pagination-Current-Page`
- `X-Pagination-Per-Page`
- `Link`

Основание:

- стр. `3-5`

### 4.5. Важное ограничение по `/games`

PDF отдельно говорит:

- на production сервере возвращается не больше `50` игр на страницу;
- `per page = 0` на production не работает;
- список игр должен кэшироваться на стороне клиента;
- запрещено публиковать URL изображений агрегатора напрямую во frontend.

Основание:

- `/games`, стр. `8`

---

## 5. Безопасность и подпись запросов

### 5.1. Заголовки авторизации

Для запросов к агрегатору и для callback-вызовов от агрегатора используются заголовки:

- `X-Merchant-Id`
- `X-Timestamp`
- `X-Nonce`
- `X-Sign`

Основание:

- запросы интегратора к агрегатору: стр. `6-7`
- callbacks агрегатора в казино: стр. `15`

### 5.2. Как считается подпись

Подпись строится так:

1. объединяются параметры запроса и авторизационные заголовки;
2. массив сортируется по ключу по возрастанию;
3. строится URL-encoded строка;
4. поверх нее считается `sha1 hmac` с `Merchant Key`.

Основание:

- стр. `6-7` для запросов к агрегатору
- стр. `15` для callback-проверки со стороны интегратора

### 5.3. Ограничение по времени

`X-Timestamp` считается просроченным, если отличается от текущего времени более чем на `30 секунд`.

Основание:

- стр. `6`
- стр. `15`

### 5.4. Что нужно реализовать на стороне казино

Backend казино должен:

- проверять подпись каждого callback-запроса;
- отклонять запросы с невалидной подписью;
- учитывать ограничение по timestamp;
- не считать подпись только по телу запроса без заголовков, потому что PDF прямо требует объединять и body, и auth-заголовки.

---

## 6. Общая схема взаимодействия

```mermaid
flowchart TD
    A["Казино: получает каталог"] --> B["GET /games"]
    B --> C["Если нужен lobby: GET /games/lobby"]
    C --> D["POST /games/init или /games/init-demo"]
    D --> E["Игрок редиректится в игру"]
    E --> F["Агрегатор вызывает callback казино"]
    F --> G["balance"]
    F --> H["bet"]
    F --> I["win"]
    F --> J["refund"]
    F --> K["rollback"]
```

### 6.1. Что делает казино

Казино:

- получает игры;
- показывает каталог;
- инициализирует игровые сессии;
- хранит баланс игрока;
- отвечает на callback-вызовы агрегатора.

### 6.2. Что делает агрегатор

Агрегатор:

- отдает каталог игр;
- готовит launch URL;
- во время игры сообщает о денежных событиях;
- запускает freespins и freevouchers механики;
- может прогонять self-validation.

---

## 7. Каталог игр и запуск игры

### 7.1. `GET /games`

**Назначение**

Получение списка игр, доступных для Merchant ID.

**Основание**

- стр. `8-10`

**Ключевые поля игры**

- `uuid`
- `name`
- `image`
- `type`
- `provider`
- `provider_id`
- `technology`
- `has_lobby`
- `is_mobile`
- `has_freespins`
- `has_tables`
- `freespin_valid_until_full_day`
- `label`

**Параметры**

- `expand`
- `filter[provider]`
- `filter[is_mobile]`
- `filter[has_freespins]`

**Доступные expansions**

- `tags`
- `parameters`
- `images`
- `related_games`

**Что важно**

- список игр нужно кэшировать;
- статические изображения тоже нужно кэшировать;
- прямые URL картинок агрегатора не должны светиться во frontend.

### 7.2. `GET /game-tags`

**Назначение**

Получение списка игровых тегов.

**Основание**

- стр. `10-11`

**Поля**

- `code`
- `label`

**Expansion**

- `category`

### 7.3. `GET /games/lobby`

**Назначение**

Если у игры есть lobby, этот метод возвращает список столов и данные lobby.

**Основание**

- стр. `11-12`

**Параметры**

- `game_uuid`
- `currency`
- `technology` — необязательный фильтр по `html5` или `flash`

**Что приходит**

Внутри `lobby`:

- `lobbyData`
- `name`
- `isOpen`
- `openTime`
- `closeTime`
- `dealerName`
- `dealerAvatar`
- `technology`
- `limits`
- `tableId`

**Что важно**

- `lobbyData` потом нужен для `/games/init`;
- `tableId` позже может использоваться в `/freevouchers/set`.

### 7.4. `POST /games/init`

**Назначение**

Подготовка игры к запуску и получение финального URL для редиректа игрока.

**Основание**

- стр. `12-13`

**Параметры**

- `game_uuid`
- `player_id`
- `player_name`
- `currency`
- `session_id`
- `device` — опционально, `desktop` или `mobile`
- `return_url`
- `language`
- `email`
- `lobby_data` — если игра использует lobby

**Ответ**

- `url`

### 7.5. `POST /games/init-demo`

**Назначение**

Подготовка demo-сессии для игры, если провайдер поддерживает demo mode.

**Основание**

- стр. `13-14`

**Параметры**

В PDF явно видны:

- `game_uuid`
- `device`
- `return_url`
- `language`

**Недостаточно данных в PDF**

На видимом фрагменте PDF список полей `/games/init-demo` показан не полностью. Поэтому в документе нельзя утверждать, что это полный список параметров.

**Ответ**

- `url`

### 7.6. Особенности launch flow

PDF прямо предупреждает о двух важных сценариях:

1. После `init` внутри игры может быть запущена другая игра, и тогда callbacks могут прийти с другим `game_uuid`, но с тем же `session_id`.
2. Ставка может пройти в mobile-версии, а выигрыш — в desktop-версии, и тогда `session_id` у `bet` и `win` может отличаться, но `round_id` будет один и тот же.

Основание:

- стр. `12-14`

Это важнейшее требование к внутренней модели данных казино: привязка только к `session_id` недостаточна, нужно хранить и `round_id`.

---

## 8. Callback API: как агрегатор ходит в backend казино

### 8.1. Общие правила callback API

По PDF:

- все callback-вызовы идут через `POST`;
- body передается как `application/x-www-form-urlencoded`;
- ответ интегратора должен быть `HTTP 200 OK`;
- `Content-Type: application/json`;
- агрегатор ждет ответ не более `3 секунд`.

Основание:

- стр. `14-15`

### 8.2. Почему важны 3 секунды

PDF прямо пишет:

- если казино не успело ответить за `3 секунды`, агрегатор считает запрос неуспешным;
- после этого агрегатор следует своему "типичному flow":
  - шлет `refund` для неуспешного `bet`;
  - либо повторно шлет `win`, `refund` или `rollback`.

Основание:

- стр. `14`

Это означает, что backend казино обязан быть:

- быстрым;
- идемпотентным;
- готовым к повторной доставке.

### 8.3. Формат ошибок

В случае ошибки интегратор должен вернуть JSON:

- `error_code`
- `error_description`

При этом HTTP-код все равно `200 OK`.

Основание:

- стр. `14-15`

### 8.4. Поддерживаемые error_code

В PDF явно указаны:

- `INSUFFICIENT_FUNDS` — для `bet`, когда у игрока недостаточно денег
- `INTERNAL_ERROR` — для остальных общих ошибок

Основание:

- стр. `15`

### 8.5. `action=balance`

**Назначение**

Получить актуальный баланс игрока.

**Основание**

- стр. `16-17`

**Поля запроса**

- `action = balance`
- `player_id`
- `currency`
- `session_id` — если эта опция включена

**Ответ**

- `balance`

### 8.6. `action=bet`

**Назначение**

Списание ставки игрока.

**Основание**

- стр. `17-18`

**Типы ставки**

- `bet`
- `tip`
- `freespin`

**Поля запроса**

- `action = bet`
- `amount`
- `currency`
- `game_uuid`
- `player_id`
- `transaction_id`
- `session_id`
- `type`
- `freespin_id`
- `quantity`
- `round_id`
- `finished`
- `transaction_datetime`
- `casino_request_retry_count`

**Ответ**

- `balance`
- `transaction_id`

**Критически важное правило**

PDF прямо требует:

- bet с данным `transaction_id` должен быть обработан только один раз;
- если он уже был обработан, нужно вернуть успешный идемпотентный ответ с тем же внутренним `transaction_id`.

### 8.7. `action=win`

**Назначение**

Начисление выигрыша игроку.

**Основание**

- стр. `18-19`

**Типы выигрыша**

Базовые:

- `win`
- `jackpot`
- `freespin`

Дополнительные типы, перечисленные в PDF:

- `bonus`
- `pragmatic_prize_drop`
- `pragmatic_tournament`
- `promo`
- `prize_drop`
- `tournament`
- `unaccounted_promo`
- `loyalty_win`

**Поля запроса**

- `action = win`
- `amount`
- `currency`
- `game_uuid`
- `player_id`
- `transaction_id`
- `session_id`
- `type`
- `freespin_id`
- `quantity`
- `round_id`
- `finished`
- `transaction_datetime`
- `casino_request_retry_count`

**Ответ**

- `balance`
- `transaction_id`

**Критически важное правило**

Как и для bet:

- win с данным `transaction_id` обрабатывается только один раз;
- повтор должен вернуть идемпотентный успешный ответ.

**Отдельная оговорка из PDF**

Для freespin wins провайдера ELK `round_id` может не передаваться.

Основание:

- стр. `19`

### 8.8. `action=refund`

**Назначение**

Возврат денег в случае проблем со ставкой.

**Основание**

- стр. `19-20`

**Поля запроса**

- `action = refund`
- `amount`
- `currency`
- `game_uuid`
- `player_id`
- `transaction_id`
- `session_id`
- `type`
- `bet_transaction_id`
- `freespin_id`
- `quantity`
- `round_id`
- `finished`
- `transaction_datetime`
- `casino_request_retry_count`

**Ответ**

- `balance`
- `transaction_id`

**Самое важное правило**

PDF прямо требует:

- после `refund` интегратор должен отменить соответствующую `bet`-транзакцию и вернуть деньги игроку;
- если такой `bet`-транзакции еще нет, интегратор должен **сохранить refund** и все равно ответить успехом.

Это значит:

- `refund` может приходить раньше, чем найден исходный `bet`;
- без таблицы отложенных операций это безопасно реализовать нельзя.

### 8.9. `action=rollback`

**Назначение**

Откат целого раунда или части сессии, если провайдер не поддерживает раунды.

**Основание**

- стр. `20-21`

**Поля запроса**

- `action = rollback`
- `currency`
- `game_uuid`
- `player_id`
- `transaction_id`
- `rollback_transactions[]`
- `session_id`
- `type = rollback`
- `provider_round_id`
- `round_id`
- `transaction_datetime`
- `casino_request_retry_count`

**Содержимое `rollback_transactions[]`**

Для каждого элемента массива:

- `action` — `bet`, `win` или `refund`
- `amount`
- `transaction_id`
- `type`

**Ответ**

- `balance`
- `transaction_id`
- `rollback_transactions`

**Самое важное правило**

PDF прямо говорит:

- интегратор должен откатывать **только** транзакции из `rollback_transactions`;
- нельзя на своей стороне расширять rollback на весь раунд только по `provider_round_id`, потому что это может породить ошибки из-за рассинхронизации данных.

Это одно из самых важных архитектурных требований документа.

---

## 9. Дополнительные методы агрегатора

### 9.1. `GET /limits`

**Назначение**

Возвращает список лимитов для мерчанта.

**Основание**

- стр. `22`

**Ответ**

Массив объектов:

- `amount`
- `currency`
- `providers`

### 9.2. `GET /limits/freespin`

**Назначение**

Возвращает freespin limits для мерчанта.

**Основание**

- стр. `23`

**Ответ**

Массив объектов:

- `quantity`
- `currency`
- `providers`

### 9.3. `GET /jackpots` (LEGACY)

**Назначение**

Возвращает список jackpots по провайдерам.

**Основание**

- стр. `23-24`

**Важно**

В PDF этот метод помечен как `LEGACY`.

**Ответ**

- `name`
- `amount`
- `currency`
- `provider`

### 9.4. `POST /balance/notify` (LEGACY)

**Назначение**

Уведомить провайдеров о смене баланса игрока.

**Основание**

- стр. `24-25`

**Поля запроса**

- `balance`
- `session_id`

**Важно**

В PDF этот метод тоже помечен как `LEGACY`.

**Недостаточно данных в PDF**

Из показанного фрагмента видно только пример с `200 OK` и отдельный пример `500 Internal Server Error`, но не раскрыта полная спецификация возможных ошибок и сценариев повторной отправки.

---

## 10. Freespins API

### 10.1. `GET /freespins/bets`

**Назначение**

Получить список допустимых freespin bet values для конкретной игры и валюты.

**Основание**

- стр. `25-26`

**Параметры**

- `game_uuid`
- `currency`

**Ответ**

- `denominations`
- `bets`
- `total_bets`

### 10.2. `POST /freespins/set`

**Назначение**

Создать freespin campaign для игрока.

**Основание**

- стр. `26-27`

**Поля запроса**

- `player_id`
- `player_name`
- `currency`
- `quantity`
- `valid_from`
- `valid_until`
- `freespin_id`
- `bet_id`
- `total_bet_id`
- `denomination`
- `game_uuid`

**Ключевое правило**

Из PDF:

- требуется либо пара `bet_id + denomination`, либо `total_bet_id`.

### 10.3. `GET /freespins/get`

**Назначение**

Получить статус freespin campaign.

**Основание**

- стр. `27-28`

**Параметры**

- `freespin_id`

**Ответ**

- `player_id`
- `currency`
- `quantity`
- `quantity_left`
- `valid_from`
- `valid_until`
- `freespin_id`
- `bet_id`
- `total_bet_id`
- `denomination`
- `game_uuid`
- `status`
- `is_canceled`
- `total_win`

### 10.4. `POST /freespins/cancel`

**Назначение**

Отменить ранее созданную freespin campaign.

**Основание**

- стр. `28`

**Поля запроса**

- `freespin_id`

---

## 11. Freevouchers API

### 11.1. `POST /freevouchers/set`

**Назначение**

Создать freevoucher campaign для игрока.

**Основание**

- стр. `29`

**Поля запроса**

- `player_id`
- `title`
- `currency`
- `initial_balance`
- `max_winnings`
- `valid_until`
- `voucher_id`
- `table_ids[]`
- `short_terms`
- `terms_and_conds`

**Что важно**

- `table_ids[]` берется из `tableId`, который приходит в `/games/lobby`.

### 11.2. `GET /freevouchers/get`

**Назначение**

Получить статус и параметры freevoucher campaign.

**Основание**

- стр. `30`

**Параметры**

- `voucher_id`

**Ответ**

- `player_id`
- `currency`
- `title`
- `state`
- `valid_from`
- `valid_until`
- `voucher_id`
- `playable`
- `winnings`

### 11.3. `POST /freevouchers/cancel`

**Назначение**

Отменить freevoucher campaign.

**Основание**

- стр. `31`

**Поля запроса**

- `voucher_id`
- `reason`

**Разрешенные значения `reason`**

- `Canceled`
- `Forfeited`

---

## 12. Self validation

### 12.1. `POST /self-validate`

**Назначение**

Проверить, правильно ли реализована интеграция на стороне казино.

**Основание**

- стр. `31`

**Как работает**

По PDF:

- у интегратора должна быть активная игровая сессия;
- сессия должна быть открыта не более `15 минут` назад;
- агрегатор сам отправит набор запросов вроде `bet`, `win` и других;
- затем вернет результат в ответе.

**Поля ответа**

- `success`
- `log`

**Недостаточно данных в PDF**

В видимом фрагменте нет подробного списка тестовых шагов self-validation и нет точной схемы request body для самого вызова `/self-validate`.

---

## 13. Схема прохождения транзакций

### 13.1. Нормальный сценарий

```mermaid
sequenceDiagram
    participant Casino as Казино
    participant Agg as Slotegrator
    participant Game as Игра

    Casino->>Agg: GET /games
    Casino->>Agg: POST /games/init
    Agg-->>Casino: launch URL
    Casino->>Game: redirect игрока

    Agg->>Casino: action=balance
    Casino-->>Agg: { balance }

    Game->>Agg: игрок нажал spin
    Agg->>Casino: action=bet
    Casino-->>Agg: { balance, transaction_id }

    Game->>Agg: игра посчитала результат
    Agg->>Casino: action=win
    Casino-->>Agg: { balance, transaction_id }
```

### 13.2. Сценарий со сбоем на ставке

```mermaid
sequenceDiagram
    participant Agg as Slotegrator
    participant Casino as Казино

    Agg->>Casino: action=bet
    Note over Casino: ставка уже могла быть списана
    Note over Agg,Casino: ответ не дошел за 3 секунды
    Agg->>Casino: action=refund
    Casino-->>Agg: success
```

Основание:

- таймаут `3 секунды` и дальнейший flow с `refund`: стр. `14`
- логика `refund`: стр. `19-20`

### 13.3. Сценарий с rollback

```mermaid
sequenceDiagram
    participant Agg as Slotegrator
    participant Casino as Казино

    Agg->>Casino: action=rollback + rollback_transactions[]
    Note over Casino: откатываются только транзакции из списка
    Casino-->>Agg: balance + transaction_id + rollback_transactions
```

Основание:

- стр. `20-21`

### 13.4. Что это значит для backend казино

Backend казино обязан:

1. связывать внешние транзакции по `transaction_id`;
2. отдельно связывать игровые действия по `round_id`;
3. не полагаться только на `session_id`;
4. уметь сохранять `refund`, если исходной ставки еще нет;
5. уметь идемпотентно отвечать на повторные `bet`, `win`, `refund`, `rollback`.

---

## 14. Что обязательно учесть при реализации

### 14.1. Идемпотентность

PDF прямо требует одноразовую обработку как минимум для:

- `bet`
- `win`
- `refund`
- `rollback`

Практически это означает:

- внешний `transaction_id` должен храниться в базе;
- повторы должны возвращать тот же успешный результат;
- повтор не должен двигать деньги второй раз.

### 14.2. Привязка к `round_id`

Из-за явно описанных в PDF сценариев:

- `bet` и `win` могут иметь разный `session_id`;
- разные `game_uuid` могут приходить в рамках одного `session_id`;

следовательно:

- раунд должен быть отдельной сущностью;
- привязка только к `session_id` ненадежна.

### 14.3. Отложенные операции

Так как `refund` может прийти раньше исходного `bet`, нужно отдельное состояние "операция пока не связана".

Иначе невозможно выполнить прямое требование PDF:

- "сохранить refund и ответить success".

### 14.4. Rollback только по списку

При `rollback` нельзя:

- откатывать весь раунд "по догадке";
- откатывать дополнительные транзакции вне `rollback_transactions`.

Это прямо запрещено PDF.

### 14.5. Время ответа

Если callback обрабатывается дольше `3 секунд`, агрегатор может:

- отправить `refund`;
- повторно отправить `win`;
- повторно отправить `refund`;
- повторно отправить `rollback`.

То есть медленный backend автоматически порождает больше сложных сценариев.

---

## 15. Что в PDF не раскрыто полностью

Ниже перечислены места, где документ не дает полного описания, и поэтому по ним нельзя делать жесткие выводы без дополнительных материалов от интеграционного менеджера.

### 15.1. Полный список параметров `/games/init-demo`

В видимом фрагменте PDF часть полей есть, но блок выглядит неполным.

### 15.2. Полный request body `/self-validate`

В PDF видна общая идея метода и response fields, но не раскрыт полный формат запроса.

### 15.3. Полная матрица ошибок по методам

В PDF перечислены только:

- `INSUFFICIENT_FUNDS`
- `INTERNAL_ERROR`

Недостаточно данных, чтобы утверждать, что других прикладных `error_code` нет вообще.

### 15.4. Поведение legacy-методов в актуальной production-практике

`/jackpots` и `/balance/notify` помечены как `LEGACY`, но PDF не описывает:

- насколько активно они реально используются;
- рекомендованы ли они для новых интеграций;
- есть ли planned deprecation.

### 15.5. Поведение некоторых промо-потоков при сбое

PDF описывает методы freespins и freevouchers, но не дает полного протокола retry/rollback для этих кампаний.

Поэтому любые выводы о точном поведении промо-операций при сетевых ошибках без дополнительных данных были бы домыслом.

---

## 16. Примеры из PDF

> Ниже собраны JSON-примеры, которые есть в самом PDF.
>
> Важное замечание:
>
> - часть примеров в PDF показана корректно;
> - часть после текстового извлечения из PDF повреждается переносами строк и пробелами;
> - в таких местах ниже стоит пометка `Формат в PDF/извлечении частично поврежден`.
>
> Я не исправляю такие примеры "по догадке", а оставляю их в том виде, в каком они надежно читаются из документа.

### 16.1. Пример ответа `GET /games`

Источник:

- PDF, стр. `9-10`

```json
{
  "items": [
    {
      "uuid": "abcd12345",
      "name": "Book of Ra",
      "image": "https://static.game-aggregator.com/games/4694605316aa1ca969fe89227aabe51c1e8b091b.jpg",
      "type": "Slots",
      "provider": "abcd12345",
      "provider_id": 12345,
      "technology": "Flash",
      "has_lobby": 0,
      "is_mobile": 0,
      "label": "Some Sub Provider Legal Name GmBH.",
      "tags": [
        {
          "code": "jackpots",
          "label": "Jackpots"
        },
        {
          "code": "freespins",
          "label": "FreeSpins"
        }
      ],
      "parameters": {
        "rtp": 98.72,
        "volatility": "medium-high",
        "reels_count": "5+1",
        "lines_count": 20
      },
      "images": [
        {
          "name": "4694605316aa1ca969fe89227aabe51c1e8b091b.jpg",
          "file": "games/4694605316aa1ca969fe89227aabe51c1e8b091b.jpg",
          "url": "https://static.game-aggregator.com/games/4694605316aa1ca969fe89227aabe51c1e8b091b.jpg",
          "type": "regular"
        },
        {
          "name": "4694605316aa1ca969fe89227aabe51c1e8b091b_hq.png",
          "file": "games/4694605316aa1ca969fe89227aabe51c1e8b091b_hq.png",
          "url": "https://static.game-aggregator.com/games/4694605316aa1ca969fe89227aabe51c1e8b091b_hq.png",
          "type": "high-quality"
        }
      ],
      "related_games": [
        {
          "uuid": "4694605316aa1ca969fe89227aabe51c1e8b091b",
          "is_mobile": 0
        },
        {
          "uuid": "4694605316aa1ca969fe89227aabe51c1e8b091c",
          "is_mobile": 1
        }
      ]
    }
  ],
  "_links": {
    "self": {
      "href": "https://game-aggregator.com/endpoint?page=1"
    },
    "next": {
      "href": "https://game-aggregator.com/endpoint?page=2"
    },
    "last": {
      "href": "https://game-aggregator.com/endpoint?page=50"
    }
  },
  "_meta": {
    "totalCount": 1000,
    "pageCount": 50,
    "currentPage": 1,
    "perPage": 20
  }
}
```

Что здесь происходит:

- агрегатор отдает коллекцию игр;
- у каждой игры есть базовые поля витрины;
- при `expand` могут прийти `tags`, `parameters`, `images`, `related_games`;
- пагинация идет через `_links` и `_meta`.

### 16.2. Пример ответа `GET /game-tags`

Источник:

- PDF, стр. `11`

```json
{
  "items": [
    {
      "code": "jackpots",
      "label": "Jackpots",
      "category": {
        "code": "financial",
        "label": "Financial"
      }
    },
    {
      "code": "freespins",
      "label": "FreeSpins",
      "category": {
        "code": "financial",
        "label": "Financial"
      }
    }
  ],
  "_links": {
    "self": {
      "href": "https://game-aggregator.com/endpoint?page=1"
    },
    "next": {
      "href": "https://game-aggregator.com/endpoint?page=1"
    },
    "last": {
      "href": "https://game-aggregator.com/endpoint?page=1"
    }
  },
  "_meta": {
    "totalCount": 20,
    "pageCount": 1,
    "currentPage": 1,
    "perPage": 20
  }
}
```

Что здесь происходит:

- агрегатор отдает словарь тегов;
- если запросить `expand=category`, у тега появляется категория.

### 16.3. Пример ответа `GET /games/lobby`

Источник:

- PDF, стр. `12`

```json
{
  "lobby": {
    "lobbyData": "abcd12345",
    "name": "Baccarat",
    "isOpen": true,
    "openTime": "11:00:00",
    "closeTime": "12:00:00",
    "dealerName": "abcd12345",
    "dealerAvatar": "https://avatar-url.com",
    "technology": "html5",
    "limits": [
      {
        "currency": "USD",
        "min": 1,
        "max": 100
      }
    ]
  }
}
```

Что здесь происходит:

- агрегатор отдает данные lobby для live/table game;
- `lobbyData` потом передается в `/games/init`;
- `limits` описывают допустимые лимиты для выбранного стола.

### 16.4. Пример ответа `POST /games/init`

Источник:

- PDF, стр. `13`

```json
{
  "url": "https://example.com/endpoint"
}
```

Что здесь происходит:

- казино подготовило сессию через `/games/init`;
- агрегатор вернул финальный URL;
- игрок должен быть редиректнут в игру по этому URL.

### 16.5. Пример ответа на `action=balance`

Источник:

- PDF, стр. `17`

```json
{
  "balance": 57.12
}
```

Что здесь происходит:

- агрегатор обратился в backend казино;
- казино вернуло актуальный баланс игрока.

### 16.6. Пример ответа на `action=bet`

Источник:

- PDF, стр. `18`

```json
{
  "balance": 27.18,
  "transaction_id": "abcd12345"
}
```

Что здесь происходит:

- ставка успешно списана;
- казино вернуло новый баланс после списания;
- `transaction_id` в ответе — внутренний идентификатор обработанной транзакции на стороне интегратора.

### 16.7. Пример ответа на `action=win`

Источник:

- PDF, стр. `19`

```json
{
  "balance": 170.21,
  "transaction_id": "abcd12345"
}
```

Что здесь происходит:

- выигрыш был зачислен;
- агрегатор получил новый баланс;
- `transaction_id` подтверждает, что операция принята и проведена.

### 16.8. Пример ответа на `action=refund`

Источник:

- PDF, стр. `20`

```json
{
  "balance": 27.18,
  "transaction_id": "abcd12345"
}
```

Что здесь происходит:

- refund был принят как успешный;
- деньги возвращены игроку или операция зарегистрирована как отложенная;
- внешний запрос считается обработанным.

### 16.9. Пример ответа на `action=rollback`

Источник:

- PDF, стр. `21`

```json
{
  "balance": 27.18,
  "transaction_id": "12345",
  "rollback_transactions": {
    "12346",
    "12347"
  }
}
```

Формат в PDF/извлечении частично поврежден:

- в текстовом извлечении `rollback_transactions` выглядит не как валидный JSON-массив, а как фигурные скобки с перечнем ID;
- поэтому выше пример приведен в том виде, в каком он надежно читается из PDF-фрагмента.

Что здесь происходит:

- казино подтверждает rollback;
- возвращает итоговый баланс;
- возвращает список транзакций, которые считает успешно откатанными.

### 16.10. Пример ответа `GET /limits`

Источник:

- PDF, стр. `22`

```json
[
  {
    "amount": "1000.00",
    "currency": "USD",
    "providers": [
      "Provider1",
      "Provider2",
      "Provider3"
    ]
  },
  {
    "amount": "1000.00",
    "currency": "EUR",
    "providers": [
      "Provider1"
    ]
  }
]
```

Что здесь происходит:

- агрегатор показывает остатки лимитов по валютам;
- лимит привязывается к списку провайдеров.

### 16.11. Пример ответа `GET /limits/freespin`

Источник:

- PDF, стр. `23`

```json
[
  {
    "quantity": 17,
    "currency": "USD",
    "providers": [
      "Provider1",
      "Provider2",
      "Provider3"
    ]
  },
  {
    "quantity": 1000,
    "currency": "EUR",
    "providers": [
      "Provider1"
    ]
  }
]
```

Что здесь происходит:

- агрегатор показывает доступные freespin-лимиты по валютам;
- лимит также привязывается к списку провайдеров.

### 16.12. Пример ответа `GET /jackpots`

Источник:

- PDF, стр. `24`

```json
[
  {
    "name": "jackpot name",
    "amount": "1000.00",
    "currency": "USD",
    "provider": "Provider1"
  },
  {
    "name": null,
    "amount": "1000.00",
    "currency": "EUR",
    "provider": "Provider2"
  }
]
```

Что здесь происходит:

- агрегатор отдает текущие jackpot values по провайдерам;
- имя jackpot может отсутствовать.

### 16.13. Пример ответа `GET /freespins/bets`

Источник:

- PDF, стр. `26`

```json
{
  "denominations": ["0.01", "0.1", "1"],
  "bets": [
    {
      "bet_id": "0",
      "bet_per_line": 1,
      "lines": 25
    },
    {
      "bet_id": "1",
      "bet_per_line": 2,
      "lines": 25
    }
  ],
  "total_bets": [
    {
      "bet_id": 0,
      "amount": 10.0
    },
    {
      "bet_id": 1,
      "amount": 25.0
    }
  ]
}
```

Что здесь происходит:

- агрегатор возвращает допустимые ставки для freespin campaign;
- можно оперировать либо комбинацией `bet_id + denomination`, либо значениями из `total_bets`.

### 16.14. Пример ответа `GET /freespins/get`

Источник:

- PDF, стр. `28`

```json
{
  "player_id": "abcd12345",
  "currency": "USD",
  "quantity": 10,
  "quantity_left": 8,
  "freespin_id": "abcd12345"
}
```

Что здесь происходит:

- агрегатор возвращает текущее состояние freespin campaign;
- видно, сколько спинов было задано и сколько осталось.

### 16.15. Пример ответа `GET /freevouchers/get`

Источник:

- PDF, стр. `30`

```json
{
  "player_id": "abcd12345",
  "voucher_id": "abcd12345",
  "currency": "USD",
  "valid_from": 1519610000,
  "valid_until": 1519810000,
  "title": "delight",
  "state": "Active",
  "playable": 50,
  "winnings": 12.5
}
```

Формат в PDF/извлечении частично поврежден:

- в извлеченном тексте часть ключей и двоеточий распалась из-за PDF-разметки;
- выше сохранены только те поля и значения, которые читаются из фрагмента однозначно.

Что здесь происходит:

- агрегатор отдает состояние freevoucher campaign;
- видно доступный playable balance и уже накопленные winnings.
