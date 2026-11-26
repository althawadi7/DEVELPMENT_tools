# MyFatoorah Full-Stack Integration Guide (Backend + Security + Apple Pay + Save Card + Checkout UI + Flutter APIs)

This is the complete blueprint to build the **MyFatoorah v2/v3** payment integration from scratch, including **Embedded Web Checkout UI**, **Apple Pay**, **Save Card / Tokenization**, **Security Hardening**, and **Flutter Native SDK APIs**.

---

## 1. Architecture & Environment Configuration

### Separation of Keys (CRITICAL Security Rule)
Never send the server secret key to the browser or Flutter app:
- **`MYFATOORAH_API_KEY`**: Server-only Secret Key (used for `/v2/*`, `/v3/*`, webhooks, verification, charging tokens). **NEVER exposed to clients.**
- **`MYFATOORAH_APP_API_KEY`**: Restricted Client/App Key (Create Payments permission only) returned to Flutter via `GET /api/v1/payment/config` for `MFSDK.init(...)`.

### Environment Variables (`project/core/config.py`)
```python
MYFATOORAH_PORTAL_URL: str = config("MYFATOORAH_PORTAL_URL", default="https://demo.myfatoorah.com/")
MYFATOORAH_API_URL: str = config("MYFATOORAH_API_URL", default="https://apitest.myfatoorah.com/")
MYFATOORAH_API_KEY: str = config("MYFATOORAH_API_KEY", default="")
MYFATOORAH_APP_API_KEY: str = config("MYFATOORAH_APP_API_KEY", default="")
MYFATOORAH_SDK_ENV: str = config("MYFATOORAH_SDK_ENV", default="TEST")  # TEST | LIVE
MYFATOORAH_GOOGLE_MERCHANT_ID: str = config("MYFATOORAH_GOOGLE_MERCHANT_ID", default="")
MYFATOORAH_ENABLED: bool = config("MYFATOORAH_ENABLED", default=True, cast=bool)
MYFATOORAH_WEBHOOK_SECRET: str = config("MYFATOORAH_WEBHOOK_SECRET", default="")
MYFATOORAH_COUNTRY_ISO: str = config("MYFATOORAH_COUNTRY_ISO", default="BHR")
MYFATOORAH_CURRENCY: str = config("MYFATOORAH_CURRENCY", default="BHD")
MYFATOORAH_LANGUAGE: str = config("MYFATOORAH_LANGUAGE", default="EN")
MYFATOORAH_TIMEOUT: int = config("MYFATOORAH_TIMEOUT", default=40, cast=int)
MYFATOORAH_METHODS_CACHE_TTL: int = config("MYFATOORAH_METHODS_CACHE_TTL", default=3600, cast=int)
MYFATOORAH_TOKENIZATION_ENABLED: bool = config("MYFATOORAH_TOKENIZATION_ENABLED", default=False, cast=bool)
MYFATOORAH_BYPASS_3DS: bool = config("MYFATOORAH_BYPASS_3DS", default=False, cast=bool)
MYFATOORAH_BYPASS_CVV: bool = config("MYFATOORAH_BYPASS_CVV", default=False, cast=bool)
```

> **Live vs Test URLs:**
> - Live API: `https://api.myfatoorah.com/` (or regional `.sa` / `.ae` / `.qa` if applicable) | Portal: `https://portal.myfatoorah.com/`
> - Test API: `https://apitest.myfatoorah.com/` | Portal: `https://demo.myfatoorah.com/`
> - Embedded JS URL: `{MYFATOORAH_PORTAL_URL}/sessions/v1/session.js`

---

## 2. Database Models & Admin Controls

### A. `GeneralSettings` (Admin Gateway & Surface Toggles)
Store runtime toggles in DB so admins can switch gateways or surfaces without redeploying:
```python
myfatoorah_enabled = Column(Boolean, default=False, nullable=False)
myfatoorah_surface = Column(String(16), default="both", nullable=False)  # "both" | "browser" | "app"
myfatoorah_save_card_enabled = Column(Boolean, default=False, nullable=False)
myfatoorah_fastpay_enabled = Column(Boolean, default=False, nullable=False)
myfatoorah_bypass_cvv_enabled = Column(Boolean, default=False, nullable=False)
myfatoorah_enabled_methods = Column(JSON, nullable=True)  # e.g. ["CARD", "BENEFIT", "APPLEPAY"] or null = all live
```

### B. `Order` & `PaymentIntent`
- `Order.payment_method`: `"cod" | "myfatoorah" | "myfatoorah:CARD" | "myfatoorah:BENEFIT" | "myfatoorah:APPLEPAY" | "loyalty_wallet"`
- `Order.payment_gateway`: `"myfatoorah" | "fatora" | None`
- `Order.payment_status`: `"pending" | "paid" | "failed"`
- `Order.gateway_reference` / `fatora_transaction_id`: stores `PaymentId` / `InvoiceId` idempotently.
- `PaymentIntent`: stores `intent_id`, `user_id`, `amount`, `currency`, `status` (`pending` / `completed`), and `payload` JSON (including `{"save_card": true/false}` consent).

### C. `UserSavedCard` (Tokenized Cards)
Stores: `id`, `user_id`, `gateway` (`"myfatoorah"`), `token` (MyFatoorah card token), `brand` (`VISA` / `MASTERCARD` / `MADA`), `last4`, `expiry_month`, `expiry_year`, `is_default`, `is_active`, `created_at`.

---

## 3. Core Backend Service (`main_app/utils/myfatoorah_integration.py`)

### Key Functions to Implement:
1. **`create_session(amount, currency, customer_reference, save_token=False, ...)`**
   - Calls `POST /v3/sessions` (Unified Embedded Session for Card, Apple Pay, Benefit, Google Pay).
   - If `save_token` is `True` AND `feature_flags(db)["save_card"]` is `True`:
     - Sends `Customer.Reference = f"HIRFA-U{user_id}"`
     - Sends `SaveCardOptions: {"SaveToken": True}`
   - Otherwise explicitly sends `SaveCardOptions: {"SaveToken": False}`.
   - Returns `{ "session_id", "country_code", "encryption_key", ... }`.
2. **`create_payment(order_number, amount, currency, method_code, customer_name, customer_email, customer_phone, callback_url, error_url, save_token=False)`**
   - Calls `POST /v2/ExecutePayment` (or `/v3/payments`) with `PaymentMethodId` / `SessionId`.
   - Returns `{ "invoice_id", "payment_url", "is_direct_payment", ... }`.
3. **`get_payment_details(payment_id)` / `get_invoice_details(invoice_id)`**
   - Calls `POST /v2/GetPaymentStatus` (`{"Key": payment_id, "KeyType": "PaymentId"}` or `"InvoiceId"`).
   - **Never trust client-reported success** — always verify `InvoiceStatus == "Paid"` and `TransactionStatus == "Succss"` server-side via `is_paid_payment_details(details)`.
4. **`get_payment_methods(force=False)`**
   - Calls `POST /v2/InitiatePayment` (`InvoiceAmount`, `CurrencyIso`) and caches normalized methods in memory/Redis for `MYFATOORAH_METHODS_CACHE_TTL` seconds.
   - Normalizes each method to: `code` / `api_name` (`CARD`, `BENEFIT`, `APPLEPAY`, `GOOGLEPAY`, `STC_PAY`), `title_en`, `title_ar`, `image_url`, `is_embedded`, `session_name` (`"card"` | `"applePay"` | `"benefit"`).
   - Filters by country + `myfatoorah_enabled_methods` admin allowlist (`filter_methods_by_admin`).
5. **`save_user_card_from_details(db, user_id, details, opted_in=False)`**
   - Only saves if **both** `feature_flags(db)["save_card"]` is `True` **and** `opted_in` is `True` (explicit user consent stored on `PaymentIntent`).
   - Extracts `Token`, `CardInfo` (`Brand`, `Number` last 4, `ExpiryMonth`, `ExpiryYear`) from `GetPaymentStatus` response and upserts into `UserSavedCard`.
6. **`charge_saved_token(token, amount, currency, order_number, ...)` & `cancel_token(token)`**
   - Charges a saved token only when `saved_card_payment` (`save_card && bypass_cvv`) is approved and enabled.
   - Calls MyFatoorah `CancelToken` when user deletes a saved card.

---

## 4. Capability & Feature Flags (`mf_capabilities.py`)

Because MyFatoorah accounts require merchant-level approval for **Tokenization (Save Card)**, **FastPay**, **Bypass 3DS**, and **Bypass CVV**:
- Store verified account approvals in `config/myfatoorah_capabilities.json` keyed by `sha256(MYFATOORAH_API_KEY)` (`account_fingerprint`) so switching keys/environments never leaks approvals.
- **Fail closed**: if not explicitly approved, `effective_flags()` forces `save_card = False`, `fastpay = False`, `saved_card_payment = False`.
- **`observe_rejection(response)`**: Inspects live MyFatoorah error responses; if MyFatoorah returns `"not enabled"` / `"not activated"` for tokenization/fastpay/bypass3ds/bypasscvv, it automatically revokes that capability in `myfatoorah_capabilities.json` so the UI immediately hides unsupported toggles!

---

## 5. Security Architecture (Must-Have Checks)

1. **HMAC-SHA256 Webhook Signature Verification (`verify_webhook_signature`)**:
   - Read raw request body (`await request.body()`) and `MyFatoorah-Signature` header.
   - Build canonical signature string from payload fields (sorted keys: `Data.*` depending on event type), compute HMAC-SHA256 with `MYFATOORAH_WEBHOOK_SECRET` (base64 encoded), and compare using `hmac.compare_digest()`.
   - Reject (`401`) any invalid/missing signature.
2. **Server-Side Ownership & Amount Validation (`mf_checkout_context.py`)**:
   - `owned_context(db, current_user.id, order_number)` verifies that the `Order`, `PaymentIntent`, or `PaymentLink` belongs to `current_user.id`, is not cancelled/already paid, and has a finite positive `Decimal` amount.
   - During `complete_myfatoorah_payment` and `myfatoorah_webhook`, compare `paid_amount(details)` against `_expected_amount(db, order_number)` (with `0.005` BHD tolerance) before marking the order paid.
3. **Encrypted Session Payload Decryption (`decrypt_payment_data`)**:
   - Supports AES/CBC decryption of MyFatoorah encrypted callback payloads when enabled.
4. **Idempotent Order Finalisation (`_finalise_order_paid`)**:
   - Locks/checks `order.payment_status == "paid"` before crediting shop wallets, clearing cart items, deducting stock, or sending notifications — preventing double-crediting if both `/return` (redirect), `/complete` (app/JS), and `/webhook` fire simultaneously.

---

## 6. Complete Backend & Flutter API Reference

Base prefix: `/api/v1/payment` (plus `/payment/myfatoorah/features` for lightweight web polling).

### 1) `GET /api/v1/payment/config?client=app|browser`
Returns gateway visibility, feature flags, allowed methods, and **restricted Flutter SDK init config**:
```json
{
  "gateway": "myfatoorah",
  "fatora_enabled": false,
  "myfatoorah_enabled": true,
  "myfatoorah_visible_browser": true,
  "myfatoorah_visible_app": true,
  "myfatoorah_save_card_enabled": false,
  "myfatoorah_fastpay_enabled": false,
  "myfatoorah_features": {
    "saved_card_payment": false,
    "save_card": false,
    "fastpay": false,
    "bypass_3ds": false,
    "bypass_cvv": false
  },
  "myfatoorah_enabled_methods": ["CARD", "BENEFIT", "APPLEPAY"],
  "currency": "BHD",
  "country": "BHR",
  "sdk": {
    "api_key": "<MYFATOORAH_APP_API_KEY>",
    "configured": true,
    "environment": "LIVE",
    "country_iso": "BHR",
    "currency_iso": "BHD",
    "google_merchant_id": null
  }
}
```
> **Flutter startup:** Call `GET /api/v1/payment/config?client=app`, then `MFSDK.init(sdk.api_key, MFCountry.BAHRAIN, sdk.environment == 'LIVE' ? MFEnvironment.LIVE : MFEnvironment.TEST)`.

### 2) `GET /api/v1/payment/methods`
Returns filtered live payment methods (`CARD`, `BENEFIT`, `APPLEPAY`, etc.) with localized titles, icons, and `session_name`.

### 3) `GET /api/v1/payment/myfatoorah/checkout-context?order_number=...`
Auth required. Returns `{ "order_number", "amount", "currency", "paid" }` for the authenticated owner.

### 4) `POST /api/v1/payment/myfatoorah/initiate`
Auth required. Body:
```json
{
  "order_number": "ORD-...",
  "method": "CARD",        // CARD | BENEFIT | APPLEPAY | GOOGLEPAY
  "embedded": true,        // true for in-page web session.js; false for redirect/app
  "save_card": false
}
```
Returns:
- When `embedded: true`: `{ "session_id", "country_code", "amount", "currency", "session_name", "method" }`
- When `embedded: false`: `{ "payment_url", "invoice_id", "method" }`

### 5) `POST /api/v1/payment/myfatoorah/complete`
Auth required. Called by Web embedded callback or Flutter SDK after payment finishes.
Body: `{ "order_number": "ORD-...", "payment_id": "...", "session_id": "..." }`
Verifies payment status + amount server-side via `GetPaymentStatus`, saves card if `save_card` consent was set, marks order `paid`, clears cart, and returns `{ "success": true, "paid": true, "order_number": "...", "redirect_url": "/order/..." }`.

### 6) `GET /api/v1/payment/myfatoorah/status?order_number=...`
Auth required. Polls server-side order payment state (`paid`, `pending`, `failed`).

### 7) `POST /api/v1/payment/myfatoorah/card-consent`
Auth required. Stores explicit user opt-in `{ "order_number": "...", "save_card": true }` on the `PaymentIntent` before executing payment.

### 8) `GET /api/v1/payment/cards` & `DELETE /api/v1/payment/cards/{card_id}` & `POST /api/v1/payment/myfatoorah/saved-card/charge`
- List user's active saved cards (`brand`, `last4`, `expiry_month`, `expiry_year`).
- Delete saved card (cancels token at MyFatoorah + deactivates locally).
- Charge a saved card directly when `saved_card_payment` is enabled.

### 9) `GET /api/v1/payment/myfatoorah/return` & `POST /api/v1/payment/myfatoorah/webhook`
- `/return`: Handles 3DS / Benefit browser redirect (`?paymentId=...`), verifies with `GetPaymentStatus`, finalises order, and redirects to `/order/{order_number}` (or `/checkout` on failure).
- `/webhook`: Background server-to-server notification, verifies HMAC-SHA256 signature, and idempotently finalises order.

---

## 7. Checkout UI Design & Embedded Flow (`templates/shop/checkout.html`)

### Visual Design System (Payment Section `#payment-method-section.pay-details-shell`)
- **Tile Grid (`#payment-method-options.pay-method-grid`)**:
  - `grid-template-columns: repeat(2, minmax(0, 1fr))` on mobile/tablet, expanding to `repeat(4, minmax(0, 1fr))` on desktop (`>= 900px`).
  - Each `.payment-option` tile: `min-height: 5.25rem`, `padding: 0.95rem 1rem`, `border-radius: 0.85rem`, `border: 1.5px solid hsl(var(--border))`, `background: hsl(var(--muted))`.
  - Selected tile (`.is-selected` / `:has(input:checked)`): `border-width: 2px; border-color: hsl(var(--primary)); background: hsl(var(--primary) / 0.1); box-shadow: 0 0 0 1px hsl(var(--primary) / 0.2)`.
  - Inside each tile: `.pay-opt-icon` (`2.25rem × 2.25rem`, `border-radius: 0.55rem`, card background) + `.pay-opt-title` (`font-weight: 700; font-size: 0.88rem`).
- **Saved Cards Row (`#mf-saved-cards`)**:
  - Shown above the method tiles only when `MF_SAVED_CARD_PAYMENT_ENABLED` is `true` and user has saved cards (`Brand •••• 4242` + Delete button).
- **Detail Pane (`#pay-method-detail`)**:
  - **`#mf-embedded-panel`**: `border-radius: 0.85rem; border: 1.5px solid hsl(var(--border)); background: hsl(var(--muted)); padding: 1rem 1.05rem`.
  - **`#mf-unified-session`**: Hosts MyFatoorah's `session.js` iframe (`background: hsl(var(--card)); border-radius: 0.65rem; padding: 0.45rem; overflow: hidden`).
  - **`#mf-save-card-wrap`**: Checkbox *"Save this card for next time"* (`hidden` unless `MF_SAVE_CARD_ENABLED === true`).
- **Fixed Bottom Pay Bar (`#co-float-pay`)**:
  - Pinned at bottom across desktop & mobile (on phone Step 2) with frosted backdrop (`backdrop-filter: blur(10px)`), showing **Total** on one side and **Confirm Order** pill button on the other.

### Apple Pay & Embedded Session Flow (Web)
1. **Apple Pay Availability**:
   - Check `window.ApplePaySession && ApplePaySession.canMakePayments()` on Safari/iOS/macOS.
   - Host the Apple Pay Domain Association file at `/.well-known/apple-developer-merchantid-domain-association` on your domain and register the domain in MyFatoorah Portal.
   - On non-Apple browsers, either hide the Apple Pay tile or show an Apple Pay QR/redirect prompt (`#mf-applepay-modal`).
2. **Order Submission (`placeOrder()`)**:
   - User clicks **Confirm Order** → `POST /api/v1/orders` creates the pending order (`order_number`).
   - Frontend calls `POST /api/v1/payment/myfatoorah/initiate` with `embedded: true`.
   - Load `MF_SESSION_JS_URL` (`session.js`) dynamically, mount `myfatoorah.init({ sessionId, countryCode, containerId: "mf-unified-session", callback: onMfPaymentCallback, ... })`.
   - Save pending order in `localStorage` (`hirfa_mf_pending_payment`) and URL hash `#mf-pay=ORD-...` so if 3DS redirects/reloads the page, `resumePendingMyFatoorah()` recovers cleanly.
   - When `onMfPaymentCallback(response)` fires with `paymentId` / `sessionId`, call `POST /api/v1/payment/myfatoorah/complete` → redirect to `/order/{order_number}`!

---

## 8. Flutter App Integration Flow

1. **App Startup:** `GET /api/v1/payment/config?client=app` → initialize `MFSDK.init(sdk.api_key, country, env)`.
2. **Checkout Screen:** `GET /api/v1/payment/methods` → display **Card**, **Benefit**, **Apple Pay** (iOS only), **Google Pay** (Android), **Saved Cards** (if `myfatoorah_features.saved_card_payment`), and **COD**.
3. **Place Order:** `POST /api/v1/orders` → returns `order_number` and `total_amount`.
4. **Initiate Payment:**
   - Option A (Native SDK): Pass `order_number` + `total_amount` to `MFSDK.executePayment(...)` (or `MFApplePay` / `MFCardPaymentView`) with `CustomerReference = "HIRFA-U{user_id}"`.
   - Option B (Hosted Fallback): `POST /api/v1/payment/myfatoorah/initiate` with `embedded: false` → open `payment_url` in WebView and intercept `/return`.
   - If `save_card` checkbox is checked, call `POST /api/v1/payment/myfatoorah/card-consent` first.
5. **Verify & Complete:** On SDK callback success, immediately call `POST /api/v1/payment/myfatoorah/complete` with `{ order_number, payment_id }` (or poll `GET /api/v1/payment/myfatoorah/status`), then navigate to the Order Confirmation screen.
