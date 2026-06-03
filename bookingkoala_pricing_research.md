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
| **Toronto Shine Cleaning** | torontoshinecleaning.ca | torontoshinecleaning.bookingkoala.com | Regular, Deep, Move-in/out, Office, Post-Construction, Airbnb Turnover | Скрыты до booking-формы | Видно перечень (10+) | «Up to 15% Off for Regular Services» | Прозрачные cancellation-fees: $50 / 60% / 100% |
| **Cleaning Hive** | cleaninghive.ca | cleaninghive.bookingkoala.com | Regular, Deep, Move-in/out, Post-Renovation, Office/Commercial | Скрыты до booking-формы | Не показаны на маркетинге | Не показан | Один из чистейших BookingKoala-шаблонов |
| **Beam Cleaning** | beamcleaning.ca | beamcleaning.bookingkoala.com | Condo, Office, Standard, Deep, Move-in/out, Airbnb | **Полностью открыты** | **Полностью открыты** | Weekly $93–$110+, плюс 15% OFF (regular) / 10% OFF (one-time, code APPRECIATE) | Самый прозрачный прайс в выборке |
| **Take Care Cleaners** | takecarecleaners.com | takecarecleaners.bookingkoala.com | Deep, Move-Out, Standard, Condo, Airbnb, Janitorial, Commercial, Office, Restaurant | Скрыты | Не показаны | Не показан | Только quote-form, никаких цен на маркетинге |
| **The Cleaning Hero** | thecleaninghero.ca | thecleaninghero.bookingkoala.com | House, Deep, Moving, Office, Post-Construction | Скрыты | Не показаны | Не показан | Локализация по Scarborough/Markham/Vaughan/RichmondHill |
| **RNV Cleaning** | rnvcleaning.com | rnvcleaningservices.bookingkoala.com | Residential, Commercial, Real Estate, Airbnb, New Baby | **Частично:** Standard $150 / Deep $180 / Commercial $99 (starting) | Не показаны | Не показан | Cleaning-flow прямо на сайте: `/booknow/book-residential-cleaning/` и т.п. |
| **Deepcleanin6 (subdomain)** | deepcleanin6.bookingkoala.com | сам сабдомен | n/a | Скрыты (403/CF) | n/a | n/a | Главное домен не отдаёт SPA без CF-байпаса |
| **Spotless Sweep Cleaning** | spotlesssweepcleaning.bookingkoala.com | сам сабдомен | Move-in/out clean (по URL пути) | Скрыты | n/a | n/a | Глубинная страница `/move-in-out-clean` существует |
| **PureSpace Cleaning CA** | purespacecleaningca.bookingkoala.com | сам сабдомен | n/a (booknow flow) | Скрыты | n/a | n/a | Маркетинг-сайт `.ca` ещё «Launching Soon» |
| **MaidForHomes CA** | maidforhomesca.bookingkoala.com | сам сабдомен | n/a (только privacy-policy открылся в curl) | Скрыты | n/a | n/a | Маркетинговый домен закрыт (CF 403) |

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
- **Full prices:** **не опубликованы** (gated by booking form).
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
- **Services:** не подтверждено публично.
- **Pricing model:** не подтверждено.
- **Full prices:** скрыты за BookingKoala-формой + Cloudflare.
- **Notes:** сабдомен похож на test/staging account («deepcleanin» + цифра 6).

### 8. Spotless Sweep Cleaning

- **URL:** https://spotlesssweepcleaning.bookingkoala.com/move-in-out-clean (прямой сабдомен)
- **Services:** по URL — точно есть **Move-In / Move-Out Cleaning** как отдельный flow (вероятно конфиг с pre-selected `items`).
- **Pricing model:** не подтверждён публично.
- **Full prices:** скрыты.
- **Notes:** URL-pattern `/move-in-out-clean` — это «landing page» через BookingKoala-кастом, который сразу открывает определённый service-tab.

### 9. PureSpace Cleaning CA

- **URL:** https://purespacecleaningca.bookingkoala.com/booknow
- **Маркетинговый сайт:** `purespacecleaning.ca` — статус «Launching Soon».
- **Services / prices:** скрыты + маркетинг не запущен.
- **Notes:** свежий BookingKoala-аккаунт, ещё не наполненный.

### 10. MaidForHomes CA

- **URL:** https://maidforhomesca.bookingkoala.com/privacy-policy (открывается)
- **Сам booknow / маркетинг:** недоступны через WebFetch/curl без байпаса CF.
- **Services / prices:** не подтверждены публично.

---

## Какие сайты дали полные / частичные / нулевые цены

### Полные (рабочий прайслист в открытом доступе)
- **Beam Cleaning** — по 5 категориям + по bedroom-tier для condo + по sq.ft для office, плюс 11 extras для condo и 4 для office, плюс recurring discounts и hourly rate $40.

### Частичные (видны стартовые цены)
- **RNV Cleaning** — 3 стартовые цены по основным сервисам ($99/$150/$180).
- **Toronto Shine Cleaning** — opaque на цены, но **открытые cancellation fees** ($50/60%/100%) и список extras без цен.

### Скрыто до адреса/логина/контактов
- **Cleaning Hive, Take Care Cleaners, The Cleaning Hero, Deepcleanin6, Spotless Sweep, PureSpace, MaidForHomes** — все требуют либо заполнения lead-form, либо загрузки SPA-формы (которую публично не парсится из-за HMAC-auth + Cloudflare).

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

## Files / endpoints для дальнейшей работы

- Полная копия JS: `/tmp/bk_research/bk_main.js` (~4 МБ), `tejas.js` (~22 КБ), `beam_embed.js` (iFrame Resizer).
- Архивированные HTML SPA-оболочки: `/tmp/bk_research/{torontoshinecleaning,cleaninghive,deepcleanin6,spotlesssweep,purespace,beamcleaning,takecarecleaners,thecleaninghero,rnvcleaning,maidforhomesca}.html`.
- Если будет нужен живой прайслист — следующий шаг: headless-браузер (Playwright/Puppeteer) с реальным выполнением JS до получения ответа `/api/v1/pricing-parameters` (полностью описывает структуру калькулятора каждой компании).
