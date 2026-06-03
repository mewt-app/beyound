# Rally KYC Integration — Webview Approach (zero native HyperVerge SDK)

Goal: Rally mobile app does KYC **without bundling the HyperVerge native SDK** (app-size concern). Achieved by embedding a hosted launcher page (`kyc.<brand>.in`) in a native WebView. The launcher runs the HyperVerge **web** SDK; Rally only embeds a URL and listens for a result message.

Reference impl: beyound wallet (`supermade/apps/wallet`) embeds `kyc.beyound.in` in an iframe. Rally reproduces the same contract in a native WebView.

---

## 0. Architecture (who does what)

```
Rally app (native WebView)
   │  embeds  https://kyc.<brand>.in/?sid=<sessionId>&mid=<merchantId>&return_url=<app deeplink/https>
   ▼
kyc.<brand>.in  (SEPARATE hosted launcher app — NOT in supermade/backend repos)
   │  server-side, using mid:
   │    POST /v2/zokudo/wallet/register            (ensure wallet active, MIN_KYC)
   │    POST /v2/zokudo/wallet/generate-kyc-request → {access_token, workflow_id, transaction_id}
   │  runs HyperVerge WEB SDK with those tokens (camera/mic via WebView perms)
   │  on finish → emits result back to Rally (postMessage / JS bridge / return_url redirect)
   ▼
Zokudo server → POST /v2/zokudo/wallet/kyc-callback (server-to-server) → wallet kyc_status = FULL_KYC
```

Key point: **Rally never calls `generate-kyc-request` and never touches HyperVerge tokens.** Those are the launcher's job. Rally's surface = (1) auth/OTP to get `sessionId`+`merchantId`, (2) embed launcher URL, (3) receive result. This is exactly why app-size stays flat — no SDK in the binary.

⚠️ The current beyound launcher posts results via `window.parent.postMessage` (works for web iframe). A **native** WebView has no parent window → the launcher needs a native-compatible result channel (see §4). That is a **launcher-side change**, not Rally-side.

---

## 1. Backend contract — curls Rally (or the launcher) needs

`BASE_URL = https://api.spaid.in/backend`
Path prefix `ROUTE_PREFIX_V1 = /v2` → every path is `${BASE_URL}/v2/...`.
Auth: **no token header** on wallet/KYC calls — handlers authorize by `merchant_id` in body. Carry `merchantId` forward; that's the load-bearing value.

### Step 1a — Send OTP (Rally calls)
```bash
curl -X POST "$BASE_URL/v2/authentication/notification-otp/send" \
  -H "Content-Type: application/json" \
  -d '{ "phone": "'"$PHONE"'", "brand": "rally", "company": "Rally" }'
```
→ `200 { "sid": "SM...", "status": "sent", "brand": "rally", "expires_in_seconds": 300 }`
Keep `sid`, echo into validate. (`rally` must be whitelisted — see §6.)

### Step 1b — Validate OTP (Rally calls) → gives sessionId + merchantId
```bash
curl -X POST "$BASE_URL/v2/authentication/notification-otp/validate" \
  -H "Content-Type: application/json" \
  -d '{ "phone": "'"$PHONE"'", "brand": "rally", "sid": "'"$SID"'", "otp": "123456" }'
```
→ `200 { "isValid": true, "sessionId": "<64hex>", "merchantId": "<mid>", "brand": "rally", "phone": "9876543210" }`
**`merchantId` = `$MID` for everything below. `sessionId` = `sid` query param for launcher URL.**

### Step 2 — Wallet exists? (launcher or Rally)
```bash
curl -X GET "$BASE_URL/v2/zokudo/wallet/wallet-exists?merchant_id=$MID"
```
→ `200 { "exists": true, "wallet_id": "<id>" }` | `{ "exists": false, "wallet_id": null }`

### Step 3 — Register wallet (launcher, server-side)
```bash
curl -X POST "$BASE_URL/v2/zokudo/wallet/register" \
  -H "Content-Type: application/json" \
  -d '{ "merchant_id": "'"$MID"'" }'
```
→ `200 { "success": true, "wallet_id": "...", "user_hash_id": "...", "kyc_status": "MIN_KYC", "status": "active" }`
Preconditions (else 4xx): merchant exists; **Aadhaar data present** (`422 AADHAAR_DATA_MISSING`); merchant email present (`422 EMAIL_MISSING`).

### Step 4 — Generate KYC request (launcher, server-side) → HyperVerge SDK tokens
```bash
curl -X POST "$BASE_URL/v2/zokudo/wallet/generate-kyc-request" \
  -H "Content-Type: application/json" \
  -d '{ "merchant_id": "'"$MID"'" }'
```
→ `200 { "success": true, "access_token": "<hv>", "workflow_id": "<wf>", "transaction_id": "<txn>", "kyc_record_id": "<id>" }`
Requires active wallet (`404 WALLET_NOT_FOUND` → register first). The launcher feeds `access_token`/`workflow_id`/`transaction_id` into the **HyperVerge web SDK**. No `kyc.<brand>` URL is built by the backend — the launcher IS that page.

### Step 5 — KYC callback (Zokudo → backend, server-to-server; nobody on Rally calls this)
```bash
# Zokudo posts this after HyperVerge journey; documented only
curl -X POST "$BASE_URL/v2/zokudo/wallet/kyc-callback" \
  -H "Content-Type: application/json" \
  -d '{ "transactionId": "'"$TXN"'", "status": "SUCCESS", "customerHashId": "<userHashId>" }'
```
Effect: matches by `transaction_id`, sets wallet `last_response.kyc_status = FULL_KYC` on success. Always returns `200`.

### Step 6 — KYC status poll — ⚠️ NO ENDPOINT EXISTS
No GET returns the final KYC tier. Today: re-call `register` (already-active branch) or trust the launcher's result message. **Recommend backend add** `GET /v2/zokudo/wallet/kyc-status?merchant_id=` returning `last_response.kyc_status` for a clean server-confirmed poll.

Error envelope (wallet/KYC): `{ "detail": { "message": "...", "error_code": "..." } }`. OTP errors: `{ "detail": "..." }`.

---

## 2. Launcher URL Rally embeds

```
https://kyc.<brand>.in/?sid=<sessionId>&mid=<merchantId>&return_url=<return>
```
- `sid` = `sessionId` from OTP validate.
- `mid` = `merchantId` from OTP validate.
- `return_url` = where launcher sends user on finish. Web wallet uses `window.location.origin + "/"`. For a native app: an **app deeplink** (`rally://kyc-done`) or an https page the app intercepts. Launcher only honours `https://*.<brand>.in` today → if using a custom-scheme deeplink, launcher allow-list must be widened (launcher-side).
- Null guard: if `sid` or `mid` missing → don't build/load (would fail).

---

## 3. Rally load sequence

1. OTP send → validate → store `sessionId`, `merchantId`.
2. (Optional) `wallet-exists?merchant_id=mid` to decide if KYC needed.
3. Build launcher URL with `sid`/`mid`/`return_url`.
4. Open native WebView on that URL. Grant camera + mic WebView permissions (HyperVerge needs camera).
5. Listen for result (§4). On success → close WebView, refresh wallet state.

WebView permission flags by stack:
- **React Native (`react-native-webview`)**: `mediaCapturePermissionGrantType="grant"`, `allowsInlineMediaPlayback`, `javaScriptEnabled`, Android `onPermissionRequest` → grant camera/mic; iOS `Info.plist` `NSCameraUsageDescription`+`NSMicrophoneUsageDescription`; Android `<uses-permission CAMERA/RECORD_AUDIO>` + runtime grant.
- **Flutter (`webview_flutter` / `flutter_inappwebview`)**: inappwebview `onPermissionRequest` → `PermissionResponseAction.GRANT`; same native manifest/plist perms.
- **Native iOS (`WKWebView`)**: `WKUIDelegate` media-capture grant; plist perms.
- **Native Android (`WebView`)**: `WebChromeClient.onPermissionRequest(grant CAMERA/AUDIO)`; `setMediaPlaybackRequiresUserGesture(false)`.

---

## 4. Result callback bridge — the launcher-side delta

Web iframe (beyound today): launcher does
```js
window.parent.postMessage({ type: "beyound-kyc-result", status: "<status>", code: <n> }, targetOrigin)
```
Native WebView has no `window.parent`. Launcher must ALSO emit one of:

**Option A — JS→native bridge (recommended).** Launcher detects native context and calls:
```js
// React Native
if (window.ReactNativeWebView) window.ReactNativeWebView.postMessage(JSON.stringify(result));
// Flutter (flutter_inappwebview channel named e.g. "kycResult")
else if (window.flutter_inappwebview) window.flutter_inappwebview.callHandler("kycResult", result);
// Native iOS (message handler "kycResult")
else if (window.webkit?.messageHandlers?.kycResult) window.webkit.messageHandlers.kycResult.postMessage(result);
// Native Android (addJavascriptInterface name e.g. "KycBridge")
else if (window.KycBridge) window.KycBridge.onResult(JSON.stringify(result));
else window.parent?.postMessage(result, "*"); // web fallback
```
Rally side: RN `onMessage={e => handle(JSON.parse(e.nativeEvent.data))}`; Flutter `addJavaScriptHandler(handlerName:"kycResult")`; iOS `WKScriptMessageHandler` name `kycResult`; Android `@JavascriptInterface onResult(String)`.

**Option B — return_url redirect interception.** Launcher redirects to `return_url?status=<status>&txn=<txn>` on finish. Rally intercepts nav: RN `onShouldStartLoadWithRequest`, Flutter `onNavigationRequest`/`shouldOverrideUrlLoading`, iOS `decidePolicyFor navigationAction`, Android `shouldOverrideUrlLoading`. Parse query, close WebView. Works even if JS bridge unavailable; needs launcher to support redirect-on-finish + allow-listed `return_url`.

**Result status vocabulary** (from reference impl `classify`):
- success ← `auto_approved | needs_review | already_completed | approved`
- cancelled ← `user_cancelled | cancelled`
- failed ← everything else (`auto_declined`, `error`, ...). Product rule in beyound: **no retry on failure** → show "contact support".

On success: close WebView, re-check wallet/KYC state (poll gap — see §1 Step 6).

---

## 5. Reusable guards to copy from reference impl

- **Stall watchdog** (6s): if WebView/launcher never signals load, show "Continue / Open externally" escape (covers frame-bust / CSP / dead launcher). Native: a timeout + a "retry / open in browser" affordance.
- **Fire-once**: launch KYC once per session; guard with a flag so closing the WebView doesn't re-pop in a loop.
- **Failure-safe**: if launcher URL can't be built (missing sid/mid) → manual fallback, never load a dead URL.
- **Pending-only**: only push users into KYC when `kyc_completed=false`; never re-KYC a completed user.
- **Origin/type validation** on any postMessage result before trusting it.

---

## 6. Backend prerequisites for Rally (BLOCKERS — do before any integration)

1. **Add `"rally"` to `VALID_BRANDS`** — `mewt-backend-v3 .../routers/notification_otp.py:94`. Hard blocker: OTP send/validate returns `400 Unknown brand 'rally'` until added. (Same one-line change beyound needed.)
2. **OTP SMS** — default twilio direct-send is brand-agnostic; works once whitelisted. Pass `company:"Rally"` for branded copy. No per-brand Twilio number wired.
3. **Zokudo creds are hardcoded to beyound** — `zokudo_wallet.py:55-56` (single program `zokdpm`, beyound client). The wallet/KYC router is NOT per-brand → **Rally wallets would provision under beyound's Zokudo program as-is.** DECISION: if Rally needs its own Zokudo program/funding, parameterize creds per brand backend-side. Currently not parameterized.
4. **Aadhaar gate still enforced** — `zokudo_wallet.py:386` → register 422s without `merchant_aadhaar_data.aadhaar_json_dump` (+ merchant email). ⚠️ DISCREPANCY: user stated "register no longer needs Aadhaar" but current backend code still gates. VERIFY against deployed prod before building. If Rally's pre-register flow doesn't capture Aadhaar, register fails → no wallet → KYC can't launch. Options: capture Aadhaar pre-register, OR relax the gate for rally backend-side.
5. **Stand up `kyc.rally.in` launcher** — the HyperVerge-web-SDK host. This is where app-size savings come from. Must: take `sid`/`mid`, call register + generate-kyc-request server-side, run HyperVerge web SDK, emit result via native bridge (§4 Option A) AND/OR return_url redirect (§4 Option B). Likely a fork of the beyound launcher with brand swap + native-bridge addition.
6. **(Recommended) KYC-status read endpoint** — none today (§1 Step 6). Add for clean server-confirmed completion.

---

## 7. Open question for Rally team

Rally stack? React Native / Flutter / native iOS / native Android — determines exact WebView component + result-bridge code. §3–4 cover all four; pick one to finalize snippets.

---

## Source citations
- Backend: `mewt-backend-v3/projects/mewt_backend/src/routers/notification_otp.py` (otp send/validate, VALID_BRANDS:94), `routers/zokudo_wallet.py` (register:338, wallet-exists:520, generate-kyc:1045, callback:1173, creds:55, aadhaar gate:386), `routers/api.py` (mounts:282,463), `.env:221` (prefix `/v2`).
- FE reference: `supermade/apps/wallet/src/components/wallet/{KycInitiationSheet,KycWebviewSheet,WalletDashboard}.tsx`, `store/StoreProvider.tsx`, `store/slices/{authStore,walletStore}.ts`.
