# KhuMart — Android grocery app + WooCommerce delivery backend

```
Flutter app (app/)  ──HTTPS──►  WordPress / WooCommerce (web.netoday.in)
  • browses products via WooCommerce's PUBLIC Store API  (/wc/store/v1)
  • everything privileged via plugin khumart/v1          (backend/khumart-delivery)
        config · quote · orders · auth · notifications · rider
Routing (OSRM or Google Directions) is called ONLY by the plugin (server side).
```

No WooCommerce consumer key/secret, routing key, payment or OTP secret is in the app. The existing store, products, customers and orders are untouched: the plugin only adds endpoints, one admin page, a rider role and an "Out for Delivery" order status.

## Status (honest)
| Part | State |
|---|---|
| Fee engine (PHP) | Written, **38/38 unit tests executed and passing** |
| WordPress plugin | Written, PHP syntax-checked. **Not yet run inside WordPress** (no site access from the build workspace) |
| Flutter app | Written (27 Dart files). **Not compiled or run** — Flutter/Android SDK were unreachable from the build workspace |
| APK | **Not built.** Build it with the included GitHub Actions workflow or locally (below) |
| Live site inspection | **Not possible from here** — `web.netoday.in` was blocked. Verify items in docs/BACKEND.md §1 |

Expect a first-compile round of small fixes (typos/API drift) — send the `flutter analyze` output and they can be fixed quickly.

## 1. Install the backend
1. Zip `backend/khumart-delivery/` and upload via **Plugins → Add New → Upload** (needs WooCommerce). Activate.
2. **WooCommerce → KhuMart Delivery**: click the map to set the **dispatch point**, switch to *Draw boundary* and click the corners of the Khumulwng service area, set fees (standard fee, ₹5/km, rounding, max distance, min order, hours/cutoff), choose payments, routing provider. Save.
3. Create riders: **Users → Add New**, role *KhuMart Rider*; give the user a billing phone. Assign a rider on each order's edit page.
4. Details: `docs/BACKEND.md`. Security checklist: `docs/SECURITY.md`.

## 2. Build the APK
**Option A — GitHub Actions (no local setup):** push this folder to a GitHub repo, then *Actions → Build KhuMart APK → Run workflow*. Download the `khumart-apk` artifact (debug + release APKs).

**Option B — local:** install Flutter 3.27.x + Android Studio (Android SDK, accept licences), then
```bash
cd app
bash tool/setup_android.sh        # flutter create (android/), manifest permissions, launcher icons
flutter analyze && flutter test
flutter build apk --debug \
  --dart-define=KHUMART_SITE=https://web.netoday.in \
  --dart-define=SUPPORT_PHONE=+91XXXXXXXXXX
# → app/build/app/outputs/flutter-apk/app-debug.apk
```
Install: enable *Install unknown apps* on the phone, copy the APK over, tap it — or `adb install -r app-debug.apk`.
Release/Play Store: `docs/RELEASE.md`.

## 3. What the app does
Splash + onboarding · Home (location chip, search, banners, categories, offers, popular, new, recently viewed) · category/search/sort/filter listing with paging · product detail (gallery, discount badge, stock, qty, pulsing Add to Cart, Buy Now, wishlist) · persistent cart · GPS/map pin picker (permission asked only on tap; manual search & pin drag fallback) · address form + saved addresses · checkout with server-quoted fee breakdown and explicit fee consent · COD / online (gateway page in browser) · idempotent order placement · order history, tracking timeline, notifications, support · rider/staff delivery view with Call + Google Maps navigation and status updates.

## 4. Business rules implemented (server-side, `class-fee-engine.php`)
* Inside Khumulwng (polygon, or radius fallback): standard fee only (can be free above a threshold).
* Outside: standard fee + **₹5 × billable road km** (road distance from routing API, never straight-line). Default rule = full distance from dispatch; alternative = distance minus N included km. Rounding: up to whole km (default) / nearest km / up to 0.5 km / exact.
* Free delivery waives the *standard* fee only; the outside charge still applies.
* Max distance, min order, operating hours, same-day cutoff, PIN-code exceptions are admin settings.
* Routing failure ⇒ no distance/fee is invented: inside-area orders use the standard fee; outside-area orders are rejected, or (COD only, if enabled) placed **on-hold** for manual fee confirmation.
* The app's displayed fee is never trusted: `POST /orders` recomputes and returns **409 + new quote** if it differs from what the customer accepted.

## 5. Not implemented / needs your input
* Mobile OTP & WhatsApp/FCM push: endpoints/hooks exist (`khumart_send_otp`, `khumart_verify_otp`, `khumart_notify`) but **nothing is sent** until you pick a provider. Notifications currently appear in-app (list refreshes on open) — no fake sends.
* Variable (size/variant) products are shown but purchasable on the website only.
* Coupons are not applied in-app. Live rider tracking is not implemented (by design, privacy-first).
* Fonts use the system font; add a bundled font for further brand polish.
