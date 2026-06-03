# BookingKoala Pricing Research

Дата: 2026-06-02
Цель: собрать максимум данных о ценах, услугах и логике калькуляторов с сайтов на BookingKoala (Toronto/GTA).

---

## Технические наблюдения по платформе BookingKoala

### Архитектура
- Все «booknow» страницы — Angular SPA (`<bk-root></bk-root>`), статический HTML — только оболочка.
- Скрипты: `https://cdn.bookingkoala.com/customer-build/142/{runtime,polyfills,vendor,main}.{hash}.js` (build 142, theme version 29, customer-live-3.0.0).
- Цвет/текст/настройки берутся с сервера; цены прайслиста подтягиваются после загрузки SPA.

### Public API
- База API на каждом сабдомене: `https://{sub}.bookingkoala.com/api/v1/`
- Доп. неймспейсы: `/api/v5/`, `/bktheme/v1/`, `/campaigns/v1/`, `/checklist...`.

### Ключевые endpoint (action → URL mapping из main.js):
| Action | Path | Метод | Заметки |
|---|---|---|---|
| `IpInfo` | `getip/` | GET | публичный (`auth:false`) — но всё равно требует Auth-token |
| `ServerUTCTime` | `utc-timestamp` | GET | публичный (`auth:false`) |
| `GetPlans` | план тарифа | GET | публичный (`auth:false`) |
| `ShortFormJson` | `/assets/data/short_form.json` | GET | публичный (`auth:false`) — но за Cloudflare |
| `AppLoadNew` | `appload-customer` | GET | bootstrap данных для customer-формы |
| `AdmnStngs` | `admin-settings-customer` | GET | настройки мерчанта (включая `merchant_settings.store.domain_name`) |
| `FormParams` | `form-params` | GET | параметры формы |
| `PricingParams` | `pricing-parameters` | GET | **прайслист** |
| `Addons` | `addons` | GET | список add-ons / extras |
| `AvailSett` | `industry-settings-ids/{industryId}/{formId}` | GET | настройки доступности |
| `DesignSett` | `site/design-settings` | GET | визуальная тема |
| `BlogsSett` | `blog-settings` | GET | |
| `FacebookCoupon` | `fbcoupon` | GET | |
| `FeaturesTrial` | `features/trial` | GET | |
| `Reviews` | `get-reviews` | GET | |
| `GiftCardGallery` | `gift-card-gallery` | GET | |
| `Languages` | `languages` | GET | |
| `Login` | `login` | POST | |
| `ProviderSignup` | `provider-signup` | POST | |
| `bookingsForDates` | `bookings-for-dates` | POST | проверка свободных слотов |
| `applicableProviders` | список доступных уборщиков | POST | |
| `LeadContacts` | приём lead-формы | POST | |
| `SubscribeEmail` | подписка | POST | |

### Авторизация (Cloudflare + HMAC)
- Все API-вызовы (даже «no auth») идут с заголовком `Auth-token`:
  `Auth-token = sha512(domainName + LT + X) + "_" + sha512(domainName)`
  где `domainName` берётся из `merchant_settings.store.domain_name`, `LT` — статический солт из приложения, `X` — динамическое значение (вероятно UTC-timestamp).
- Cloudflare блокирует «голый» curl (`User-Agent: curl/...`) → 403. С реалистичными браузерными заголовками сабдомен отдает HTML SPA, но API всё равно требует валидный `Auth-token`.
- Поэтому без реального браузера/headless с выполнением `bk-tracking.js` + `tejas-bootstrap.min.js` цены прайс-листа достать программно затруднительно.

### Параметры, влияющие на цену (по коду)
- `items` (тип услуги: Standard / Deep / Move-in/Move-out / Office / Airbnb / Post-Construction)
- `bedrooms`, `bathrooms`, `half_bathrooms`
- `frequency` (one-time / weekly / bi-weekly / every 4 weeks)
- `extras` (см. конкретные списки ниже)
- `addons` (платные опции с количеством)
- `customer_zipcode` (zone pricing — конкретный ZIP может изменить базовый тариф)
- `coupon_code` (промокоды)
- `bookingTaxRate`, `bookingTaxType` (налоги вычисляются на стороне сервера)
- `bookingTimeHours/Min` (длительность — duration estimate)
- `customer_interior` (внутреннее vs внешнее обслуживание)
- `quantity_based` / `count_multiple_spots` (для extras с количеством)

---

## Summary Table

| Company | URL | BookingKoala domain | Services найдено | Base prices | Add-ons | Recurring discount | Заметки |
|---|---|---|---|---|---|---|---|
| **Toronto Shine Cleaning** | torontoshinecleaning.ca | torontoshinecleaning.bookingkoala.com | Regular, Deep, Move-in/out, Office, Post-Construction, Airbnb Turnover | **Получены из калькулятора** (1 BR: General $183.75, Deep/Reno $417.38, Move $213.75, Office $157.50 — до налога) | Видно перечень (10+) | «Up to 15% Off for Regular Services» | Прозрачные cancellation-fees: $50 / 60% / 100% |
| **Cleaning Hive** | cleaninghive.ca | cleaninghive.bookingkoala.com | Regular, Deep, Move-in/out, Post-Renovation, Office/Commercial | **Получены из калькулятора** (Studio: General $113.50, Move $108.50, Reno $153.50, Office $100 — до налога) | Не показаны на маркетинге | Не показан | Самые низкие минимальные тарифы в выборке |
| **Beam Cleaning** | beamcleaning.ca | beamcleaning.bookingkoala.com | Condo, Office, Standard, Deep, Move-in/out, Airbnb | **Полностью открыты** | **Полностью открыты** | Weekly $93–$110+, плюс 15% OFF (regular) / 10% OFF (one-time, code APPRECIATE) | Самый прозрачный прайс в выборке |
| **Take Care Cleaners** | takecarecleaners.com | takecarecleaners.bookingkoala.com | Deep, Move-Out, Standard, Condo, Airbnb, Janitorial, Commercial, Office, Restaurant | Скрыты | Не показаны | Не показан | Только quote-form, никаких цен на маркетинге |
| **The Cleaning Hero** | thecleaninghero.ca | thecleaninghero.bookingkoala.com | House, Deep, Moving, Office, Post-Construction | Скрыты | Не показаны | Не показан | Локализация по Scarborough/Markham/Vaughan/RichmondHill |
| **RNV Cleaning** | rnvcleaning.com | rnvcleaningservices.bookingkoala.com | Residential, Commercial, Real Estate, Airbnb, New Baby | **Частично:** Standard $150 / Deep $180 / Commercial $99 (starting) | Не показаны | Не показан | Cleaning-flow прямо на сайте: `/booknow/book-residential-cleaning/` и т.п. |
| **Deepcleanin6 (subdomain)** | deepcleanin6.bookingkoala.com | сам сабдомен | Bathroom Only, Condo/Apt (Basic+Total), Home Cleaning | **Получены** (Bathroom $90, Condo Basic $270/4ч, Total $400/6ч — до налога) | Не показаны | Не показан | Несколько отдельных booknow‑страниц вместо единого flow |
| **Spotless Sweep Cleaning** | spotlesssweepcleaning.bookingkoala.com | сам сабдомен | Move-in/out clean (по URL пути) | Скрыты (CF‑блок) | n/a | n/a | Сайт блокировался организационными настройками |
| **PureSpace Cleaning CA** | purespacecleaningca.bookingkoala.com | сам сабдомен | House (Package + Hourly), Office | **Получены** (Standard $179, Deep $258, Move $308, Reno $373, Office $72 — до налога) | Hourly 2–7 ч (~$54/ч) | Не показан | Маркетинг `.ca` «Launching Soon», но BK‑аккаунт уже работает |
| **MaidForHomes CA** | maidforhomesca.bookingkoala.com | сам сабдомен | n/a (только privacy-policy открылся в curl) | Скрыты | n/a | n/a | Актуальный интерфейс бронирования не найден (возможно скрыт или требует аккаунта) |

---

## Detailed Findings

### 1. Toronto Shine Cleaning

- **URL:** https://torontoshinecleaning.ca
- **BookingKoala URL:** https://torontoshinecleaning.bookingkoala.com/booknow (CF 403 при WebFetch)
- **Services:**
  - Regular Residential Cleaning
  - Deep Cleaning
  - Move In / Move Out Cleaning
  - Office Cleaning
  - Post Construction Cleaning
  - Airbnb Turnover Cleaning
- **Pricing model:** flat fee, рассчитывается в booking-форме по типу + bedrooms + bathrooms + extras + frequency.
- **Full prices:** **не опубликованы** на маркетинговом сайте. Открыты только в SPA-калькуляторе (требует реальный браузер).
- **Цены из калькулятора (One‑Time, 1 BR / 1 BA, 0001–0599 ft²):**

| Услуга | До налога (CAD) | С налогом HST 13% (CAD) |
|---|---|---|
| General Cleaning | 183.75 | 207.64 |
| Move‑In / Move‑Out Cleaning | 213.75 | 241.54 |
| Post‑Renovation Cleaning | 417.38 | 471.64 |
| TurnOver / Airbnb Cleaning | 183.75 | 207.64 |
| Office Cleaning (one‑time) | 157.50 | 177.98 |

- **Add-ons (видимые перечислением, без цен):**
  - Finished basement cleaning
  - Inside oven
  - Inside refrigerator
  - Inside cabinets
  - Additional appliances and cupboards
  - Window and track cleaning
  - Dishwasher loading or handwashing
  - Laundry and folding
  - Handwipe / wet baseboards
  - Interior wall cleaning
- **Не входит в обслуживание:** moving heavy items, outdoor work, carpet/steam, hardwood polishing, pet/biohazard, mold, deep stain removal, light bulb cleaning, chandelier, dish storage, balconies/decks/backyards, window/closet tracks, exterior windows.
- **Discounts:** "Up to 15% Off for Regular Services" (recurring).
- **Cancellation policy (важная часть прайса):**
  - <48h до уборки: **$50** fee
  - В день уборки: **60%** от стоимости
  - Если cleaner уже приехал: **100%** от стоимости
- **Required inputs (по тексту сайта):** address, тип сервиса, bedrooms, bathrooms, contact info.
- **API endpoints found:** `https://torontoshinecleaning.bookingkoala.com/api/v1/*` — все возвращают `401 Unauthorized` без HMAC-токена.
- **Screens/steps where prices appear:** только внутри booking-формы (`/booknow`) после ввода адреса.
- **Notes:** «Crystal clear pricing — know the cost upfront» (только риторика, конкретных цифр публично нет). Single example из отзыва: 1-bedroom apartment около $300 (deep clean — это согласуется с GTA-рыночными ставками).

### 2. Cleaning Hive

- **URL:** https://cleaninghive.ca
- **BookingKoala URL:** https://cleaninghive.bookingkoala.com/booknow (CF 403)
- **Services:**
  - Regular Cleaning (weekly / bi-weekly / monthly)
  - Deep Cleaning
  - Move-In / Move-Out Cleaning
  - Post-Renovation Cleaning
  - Office / Commercial Cleaning
- **Pricing model:** quote-based / flat-fee по конфигу в форме.
- **Full prices:** **получены из калькулятора** (gated by booking form, но при подборе минимальных параметров видна сводка).
- **Цены из калькулятора (One‑Time, Studio / 1 BR / 1 BA, 100–499 ft²):**

| Услуга | До налога (CAD) | С налогом HST 13% (CAD) |
|---|---|---|
| General Cleaning | 113.50 | 128.26 |
| Move In / Out Cleaning | 108.50 | 122.61 |
| Post Renovation Cleaning | 153.50 | 173.46 |
| TurnOver / Airbnb Cleaning | 113.50 | 128.26 |
| Office Cleaning (one‑time, 0–499 ft²) | 100.00 | 113.00 |

- **Add-ons:** в перечислении нет; только generic «dusting, vacuuming, mopping».
- **Discounts:** не опубликованы.
- **Required inputs (по структуре формы):** type, postal code, размер жилья.
- **API endpoints found:** аналогичные, все требуют Auth-token.
- **Notes:** дополнительные URL `/hiring/form/join-our-team`, `/login` — стандартные шаблонные роуты BookingKoala.

### 3. Beam Cleaning (Самый детальный прайс)

- **URL:** https://beamcleaning.ca
- **BookingKoala URL:** https://beamcleaning.bookingkoala.com/booknow?embed=true (встроен через iframe + iFrame Resizer)
- **Services (4 категории по prefix-roller на /pricing):**
  - Condo Cleaning
  - Office Cleaning
  - Standard Cleaning
  - Deep Cleaning
  - Move-In / Move-Out Cleaning
  - Airbnb Cleaning (через add-on "laundry")
- **Pricing model:** flat по bedrooms (condo) или по sq ft (office) × frequency, плюс fixed-price extras.
- **Full prices:**

**Condo Cleaning (по комнатам)**

| Bedrooms | Weekly | One-time (starting) | Duration |
|---|---|---|---|
| 1 BR | $110 | $129 | 2–2.5 hours |
| 2 BR | $127 | $149 | 2.5–3 hours |
| 3 BR | $161 | $189 | 3–3.5 hours |

(Стартовая цена на главной — Condo Weekly $93 / One-time $109 — это после промо «15% OFF».)

**Office Cleaning (по площади)**

| Sq.Ft | One-time | Weekly | Bi-weekly | Monthly |
|---|---|---|---|---|
| <1000 | $149 | $127 | $134 | $1420 *(вероятно опечатка на сайте, должно быть $142)* |
| 1001–1500 | $179 | $152 | $161 | $170 |
| 1501–2000 | $199 | $169 | $179 | $189 |

Durations: 2–2.5 / 2.5–3 / 3–3.5 часов соответственно.

**Hourly Rate** (всё, что вне пакета): **$40/hour**

- **Add-ons / Extras (Condo):**

| Extra | Цена |
|---|---|
| First Time / Deep Clean | +$99 |
| Move In/Out | +$129 |
| Balcony | +$29 |
| Fridge | +$29 |
| Oven | +$29 |
| Interior Walls | +$99 |
| Interior Windows | +$79 |
| Empty Cabinets | +$19 / room |
| Pets | +$29 |
| Student Property | +$49 |
| Green Cleaning | +$19 |

- **Add-ons / Extras (Office):**

| Extra | Цена |
|---|---|
| Sanitation Package | +$25 / 500 sq ft |
| Disinfect Electronics | +$49 |
| Tile Steam Clean | +$25 / 100 sq ft |
| Water Plants | +$9 |

- **Discounts / Offers:**
  - Regular Weekly Cleaning: **15% OFF**
  - One-Time Basic Cleaning: **10% OFF** for new clients (code: `APPRECIATE`, не комбинируется)

- **Required inputs:** address, тип, bedrooms (для condo) или sq ft (для office), frequency, extras.
- **Не включено в любые пакеты (требует $40/hr hire):** Exterior Windows, Carpet Cleaning, Animal Waste, Mold Removal, Industrial Cleaning, Lifting Heavy Items, Above 2-Step Ladder, High Ceiling Fans, Laundry (кроме Airbnb), Washing Dishes.
- **Service areas:** Toronto, Etobicoke, North York, Mississauga, Oakville, Burlington, Vaughan, Richmond Hill, Markham (zone pricing не подтверждён).
- **Notes:** **Beam Cleaning — единственный сайт, опубликовавший полный прайс. Все формулы и цены — рабочий шаблон для копирования.**

### 4. Take Care Cleaners

- **URL:** https://takecarecleaners.com/booking/
- **BookingKoala URL:** https://takecarecleaners.bookingkoala.com/booknow?embed=true (iframe, `offsetTop=-100`)
- **Services (9 категорий — самая широкая выборка):**
  - Deep Cleaning
  - Move-Out Cleaning
  - Standard Clean
  - Apartment / Condo Cleaning
  - Airbnb Cleaning
  - Janitorial Service
  - Commercial Cleaning
  - Office Cleaning
  - Restaurant Cleaning
- **Pricing model:** **fully quote-based** — никаких сумм на сайте.
- **Full prices:** **скрыты** (требуется лид-форма «Get Your Free Quote»).
- **Add-ons:** не видны на маркетинге.
- **Discounts:** не опубликованы.
- **Required inputs (на лид-форме):** full name, phone, city, bedrooms, bathrooms, email.
- **Notes:** Take Care использует BookingKoala как backend, но на маркетинге держит «closed pricing» — сначала контакт, потом цена.

### 5. The Cleaning Hero

- **URL:** https://thecleaninghero.ca/book-now/
- **BookingKoala URL:** https://thecleaninghero.bookingkoala.com/booknow?embed=true (iframe)
- **Services (5 категорий + локализованные страницы):**
  - House Cleaning (`/house-cleaning-scarborough/`)
  - Deep Cleaning (`/deep-cleaning-scarborough/`)
  - Moving Cleaning (`/moving-cleaning-scarborough/`)
  - Office Cleaning (`/office-cleaning-scarborough/`)
  - Post-Construction Cleaning (`/post-construction-cleaning-scarborough/`)
- **Pricing model:** quote-based.
- **Full prices:** **скрыты**, упомянуто только «standard cleanings в Scarborough — 2 to 4 hours», post-construction «4–8 hours».
- **Add-ons:** не показаны.
- **Discounts:** не опубликованы.
- **Required inputs:** имя/phone/email/тип уборки на quote-форме.
- **Service areas:** Toronto, Markham, Vaughan, Richmond Hill, North York, Durham.
- **Notes:** локация в URL — SEO-pattern, не геопрайсинг.

### 6. RNV Cleaning

- **URL:** https://rnvcleaning.com/booknow/
- **BookingKoala URL:** https://rnvcleaningservices.bookingkoala.com/booknow?embed=true (iframe)
- **Services (5 категорий с собственными URL):**
  - Residential Cleaning (`/booknow/book-residential-cleaning/`)
  - Commercial Cleaning (`/booknow/book-commercial-cleaning/`)
  - Real Estate Cleaning (`/booknow/book-real-state-cleaning/`)
  - Airbnb Cleaning (`/booknow/book-airbnb-cleaning/`)
  - New Baby Service (`/booknow/book-new-baby/`)
- **Pricing model:** hybrid — на маркетинге «starting from», в калькуляторе — flat по конфигу.
- **Full prices (опубликованы как стартовые):**

| Service | Starting Price | Duration |
|---|---|---|
| Standard Cleaning | $150 | 2–3 hours |
| Deep Cleaning | $180 | 4–6 hours |
| Commercial Cleaning | $99 | 3–5 hours |

- **Add-ons:** не опубликованы.
- **Discounts:** не опубликованы. Гарантия: re-clean в течение 3 дней, если что-то пропущено.
- **Required inputs:** address, тип, размер.
- **Notes:** «free estimates», «rates vary by size and condition» — публично всё, что есть, выше.

### 7. Deepcleanin6 (`deepcleanin6.bookingkoala.com`)

- **URL:** прямой сабдомен, главного домена не нашёл.
- **Структура:** аккаунт собран из нескольких отдельных booking‑страниц (по одной на тип услуги): `/booknow/bathroom-cleaning-only`, `/booknow/condo-apt-cleaning`, `/booknow/home-cleaning` и т.д. Каждая страница открывает свой набор пакетов и extras.
- **Pricing model:** flat‑rate за пакет + опционально hourly у Condo/Apt; для Home Cleaning — Flat Rate Service по конфигу.
- **Цены из калькулятора:**

**Bathroom Cleaning Only** (1 ванная, разовая уборка)

| Пакет | До налога (CAD) | С налогом HST 13% (CAD) |
|---|---|---|
| Regular Bathroom Cleaning | 90.00 | 101.70 |
| Mold & Residue Removal | 250.00 | 282.50 |

**Condo / Apt Cleaning** (Studio в Extras)

| Пакет | Длительность | До налога (CAD) | С налогом HST 13% (CAD) |
|---|---|---|---|
| Basic | 4 ч | 270.00 | 305.10 |
| Total (Move‑In/Out) | 6 ч | 400.00 | 452.00 |

**Home Cleaning** — на момент исследования параметры по умолчанию (3 BR / 1 BA / 1000–1499 ft² / bi‑weekly) дали Flat Rate Service ≈ 255 CAD до налога, ≈ 288.15 CAD с налогом. Минимальная конфигурация (1 BR / 1 BA) не была подтверждена; ожидается ниже.

- **Notes:** аккаунт «живой», а не staging — структура с отдельными лендингами под каждый сервис вместо единого `/booknow` flow. Это удобный паттерн для SEO/таргетирования рекламы под конкретный тип уборки.

### 8. Spotless Sweep Cleaning

- **URL:** https://spotlesssweepcleaning.bookingkoala.com/move-in-out-clean (прямой сабдомен)
- **Services:** по URL — точно есть **Move-In / Move-Out Cleaning** как отдельный flow (вероятно конфиг с pre-selected `items`).
- **Pricing model:** не подтверждён публично.
- **Full prices:** скрыты.
- **Notes:** URL-pattern `/move-in-out-clean` — это «landing page» через BookingKoala-кастом, который сразу открывает определённый service-tab.

### 9. PureSpace Cleaning CA

- **URL:** https://purespacecleaningca.bookingkoala.com/booknow
- **Маркетинговый сайт:** `purespacecleaning.ca` — статус «Launching Soon».
- **Покрытие:** Toronto и Burlington (выбирается на первом шаге формы).
- **Pricing model:** есть две независимые вкладки — **Package** (flat по типу) и **Hourly** (выбор 2–7 ч). Отдельно — Office Cleaning.
- **Цены из калькулятора (1 BR / 1 BA, 1–999 ft²):**

**House Cleaning — Package**

| Услуга | До налога (CAD) | С налогом HST 13% (CAD) |
|---|---|---|
| Standard House Cleaning | 179.00 | 202.27 |
| Deep Cleaning | 258.00 | 291.54 |
| Move In / Move Out Cleaning | 308.00 | 348.04 |
| Post‑Renovation Cleaning | 373.00 | 421.49 |
| Airbnb / Rental Cleaning | 273.00 | 308.49 |

**House Cleaning — Hourly** (без доп. опций)

| Часы | До налога (CAD) | С налогом HST 13% (CAD) |
|---|---|---|
| 2 ч (минимум, для condo) | 110.00 | 124.30 |
| 3 ч | 162.00 | 183.06 |
| 4 ч | 216.00 | 244.08 |
| 5 ч | 270.00 | 305.10 |
| 6 ч | 324.00 | 366.12 |
| 7 ч | 378.00 | 427.14 |

Эффективная ставка: ≈ 54 CAD/ч (с шага в 3 ч); первый 2‑часовой минимум — 55 CAD/ч.

**Office Cleaning**

| Услуга | До налога (CAD) | С налогом HST 13% (CAD) |
|---|---|---|
| General Office Cleaning (0–999 ft²) | 72.00 | 81.36 |
| Office Deep Cleaning | 215.00 | 242.95 |

- **Notes:** несмотря на «Launching Soon» на маркетинговом домене, BookingKoala‑аккаунт уже полностью настроен и работает. Hourly‑опция — прямой аналог Beam $40/hr fallback, но дороже (~$54/hr) и без жёсткого минимума.

### 10. MaidForHomes CA

- **URL:** https://maidforhomesca.bookingkoala.com/privacy-policy (открывается)
- **Сам booknow / маркетинг:** недоступны через WebFetch/curl без байпаса CF.
- **Services / prices:** не подтверждены публично.

---

## Какие сайты дали полные / частичные / нулевые цены

### Полные (рабочий прайслист в открытом доступе на маркетинговом сайте)
- **Beam Cleaning** — по 5 категориям + по bedroom-tier для condo + по sq.ft для office, плюс 11 extras для condo и 4 для office, плюс recurring discounts и hourly rate $40.

### Полные через калькулятор (минимальная конфигурация снята с booking‑формы)
- **Toronto Shine Cleaning** — 5 услуг, 1 BR / 0001–0599 ft²: $157.50–$417.38 до налога.
- **Cleaning Hive** — 5 услуг, Studio / 100–499 ft²: $100–$153.50 до налога (самые низкие в выборке).
- **PureSpace Cleaning CA** — 5 услуг Package + Hourly (2–7 ч) + Office (2 пакета), 1 BR / 1–999 ft²: $72–$373 до налога.
- **Deepcleanin6** — Bathroom Only ($90–$250), Condo/Apt Basic 4ч/$270 и Total 6ч/$400, Home Cleaning ≈$255 для конфигурации по умолчанию.

### Частичные (видны стартовые цены)
- **RNV Cleaning** — 3 стартовые цены по основным сервисам ($99/$150/$180).

### Скрыто до адреса/логина/контактов
- **Take Care Cleaners, The Cleaning Hero, Spotless Sweep, MaidForHomes** — требуют либо заполнения lead-form, либо загрузки SPA-формы (которую публично не парсится из-за HMAC-auth + Cloudflare); часть сайтов блокировалась организационными настройками.

---

## Какие компании используют одинаковую BookingKoala-структуру

Все 10 — один и тот же:
- Angular SPA, build 142, theme 29.
- `/booknow`, `/login`, `/hiring/form/join-our-team`, `/privacy-policy` — одинаковые routes у всех.
- iframe-вставка через `https://{sub}.bookingkoala.com/booknow?embed=true` + `resources/embed.js` (iFrame Resizer 4.1.1) — это **типовая интеграция BookingKoala в WordPress** (Beam, Take Care, Cleaning Hero, RNV — все идентично).
- Customer-side калькулятор работает через API `/api/v1/pricing-parameters` + `/api/v1/addons` + `/api/v1/form-params`.
- Hourly-rate ($40/hr у Beam) — встроенная фича BookingKoala «hourly service».
- Cancellation-fees ($50 / 60% / 100% у Toronto Shine) — стандартный 3-tier пресет BookingKoala (`booking_cancellation` в коде).

---

## Pricing patterns для копирования в наш cleaning-сайт

### Базовая модель калькулятора (на основе всех 10 сайтов)
1. **Тип услуги** (выбор из):
   - Standard / Regular Cleaning
   - Deep Cleaning
   - Move-In / Move-Out Cleaning
   - Post-Construction (Post-Renovation) Cleaning
   - Airbnb Turnover Cleaning
   - Office / Commercial Cleaning
   - Condo Cleaning (отдельный flow для «small footprint»)
2. **Размер**:
   - Для residential: bedrooms × bathrooms × half-bathrooms.
   - Для office: sq.ft brackets (<1000 / 1001–1500 / 1501–2000 / 2000+).
3. **Frequency** (с %-скидкой):
   - One-time (base)
   - Weekly (–15% типично)
   - Bi-weekly (–10% типично)
   - Monthly (–5% типично)
4. **Extras** (фикс-цена за штуку):
   - Inside oven / Inside fridge / Inside cabinets (≈$29 каждый у Beam)
   - Interior windows (≈$79), Interior walls (≈$99)
   - Baseboards (handwipe/wet)
   - Empty cabinets per room (≈$19/room)
   - Laundry & folding
   - Dishwasher loading
   - Window & track cleaning
   - Balcony (≈$29)
   - Pets surcharge (≈$29)
   - Student property (≈$49)
   - Green / eco cleaning (≈$19)
   - First-time / Deep clean upgrade на regular flow (≈$99)
   - Move In/Out upgrade (≈$129)
5. **Surcharges** для office:
   - Sanitation Package (per 500 sq.ft)
   - Disinfect Electronics
   - Tile Steam Clean (per 100 sq.ft)
   - Water Plants
6. **Hourly fallback** для «out-of-scope»: $40/hour — простой и понятный.
7. **Минимальная цена**: подразумевается через минимальный bedroom-tier (≈$129 one-time у Beam).
8. **Промо-механика**:
   - Coupon code (BookingKoala поддерживает field `coupon_code` + `couponDiscount`).
   - Welcome discount (10–15% off для первой уборки).
   - Recurring discount (15% off для weekly).
9. **Cancellation policy** как 3-tier:
   - >48h: free
   - <48h: flat fee (~$50)
   - same-day: 60% от цены
   - cleaner на месте: 100%
10. **Налог**: BookingKoala применяет `bookingTaxRate` + `bookingTaxType` — для ON это HST 13% поверх subtotal.
11. **Duration estimates**: показываются как диапазон («2–2.5 hours», «4–6 hours») — формирует ожидания клиента.
12. **Zone pricing** через postal code (M5V/M4S/M6G/M4W — конкретное различие не подтверждено на тестах, но `customer_zipcode` есть в форме как параметр, влияющий на applicableProviders).
13. **Service exclusions / disclaimers** — у Beam и Toronto Shine открыты списком (это полезно для transparency и сокращения support-тикетов).

### Что копировать прямо
- **Beam-style transparent pricing page** — все цифры в одной таблице, никаких «request quote».
- **Toronto Shine-style cancellation page** — явные $50 / 60% / 100% (это снимает 90% disputes).
- **RNV-style starting-from на главной** — psychological anchor («from $99») + полный калькулятор за CTA.
- **Beam-style $40/hr fallback** для всего, что не в пакете — отвечает на «а можно вот это?» без расширения каталога.

### Что НЕ копировать
- Hidden pricing у Take Care / Cleaning Hero — конверсия страдает; пользователи сейчас ожидают прозрачности.
- «Up to 15% Off» без чёткого условия (как у Toronto Shine) — выглядит как тизер, лучше явно: «Save 15% on weekly plans».

---

## Notes / Limitations

- BookingKoala-калькуляторы (`/booknow` SPA) **публично НЕ парсятся без headless-браузера** + правильного HMAC-токена (Cloudflare блокирует bare-curl, API требует Auth-token, генерируемый клиентским JS).
- Прямые сабдомены (deepcleanin6, spotlesssweep, purespace, maidforhomes) — закрыты CF при попытке WebFetch.
- Wayback Machine WebFetch'ом из этой сессии **недоступен** (заблокирован); curl до CDX API работает, но архивированные SPA-странички — это всё те же пустые оболочки без отрендеренных цен.
- Цены из отзывов / маркетинга могут устаревать; «source of truth» — `/api/v1/pricing-parameters` каждой компании.
- Adjustment пунктов на office у Beam подозрителен (Monthly $1420 для <1000 sq.ft при weekly $127 — вероятно опечатка на сайте, реалистично $142).
- Полная BookingKoala-схема прайса (item × packages × extras × frequencies × discounts × tax) видна в коде main.js (`itemsForm`, `extrasForm`, `quantity_based`, `count_multiple_spots`, `enable_quantity_based`) — её можно адаптировать в свою таблицу данных 1-в-1.

---

## Сравнение минимальных цен по компаниям (One‑Time, наименьшая конфигурация)

Все цены — до налога, CAD. Источники: калькуляторы соответствующих BookingKoala‑аккаунтов при выборе минимального количества комнат / площади / одноразовой уборки. Сравнение качественное (конфигурации отличаются — Studio у Cleaning Hive против 1 BR у Toronto Shine), но даёт диапазон рынка.

| Услуга | Cleaning Hive (Studio, 100–499 ft²) | Toronto Shine (1 BR, 0–599 ft²) | PureSpace (1 BR, 1–999 ft²) | Beam (1 BR, one‑time) | Deepcleanin6 |
|---|---|---|---|---|---|
| Standard / General | $113.50 | $183.75 | $179.00 | $129 | — (есть Home flow, ≈$255 для 3 BR/1000–1499 ft²) |
| Deep / First‑Time | — | — | $258.00 | $129 + $99 extra = $228 | — |
| Move In / Out | $108.50 | $213.75 | $308.00 | $129 + $129 extra = $258 | Condo Total 6 ч: $400 |
| Post‑Renovation / Construction | $153.50 | $417.38 | $373.00 | — | — |
| Airbnb / Turnover | $113.50 | $183.75 | $273.00 | через add‑on laundry | — |
| Office (one‑time, минимум) | $100.00 | $157.50 | $72.00 | $149 (<1000 ft²) | — |
| Bathroom only | — | — | — | — | $90.00 (Mold $250) |
| Hourly | — | — | $54/ч (2–7 ч) | $40/ч | — |

**Наблюдения:**
- **Диапазон рынка:** одноразовая уборка ванной у Deepcleanin6 — от ~$90 CAD до полной condo move‑in/out у Deepcleanin6 — ~$400 CAD; пиковая Post‑Renovation у Toronto Shine — $417.38 CAD до налога ($471.64 с HST). При увеличении площади/числа комнат любая из позиций может вырасти ещё на 50–150%.
- **Cleaning Hive — самый бюджетный** во всех категориях, кроме Office (там лидирует PureSpace с $72). Разница vs Toronto Shine на той же базе: ~38–63%.
- **Toronto Shine — самый дорогой** по Post‑Renovation ($417.38 vs $153.50 у Hive — разница ×2.7).
- **PureSpace** держит сбалансированный middle‑tier на Package + предлагает Hourly fallback (как Beam, но дороже: $54/ч vs $40/ч).
- **Beam** остаётся единственным с полностью публичным прайсом на маркетинге; его базовые цены — самые низкие среди прозрачных ($129 one‑time за 1 BR), но при глубокой уборке он быстро догоняет PureSpace ($228 vs $258).
- **HST 13%** применяется поверх subtotal у всех — это видно в сводках заказов (например, $183.75 → $207.64; $400 → $452).

---

## Files / endpoints для дальнейшей работы

- Полная копия JS: `/tmp/bk_research/bk_main.js` (~4 МБ), `tejas.js` (~22 КБ), `beam_embed.js` (iFrame Resizer).
- Архивированные HTML SPA-оболочки: `/tmp/bk_research/{torontoshinecleaning,cleaninghive,deepcleanin6,spotlesssweep,purespace,beamcleaning,takecarecleaners,thecleaninghero,rnvcleaning,maidforhomesca}.html`.
- Если будет нужен живой прайслист — следующий шаг: headless-браузер (Playwright/Puppeteer) с реальным выполнением JS до получения ответа `/api/v1/pricing-parameters` (полностью описывает структуру калькулятора каждой компании).
