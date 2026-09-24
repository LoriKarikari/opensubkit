# SDK wire contract for v0 endpoints

Research for [SDK wire contract for v0 endpoints](https://github.com/LoriKarikari/opensubkit/issues/2).

Everything here is read from SDK source, pinned to these commits (all from 2026-09-24):

- `RevenueCat/purchases-android` at [`8d0a177`](https://github.com/RevenueCat/purchases-android/tree/8d0a1775e62317a79b70f41da15e066005dc2c26), abbreviated **A** below.
- `RevenueCat/purchases-ios` at [`9b8a4ef`](https://github.com/RevenueCat/purchases-ios/tree/9b8a4efb7f36f6173329653845b7942ebfb234e0), abbreviated **I** below.
- `RevenueCat/purchases-hybrid-common` at [`ee92868`](https://github.com/RevenueCat/purchases-hybrid-common/tree/ee9286818421c97ccf35e81aa83c23b9c918a68c).

No live traffic was captured. The Verification strategy ticket should confirm these shapes against a real SDK build.

## Findings that shape the spec

1. **Never answer post receipt with a 4xx for a server-side problem.** Both SDKs treat any 4xx on `POST /v1/receipts` as final. iOS finishes the StoreKit transaction ([I `NetworkError.swift#L237`][i-finishable]). Android acknowledges the purchase and records the token as posted, so it never sends it again ([A `Backend.kt#L1359`][a-post-receipt-errors], [A `PostReceiptHelper.kt`][a-post-receipt-helper]). A Google purchase acknowledged this way is no longer auto-refunded after 3 days. A store outage, a missing credential, or a bug must be a 5xx, which keeps the purchase queued for retry. This includes 404 on iOS and 429 on both.
2. **Consumables need a flag in config.** Android only consumes a one-time purchase when the response carries `purchased_products[<product_id>].should_consume: true` ([A `PostReceiptResponse.kt`][a-post-receipt-response], [A `BillingWrapper.kt#L428`][a-consume]). Otherwise it acknowledges, and the user can't buy that consumable again.
3. **StoreKit 2 restore posts one transaction.** On iOS 15+ with the default StoreKit 2, restore and sync post the first verified transaction's JWS plus the `app_transaction` JWS, or only `app_transaction` when there are no transactions ([I `PurchasesOrchestrator.swift#L1922`][i-sync-sk2]). OpenSubKit has to fetch the rest of the Customer's history from the App Store Server API.
4. **Signature checks fail open, as expected.** Default mode is informational on both platforms ([I `Signing.swift#L179`][i-signing-default], [A `EntitlementVerificationMode.kt`][a-verification-mode]). Without an `X-Signature` header every signed response is `FAILED` ([A `SigningManager.kt#L179`][a-verify-signed]). Access is still granted. The two side effects are an error log per request and no ETag caching, because failed responses are never cached ([A `ETagManager.kt#L266`][a-should-use-etag], [I `ETagManager.swift#L159`][i-should-use-etag]). OpenSubKit should not implement ETags or 304s in v0. The one real risk is an app that reads `EntitlementInfo.verification` itself and denies on `FAILED`.
5. **The proxy URL carries everything a v0 app sends.** With a proxy URL, fallback hosts and API-source failover are off on both platforms ([A `AppConfig.kt#L58`][a-appconfig-baseurl], [I `HTTPRequestPath.swift#L139`][i-url-proxy]). Android still sends diagnostics and feature events to RevenueCat hosts ([A `AppConfig.kt#L34`][a-diagnostics-url]), but diagnostics are off by default and feature events only come from paywalls, Customer Center, and ads, all out of scope. So a v0 app sends nothing to RevenueCat.
6. **API keys are only checked by prefix, and only for logging.** Nothing is blocked. Use `appl_` for App Store Apps and `goog_` for Google Play Apps to avoid error logs. Never issue a key starting with `test_`, which switches the SDK into its simulated Test Store ([A `APIKeyValidator.kt#L98`][a-api-key], [I `ConfiguredStoreEnvironment.swift#L34`][i-api-key]). The key reaches OpenSubKit as `Authorization: Bearer <key>` ([A `BackendHelper.kt`][a-auth-header]), so it identifies the App and therefore the Project.

## Request conventions

- **Auth.** `Authorization: Bearer <public SDK key>` on every v0 endpoint. The newer IAM token flow (`/auth/*`) is off by default (`iamEnabled = false`, [A `AppConfig.kt`][a-appconfig]) and out of scope.
- **Body.** JSON, `Content-Type: application/json`. Android sends every request with a body as POST ([A `HTTPClient.kt`][a-http-client]).
- **Headers worth reading.** `X-Platform` (`android`, `iOS`, and so on), `X-Version` (SDK version), `X-Client-Bundle-ID`, `X-Observer-Mode-Enabled`, `X-Storefront`, `X-Is-Sandbox` (iOS always, Android only for its Test Store), `X-StoreKit-Version` (iOS), `X-Retry-Count` (iOS). Full lists at [A `HTTPClient.kt#L673`][a-headers] and [I `HTTPClient.swift#L113`][i-headers].
- **Headers to ignore.** `X-Nonce`, `X-Post-Params-Hash`, `X-Headers-Hash` (signature inputs), `X-RevenueCat-ETag` and `X-RC-Last-Refresh-Time` (ETag cache, see finding 4).
- **Paths.** App User IDs are URL-encoded in paths. Anonymous IDs look like `$RCAnonymousID:<32 lowercase hex>` ([A `IdentityManager.kt#L290`][a-anon-id]).
- **Errors.** A JSON body `{"code": <int>, "message": "<text>"}`. The SDK maps known codes to its public error codes ([A `errors.kt#L78`][a-errors]). Useful ones are `7225` invalid API key, `7220` empty App User ID, `7256` invalid App User ID, `7103` invalid receipt, `7102` receipt belongs to another user, `7101` store problem, `7263` invalid attributes.
- **Retries.** iOS retries 429 on its own and honors `Retry-After` and `Is-Retryable` ([I `HTTPClient.swift#L57`][i-retry]). If retries run out on post receipt, that 429 is still a 4xx and finishes the transaction (finding 1).
- **Attributes "synced" rule.** For attributes, any status except 5xx and 404 counts as synced and is not re-sent ([A `RCHTTPStatusCodes.kt#L21`][a-is-synced]).

## Endpoints

Paths are relative to the proxy URL. "Signed" means the SDK will try to verify the response (see finding 4).

### Must implement

| Endpoint | Request body | Success response | Notes |
| --- | --- | --- | --- |
| `GET /v1/subscribers/{app_user_id}` | none | CustomerInfo | Signed. Called at startup and on foreground. |
| `POST /v1/receipts` | see below | CustomerInfo plus `purchased_products` | Signed. Findings 1 to 3. |
| `POST /v1/subscribers/identify` | `{"app_user_id", "new_app_user_id"}` | CustomerInfo | Signed. `201` means the new App User ID was created, `200` means it existed ([A `Backend.kt#L486`][a-login]). An empty body is an error. |
| `GET /v1/subscribers/{app_user_id}/offerings` | none | Offerings | Signed. Android falls back to cached Offerings on error. |
| `GET /v1/product_entitlement_mapping` | none | Product to Entitlement map | Signed. Powers offline entitlements when OpenSubKit is unreachable. |
| `POST /v1/subscribers/{app_user_id}/attributes` | `{"attributes": {"<key>": {"value", "updated_at_ms"}}}` | any 2xx | Not signed. |
| `POST /v1/subscribers/{app_user_id}/alias` | `{"app_user_id", "new_app_user_id"}` | any 2xx | Android only. Called when Google Block Store restores an anonymous ID after reinstall ([A `BlockstoreHelper.kt#L100`][a-blockstore]). Identity rules decide what it does. |

#### `POST /v1/receipts` body

Common fields: `fetch_token`, `app_user_id`, `is_restore`, `observer_mode`, `purchase_completed_by`, `initiation_source` (`purchase`, `restore`, `queue`), `attributes`, `presented_offering_identifier`, `presented_placement_identifier`, `sdk_originated`, `payload_version` ([A `Backend.kt#L304`][a-post-receipt], [I `PostReceiptDataOperation.swift#L328`][i-post-receipt]).

- **Google Play.** `fetch_token` is the Play purchase token. Also `product_ids`, `platform_product_ids`, `price`, `currency`, `normal_duration`, `pricing_phases`, `store_user_id`, `proration_mode`. Android also sends `price_string` and `marketplace` as request headers.
- **App Store.** `fetch_token` is one of three things ([I `TransactionPoster.swift#L433`][i-fetch-token]).
  - The signed transaction JWS. This is the default on iOS 15+ with StoreKit 2.
  - A base64 JSON receipt the SDK builds itself. Only in Xcode's local StoreKit testing environment.
  - The base64 app receipt. The StoreKit 1 path.

  iOS also sends `app_transaction` (the AppTransaction JWS), `transaction_id`, and product data encoded in snake_case: `product_id`, `payment_mode`, `currency`, `store_country`, `price`, `normal_duration`, `intro_duration`, `trial_duration`, `introductory_price`, `subscription_group_id`, `offers` ([I `ProductRequestData.swift`][i-product-data]). A restore with no transactions posts no `fetch_token` and only `app_transaction`.

Response: CustomerInfo, plus `"purchased_products": {"<product_id>": {"should_consume": <bool>}}` for Android one-time purchases.

### Answer with a stub

| Endpoint | Stub | Why it's safe |
| --- | --- | --- |
| `GET /v1/config/{domain}` | a 4xx | Remote config is on by default on both platforms and expects a binary "RC Container" body. It only feeds paywalls, workflows, UI config, checkpoints, and audiences. Android treats a 4xx as a deliberate refusal and doesn't retry it ([A `RemoteConfigManager.kt#L350`][a-remote-config]). iOS behavior on a 4xx is not confirmed yet. |
| `GET /v1/subscribers/{app_user_id}/health_report_availability` | `{"report_logs": false}` | iOS debug builds only, at startup ([I `Purchases.swift#L3160`][i-health]). |
| `POST /v1/subscribers/{app_user_id}/intro_eligibility` | `{}` | iOS StoreKit 1 path only ([I `TrialOrIntroPriceEligibilityChecker.swift`][i-intro]). The response maps product ID to `true`, `false`, or `null`. An empty map leaves eligibility unknown. |
| `POST /v1/subscribers/{app_user_id}/attribution`, `/adservices_attribution` | `200 {}` | Attribution tokens. Accept and drop. |
| `POST /v1/diagnostics`, `POST /v1/events` | `200 {}` | iOS sends these to the proxy when enabled. Off by default or out of scope. |

### Out of scope

`/v1/offers` (promotional offer signing, needs the Apple subscription key), `/v1/customercenter/*`, `/v1/subscribers/{id}/virtual_currencies`, `/v1/subscribers/redeem_purchase`, `/v1/external_purchase_tokens`, `/v1/subscribers/{id}/restore/eligibility`, `/v1/subscribers/{id}/ads/*`, `/rcbilling/*`, `/v1/receipts/amazon/*`, and `/auth/*`. Full lists at [A `Endpoint.kt`][a-endpoints] and [I `HTTPRequestPath.swift#L425`][i-paths]. Answer them with a 4xx that isn't 404, so the SDK surfaces an error and doesn't retry.

## Response shapes

Example payloads live in the iOS test fixtures ([I `Fixtures/`][i-fixtures]). `CustomerInfo.json`, `Offerings.json`, and `ProductsEntitlementsWithBasePlanId.json` are the canonical references.

### CustomerInfo

Required, or parsing fails: `request_date`, `subscriber.first_seen`, `subscriber.subscriptions`, `subscriber.non_subscriptions` ([A `CustomerInfoFactory.kt#L55`][a-customer-info]).

```json
{
  "request_date": "2026-09-24T12:00:00Z",
  "request_date_ms": 1790251200000,
  "subscriber": {
    "original_app_user_id": "$RCAnonymousID:...",
    "first_seen": "...",
    "original_purchase_date": "...",
    "original_application_version": "1.0",
    "management_url": "https://apps.apple.com/account/subscriptions",
    "entitlements": {
      "pro": { "product_identifier": "monthly", "product_plan_identifier": "monthly-base", "purchase_date": "...", "expires_date": "..." }
    },
    "subscriptions": {
      "monthly": {
        "store": "app_store", "is_sandbox": true, "period_type": "normal",
        "purchase_date": "...", "original_purchase_date": "...", "expires_date": "...",
        "unsubscribe_detected_at": null, "billing_issues_detected_at": null,
        "grace_period_expires_date": null, "refunded_at": null, "ownership_type": "PURCHASED",
        "store_transaction_id": "...", "product_plan_identifier": "monthly-base"
      }
    },
    "non_subscriptions": {
      "coins_100": [ { "id": "...", "store_transaction_id": "...", "purchase_date": "...", "store": "play_store", "is_sandbox": true } ]
    }
  }
}
```

- The SDK decides whether an Entitlement is active on the device, from `expires_date` against `request_date`. A `null` `expires_date` means lifetime. So `request_date` must be the server's current time.
- Each Entitlement's `product_identifier` must be a key in `subscriptions` or `non_subscriptions`, where the SDK reads store, sandbox, period, and billing details ([A `EntitlementInfoFactories.kt`][a-entitlement-info]).
- `subscriptions` is keyed by store product ID. For Google, the SDK joins that key with `product_plan_identifier` into `<product_id>:<base_plan_id>` itself ([A `CustomerInfoFactory.kt`][a-customer-info-file]). Mapping each lifecycle state to these fields is the Subscription lifecycle ticket's job.

### Offerings

Required: `current_offering_id`, `offerings[].identifier`, `offerings[].description`, `offerings[].packages[].identifier`, `offerings[].packages[].platform_product_identifier` ([A `OfferingParser.kt#L47`][a-offerings]). Android also reads `platform_product_plan_identifier` for the base plan. `metadata`, `placements`, and `targeting` are optional. `paywall`, `paywall_components`, and `ui_config` are out of scope.

Standard Package identifiers are `$rc_weekly`, `$rc_monthly`, `$rc_two_month`, `$rc_three_month`, `$rc_six_month`, `$rc_annual`, and `$rc_lifetime`. Any other string is a custom Package.

### Product entitlement mapping

```json
{
  "product_entitlement_mapping": {
    "monthly:monthly-base": { "product_identifier": "monthly", "base_plan_id": "monthly-base", "entitlements": ["pro"] },
    "lifetime": { "product_identifier": "lifetime", "entitlements": ["pro"] }
  }
}
```

([A `ProductEntitlementMapping.kt`][a-mapping])

## Hybrid SDKs

React Native and Flutter call `setProxyURLString`, which sets the native `Purchases.proxyURL` on both platforms ([hybrid `CommonFunctionality.swift`][h-ios], [hybrid `common.kt`][h-android]). It must be set before `configure`. Native compatibility therefore covers them.

## Open questions for other tickets

- **Apple purchase verification.** Rebuild full history from one transaction JWS or only an `app_transaction` (finding 3). Handle the Xcode JSON receipt and the StoreKit 1 app receipt, or declare them unsupported.
- **Identity rules.** What `alias` and `identify` do, including the `201` versus `200` distinction.
- **Config file shape.** A consumable flag per Product (finding 2). One `appl_` or `goog_` key per App (finding 6).
- **Verification strategy.** Capture real traffic to confirm this document. Include an iOS StoreKit 2 restore, an Android consumable, and the iOS response to a 4xx from `/v1/config`.
- **Not yet specified.** Which SDK versions v0 supports. This document is pinned to 2026-09-24 main.

[a-post-receipt-errors]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/Backend.kt#L1359
[a-post-receipt-helper]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/PostReceiptHelper.kt
[a-post-receipt-response]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/networking/PostReceiptResponse.kt
[a-consume]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/google/BillingWrapper.kt#L428
[a-verification-mode]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/EntitlementVerificationMode.kt
[a-verify-signed]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/verification/SigningManager.kt#L179
[a-should-use-etag]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/networking/ETagManager.kt#L266
[a-appconfig-baseurl]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/AppConfig.kt#L58
[a-appconfig]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/AppConfig.kt
[a-diagnostics-url]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/AppConfig.kt#L34
[a-api-key]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/APIKeyValidator.kt#L98
[a-auth-header]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/BackendHelper.kt
[a-http-client]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/HTTPClient.kt
[a-headers]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/HTTPClient.kt#L673
[a-anon-id]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/identity/IdentityManager.kt#L290
[a-errors]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/errors.kt#L78
[a-is-synced]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/networking/RCHTTPStatusCodes.kt#L21
[a-login]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/Backend.kt#L486
[a-blockstore]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/blockstore/BlockstoreHelper.kt#L100
[a-post-receipt]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/Backend.kt#L304
[a-remote-config]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/remoteconfig/RemoteConfigManager.kt#L350
[a-endpoints]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/networking/Endpoint.kt
[a-customer-info]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/CustomerInfoFactory.kt#L55
[a-entitlement-info]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/EntitlementInfoFactories.kt
[a-offerings]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/OfferingParser.kt#L47
[a-mapping]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/offlineentitlements/ProductEntitlementMapping.kt
[i-finishable]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Networking/HTTPClient/NetworkError.swift#L237
[i-sync-sk2]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Purchasing/Purchases/PurchasesOrchestrator.swift#L1922
[i-signing-default]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Security/Signing.swift#L179
[i-should-use-etag]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Networking/HTTPClient/ETagManager.swift#L159
[i-url-proxy]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Networking/HTTPClient/HTTPRequestPath.swift#L139
[i-api-key]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Purchasing/Purchases/ConfiguredStoreEnvironment.swift#L34
[i-headers]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Networking/HTTPClient/HTTPClient.swift#L113
[i-retry]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Networking/HTTPClient/HTTPClient.swift#L57
[i-post-receipt]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Networking/Operations/PostReceiptDataOperation.swift#L328
[i-fetch-token]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Purchasing/Purchases/TransactionPoster.swift#L433
[i-health]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Purchasing/Purchases/Purchases.swift#L3160
[i-paths]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Networking/HTTPClient/HTTPRequestPath.swift#L425
[i-product-data]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Purchasing/ProductRequestData.swift
[i-intro]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Purchasing/TrialOrIntroPriceEligibilityChecker.swift
[a-customer-info-file]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/CustomerInfoFactory.kt
[i-fixtures]: https://github.com/RevenueCat/purchases-ios/tree/9b8a4efb7f36f6173329653845b7942ebfb234e0/Tests/UnitTests/Networking/Responses/Fixtures
[h-ios]: https://github.com/RevenueCat/purchases-hybrid-common/blob/ee9286818421c97ccf35e81aa83c23b9c918a68c/ios/PurchasesHybridCommon/PurchasesHybridCommon/CommonFunctionality.swift#L42
[h-android]: https://github.com/RevenueCat/purchases-hybrid-common/blob/ee9286818421c97ccf35e81aa83c23b9c918a68c/android/hybridcommon/src/main/java/com/revenuecat/purchases/hybridcommon/common.kt#L877
