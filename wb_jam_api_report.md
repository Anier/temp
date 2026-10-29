# Отчёт: Анализ влияния отмены подписки Wildberries «Джем» на функционал wb-tools

> **Дата составления:** 29 октября 2026 г.  
> **Объект анализа:** скрипты и API бэкенда (`public/php/`), конфигурация планировщика задач ([public/php/cron/root](file:///d:/www/React/wb-tools/public/php/cron/root)) и интерфейс фронтенда (`src/components/`).

---

## 1. Главный вывод

Подписка Wildberries **«Джем»** является обязательным условием для доступа исключительно к домену расширенной аналитики:  
`https://seller-analytics-api.wildberries.ru`

При завершении подписки «Джем» маркетплейс возвращает ошибки авторизации / доступа (**HTTP 403 Forbidden** или **HTTP 401** с сообщением об отсутствии активной подписки / тарифа) для:
1. **Воронки продаж** (просмотры карточек, добавления в корзину, конверсии).
2. **Поисковой аналитики** (индекс видимости карточки, средняя позиция в поиске).
3. **Отчёта об оборачиваемости и скорости продаж** товаров.

**Все остальные категории Wildberries API продолжают работать штатно**, поскольку входят в базовый бесплатный функционал кабинета продавца и не требуют «Джем»:
- **Цены и скидки** (`discounts-prices-api.wildberries.ru`)
- **Реклама и ставки** (`advert-api.wildberries.ru`)
- **Статистика заказов и выкупов** (`statistics-api.wildberries.ru`)
- **Остатки на складах WB** (`statistics-api.wildberries.ru`)
- **Остатки на складах FBS** (`marketplace-api.wildberries.ru`)
- **Карточки товаров и контент** (`content-api.wildberries.ru`)
- **Вопросы и отзывы** (`feedbacks-api.wildberries.ru`)
- **Календарь официальных акций** (`dp-calendar-api.wildberries.ru`)
- **Парсинг СПП Playwright** (парсинг публичной витрины маркетплейса)

---

## 2. Анализ серверного планировщика CRON (`public/php/cron/root`)

| Расписание (Cron) | Запускаемый скрипт | Назначение | Статус после отмены «Джем» |
| :--- | :--- | :--- | :---: |
| `45 7-23 * * *` (ежечасно) | [hourly_snapshots.sh](file:///d:/www/React/wb-tools/public/php/cron/hourly_snapshots.sh) | Пошаговый запуск `sales_funnel_snapshot.php`, `visibility_snapshot_single.php` и `min_prices.php` | ⚠️ **Частично деградировал** |
| `39 23 * * *` (раз в день) | [collect_analytics.py](file:///d:/www/React/wb-tools/public/php/collect_analytics.py) | Запись суточного среза метрик в SQLite `analytics.db` | ⚠️ **Деградировал** |
| `55 */2 * * *` (каждые 2 ч) | [reklama_snapshot.sh](file:///d:/www/React/wb-tools/public/php/cron/reklama_snapshot.sh) | Сбор активных рекламных кампаний и ставок в `reklama.json` через [reklama.php](file:///d:/www/React/wb-tools/public/php/reklama.php) |  **Работает штатно** |
| `26 */4 * * *` (каждые 4 ч) | [warehouse_remains_snapshot.sh](file:///d:/www/React/wb-tools/public/php/cron/warehouse_remains_snapshot.sh) | Сбор остатков товаров на складах WB в `warehouse_remains.json` через [warehouse_remains.php](file:///d:/www/React/wb-tools/public/php/warehouse_remains.php) |  **Работает штатно** |
| `18 0 * * *` (раз в день) | [promo_daily.py](file:///d:/www/React/wb-tools/public/php/promo_daily.py) | Автоматическое плановое снижение скидок на WB |  **Работает штатно** |
| `0 12 * * *` (раз в день) | [db_backup.py](file:///d:/www/React/wb-tools/public/php/db_backup.py) | Создание резервной копии базы данных SQLite с ротацией 20 дней |  **Работает штатно** |

### Детализация проблемных задач планировщика:
- **`hourly_snapshots.sh`**:
  - `sales_funnel_snapshot.php` завершается ошибкой (нет доступа к воронке продаж).
  - `visibility_snapshot_single.php` завершается ошибкой (нет доступа к поисковому отчёту). Скрипт пытается использовать локальный fallback (`visibility.last_good.json`), из-за чего данные «замораживаются» на старой дате.
  - `min_prices.php` успешно собирает цены и скидки.
- **`collect_analytics.py`**:
  - Сам скрипт выполняется без критического сбоя, однако берёт данные из `sales_funnel.json` и `visibility.json`. Поскольку свежих данных нет, в исторические таблицы `analytics.db` (`product_analytics`) за новые даты записываются нули либо устаревшие значения по показателям: `openCount` (просмотры), `cart_count` (корзины), `visibility` (видимость), `avg_position` (позиция).

---

## 3. Сводная таблица задействованных Wildberries API

| Хост и Endpoint Wildberries | Файлы проекта | Назначение (за что отвечал) | Статус | Требует «Джем» |
| :--- | :--- | :--- | :---: | :---: |
| `POST seller-analytics-api.wildberries.ru/api/analytics/v3/sales-funnel/products` | [sales_funnel_snapshot.php](file:///d:/www/React/wb-tools/public/php/sales_funnel_snapshot.php) | Воронка продаж: просмотры карточки (`openCount`), добавления в корзину (`addToCartCount`), конверсии | ❌ **Не работает** | **Да** |
| `POST seller-analytics-api.wildberries.ru/api/v2/search-report/report` | [visibility_snapshot_single.php](file:///d:/www/React/wb-tools/public/php/visibility_snapshot_single.php), [visibility_snapshot.php](file:///d:/www/React/wb-tools/public/php/visibility_snapshot.php) | Поисковый отчёт: индекс видимости карточки (`visibility`) и средняя позиция в поиске (`avgPos`) | ❌ **Не работает** | **Да** |
| `POST seller-analytics-api.wildberries.ru/api/v2/stocks-report/products/products` | [promo.php](file:///d:/www/React/wb-tools/public/php/promo.php) (`action=sync-turnover`) | Отчёт об остатках и оборачиваемости товаров (`saleRate`, `avgStockTurnover`) | ❌ **Не работает** | **Да** |
| `GET discounts-prices-api.wildberries.ru/api/v2/list/goods/filter` | [min_prices.php](file:///d:/www/React/wb-tools/public/php/min_prices.php), [promo.php](file:///d:/www/React/wb-tools/public/php/promo.php) | Получение текущих цен, скидок и минимальных цен товаров |  **Работает** | Нет |
| `POST discounts-prices-api.wildberries.ru/api/v2/upload/task` | [promo_daily.py](file:///d:/www/React/wb-tools/public/php/promo_daily.py), [promo.php](file:///d:/www/React/wb-tools/public/php/promo.php) | Загрузка и обновление цен и скидок на WB |  **Работает** | Нет |
| `GET advert-api.wildberries.ru/adv/v1/promotion/count` | [reklama.php](file:///d:/www/React/wb-tools/public/php/reklama.php) | Получение количества кампаний продвижения |  **Работает** | Нет |
| `GET advert-api.wildberries.ru/api/advert/v2/adverts` | [reklama.php](file:///d:/www/React/wb-tools/public/php/reklama.php), [export_cpc_bids.py](file:///d:/www/React/wb-tools/public/php/export_cpc_bids.py) | Список кампаний, ставки CPC/CPM, статусы, артикулы в рекламе |  **Работает** | Нет |
| `GET statistics-api.wildberries.ru/api/v1/supplier/orders` | [test.php](file:///d:/www/React/wb-tools/public/php/test.php) | Заказы по часам, датам, регионам и складам |  **Работает** | Нет |
| `GET statistics-api.wildberries.ru/api/v1/supplier/stocks` | [warehouse_remains.php](file:///d:/www/React/wb-tools/public/php/warehouse_remains.php) | Фактические остатки товаров на складах хранения WB (FBO) |  **Работает** | Нет |
| `GET marketplace-api.wildberries.ru/api/v3/stocks/{warehouse_id}` | [warehouse_snapshots.py](file:///d:/www/React/wb-tools/public/php/warehouse_snapshots.py), [manage_warehouse_stocks.py](file:///d:/www/React/wb-tools/public/php/manage_warehouse_stocks.py) | Чтение и передача остатков на собственных складах продавца (FBS `6222` и `1884105`) |  **Работает** | Нет |
| `POST content-api.wildberries.ru/content/v2/get/cards/list` | [warehouse_snapshots.py](file:///d:/www/React/wb-tools/public/php/warehouse_snapshots.py), [promo.php](file:///d:/www/React/wb-tools/public/php/promo.php) | Реестр карточек товаров, номенклатуры, штрихкоды |  **Работает** | Нет |
| `GET feedbacks-api.wildberries.ru/api/v1/feedbacks` | [feedbacks.php](file:///d:/www/React/wb-tools/public/php/feedbacks.php) | Получение отзывов и оценок покупателей |  **Работает** | Нет |
| `GET dp-calendar-api.wildberries.ru/api/v1/calendar/promotions` | [collect_analytics.py](file:///d:/www/React/wb-tools/public/php/collect_analytics.py) | Официальный календарь промо-акций Wildberries |  **Работает** | Нет |

---

## 4. Подробный разбор затронутых компонентов и интерфейса

### 4.1. Раздел «Продажи» (`src/components/Sales.jsx`)
* **Что перестало работать:**
  * Колонки **«Открытия»** (просмотры карточки товара) и **«В корзину»** — данные брались из `sales_funnel.json`.
  * Конверсии воронки продаж (переход в корзину / заказ).
  * Колонки **«Индекс видимости»** и **«Ср. позиция»** — данные брались из `visibility.json`.
  * В модальном окне детального сравнения товара ([src/components/ComparisonTable.jsx](file:///d:/www/React/wb-tools/src/components/ComparisonTable.jsx)) за новые дни эти метрики будут нулевыми или отсутствовать.
* **Что продолжает работать:**
  * Отображение текущих цен и скидок (`min_prices.json`).
  * Колонка **«Заказали»** и расчет заказанных штук (`ordersCount = ROUND(orderSum / minPrice)`), если сумма заказов поступает или рассчитывается.
  * Зеленые бейджи рекламы с типами кампаний и ставками CPC/CPM (`reklama.json`).
  * Складские остатки и модальное окно остатков по регионам/складам WB (`warehouse_remains.json`).
  * Исторические периоды за прошлые дни, когда подписка «Джем» ещё действовала.

### 4.2. Раздел «Promo» (`src/components/Promo.jsx`)
* **Что перестало работать:**
  * Кнопка синхронизации оборачиваемости товаров (`action=sync-turnover` в [public/php/promo.php](file:///d:/www/React/wb-tools/public/php/promo.php)), которая запрашивала `stocks-report`.
* **Что продолжает работать:**
  * Просмотр и редактирование цен и скидок по товарам.
  * Ручное и массовое применение скидок.
  * Экспорт и импорт Excel-шаблонов.
  * Ежедневный автопересчёт и снижение скидок по правилам ([promo_daily.py](file:///d:/www/React/wb-tools/public/php/promo_daily.py)).
  * Графики и сбор розничных цен (СПП) через Playwright ([wb_promo_price_crawler.py](file:///d:/www/React/wb-tools/public/php/wb_promo_price_crawler.py)) или расширение браузера.

### 4.3. Раздел «Статистика заказов» (`src/components/Order_statistics.jsx`)
* **Что произошло:** **Никаких изменений.**
* Заказы запрашиваются через [test.php](file:///d:/www/React/wb-tools/public/php/test.php) к эндпоинту `statistics-api.wildberries.ru/api/v1/supplier/orders`. Это стандартный бесплатный метод статистики WB. Тепловая карта по часам, дням недели и регионам работает полностью.

### 4.4. Раздел «Склады» (`src/components/Warehouses.jsx`)
* **Что произошло:** **Никаких изменений.**
* Опрос остатков складов DBS/FBS `1884105` и `6222` выполняется через Marketplace API (`marketplace-api.wildberries.ru`) и Content API. Сравнение снимков остатков работает штатно.

### 4.5. Раздел «Настройки» (`src/components/Settings.jsx`)
* **Что произошло:** Выгрузка отзывов через [feedbacks.php](file:///d:/www/React/wb-tools/public/php/feedbacks.php), загрузка XLSX автоакций и остатков работают в обычном режиме.
