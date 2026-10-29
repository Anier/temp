# Отчёт: Анализ влияния отмены подписки Wildberries «Джем»

> **Дата составления:** 29 октября 2026 г.  
> **Объект анализа:** скрипты и API бэкенда (`public/php/`), конфигурация планировщика задач ([public/php/cron/root](file:///d:/www/React/wb-tools/public/php/cron/root)) и интерфейс фронтенда (`src/components/`).



## 1. Главный вывод по подписке «Джем»

Подписка Wildberries **«Джем»** является обязательным условием для доступа исключительно к домену расширенной аналитики:  
`https://seller-analytics-api.wildberries.ru`

При завершении подписки «Джем» маркетплейс возвращает ошибку доступа (**HTTP 403 Forbidden**) для:
1. **Воронки продаж** (просмотры карточек, добавления в корзину, конверсии).
2. **Поисковой аналитики** (индекс видимости карточки, средняя позиция в поиске).
3. **Отчёта об оборачиваемости и скорости продаж** товаров.

**Все остальные категории Wildberries API работают без «Джем»** (при условии валидного, непросроченного токена):
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

| Расписание (Cron) | Запускаемый скрипт | Назначение | Статус при валидном токене (без «Джем») | Текущий статус (токен истёк) |
| :--- | :--- | :--- | :---: | :---: |
| `45 7-23 * * *` (ежечасно) | [hourly_snapshots.sh](file:///d:/www/React/wb-tools/public/php/cron/hourly_snapshots.sh) | Пошаговый запуск воронки, видимости и цен | ⚠️ **Частично деградировал** (воронка и видимость 403, цены работают) | ❌ Ошибка 401 (токены истекли) |
| `39 23 * * *` (раз в день) | [collect_analytics.py](file:///d:/www/React/wb-tools/public/php/collect_analytics.py) | Запись суточного среза метрик в SQLite `analytics.db` | ⚠️ **Деградировал** (пишет нули по воронке и позициям) | ⚠️ Пишет нули по воронке и позициям |
| `55 */2 * * *` (каждые 2 ч) | [reklama_snapshot.sh](file:///d:/www/React/wb-tools/public/php/cron/reklama_snapshot.sh) | Сбор активных рекламных кампаний и ставок в `reklama.json` через [reklama.php](file:///d:/www/React/wb-tools/public/php/reklama.php) |  **Работает штатно** | ❌ Ошибка 401 |
| `26 */4 * * *` (каждые 4 ч) | [warehouse_remains_snapshot.sh](file:///d:/www/React/wb-tools/public/php/cron/warehouse_remains_snapshot.sh) | Сбор остатков товаров на складах WB в `warehouse_remains.json` через [warehouse_remains.php](file:///d:/www/React/wb-tools/public/php/warehouse_remains.php) |  **Работает штатно** | Fallback на XLSX (API даёт 401) |
| `18 0 * * *` (раз в день) | [promo_daily.py](file:///d:/www/React/wb-tools/public/php/promo_daily.py) | Автоматическое плановое снижение скидок на WB |  **Работает штатно** | ❌ Ошибка 401 |
| `0 12 * * *` (раз в день) | [db_backup.py](file:///d:/www/React/wb-tools/public/php/db_backup.py) | Создание резервной копии базы данных SQLite с ротацией 20 дней |  **Работает штатно** |  **Работает штатно** |

---

## 3. Сводная таблица задействованных Wildberries API

| Хост и Endpoint Wildberries | Файлы проекта | Назначение (за что отвечал) | Требует «Джем»? | Причина текущего сбоя |
| :--- | :--- | :--- | :---: | :--- |
| `POST seller-analytics-api.wildberries.ru/api/analytics/v3/sales-funnel/products` | [sales_funnel_snapshot.php](file:///d:/www/React/wb-tools/public/php/sales_funnel_snapshot.php) | Воронка продаж: просмотры карточки (`openCount`), корзины (`addToCartCount`), конверсии | **Да** | Нет подписки «Джем» + истёк токен |
| `POST seller-analytics-api.wildberries.ru/api/v2/search-report/report` | [visibility_snapshot_single.php](file:///d:/www/React/wb-tools/public/php/visibility_snapshot_single.php) | Поисковый отчёт: индекс видимости карточки (`visibility`) и средняя позиция (`avgPos`) | **Да** | Нет подписки «Джем» + истёк токен |
| `POST seller-analytics-api.wildberries.ru/api/v2/stocks-report/products/products` | [promo.php](file:///d:/www/React/wb-tools/public/php/promo.php) (`action=sync-turnover`) | Отчёт об остатках и оборачиваемости товаров (`saleRate`, `avgStockTurnover`) | **Да** | Нет подписки «Джем» + истёк токен |
| `GET discounts-prices-api.wildberries.ru/api/v2/list/goods/filter` | [min_prices.php](file:///d:/www/React/wb-tools/public/php/min_prices.php), [promo.php](file:///d:/www/React/wb-tools/public/php/promo.php) | Получение текущих цен, скидок и минимальных цен товаров | Нет | **Истёк токен (401)** |
| `POST discounts-prices-api.wildberries.ru/api/v2/upload/task` | [promo_daily.py](file:///d:/www/React/wb-tools/public/php/promo_daily.py), [promo.php](file:///d:/www/React/wb-tools/public/php/promo.php) | Загрузка и обновление цен и скидок на WB | Нет | **Истёк токен (401)** |
| `GET advert-api.wildberries.ru/adv/v1/promotion/count` | [reklama.php](file:///d:/www/React/wb-tools/public/php/reklama.php) | Получение количества кампаний продвижения | Нет | **Истёк токен (401)** |
| `GET advert-api.wildberries.ru/api/advert/v2/adverts` | [reklama.php](file:///d:/www/React/wb-tools/public/php/reklama.php) | Список кампаний, ставки CPC/CPM, статусы, артикулы в рекламе | Нет | **Истёк токен (401)** |
| `GET statistics-api.wildberries.ru/api/v1/supplier/orders` | [test.php](file:///d:/www/React/wb-tools/public/php/test.php) | Заказы по часам, датам, регионам и складам | **Нет** | **Истёк токен (401)** |
| `GET statistics-api.wildberries.ru/api/v1/supplier/stocks` | [warehouse_remains.php](file:///d:/www/React/wb-tools/public/php/warehouse_remains.php) | Фактические остатки товаров на складах хранения WB (FBO) | Нет | **Истёк токен (401)** (срабатывает XLSX fallback) |
| `GET marketplace-api.wildberries.ru/api/v3/stocks/{warehouse_id}` | [warehouse_snapshots.py](file:///d:/www/React/wb-tools/public/php/warehouse_snapshots.py) | Чтение и передача остатков на складах FBS (`6222` и `1884105`) | Нет | **Истёк токен (401)** |
| `POST content-api.wildberries.ru/content/v2/get/cards/list` | [warehouse_snapshots.py](file:///d:/www/React/wb-tools/public/php/warehouse_snapshots.py) | Реестр карточек товаров, номенклатуры, штрихкоды | Нет | **Истёк токен (401)** |
| `GET feedbacks-api.wildberries.ru/api/v1/feedbacks` | [feedbacks.php](file:///d:/www/React/wb-tools/public/php/feedbacks.php) | Получение отзывов и оценок покупателей | Нет | Токен валиден до 2027 г. (работает) |
| `GET dp-calendar-api.wildberries.ru/api/v1/calendar/promotions` | [collect_analytics.py](file:///d:/www/React/wb-tools/public/php/collect_analytics.py) | Официальный календарь промо-акций Wildberries | Нет | Токен валиден / общедоступно |

---

## 4. Что нужно сделать для восстановления работы

1. Зайти в личный кабинет Wildberries: **Профиль -> Настройки -> Доступ к API / Токены**.
2. Сгенерировать новый API-токен (или токены) со следующими правами:
   - **Статистика** (для `test.php` — «Статистика заказов», и `warehouse_remains.php` — остатки FBO)
   - **Цены и скидки** (для `Promo` и `min_prices.php`)
   - **Продвижение** (для рекламы и ставок CPC/CPM)
   - **Маркетплейс** (для остатков FBS складов)
   - **Контент** (для списка карточек товаров)
3. Обновить значения в файле [public/php/.env](file:///d:/www/React/wb-tools/public/php/.env):
   - `WB_TOKEN`
   - `WB_TOKEN_ANALITICS_SEO`
   - `WB_TOKEN_PRICES_AND_SALES`
   - `WB_TOKEN_SALES`
4. **Результат:** 
   * «Статистика заказов» (тепловая карта) **сразу вернётся в строй**.
   * Раздел «Склады», «Реклама», «Цены и скидки» (Promo) также заработают штатно.
   * А вот воронка продаж и видимость продолжат сообщать о необходимости подписки «Джем» (HTTP 403), если она не будет продлена.
