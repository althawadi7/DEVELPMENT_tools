# MyFatoorah In-App Payment Guide (Flutter)

Complete architecture, SDK flow, and UI design for **In-App Card Payment**, **Native In-App Apple Pay (no URL)**, and **Benefit Slide-Up Bottom Sheet**, matching Hirfa's exact implementation.

---

## 1. Package & Initialization (`pubspec.yaml` + `MyFatoorahPaymentService`)

```yaml
dependencies:
  myfatoorah_flutter: 3.2.1
  webview_flutter: ^4.4.2
```

Initialize the SDK dynamically from your backend `/payment/config` (never hardcode keys):

```dart
await MFSDK.init(
  apiKey,
  MFCountry.BAHRAIN, // Match your API key country (e.g. BHR, KWT, SAU)
  isLive ? MFEnvironment.LIVE : MFEnvironment.TEST,
);

await MFSDK.setUpActionBar(
  toolBarTitle: 'YourApp',
  toolBarTitleColor: '#FFFFFF',
  toolBarBackgroundColor: '#A86A42', // Brand accent hex
  isShowToolBar: true,
);
```

---

## 2. Payment Option Tile & PCI Footer Design

Each payment option (Apple Pay, Benefit, Visa/Mastercard, Cash) uses a clean bordered card tile (`lib/view/home/cart_screens/widgets/checkout_payment_option_tile.dart`):

- **Border radius:** `12.r`
- **Selected state:** `1.8px` border in `accentColor`, background `accentColor.withValues(alpha: 0.08)`
- **Unselected state:** `1.2px` border in `borderColor`, background `cardColor`
- **Important:** Do **not** wrap `MFCardPaymentView` in `ClipRRect` / `ClipPath` — Android/iOS `PlatformView`s can render blank if clipped.
- **PCI Security Strip (`CheckoutPciFooter`):**
  - Background: `#F0F4F8` (Light) / `Colors.white.withValues(alpha: 0.06)` (Dark)
  - Border: `#D6E2F0` (Light) / `Colors.white.withValues(alpha: 0.12)` (Dark), radius `12.r`
  - Blue shield icon `#2F6FED` (`Icons.verified_user`) + `"Protected by PCI Data Security Standard"` + `Icons.info_outline`
  - Tapping opens a slide-up **Liquid Glass** modal (`showPciSecuritySheet`) explaining SSL/TLS encryption, no CVV storage, and PCI DSS Level 1 compliance.

---

## 3. In-App Card Payment (`CardPaymentSheet` + `MFCardPaymentView`)

Instead of opening a browser URL, cards are entered natively inside a slide-up bottom sheet (`showModalBottomSheet`):

### Sheet Chrome Design
- `isScrollControlled: true`, `enableDrag: false`, `backgroundColor: Colors.transparent`, `barrierColor: Colors.black.withValues(alpha: 0.55)`
- Container: `borderRadius: BorderRadius.vertical(top: Radius.circular(32.r))`, `color: AppColors.cardColor`
- Top pull handle (`44.w x 4.h`, rounded pill) + Order total badge + `MFCardPaymentView` + "Save card" toggle + Pay CTA button.

### Exact `MFCardViewStyle` (Dark-Mode Safe)
Because native MyFatoorah card fields keep a light fill, force dark ink inside `MFCardViewStyle` so dark mode never renders white text on a white input:

```dart
MFCardViewStyle _style(bool isArabic) {
  final s = MFCardViewStyle();
  s.cardHeight = 210;
  s.hideCardIcons = false;
  s.direction = isArabic ? 'rtl' : 'ltr';
  s.backgroundColor = getColorHexFromStr('#FFFFFF');
  s.label = MFCardViewLabel()
    ..display = true
    ..color = getColorHexFromStr('#333333')
    ..fontSize = 12
    ..fontWeight = MFFontWeight.Medium;

  final placeholders = MFCardViewPlaceHolder()
    ..holderName = isArabic ? 'الاسم على البطاقة' : 'Name on card'
    ..cardNumber = isArabic ? 'رقم البطاقة' : 'Card number'
    ..expiryDate = isArabic ? 'شهر / سنة' : 'MM / YY'
    ..securityCode = 'CVV';

  s.input = MFCardViewInput()
    ..inputMargin = 4
    ..inputHeight = 38
    ..borderRadius = 10
    ..borderColor = getColorHexFromStr('#C7C7C7')
    ..borderWidth = 1
    ..color = getColorHexFromStr('#1C1C1C')
    ..fontSize = 14
    ..placeHolder = placeholders;
  return s;
}
```

### Flow
1. `final session = await MFSDK.initSession(MFInitiateSessionRequest(customerIdentifier: customerRef), lang);`
2. `await cardView.load(session, (bin) {});`
3. On Pay tap: `final status = await cardView.pay(executePaymentRequest, lang, (invoiceId) {});`
4. Verify `status.invoiceStatus == 'Paid'` with your backend before showing order confirmation.

---

## 4. Native In-App Apple Pay (`ApplePaySheet` + `MFApplePayButton`) — Not a URL!

Apple Pay runs **100% natively in-app** using `MFApplePayButton` inside a slide-up bottom sheet (`lib/view/home/cart_screens/apple_pay_sheet.dart`):

### Styling
```dart
MFApplePayStyle _style() {
  final s = MFApplePayStyle();
  s.height = 52;
  s.borderRadius = 14;
  s.buttonText = "Pay with";
  s.hideLoadingIndicator = false;
  s.vendorName = "YourApp";
  return s;
}
```

### Flow
1. Open `showModalBottomSheet` with `32.r` top border radius, `cardColor` background, pull handle, and `SizedBox(height: 52, child: _applePayButton)`.
2. Initialize session:
   ```dart
   final session = await MFSDK.initSession(MFInitiateSessionRequest(), lang);
   ```
3. Load the native Apple Pay button and listen for completion:
   ```dart
   _applePayButton.load(session, executePaymentRequest, lang, (_) {
     // Optional: auto-trigger the native iOS Apple Pay sheet via MethodChannel
     _tryAutoTapButton();
   }).then((MFGetPaymentStatusResponse res) {
     // Payment completed natively in-app -> pop with result & confirm order
   }).catchError((e) {
     // Handle cancel or error
   });
   ```
4. **Why it feels instant:** As soon as `MFApplePayButton.load` signals the session is ready, a lightweight `MethodChannel('.../apple_pay_trigger')` invokes `triggerApplePayButton` (~250ms delay) to pop up the iOS Face ID / Touch ID Apple Pay sheet automatically — while keeping the button visible inside the sheet if the user needs to tap manually.
5. Only if native Apple Pay is unsupported/fails do we fall back to `HostedPaymentWebView` inside the slide-up sheet.

---

## 5. Benefit / 3DS Slide-Up Bottom Sheet (`MyFatoorahCheckoutSheet` + `HostedPaymentWebView`)

For **Benefit / BenefitPay** (and 3DS / OTP challenges), we do **not** launch an external browser or a jarring full-screen route. Instead, it slides up smoothly from the bottom as an `86%`-height modal sheet (`lib/view/home/cart_screens/myfatoorah_checkout_sheet.dart` + `hosted_payment_webview.dart`):

### Slide-Up Sheet Chrome (`MyFatoorahCheckoutSheet`)
```dart
showModalBottomSheet<T>(
  context: context,
  useRootNavigator: true,
  isScrollControlled: true,
  useSafeArea: true,
  isDismissible: false,
  enableDrag: false,
  backgroundColor: Colors.transparent,
  builder: (_) => MyFatoorahCheckoutSheet(title: 'Benefit', child: webView),
);
```
- **Container:** `height: (media.size.height - media.viewInsets.bottom) * 0.86`, `color: AppColors.bgColor`, `borderRadius: BorderRadius.vertical(top: Radius.circular(32))`.
- **Top Bar:**
  - Pull handle (`40 x 4`, `borderRadius: 2`)
  - Header row: `Back` button + centered bold title (`18sp`) + `Close` button
  - `1px` divider (`AppColors.borderColor`)
- **Loading State (`MyFatoorahCheckoutLoading`):** Shimmer skeleton blocks (`72px` header + three `48px` rows) while the Benefit page loads.

### Callback & Auto-Completion Inside `HostedPaymentWebView`
1. `WebViewController` loads the MyFatoorah Benefit/3DS URL with `JavaScriptMode.unrestricted` and `setBackgroundColor(AppColors.bgColor)`.
2. `NavigationDelegate` watches `onPageStarted` and `onPageFinished`:
   - Extracts `paymentId` / `PaymentId` / `Id` from query params.
   - Detects your backend callback/return URL (`_maybeFinishFromUrl`) or inspects the DOM for paid confirmation (`_inspectPageForPaid`).
3. Once detected, it polls your backend `PaymentStatusService` to verify the payment is genuinely `Paid`, then closes the slide-up sheet (`Navigator.pop(true)`) and shows the Order Confirmation screen.
