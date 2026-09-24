# Subscription lifecycle states across both stores

Research for [Subscription lifecycle states across both stores](https://github.com/LoriKarikari/opensubkit/issues/5). It builds on [`sdk-wire-contract.md`](./sdk-wire-contract.md), which defines the CustomerInfo shape.

Sources, abbreviated below:

- **SDK-A** and **SDK-I**: `RevenueCat/purchases-android` at [`8d0a177`][a-tree] and `RevenueCat/purchases-ios` at [`9b8a4ef`][i-tree], the same commits as the wire contract.
- **Apple lib**: `apple/app-store-server-library-node` at [`bb0c0f8`][apple-lib], Apple's official models for the App Store Server API and notifications.
- **Apple docs**: [notificationType][apple-notification-type], [status][apple-status], [sandbox testing][apple-sandbox], [sandbox account settings][apple-sandbox-settings].
- **Google docs**: [`purchases.subscriptionsv2`][google-v2], [subscription lifecycle][google-lifecycle], [RTDN reference][google-rtdn], [testing][google-test].
- **RevenueCat docs**: [grace periods][rc-grace], [Family Sharing][rc-family], [REST v1 subscriber][rc-rest], [data export v5][rc-export].

## The rule everything hangs on

Both SDKs decide `EntitlementInfo.isActive` from one field, `subscriber.entitlements.<id>.expires_date` ([SDK-A `EntitlementInfoFactories.kt`][a-entitlement-info], [SDK-I `CustomerInfo+ActiveDates.swift`][i-active-dates]).

- `null` means active forever.
- Otherwise it's active when `expires_date` is later than the reference date. The reference date is the response's `request_date` if the response is at most 3 days old, and the device clock after that.

Nothing else grants or removes access. `grace_period_expires_date`, `billing_issues_detected_at`, `refunded_at`, and `auto_resume_date` are informational. So **every lifecycle state reduces to one computed value per Entitlement, the moment access ends**. RevenueCat's data exports name that value `effective_end_time`: "the date a subscriber will lose access", the later of `end_time` and `grace_period_end_time`, "inclusive of each store's logic for refunds, grace periods, cancellations" ([RevenueCat data export v5][rc-export]).

Two consequences:

1. **A store grace period must extend the Entitlement's `expires_date`** to the grace period's end. Otherwise the SDK shows the user as expired while the store still grants access. RevenueCat documents that subscriptions in a grace period "will still be considered active" ([RevenueCat grace periods][rc-grace]).
2. **A refund or revocation must pull `expires_date` back** to the revocation time, or drop the Entitlement. A device can keep a cached response for up to 3 days offline, so revocation reaches it on the next fetch.

`willRenew` is derived on the device too. It's false when the store is `promotional`, `expires_date` is `null`, `unsubscribe_detected_at` is set, `billing_issues_detected_at` is set, or `period_type` is `prepaid` ([SDK-A `EntitlementInfoHelper.kt`][a-will-renew], [SDK-I `EntitlementInfo.swift`][i-will-renew]).

## Field vocabulary

Per subscription (`subscriber.subscriptions.<product_id>`) the SDK reads these fields ([SDK-A `SubscriptionInfoResponse.kt`][a-subscription-info]):

| Field | Values |
| --- | --- |
| `store` | `app_store`, `play_store`, plus others out of scope ([SDK-A `EntitlementInfo.kt`][a-store]) |
| `is_sandbox` | boolean |
| `period_type` | `normal`, `trial`, `intro`, `prepaid`. Anything else parses as `normal`. |
| `ownership_type` | `PURCHASED`, `FAMILY_SHARED` |
| `purchase_date`, `original_purchase_date`, `expires_date` | ISO 8601 |
| `unsubscribe_detected_at`, `billing_issues_detected_at`, `grace_period_expires_date`, `refunded_at`, `auto_resume_date` | ISO 8601 or `null` |
| `store_transaction_id`, `product_plan_identifier` | strings, the plan is Google's base plan ID |

Per Entitlement: `product_identifier`, `product_plan_identifier`, `purchase_date`, `expires_date`, and `grace_period_expires_date` ([RevenueCat REST v1 subscriber][rc-rest]).

## Store signals

**Apple.** The App Store Server API reports a subscription status of `ACTIVE` (1), `EXPIRED` (2), `BILLING_RETRY` (3), `BILLING_GRACE_PERIOD` (4), or `REVOKED` (5) ([Apple lib `Status.ts`][apple-status-ts]). The renewal info carries `autoRenewStatus`, `isInBillingRetryPeriod`, `gracePeriodExpiresDate`, `expirationIntent`, and `autoRenewProductId`. The transaction carries `expiresDate`, `revocationDate`, `revocationReason`, `isUpgraded`, `inAppOwnershipType`, `offerType`, `offerDiscountType`, `type`, and `environment` ([Apple lib `JWSRenewalInfoDecodedPayload.ts`][apple-renewal], [`JWSTransactionDecodedPayload.ts`][apple-transaction]).

**Google.** `purchases.subscriptionsv2` reports `subscriptionState` as `PENDING`, `ACTIVE`, `PAUSED`, `IN_GRACE_PERIOD`, `ON_HOLD`, `CANCELED`, `EXPIRED`, or `PENDING_PURCHASE_CANCELED`. Each line item has `expiryTime`, `autoRenewingPlan.autoRenewEnabled` or `prepaidPlan`, and `offerPhase` (`freeTrial`, `introductoryPrice`, `basePrice`, `prorationPeriod`). The purchase has `linkedPurchaseToken`, `pausedStateContext.autoResumeTime`, `canceledStateContext`, `testPurchase`, and `acknowledgementState` ([Google `subscriptionsv2`][google-v2]). Google's guidance is "never calculate expiry or renewal dates manually. Always use the `expiryTime` field" ([RevenueCat's Google state machine guide][rc-google-guide], restating [Google lifecycle][google-lifecycle]).

## State mapping

"Access ends" is the Entitlement's `expires_date`. "Subscription fields" are the fields that differ from a plain active subscription.

| State | Apple signal | Google signal | Access ends | Subscription fields |
| --- | --- | --- | --- | --- |
| Active, renewing | status `ACTIVE`, `autoRenewStatus` 1. Notification `SUBSCRIBED` or `DID_RENEW`. | `ACTIVE`, `autoRenewEnabled` true. RTDN `SUBSCRIPTION_PURCHASED` (4) or `SUBSCRIPTION_RENEWED` (2). | `expiresDate` / `expiryTime` | none |
| Auto-renew off, not yet expired | `autoRenewStatus` 0. `DID_CHANGE_RENEWAL_STATUS` / `AUTO_RENEW_DISABLED`. | `CANCELED`. `SUBSCRIPTION_CANCELED` (3). | `expiresDate` / `expiryTime` | `unsubscribe_detected_at` = when first seen. Cleared on `AUTO_RENEW_ENABLED` or `SUBSCRIPTION_RESTARTED` (7). |
| Expired | status `EXPIRED`. `EXPIRED` with subtype `VOLUNTARY`, `BILLING_RETRY`, `PRICE_INCREASE`, or `PRODUCT_NOT_FOR_SALE`. | `EXPIRED`. `SUBSCRIPTION_EXPIRED` (13). | past `expiresDate` | unchanged |
| Billing grace period | status `BILLING_GRACE_PERIOD`, `gracePeriodExpiresDate`. `DID_FAIL_TO_RENEW` / `GRACE_PERIOD`. | `IN_GRACE_PERIOD`. `SUBSCRIPTION_IN_GRACE_PERIOD` (6). | end of grace period | `billing_issues_detected_at`, `grace_period_expires_date` |
| Billing retry, no access | status `BILLING_RETRY`. `DID_FAIL_TO_RENEW` without subtype, or `GRACE_PERIOD_EXPIRED`. | `ON_HOLD` (account hold). `SUBSCRIPTION_ON_HOLD` (5). | past `expiresDate` / `expiryTime` | `billing_issues_detected_at` |
| Recovered from billing issue | `DID_RENEW` / `BILLING_RECOVERY` | `SUBSCRIPTION_RECOVERED` (1) | new expiry | clear `billing_issues_detected_at` and `grace_period_expires_date` |
| Paused | n/a | `PAUSED`, `autoResumeTime`. `SUBSCRIPTION_PAUSED` (10). Resuming sends `SUBSCRIPTION_RECOVERED` (1). | now, no access while paused | `auto_resume_date` |
| Refunded or revoked | `REFUND` with `revocationDate`. Status `REVOKED`. `REFUND_REVERSED` undoes it. | `VoidedPurchaseNotification`, `SUBSCRIPTION_REVOKED` (12), or the Voided Purchases API | `revocationDate` | `refunded_at` |
| Family Sharing | transaction `inAppOwnershipType` `FAMILY_SHARED`. Access removed with `REVOKE`. | n/a | as the owner's subscription, or `REVOKE` time | `ownership_type` `FAMILY_SHARED` |
| Free trial | `offerDiscountType` `FREE_TRIAL` | `offerPhase.freeTrial` | as active | `period_type` `trial` |
| Paid introductory offer | `offerType` 1 with `PAY_AS_YOU_GO` or `PAY_UP_FRONT` | `offerPhase.introductoryPrice` | as active | `period_type` `intro` |
| Promotional offer, offer code, win-back | `offerType` 2, 3, or 4 | Google offers surface as `offerPhase`, covered by the two rows above | as active | `period_type` unconfirmed, see open questions |
| Google prepaid plan | n/a | line item has `prepaidPlan` | `expiryTime` | `period_type` `prepaid`, which also forces `willRenew` false |
| Pending purchase | Ask to Buy, no transaction yet | `PENDING`. `PENDING_PURCHASE_CANCELED` (20) if abandoned. | no Entitlement | none |

Apple sources for this table are the [notificationType][apple-notification-type] event tables and [Apple lib enums][apple-lib]. Google sources are the [RTDN reference][google-rtdn] and [`subscriptionsv2`][google-v2].

### Plan changes

- **Apple upgrade, downgrade, crossgrade.** All stay in one subscription group under one `originalTransactionId`. An upgrade takes effect immediately. The old transaction gets `isUpgraded: true` and the new product starts. A downgrade shows up as `DID_CHANGE_RENEWAL_PREF` / `DOWNGRADE` and takes effect at the next renewal, visible in `autoRenewProductId` ([Apple notificationType][apple-notification-type]). OpenSubKit keys `subscriptions` by product ID, so after an upgrade the old product's entry ends at the upgrade time and the new product's entry starts.
- **Google replacement.** A plan change creates a new purchase token whose `linkedPurchaseToken` points at the old one. The old token stops granting access. The SDK sends the replacement mode as `proration_mode` on post receipt ([sdk-wire-contract.md](./sdk-wire-contract.md)). A deferred replacement keeps the old plan until it expires.
- **Which product an Entitlement points at.** When several of a Customer's subscriptions unlock the same Entitlement, the Entitlement follows the one with the latest access end. That's the product the SDK reports in `EntitlementInfo.productIdentifier`.

### One-time purchases

| Kind | Apple `type` | Google | CustomerInfo |
| --- | --- | --- | --- |
| Consumable | `Consumable` | one-time product, consumed when the response says `should_consume` | an entry appended to `non_subscriptions.<product_id>` per purchase. Usually unlocks no Entitlement. |
| Non-consumable | `Non-Consumable` | one-time product, acknowledged and never consumed | an entry in `non_subscriptions`. Entitlement `expires_date` is `null`. |
| Non-renewing subscription | `Non-Renewing Subscription` | n/a | Apple gives no expiry date for these. See open questions. |

A refunded one-time purchase is reported by Apple's `REFUND` and by Google's `VoidedPurchaseNotification`. It removes the Entitlement the purchase unlocked. Google also sends `ONE_TIME_PRODUCT_PURCHASED` (1) and `ONE_TIME_PRODUCT_CANCELED` (2) for pending one-time purchases ([Google RTDN][google-rtdn]).

## Sandbox

- **Detection.** Apple transactions carry `environment` (`Sandbox`, `Production`, `Xcode`). Google purchases carry `testPurchase`. Both map to `is_sandbox`.
- **Apple time compression.** By default "1 month = 5 minutes", configurable per sandbox account. Subscriptions "automatically renew up to 12 times" before auto-renew turns off. The same rate "also determines the length of Billing Retry and Billing Grace Period" ([Apple sandbox account settings][apple-sandbox-settings]).
- **Google time compression.** License tester subscriptions renew on shortened intervals, a monthly plan every 5 minutes, and Play Billing Lab can "accelerate subscription state transition" or move a test subscription "into grace period or account hold" ([Google testing][google-test], [Google blog 2018][google-blog]).
- **Why it matters.** Sandbox subscriptions go through every state in minutes. OpenSubKit must not assume a state lasts long enough for a polling interval to catch it. Store notifications and on-demand refresh at post receipt are the reliable paths.

## Open questions for other tickets

- **Lifecycle coverage for v0.** Which rows are day-one requirements. A candidate cut is everything except Family Sharing, plan changes beyond "the Entitlement follows the latest access end", and non-renewing subscriptions.
- **Lifecycle coverage for v0.** How RevenueCat sets `period_type` for Apple promotional offers, offer codes, and win-back offers. Not found in primary sources. A rule that matches the table is `trial` for any free phase, `intro` for a discounted first phase, and `normal` otherwise. Confirm against a RevenueCat response in the Verification strategy ticket, or accept the rule.
- **Lifecycle coverage for v0.** Non-renewing subscriptions need an access duration from config, since Apple doesn't provide one. Or they're out of v0.
- **Apple purchase verification.** Whether Apple's grace period end appears in `expiresDate` or only in `gracePeriodExpiresDate`. The mapping above assumes only the latter and computes access end as the later of the two.
- **Google Play purchase verification.** Whether `expiryTime` already includes the grace period while `IN_GRACE_PERIOD`. Same rule, the later of the two.
- **Data model.** Store the raw store state and derive the CustomerInfo fields when serving, so a mapping fix doesn't need a data migration. `request_date` must be the server's current time.

[a-tree]: https://github.com/RevenueCat/purchases-android/tree/8d0a1775e62317a79b70f41da15e066005dc2c26
[i-tree]: https://github.com/RevenueCat/purchases-ios/tree/9b8a4efb7f36f6173329653845b7942ebfb234e0
[apple-lib]: https://github.com/apple/app-store-server-library-node/tree/bb0c0f874494321ea2d005329c3dc2188e893d41/models
[apple-status-ts]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/models/Status.ts
[apple-renewal]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/models/JWSRenewalInfoDecodedPayload.ts
[apple-transaction]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/models/JWSTransactionDecodedPayload.ts
[apple-notification-type]: https://developer.apple.com/documentation/appstoreservernotifications/notificationtype
[apple-status]: https://developer.apple.com/documentation/appstoreserverapi/status
[apple-sandbox]: https://developer.apple.com/documentation/storekit/testing-in-app-purchases-with-sandbox
[apple-sandbox-settings]: https://developer.apple.com/help/app-store-connect/test-in-app-purchases/manage-sandbox-apple-account-settings/
[google-v2]: https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptionsv2
[google-lifecycle]: https://developer.android.com/google/play/billing/lifecycle/subscriptions
[google-rtdn]: https://developer.android.com/google/play/billing/rtdn-reference
[google-test]: https://developer.android.com/google/play/billing/test
[google-blog]: https://android-developers.googleblog.com/2018/01/faster-renewals-for-test-subscriptions.html
[rc-grace]: https://www.revenuecat.com/docs/subscription-guidance/how-grace-periods-work
[rc-family]: https://www.revenuecat.com/docs/platform-resources/apple-platform-resources/apple-family-sharing
[rc-rest]: https://www.revenuecat.com/docs/api-v1/entitlements
[rc-export]: https://www.revenuecat.com/docs/integrations/scheduled-data-exports/data-export-version-5
[rc-google-guide]: https://www.revenuecat.com/guides/google-play-billing/the-subscription-state-machine
[a-entitlement-info]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/EntitlementInfoFactories.kt
[a-will-renew]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/utils/EntitlementInfoHelper.kt
[a-subscription-info]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/common/responses/SubscriptionInfoResponse.kt
[a-store]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/EntitlementInfo.kt
[i-active-dates]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Identity/CustomerInfo+ActiveDates.swift
[i-will-renew]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Purchasing/EntitlementInfo.swift#L360
