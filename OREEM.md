# Oreem Delivery & Shipping Integration — Full Technical Blueprint

دليل هندسي متكامل لإعادة بناء نظام التوصيل والشحن **Oreem** (المحلي داخل البحرين + الدولي، التسعير مع هامش الربح، جدولة أوقات العمل، الطلبات اليدوية من التجار والأدمن، ومحفظة Oreem والسحب التلقائي) من الصفر على أي مشروع **FastAPI + SQLAlchemy**.

---

## 1. البنية العامة (Architecture Overview)

ينقسم تكامل Oreem إلى مسارين رئيسيين وطبقة تسعير ومحفظة:

1. **التوصيل المحلي (Domestic — داخل البحرين):**
   - يعتمد على البلوك/المجمع (`block_id`) والمدينة (`city_id`) وسرعة التوصيل (`same_day` / `next_day` / `instant`) مع تحديد المركبة (`bike`).
   - يتحقق من أوقات عمل Oreem في توقيت البحرين (`Asia/Bahrain`, `UTC+3`)، ويجدول الطلب تلقائياً إذا كان خارج أوقات العمل.
2. **الشحن الدولي (International — خارج البحرين):**
   - يتفعل تلقائياً عند كون عنوان المشتري خارج البحرين (`not is_bahrain_address`)، عبر مفاتيح `INTERNATIONAL_OREEM_*`.
3. **طبقة التسعير وهامش الربح (`delivery_pricing.py`):**
   - تضيف هامش ربح المنصة (`delivery_internal_profit`) فوق سعر Oreem الأساسي، وتوفر تسعيرة احتياطية (`delivery_internal_fallback_cost`) عند تعطل API تسعير Oreem.
4. **التوصيل اليدوي (`manual_oreem_delivery.py`):**
   - يمكّن البائعين والأدمن من طلب مندوب Oreem لطلبات الهاتف أو الطلبات الخارجية دون إنشاء `Order` تقليدي، مع التحقق من رصيد محفظة البائع وخصم التكلفة منه.
5. **محفظة Oreem والسحب التلقائي (`oreem_wallet.py`, `oreem_auto_withdraw.py`, `oreem_wallet_notifications.py`):**
   - عرض الرصيد المتاح (`available_balance`) من Oreem، السحب اليدوي إلى IBAN محفوظ، تنبيهات انخفاض الرصيد، والسحب التلقائي اليومي مع الإبقاء على رصيد احتياطي (`keep`) لتغطية رسوم التوصيل للطلبات الجديدة.

---

## 2. متغيرات البيئة والأمان (`project/core/config.py`)

يجب إعداد مفاتيح API منفصلة للمحلي والدولي، بالإضافة إلى مفاتيح التحقق من الـ Webhook:

```env
# 1) Domestic (Bahrain) Oreem API
DOMESTIC_OREEM_API_KEY=...
DOMESTIC_OREEM_API_URL=https://...
DOMESTIC_OREEM_WEBHOOK_SECRET=...

# 2) International Shipping Oreem API
INTERNATIONAL_OREEM_API_KEY=...
INTERNATIONAL_OREEM_API_URL=https://...
SHIPPING_OREEM_WEBHOOK_SECRET=...

# 3) Webhook Security
OREEM_WEBHOOK_TOKEN=...
OREEM_WEBHOOK_MAX_AGE_SECONDS=300
```

- يتم التحقق من صحة طلبات الـ Webhook عبر `OREEM_WEBHOOK_TOKEN` وتوقيع HMAC باستخدام `DOMESTIC_OREEM_WEBHOOK_SECRET` (للمحلي) أو `SHIPPING_OREEM_WEBHOOK_SECRET` (للشحن)، مع رفض أي طلب يتجاوز عمره `OREEM_WEBHOOK_MAX_AGE_SECONDS` (الافتراضي `300` ثانية) لمنع Replay Attacks.

---

## 3. جداول قاعدة البيانات (Database Models)

1. **`OreemBlock` & `OreemCity`:**
   - تخزين المجمعات والمدن المتزامنة من Oreem محلياً لتسريع البحث وعدم الاعتماد على استعلامات خارجية متكررة.
1. **`OreemBlock` (`oreem_blocks`) & `OreemCity` (`oreem_cities`):**
   - **`OreemBlock`**: `id`, `oreem_id` (unique), `block_number` (مثل `"101"`), `address` (مثل `"101 Sharq Al Hidd شرق الحد"`), `city_name`, `latitude`, `longitude`, `is_active`, `last_synced_at`, `created_at`.
   - **`OreemCity`**: `id`, `name` (unique، مثل `"Manama"`, `"Isa Town"`), `block_count`, `is_active`, `last_synced_at`, `created_at`.
2. **حقول التوصيل في `Order`:**
   - `oreem_order_id`: معرف الطلب لدى Oreem.
   - `delivery_method`: مثل `oreem_local` أو نوع الشحن.
   - `delivery_status`: حالات مثل `"dispatching"`, `"scheduled"`, أو حالات التتبع من Oreem (`pending`, `processing`, `out_for_delivery`, `delsuccess`, `pkpfailed`, `delfailed`).
3. **`ManualOreemDelivery` & `ManualDeliveryReceiverHistory`:**
   - حفظ سجل الطلبات اليدوية وحفظ بيانات المستلمين السابقين لتسهيل إعادة الاختيار السريع للبائع/الأدمن.
4. **`AdminSavedIBAN` & `OreemAutoWithdrawLog`:**
   - حفظ حسابات IBAN المعتمدة للسحب وسجل عمليات السحب التلقائي اليومية.
5. **إعدادات `GeneralSettings`:**
   - `delivery_internal_profit` (float، الافتراضي `0.0`): هامش ربح المنصة المضاف لكل طلب توصيل.
   - `delivery_internal_fallback_cost` (float، الافتراضي `0.0`): التكلفة الأساسية الاحتياطية عند تعذر جلب التسعيرة من Oreem.
   - `oreem_wallet_low_balance_threshold_bhd` و `oreem_wallet_low_balance_last_notified_at`: إعدادات تنبيه انخفاض رصيد محفظة Oreem.
   - إعدادات السحب التلقائي (التفعيل، حد السحب `threshold`، الرصيد المحتفظ به `keep`).

---

## 3.5. جلب ومزامنة المدن والبلوكات وربط الـ GPS (`oreem.py`, `oreem_api.py`, `oreem_geo.py`)

> **قاعدة أساسية:** نظام Oreem يرفض أي أسماء مدن أو مجمعات مكتوبة بحرية (Free text)؛ لذلك يجب أن تأتي جميع القيم في الويب وFlutter من جدولي `OreemCity` و`OreemBlock` المخزنين محلياً.

### أ. كيف نجلب المدن والبلوكات من Oreem (`oreem_service.get_blocks`)
1. يطلب النظام قائمة مدن البحرين من Oreem:
   `GET /api/v1/shipments/cities?country_code=BH`
2. يمر على كل مدينة ويطلب بلوكاتها:
   `GET /api/v1/shipments/blocks?city_name={city_name}`
3. يوحّد البيانات في قائمة تحتوي `id`، `block` (`block_number`)، `address`، `latitude`، `longitude`.

### ب. كيف نخزنها ونحدثها في قاعدة البيانات (`POST /api/v1/delivery/sync-blocks-cities`)
1. يستدعي `oreem_service.get_blocks()`.
2. يستخرج اسم المدينة الإنجليزي من حقل `address` بإزالة رقم البلوك من البداية وأخذ الكلمات الإنجليزية قبل أي حرف عربي (`\u0600-\u06FF`).
3. يضيف أي بلوك غير موجود في `oreem_blocks` (مقارنة بـ `existing_oreem_ids`) مع عمل `db.commit()` كل `100` سجل لتفادي استهلاك الذاكرة.
4. يعيد حساب `block_count` الفعلي لكل مدينة نشطة في `oreem_cities` ويحدثها أو ينشئها.
5. **في الواجهات (الويب وFlutter):** تُقرأ القوائم مباشرة من قاعدة البيانات عبر `GET /api/v1/delivery/bahrain/cities` و`GET /api/v1/delivery/bahrain/blocks?city_name=...` لسرعة فائقة دون انتظار API الخارجي، كما يمكن للأدمن التعديل أو الإضافة من `/admin/oreem-data`.

### ج. تحويل إحداثيات الـ GPS إلى أقرب بلوك معتمد (`oreem_geo.py`)
- **`gps_in_bahrain(lat, lng)`:** يتحقق من وجود الإحداثيات داخل نطاق البحرين (`lat: 24.0..27.5`, `lng: 49.5..51.8`).
- **`find_nearest_oreem_block(db, lat, lng)`:** يحسب المسافة عبر `haversine_km` بين نقطة الـ GPS وجميع سجلات `OreemBlock` النشطة التي تملك إحداثيات، ويرجع `(block, distance_km, confidence)`:
  - `high`: المسافة $\le 1.5$ كم
  - `medium`: المسافة $\le 5.0$ كم
  - `low`: المسافة $> 5.0$ كم
  ويتم نسخ `block.city_name` و`block.block_number` كما هما لضمان قبول الطلب في Oreem 100%.

---

## 4. التسعير وهامش الربح (`main_app/utils/delivery_pricing.py`)

جميع حسابات التوصيل تمر عبر طبقة التسعير الداخلية لضمان عدم ظهور أي سعر خام بدون ربح المنصة:

- `get_internal_delivery_profit(db)` -> يقرأ `GeneralSettings.delivery_internal_profit` (افتراضي `0.0`).
- `get_internal_fallback_base_cost(db)` -> يقرأ `GeneralSettings.delivery_internal_fallback_cost` (افتراضي `0.0`).
- `apply_internal_delivery_profit(db, oreem_fee)` -> يرجع Tuple: `(round(raw + profit, 3), raw, profit)`.
- `internal_fallback_delivery_charge(db)` -> عند فشل استعلام السعر من Oreem، إذا كان `base > 0` يرجع `base + profit`.

> يتم استخدام دقة 3 منازل عشرية (`round(..., 3)`) لتطابق عملة الدينار البحريني (`BHD` - فلس).

---

## 5. منطق الإرسال التلقائي وجدولة أوقات العمل (`main_app/utils/oreem_integration.py`)

### أ. أوقات العمل (`Asia/Bahrain` - `UTC+3`)
- **ساعات العمل الفعلية:** من `08:05:00` صباحاً حتى `23:59:59` مساءً.
- **فترة الإغلاق:** من `00:00:00` حتى `08:04:59` صباحاً (`is_oreem_outside_hours()`).
- **اكتشاف أخطاء الإغلاق من API (`is_oreem_closed_error`):**
  يطابق نصوص الخطأ التالية (case-insensitive):
  `"outside_hours"`, `"outside hours"`, `"closed right now"`, `"we're closed"`, `"currently closed"`, `"outside operating hours"`, `"operating hours"`.

### ب. سرعات التوصيل (`normalize_delivery_speed`)
يدعم النظام 3 سرعات موحدة:
- `'same_day'` (الافتراضي)
- `'next_day'`
- `'instant'`

### ج. دالة `create_delivery_for_order`
1. تتخطى الطلب إذا كان `order.oreem_order_id` موجوداً مسبقاً، أو إذا كانت جميع المنتجات رقمية (`is_digital`)، أو إذا كان التوصيل ذاتياً من التاجر (`is_self_delivered`).
2. تفحص عنوان الشحن عبر `is_bahrain_address`:
   - داخل البحرين -> `_create_domestic_delivery`
   - خارج البحرين -> `_create_international_shipment`

### د. دالة `trigger_oreem_delivery(db, order, force_now=False)`
تُنفذ دورة إرسال محمية من التكرار والتزامن (Race Conditions):
1. **قفل الصف في قاعدة البيانات:** تستخدم `with_for_update()` على سجل الطلب.
2. **منع التكرار:** ترفض التنفيذ إذا كان `oreem_order_id` مسجلاً أو `delivery_status == "dispatching"`.
3. **حظر التجار خارج البحرين:** تمنع الإرسال إذا كان البائع من خارج البحرين (`order_blocks_oreem`, `is_store_outside_bahrain`).
4. **الجدولة خارج أوقات العمل:** إذا كان `not force_now` و `is_oreem_outside_hours()` صحيحاً:
   - تضبط `db_order.delivery_status = "scheduled"`
   - تضبط `db_order.delivery_method = db_order.delivery_method or "oreem_local"`
   - يتولى الـ Cron/Scheduler الصباحي إرسال الطلبات المجدولة عند الساعة `08:05 AM`.
5. **الإرسال الفوري:**
   - تضبط `delivery_status = "dispatching"` لمنع أي محاولة متزامنة.
   - إذا كانت طريقة الدفع COD (`"cod"`, `"cash_on_delivery"`, `"cash"`)، تمرر `cash = total_amount` ليحصّلها المندوب.
   - تستدعي `create_delivery_for_order(..., vehicle="bike", trigger_mode="manual")`.
6. **معالجة الأخطاء:**
   - إذا رجع خطأ `OUTSIDE_HOURS`، تتحول الحالة تلقائياً إلى `delivery_status = "scheduled"`.
   - في الأخطاء الأخرى، تعيد `delivery_status` إلى `None` (أو `"scheduled"` إذا كان مجدولاً مسبقاً `was_previously_scheduled`).

---

## 6. التوصيل اليدوي للطلبات الخارجية (`main_app/utils/manual_oreem_delivery.py`)

يسمح للتاجر أو الأدمن بطلب مندوب Oreem لطلبات الواتساب/الهاتف مباشرة:

- **السرعات المدعومة:** `DELIVERY_METHODS = ("same_day", "next_day", "instant")`.
- **الحالات النهائية للطلب:** `TERMINAL_OREEM_DELIVERY = ("delsuccess", "pkpfailed", "delfailed")`.
- **حالات الطلبات النشطة:** `ACTIVE_OREEM_ORDER_STATUSES = ("pending", "processing", "out_for_delivery")`.
- **التحقق من محفظة البائع (`InsufficientSellerWalletError(required, available)`):**
  - قبل إرسال أي طلب يدوي، يتحقق النظام من أن رصيد محفظة المتجر يغطي إجمالي رسوم التوصيل (`required` BHD شاملة هامش الربح).
  - إذا كان الرصيد غير كافٍ، يتم إيقاف العملية برفع `InsufficientSellerWalletError`.
  - عند نجاح الطلب، يتم خصم المبلغ وتسجيله في `WalletTransaction` وحفظ الطلب في `ManualOreemDelivery` وبيانات المستلم في `ManualDeliveryReceiverHistory`.

---

## 7. محفظة Oreem في لوحة التحكم والسحب التلقائي

### أ. محفظة التاجر (`main_app/utils/oreem_wallet.py`)
- تستخدم مفاتيح `INTERNATIONAL_OREEM_API_KEY` / `INTERNATIONAL_OREEM_API_URL` مع Fallback إلى `DOMESTIC_OREEM_API_KEY` / `DOMESTIC_OREEM_API_URL`.
- **الحد الأدنى للسحب:** `MIN_WITHDRAWAL_BHD = 0.111` (لتغطية رسوم التحويل البنكي > `0.110` BHD).
- **الدوال الأساسية:**
  - `fetch_oreem_wallet_balance(timeout=30.0)` -> `GET {_BASE_URL}/api/v1/wallet/balance`
  - `submit_oreem_wallet_withdrawal(amount, iban, ...)` -> `POST {_BASE_URL}/api/v1/wallet/withdraw`

### ب. السحب التلقائي اليومي (`main_app/utils/oreem_auto_withdraw.py`)
- يعتمد **حصراً** على `oreem_wallet.get("available_balance")` (ولا يستخدم الرصيد المعلق أو الإجمالي).
- عندما يتجاوز الرصيد المتاح الحد المحدد (`Available > threshold`):
  - يسحب مبلغ `Available - keep` (BHD) إلى الـ IBAN المحدد في `AdminSavedIBAN`.
  - يُبقي مبلغ `keep` في الرصيد المتاح لتغطية رسوم توصيل الطلبات الجديدة.
- يُنفذ مرة واحدة يومياً بتوقيت البحرين (`UTC+3`) أو يدوياً عبر زر "Run now"، ويسجل النتيجة في `OreemAutoWithdrawLog` ويرسل إشعاراً عبر `notify_oreem_wallet_withdrawal`.

### ج. تنبيهات انخفاض الرصيد (`main_app/utils/oreem_wallet_notifications.py`)
- تُستدعى `refresh_oreem_wallet_and_check_low_balance(db)` عند تحديث حالات الطلبات (في `order_updates.py`).
- تقارن الرصيد بـ `GeneralSettings.oreem_wallet_low_balance_threshold_bhd` وترسل تنبيهاً للإدارة مع تحديث `oreem_wallet_low_balance_last_notified_at` لمنع تكرار الإزعاج.

---

## 8. ربط الـ Routers في التطبيق (`main.py`)

يتم ربط جميع مسارات Oreem والشحن والتطبيق في `main.py` بالشكل التالي:

```python
# 1) Webhooks العامة (بدون توثيق مستخدم — تعتمد على HMAC/Token)
app.include_router(oreem_api.public_router, prefix="/api/v1/delivery")
app.include_router(oreem_api.public_router, prefix="/api/v1/shop-owner")

# 2) واجهات التوصيل والشحن
app.include_router(oreem_api.router, prefix="/api/v1/delivery")
app.include_router(shipping_api.router, dependencies=api_dependencies)          # prefix="/api/v1/shipping"
app.include_router(manual_delivery_api.router, dependencies=api_dependencies)   # prefix="/api/v1/shop-owner/manual-delivery"

# 3) لوحة تحكم الأدمن
app.include_router(admin_oreem_data.router)
app.include_router(admin_manual_delivery.router)
```

---

## 9. تفاصيل الـ Webhooks والأمان (Domestic & International)

### أ. Webhook التوصيل المحلي (`oreem_api.py`)
- **المسارات المتاحة:**
  - `GET /api/v1/delivery/webhook` (Health Check يرجع `{"status": "ok"}`)
  - `POST /api/v1/delivery/webhook` (والمسار البديل `/api/v1/shop-owner/webhook`)
- **التحقق الأمني:**
  1. **المسار المفضل (HMAC-SHA256):** عند توفر `DOMESTIC_OREEM_WEBHOOK_SECRET` وهيدر `X-Webhook-Signature` (مع إزالة بادئة `sha256=`)، يتم حساب HMAC-SHA256 على الـ raw body ومقارنته بـ `hmac.compare_digest`.
  2. **الحماية من إعادة الإرسال (Replay Protection):** فحص `X-Webhook-Timestamp` ورفض الطلب إذا تجاوز عمره `OREEM_WEBHOOK_MAX_AGE_SECONDS` (300 ثانية).
  3. **المسار الاحتياطي (Token Fallback):** إذا لم يُعدّ توقيع HMAC، يتحقق من `token` في Query Param أو `Authorization: Bearer <token>` ومطابقته مع `OREEM_WEBHOOK_TOKEN`.
- **توجيه التحديث حسب نوع الطلب (`local_order_id`):**
  - إذا بدأ بـ `GIG-` -> يحدّث طلب الخدمة (`GigOrder`).
  - إذا بدأ بـ `PHONE-` أو `RETURN-` -> يحدّث `ManualOreemDelivery` عبر `apply_manual_oreem_webhook_update` ويرسل تحديثاً فورياً عبر `broadcast_manual_delivery_update`.
  - الطلبات العادية -> يبحث عن `Order.order_number` (أو `Order.oreem_order_id` كخيار احتياطي)، ويحوّل الحالة عبر `normalize_oreem_status` و `map_oreem_status_to_order_status`.
- **صيغ الـ Payload المدعومة:** يدعم الصيغة الحديثة (`notification_type` + `data.order_id` + `data.customer_order_id` / `other_system_order_id` + `tracking_url`) والصيغة القديمة (`oreem_order_id`, `local_order_id`, `status`).

### ب. Webhook الشحن الدولي (`shipping_api.py`)
- `GET /api/v1/shipping/webhook` و `POST /api/v1/shipping/webhook`
- يتحقق باستخدام `SHIPPING_OREEM_WEBHOOK_SECRET` ويحدّث حالات الشحنات الدولية ويرسل إشعاراً للمستخدم عبر `create_user_notification`.

---

## 10. واجهات تطبيق Flutter للتاجر (`shop_owner_api.py` & `manual_delivery_api.py`)

### أ. إعدادات وطلب المندوب للتاجر (`/api/v1/shop-owner/*`)
| Method | Endpoint | الوظيفة |
|---|---|---|
| `GET` | `/api/v1/shop-owner/settings/pickup-mode` | جلب وضع الاستلام (`auto` أو `manual`) — محظور للمتاجر خارج البحرين |
| `POST` | `/api/v1/shop-owner/settings/pickup-mode?mode=auto\|manual` | تغيير وضع طلب المندوب بين التلقائي واليدوي |
| `POST` | `/api/v1/shop-owner/orders/{order_id}/trigger-delivery` | استدعاء مندوب Oreem يدوياً للطلب؛ يرجع `scheduled`, `oreem_order_id`, `tracking_url`, `delivery_method`, `delivery_status` |

> في استجابات تفاصيل الطلبات لتطبيق التاجر، يتم إرسال الحقلين `can_call_oreem` و `oreem_available_for_store` للتحكم في إظهار زر طلب المندوب.

### ب. التوصيل اليدوي في Flutter (`/api/v1/shop-owner/manual-delivery/*`)
| Method | Endpoint | الوظيفة |
|---|---|---|
| `GET` | `/api/v1/shop-owner/manual-delivery/bootstrap` | تهيئة الشاشة (الرصيد، العناوين، البلوكات والمدن، الإعدادات) |
| `GET` | `/api/v1/shop-owner/manual-delivery/pickup-addresses` | جلب عناوين استلام المتجر |
| `GET` | `/api/v1/shop-owner/manual-delivery/receivers` | جلب سجل المستلمين السابقين |
| `GET` | `/api/v1/shop-owner/manual-delivery/active-orders` | جلب الطلبات اليدوية النشطة |
| `POST` | `/api/v1/shop-owner/manual-delivery/quote` | تسعيرة التوصيل اليدوي مع التحقق من كفاية رصيد المحفظة |
| `POST` | `/api/v1/shop-owner/manual-delivery/create` | إنشاء طلب توصيل يدوي وخصم التكلفة من محفظة المتجر |

---

## 11. مرجع جميع نقاط النهاية (API & Admin Endpoints)

### أ. واجهات التوصيل المحلي (`/api/v1/delivery/*` — `oreem_api.py`)
| Method | Endpoint | الوظيفة |
|---|---|---|
| `GET` | `/api/v1/delivery/blocks` | جلب البلوكات/المجمعات |
| `GET` | `/api/v1/delivery/cities` | جلب المدن |
| `POST` | `/api/v1/delivery/sync-blocks-cities` | مزامنة البلوكات والمدن من Oreem وحفظها محلياً |
| `GET` | `/api/v1/delivery/bahrain/blocks` | جلب بلوكات البحرين من قاعدة البيانات المحلية |
| `GET` | `/api/v1/delivery/bahrain/cities` | جلب مدن البحرين من قاعدة البيانات المحلية |
| `POST` | `/api/v1/delivery/price` | حساب تسعيرة التوصيل (مع إضافة `delivery_internal_profit` أو `fallback`) |
| `POST` | `/api/v1/delivery/create` | إنشاء طلب توصيل في Oreem |
| `GET` | `/api/v1/delivery/track/{oreem_order_id}` | تتبع حالة الشحنة عبر معرف Oreem |
| `POST` | `/api/v1/delivery/cancel/{oreem_order_id}` | إلغاء طلب توصيل في Oreem |

### ب. واجهات الشحن الدولي (`/api/v1/shipping/*` — `shipping_api.py`)
| Method | Endpoint | الوظيفة |
|---|---|---|
| `GET` | `/api/v1/shipping/track/{shipment_id}` | تتبع شحنة دولية |
| `GET` | `/api/v1/shipping/details/{shipment_id}` | تفاصيل الشحنة الدولية |
| `GET` | `/api/v1/shipping/countries` | قائمة دول المنشأ المعتمدة |
| `GET` | `/api/v1/shipping/hscodes` | رموز التعرفة الجمركية `HS Codes` |

### ج. لوحة تحكم الأدمن — بيانات Oreem، التوصيل اليدوي، والمحفظة
| Method | Endpoint | الوظيفة |
|---|---|---|
| `GET` | `/admin/oreem-data` | عرض وإدارة البلوكات والمدن (`/blocks/add`, `/blocks/{id}/edit`, `/cities/add`, `/cities/{id}/edit`, `/cities/sync-counts`) |
| `GET` / `POST` | `/admin/manual-delivery/*` | إدارة التوصيل اليدوي، التصدير CSV، البحث عن التجار، التسعير والإنشاء (`/export`, `/pickup-addresses`, `/receivers`, `/search-users`, `/user-receivers`, `/active-orders`, `/quote`, `/create`) |
| `GET` / `POST` | `/admin/wallet` & `/admin/wallet/oreem-auto-withdraw*` | عرض محفظة Oreem، السحب اليدوي، حفظ إعدادات السحب التلقائي، والتشغيل الفوري (`/run`) |

---

## 12. ملخص تدفق العمل الكامل (End-to-End Flow)

1. **في السلة والـ Checkout:** يستعلم الواجهة عن `POST /price`، فتعود التسعيرة شاملة `delivery_internal_profit` (أو التسعيرة الاحتياطية عند تعطل API).
2. **بعد الدفع أو عند اختيار COD:** يُستدعى `trigger_oreem_delivery(db, order)` مع قفل الصف `with_for_update()`.
   - إذا كان الوقت بين `00:00:00` و `08:04:59` بتوقيت البحرين، يُحفظ الطلب بحالة `"scheduled"` ليُرسل تلقائياً الساعة `08:05 AM`.
   - وإلا يتم الإرسال مباشرة إلى Oreem وتُحفظ `oreem_order_id`.
3. **أثناء التوصيل:** يرسل Oreem تحديثاته عبر Webhook المحمي بالتوقيع والتوكن، وتتحدث حالة الطلب وتفحص `refresh_oreem_wallet_and_check_low_balance(db)` رصيد المحفظة.
4. **إدارة المحفظة:** يراقب الأدمن الرصيد المتاح في الداشبورد ويسحب يدوياً أو يعتمد على السحب التلقائي اليومي الذي يترك مبلغ `keep` لتغطية الطلبات القادمة.
