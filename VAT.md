# NBR Bahrain VAT Threshold Tracker & Alert — Full Implementation Guide

دليل تقني متكامل لتنفيذ نظام تتبع حد التسجيل الإلزامي لضريبة القيمة المضافة في البحرين (**BHD 37,500** خلال آخر 12 شهراً)، مع شريط المتابعة والـ Popup التنبيهي في لوحة التحكم (`/admin/dashboard`) وصفحة الأرباح (`/admin/profits`).

---

## 1. القاعدة المعتمدة والروابط الرسمية (NBR Bahrain)

- **الحد الإلزامي للتسجيل:** `37,500.000 BHD` من إجمالي التوريدات/المبيعات (**Platform Volume / GMV** وليس صافي الربح) خلال أي فترة **12 شهراً متواصلة** (آخر 365 يوماً).
- **حد التنبيه المبكر (قبل الوصول بـ 1,000 د.ب):** `36,500.000 BHD` (`37,500 - 1,000`).
- **إيقاف التنبيه بعد التسجيل:** عند تفعيل `GeneralSettings.vat_enabled = True` (من صفحة `/admin/wallet` -> `Enable VAT Deduction`).
- **الروابط الرسمية:**
  - دليل التسجيل في الضريبة (**VAT Registration Guide**): `https://www.nbr.gov.bh/publications/view/VAT_Registration_guide`
  - بوابة التسجيل (**VAT Registration - NBR eServices**): `https://eservices.nbr.gov.bh/sap/bc/ui5_ui5/sap/z_mcf_fmca/index.html#/Logon`

---

## 2. منطق الحساب في الباك إند (`main_app/utils/vat_threshold.py`)

يتم حساب إجمالي التوريدات خلال آخر 365 يوماً (`cutoff = datetime.utcnow() - timedelta(days=365)`) من 3 مصادر:

1. **طلبات المنتجات (`Order`):**
   - `func.sum(Order.total_amount)` حيث `Order.status.in_(["pending", "processing", "out_for_delivery", "delivered"])` و `Order.created_at >= cutoff`.
2. **طلبات الخدمات المدفوعة (`GigOrder`):**
   - `func.sum(GigOrder.agreed_price)` حيث `GigOrder.payment_status == "paid"` و `GigOrder.created_at >= cutoff`.
3. **تسجيلات الدورات المدفوعة (`CourseRegistration`):**
   - `func.sum(CourseRegistration.amount_paid)` حيث `CourseRegistration.payment_status == "paid"` و `CourseRegistration.created_at >= cutoff`.

### الكود الكامل (`main_app/utils/vat_threshold.py`)

```python
from __future__ import annotations

from datetime import datetime, timedelta
from typing import Any, Dict

from sqlalchemy import func
from sqlalchemy.orm import Session

from main_app.models import CourseRegistration, GeneralSettings, GigOrder, Order

VAT_MANDATORY_THRESHOLD_BHD = 37500.0
VAT_WARNING_BUFFER_BHD = 1000.0
VAT_ALERT_TRIGGER_BHD = VAT_MANDATORY_THRESHOLD_BHD - VAT_WARNING_BUFFER_BHD  # 36,500.0

NBR_VAT_GUIDE_URL = "https://www.nbr.gov.bh/publications/view/VAT_Registration_guide"
NBR_VAT_REGISTER_URL = "https://eservices.nbr.gov.bh/sap/bc/ui5_ui5/sap/z_mcf_fmca/index.html#/Logon"


def get_vat_threshold_status(db: Session) -> Dict[str, Any]:
    """Return rolling 12-month GMV vs BHD 37,500 VAT registration threshold status."""
    cutoff = datetime.utcnow() - timedelta(days=365)

    product_gmv_12m = (
        db.query(func.coalesce(func.sum(Order.total_amount), 0))
        .filter(
            Order.status.in_(["pending", "processing", "out_for_delivery", "delivered"]),
            Order.created_at >= cutoff,
        )
        .scalar()
        or 0.0
    )
    gig_gmv_12m = (
        db.query(func.coalesce(func.sum(GigOrder.agreed_price), 0))
        .filter(
            GigOrder.payment_status == "paid",
            GigOrder.created_at >= cutoff,
        )
        .scalar()
        or 0.0
    )
    course_gmv_12m = (
        db.query(func.coalesce(func.sum(CourseRegistration.amount_paid), 0))
        .filter(
            CourseRegistration.payment_status == "paid",
            CourseRegistration.created_at >= cutoff,
        )
        .scalar()
        or 0.0
    )

    rolling_12m_gmv = round(
        float(product_gmv_12m) + float(gig_gmv_12m) + float(course_gmv_12m), 3
    )

    settings_row = db.query(GeneralSettings).first()
    vat_enabled = bool(getattr(settings_row, "vat_enabled", False)) if settings_row else False

    remaining_bhd = max(0.0, round(VAT_MANDATORY_THRESHOLD_BHD - rolling_12m_gmv, 3))
    excess_bhd = max(0.0, round(rolling_12m_gmv - VAT_MANDATORY_THRESHOLD_BHD, 3))
    progress_pct = min(100.0, round((rolling_12m_gmv / VAT_MANDATORY_THRESHOLD_BHD) * 100.0, 1))

    if vat_enabled:
        status = "registered"
    elif rolling_12m_gmv >= VAT_MANDATORY_THRESHOLD_BHD:
        status = "exceeded"
    elif rolling_12m_gmv >= VAT_ALERT_TRIGGER_BHD:
        status = "approaching"
    else:
        status = "normal"

    should_show_popup = (not vat_enabled) and (rolling_12m_gmv >= VAT_ALERT_TRIGGER_BHD)

    return {
        "rolling_12m_gmv": rolling_12m_gmv,
        "product_gmv_12m": round(float(product_gmv_12m), 3),
        "gig_gmv_12m": round(float(gig_gmv_12m), 3),
        "course_gmv_12m": round(float(course_gmv_12m), 3),
        "threshold_bhd": VAT_MANDATORY_THRESHOLD_BHD,
        "warning_trigger_bhd": VAT_ALERT_TRIGGER_BHD,
        "warning_buffer_bhd": VAT_WARNING_BUFFER_BHD,
        "remaining_bhd": remaining_bhd,
        "excess_bhd": excess_bhd,
        "progress_pct": progress_pct,
        "vat_enabled": vat_enabled,
        "status": status,
        "should_show_popup": should_show_popup,
        "guide_url": NBR_VAT_GUIDE_URL,
        "register_url": NBR_VAT_REGISTER_URL,
    }
```

### حالات المؤشر (`status`)
- **`normal`:** أقل من `36,500 BHD` و `vat_enabled == False` (شريط أزرق `bg-primary`).
- **`approaching`:** بين `36,500` و `37,499.999 BHD` و `vat_enabled == False` (شريط أصفر متحرك `bg-warning` + تفعيل الـ Popup).
- **`exceeded`:** `>= 37,500 BHD` و `vat_enabled == False` (شريط أحمر متحرك `bg-danger` + تفعيل الـ Popup).
- **`registered`:** عند تفعيل `vat_enabled == True` (شريط أخضر `bg-success` + إيقاف الـ Popup).

---

## 3. واجهة البطاقة والـ Popup (`templates/admin_panel/partials/vat_threshold_tracker.html`)

### أ. بطاقة المتابعة (`#nbr-vat-tracker-card`)
- تعرض شريط التقدم (`progress-bar`) بنسبة `progress_pct%`، والمبلغ الحالي مقابل `37,500.000 BHD`، والمبلغ المتبقي (`remaining_bhd`) أو الزيادة (`excess_bhd`).
- تحتوي أزراراً سريعة لـ:
  - **VAT Registration Guide** (يفتح دليل NBR في تبويب جديد)
  - **VAT Registration (NBR)** (يفتح بوابة التسجيل في تبويب جديد)
  - **VAT Settings** (`/admin/wallet`)
  - زر **VAT Registration Notice** لإعادة فتح الـ Popup يدوياً عند الحاجة.

### ب. نافذة التنبيه المنبثقة (`#vatRegistrationAlertModal`)
- تظهر تلقائياً فقط عندما `vat_tracker.should_show_popup == True` (أي `rolling_12m_gmv >= 36,500.0` و `vat_enabled == False`).
- تعرض النص الرسمي لـ NBR وشريط التقدم، و3 أزرار رئيسية:
  1. `Register for VAT (NBR)`
  2. `VAT Registration Guide`
  3. `Enable VAT Deduction` (`/admin/wallet`)
- تستخدم `sessionStorage.getItem('nbr_vat_alert_dismissed')` حتى لا تزعج الأدمن في كل تنقل بعد إغلاقها خلال نفس الجلسة، وتعود للظهور في الجلسة الجديدة طالما لم يتم تفعيل الضريبة.

---

## 4. الربط في الصفحات (Wiring)

1. **في `/admin/dashboard`:**
   - في `main_app/utils/dashboard_stats.py` داخل `build_dashboard_context`:
     ```python
     from main_app.utils.vat_threshold import get_vat_threshold_status
     vat_tracker = get_vat_threshold_status(db) if not shop_owner_mode else None
     ```
     وتمرير `"vat_tracker": vat_tracker` في القاموس المُرجع.
   - في `templates/admin_panel/dashboard_stats_fragment.html` (أسفل `#platform-volume-row` للأدمن فقط):
     ```jinja2
     {% if not shop_owner_mode and vat_tracker %}
       {% include "admin_panel/partials/vat_threshold_tracker.html" %}
     {% endif %}
     ```

2. **في `/admin/profits`:**
   - في `main_app/routers/admin/admin_profits.py`:
     ```python
     from main_app.utils.vat_threshold import get_vat_threshold_status
     vat_tracker = get_vat_threshold_status(db)
     ```
     وتمرير `"vat_tracker": vat_tracker` إلى `platform_profits.html`.
   - في `templates/admin_panel/platform_profits.html` (في أعلى الصفحة مباشرة بعد شريط الفلتر وقبل بطاقات النمو):
     ```jinja2
     {% if vat_tracker %}
       {% include "admin_panel/partials/vat_threshold_tracker.html" %}
     {% endif %}
     ```
