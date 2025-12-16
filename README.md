**Описание написано с помощью Чата-гпт, надеюсь оно читабельно ...

# T‑Portfolio Sheet — Tinkoff Invest → Google Sheets

Открытый скрипт на **Google Apps Script** для загрузки портфеля из публичного API Т‑Инвестиций в Google Sheets с автоматическим расчётом ₽/лот, средней цены за 1 шт., агрегированием по имени/тикеру/FIGI и удобным форматированием.

> ⚠️ Проект не является продуктом Тинькофф/Т‑банка. Используйте на свой риск. Не инвестиционная рекомендация.

---
![Пример работы скрипта](/screenshot-summary.png)

## Содержание

* [Возможности](#возможности)
* [Быстрый старт (5 минут)](#быстрый-старт-5-минут)
* [Настройки (константы в коде)](#настройки-константы-в-коде)
* [Какие колонки пишет скрипт](#какие-колонки-пишет-скрипт)
* [Ограничения и заметки](#ограничения-и-заметки)
* [Решение проблем](#решение-проблем)
* [Дорожная карта](#дорожная-карта)
* [Безопасность и бренд](#безопасность-и-бренд)
* [Лицензия](#лицензия)
* [Contributing](#contributing)
* [English version (short)](#english-version-short)

---

## Возможности

* Получает **все открытые счета** пользователя и их позиции (`UsersService/GetAccounts` + `OperationsService/GetPortfolio`).
* Подтягивает **справочник по FIGI** (лот, тикер, валюта, тип инструмента) с кэшем на 6 часов.
* Считает:

  * **₽ за 1 шт.** (текущая) и **среднюю покупочную** ₽/шт,
  * **₽ за лот** и **стоимость позиции** в ₽,
  * количество в **штуках** и **лотах**.
* Делает **агрегацию** по `name` / `ticker` / `figi` (переключается константой).
* Пишет данные на два листа:

  * `Positions` — сырые позиции,
  * `Positions_Aggregated` — сводка по выбранному ключу.
* Добавляет меню в таблицу: **Tinkoff → Синхронизировать (лоты + ₽ + аггр.)**
* Имеет **фолбэк** на второй базовый URL API.

---

## Быстрый старт (5 минут)

1. Создайте пустую таблицу Google Sheets с двумя листами:

* `Positions`
* `Positions_Aggregated`

2. Откройте **Расширения → Apps Script** и вставьте код из `apps-script/Code.gs`.

3. В редакторе скриптов откройте **Project Settings → Script properties** и добавьте:

```
TINKOFF_TOKEN = t.xxxxxxx...
```

4. Запустите функцию `syncTinkoffPositions` (первый запуск запросит права).

5. В самой таблице появится меню **Tinkoff** — запускайте синхронизацию оттуда.

6. (Опционально) Поставьте триггер автообновления:
   **Apps Script → Triggers → Add Trigger →** `syncTinkoffPositions` → *Time-driven* (например, раз в день).

---

## Настройки (константы в коде)

* `AGGREGATE_BY` — ключ сводки: `'name' | 'ticker' | 'figi'` (по умолчанию `'name'`).
* `CACHE_TTL_SEC` — TTL кэша справочника FIGI (по умолчанию 6 часов).
* `BASES` — список базовых URL API; используется фолбэк при ошибке.
* Форматы ячеек для количеств и денег проставляются автоматически.

---

## Какие колонки пишет скрипт

Лист `Positions` (и та же структура в агрегате):

```
accountId
type
figi
ticker
name
lot
quantity_lots               // кол-во лотов
quantity_pcs                // кол-во штук
current_price_rub_per_piece // ₽ за 1 шт (текущая)
avg_price_rub_per_piece     // ₽ за 1 шт (средняя покупочная)
price_rub_per_lot           // ₽ за 1 лот (по текущей цене)
position_value_rub          // ₽ за всю позицию (по текущей цене)
instrument_currency         // валюта инструмента (по справочнику)
```

`Positions_Aggregated` содержит те же поля, но свёрнутые по ключу (`AGGREGATE_BY`) и отсортированные по `position_value_rub`.

---

## Ограничения и заметки

* Портфель запрашивается с `currency: 'RUB'`. Для USD/других валют нужна отдельная доработка.
* Средняя цена от API обычно **без комиссий** — учитывайте это при сверке с брокерским отчётом.
* Для некоторых внебиржевых/экзотических инструментов отдельные поля могут отсутствовать.
* В коде есть лёгкий троттлинг (`Utilities.sleep`) и кэш справочника; не запускайте синхронизацию слишком часто.

---

## Решение проблем

* **401/403** — проверьте `TINKOFF_TOKEN` в *Script properties* и права на выполнение.
* **404** — временная недоступность эндпоинта; используется фолбэк на второй базовый URL.
* **429 / Too Many Requests** — уменьшите частоту, увеличьте паузы.
* **Пустые данные** — убедитесь, что у аккаунтов статус `ACCOUNT_STATUS_OPEN`.
* **Формат денег** — столбцы форматируются как `#,##0.00 [$₽-ru-RU]`. Если в значение попал символ `₽`, проверьте, что он задан только форматированием, а не в теле ячейки.

---

## Дорожная карта

* NAV‑кривая и **сравнение с IMOEX**
* **XIRR/TWR** для портфеля
* **Календарь купонов/дивидендов** (7/30/90 дней)
* Взвешенный **YTM** и Duration по облигациям
* Разделение на слои `DATA_*` (данные) и `VIEW_*` (витрины)

---

## Безопасность и бренд

* Не храните и не публикуйте токен в коде или ячейках — только в **Script properties**.
* Публичную таблицу делайте **только на чтение** и без персональных данных.
* Репозиторий **не аффилирован** с Тинькофф/Т‑банком.

---

## Лицензия

MIT (см. `LICENSE`).

---

## Contributing

PR и issue приветствуются. Пожалуйста, не прикладывайте реальные ключи/данные.

---

# English version (short)

## T‑Portfolio Sheet — Tinkoff Invest → Google Sheets

An open Google Apps Script that pulls your portfolio from **Tinkoff Invest public API** into Google Sheets, computes price per lot and per piece in RUB, aggregates by name/ticker/FIGI, and formats the output nicely.

> ⚠️ Not affiliated with Tinkoff/T‑Bank. Use at your own risk. Not an investment recommendation.

### Features

* Fetches **all open accounts** and their positions.
* Queries **instrument reference** by FIGI (lot, ticker, currency, type) with a 6‑hour cache.
* Calculates:

  * **RUB per piece** (current) and **average buy** RUB/pc,
  * **RUB per lot** and **position value** in RUB,
  * quantities in **lots** and **pieces**.
* **Aggregates** by `name` / `ticker` / `figi` (`AGGREGATE_BY`).
* Writes to:

  * `Positions` — raw positions,
  * `Positions_Aggregated` — grouped summary.
* Adds a Sheets menu: **Tinkoff → Sync (lots + RUB + aggregation)**.
* Has a **fallback** base URL for the API.

### Quick start

1. Create a Google Sheet with two sheets: `Positions`, `Positions_Aggregated`.
2. Open **Extensions → Apps Script** and paste `apps-script/Code.gs`.
3. Set **Project Settings → Script properties**:

```
TINKOFF_TOKEN = t.xxxxxxx...
```

4. Run `syncTinkoffPositions` once to authorize; later use the **Tinkoff** menu.
5. (Optional) Add a time‑driven trigger for auto‑refresh.

### Settings

* `AGGREGATE_BY`: `'name' | 'ticker' | 'figi'` (default `'name'`).
* `CACHE_TTL_SEC`: FIGI reference cache TTL (default 6h).
* `BASES`: API base URLs with fallback.

### Columns produced

`accountId`, `type`, `figi`, `ticker`, `name`, `lot`,
`quantity_lots`, `quantity_pcs`,
`current_price_rub_per_piece`, `avg_price_rub_per_piece`,
`price_rub_per_lot`, `position_value_rub`, `instrument_currency`.

### Notes & limits

* Portfolio is requested with `currency: 'RUB'` only (extend as needed).
* Average price may **exclude commissions** (check your broker report).
* Some OTC/exotic instruments may lack fields.
* Throttling and FIGI cache are built‑in; avoid excessive refreshes.

### Roadmap

NAV curve & **IMOEX comparison**, **XIRR/TWR**, **cashflow calendar**, weighted **YTM/Duration**, `DATA_*` vs `VIEW_*` separation.

### License

MIT.

### Contributing

Issues and PRs are welcome. Do not share real tokens or personal data.
