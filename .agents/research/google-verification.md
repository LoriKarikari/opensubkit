# Google Play purchase verification and real-time notifications

Research for [Google Play purchase verification and real-time notifications](https://github.com/LoriKarikari/opensubkit/issues/4). It builds on [`sdk-wire-contract.md`](./sdk-wire-contract.md) and [`subscription-lifecycle.md`](./subscription-lifecycle.md).

Sources are Google's documentation, linked inline, plus `RevenueCat/purchases-android` at [`8d0a177`][a-tree] (**SDK-A**).

## Findings that shape the spec

1. **The purchase token is the primary key.** "`purchaseToken` is globally unique, so you can safely use this value as a primary key in your database." Verify every token with the Play Developer API before granting access ([Fight fraud and abuse][g-security]).
2. **`expiryTime` already includes the grace period.** "Google Play dynamically extends the `expiryTime` value until the grace period has expired." During account hold `expiryTime` "is set to a past timestamp" ([Subscription lifecycle][g-lifecycle]). On Google, access end is `expiryTime` alone.
3. **Plan changes retire the old token.** Upgrades, downgrades, resubscribes before expiry, and prepaid top-ups create a new token whose `linkedPurchaseToken` names the old one. Google says to "remove the linkedPurchaseToken from your database and revoke the entitlement that is granted to the linkedPurchaseToken to ensure that multiple users are not entitled for the same purchase" ([Fight fraud and abuse][g-security]).
4. **A resubscribe after expiry has no link.** It's a new token with no `linkedPurchaseToken` ([Subscription lifecycle][g-lifecycle]). The SDK sets `obfuscatedAccountId` to the SHA-256 of the App User ID ([SDK-A `BillingWrapper.kt#L981`][a-obfuscated]), and the API returns it as `externalAccountIdentifiers.obfuscatedExternalAccountId`. So OpenSubKit can match these tokens to a Customer when it hashes the App User IDs it knows.
5. **The SDK acknowledges, OpenSubKit usually doesn't have to.** Unacknowledged purchases are refunded after three days ([Integrate][g-integrate]). With default settings the SDK acknowledges or consumes after a successful post receipt ([sdk-wire-contract.md](./sdk-wire-contract.md)). A purchase made outside the app, such as a Play Store resubscribe or a promo code, is only acknowledged when the app next opens and posts it.
6. **The notification endpoint must return non-2xx on failure.** Pub/Sub push treats 102, 200, 201, 202, and 204 as acknowledged. Anything else is redelivered ([Pub/Sub push][ps-push]).

## What the SDK posts

`fetch_token` is the Play purchase token, plus `product_ids`, `platform_product_ids`, pricing, and `proration_mode` for plan changes ([sdk-wire-contract.md](./sdk-wire-contract.md)). The SDK doesn't say whether a token is a subscription or a one-time purchase. The Product's type in config does.

## Play Developer API

Base URL `https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{packageName}`, OAuth scope `https://www.googleapis.com/auth/androidpublisher`.

| Call | Use |
| --- | --- |
| `GET purchases/subscriptionsv2/tokens/{token}` | Subscription state. `subscriptionState`, `lineItems[].expiryTime`, `autoRenewingPlan` or `prepaidPlan`, `offerPhase`, `linkedPurchaseToken`, `acknowledgementState`, `testPurchase`, `externalAccountIdentifiers` ([subscriptionsv2][g-v2]). |
| `GET purchases/products/{productId}/tokens/{token}` | One-time purchase. `purchaseState` (0 purchased, 1 canceled, 2 pending), `consumptionState`, `acknowledgementState`, `purchaseType` (0 test, 1 promo, 2 rewarded), `obfuscatedExternalAccountId`, `quantity`, `orderId` ([purchases.products][g-products]). |
| `POST purchases/subscriptions/{subscriptionId}/tokens/{token}:acknowledge` and `purchases/products/{productId}/tokens/{token}:acknowledge` | Server-side acknowledgement, only for the case in finding 5. |
| `GET purchases/voidedpurchases?type=1` | Refunds, cancellations, and chargebacks from the last 30 days at most. Each entry has `purchaseToken`, `orderId`, `voidedTimeMillis`, `voidedReason`, `voidedSource`, `voidedQuantity` ([voidedpurchases.list][g-voided]). |

- **Quotas.** Separate buckets for Subscriptions, One-time Purchases, and Orders (which holds Voided Purchases). 3000 queries per minute each by default ([Quotas][g-quotas]).
- **Token lifetime.** A subscription token works with the API "from subscription signup until 60 days after expiration" ([Subscription lifecycle][g-lifecycle]). Refreshing old history later isn't possible, so store what the API returns.
- **Client.** `@googleapis/androidpublisher` for the API and `google-auth-library` for service account auth, both Apache-2.0 and from Google ([npm][npm-publisher], [npm][npm-auth]).

## Real-time developer notifications

RTDN goes through Cloud Pub/Sub in the self-hoster's own Google Cloud project ([Getting ready][g-ready]).

1. Create a topic. Grant `google-play-developer-notifications@system.gserviceaccount.com` the **Pub/Sub Publisher** role on it.
2. Create a subscription on the topic, push or pull.
3. In Play Console, **Monetize > Monetization setup**, enable real-time notifications with the topic name `projects/{project_id}/topics/{topic_name}`. Choose "Get all notifications for subscriptions and one-time products" so pending one-time purchases are reported too. **Send Test Message** checks the setup.

**Push versus pull.**

- **Push** POSTs `{"message": {"data": "<base64>", "messageId", "publishTime"}, "subscription"}` to an HTTPS endpoint ([Pub/Sub push][ps-push]). OpenSubKit already needs a public HTTPS endpoint for the SDK, so push adds nothing to host. Authenticate it with an OIDC token. Pub/Sub signs a JWT in the `Authorization` header, and OpenSubKit verifies its signature, `email`, and `aud` ([Pub/Sub push authentication][ps-auth]). `google-auth-library`'s `verifyIdToken` does this.
- **Pull** works behind NAT but needs Pub/Sub credentials and a long-running consumer (`@google-cloud/pubsub`).

The decoded `data` is a `DeveloperNotification` with `packageName`, `eventTimeMillis`, and exactly one of `subscriptionNotification`, `oneTimeProductNotification`, `voidedPurchaseNotification`, `pendingRefundReviewNotification`, or `testNotification` ([RTDN reference][g-rtdn]). The notification carries only the token and a type. Google's guidance is to call the API for current state every time. The type numbers that change access are mapped in [`subscription-lifecycle.md`](./subscription-lifecycle.md).

**Delivery.** Pub/Sub redelivers until acknowledged and backs off after repeated failures ([Pub/Sub push][ps-push]). Treat `messageId` as the dedupe key and don't rely on ordering. Reload from the API on every notification.

## Credentials a self-hoster provides per Google Play App

| Credential | Where it comes from | Used for |
| --- | --- | --- |
| Package name | the app | API paths, notification checks |
| Service account JSON key | Google Cloud project, service account linked in Play Console with the **View financial data** permission ([Getting ready][g-ready]) | Play Developer API |
| Pub/Sub topic and subscription | the same Google Cloud project | RTDN |
| Push auth audience and service account email | the push subscription's auth settings | verifying push requests |

## Sandbox testing

- License testers make test purchases. `testPurchase` is present on subscriptions and `purchaseType` is 0 on one-time purchases. Both map to `is_sandbox`.
- The app must be published to a track, and the internal test track is enough ([Getting ready][g-ready]).
- Renewal compression and Play Billing Lab are covered in [`subscription-lifecycle.md`](./subscription-lifecycle.md).

## Open questions for other tickets

- **Config file shape.** Per Google Play App: package name, a reference to the service account JSON, Pub/Sub mode (push or pull), and the push auth audience. A consumable flag per Product, already noted.
- **Data model.** Key Google purchases by purchase token. Link tokens through `linkedPurchaseToken` and mark the old one superseded. Store the SHA-256 of each App User ID for matching `obfuscatedExternalAccountId`. Keep processed Pub/Sub `messageId`s.
- **Lifecycle coverage for v0.** Whether OpenSubKit acknowledges purchases that arrive by notification before the app posts them. Acknowledging stops the three-day auto-refund even if no Customer ever claims the purchase. Not acknowledging risks refunding a real purchase the user made from the Play Store.
- **Verification strategy.** Use a license tester, an internal testing track build, Play Console's Send Test Message, and a Play Store resubscribe to exercise the unlinked-token path.

[a-tree]: https://github.com/RevenueCat/purchases-android/tree/8d0a1775e62317a79b70f41da15e066005dc2c26
[a-obfuscated]: https://github.com/RevenueCat/purchases-android/blob/8d0a1775e62317a79b70f41da15e066005dc2c26/purchases/src/main/kotlin/com/revenuecat/purchases/google/BillingWrapper.kt#L981
[g-security]: https://developer.android.com/google/play/billing/security
[g-lifecycle]: https://developer.android.com/google/play/billing/lifecycle/subscriptions
[g-integrate]: https://developer.android.com/google/play/billing/integrate
[g-ready]: https://developer.android.com/google/play/billing/getting-ready
[g-rtdn]: https://developer.android.com/google/play/billing/rtdn-reference
[g-v2]: https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptionsv2
[g-products]: https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.products
[g-voided]: https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.voidedpurchases/list
[g-quotas]: https://developers.google.com/android-publisher/quotas
[ps-push]: https://cloud.google.com/pubsub/docs/push
[ps-auth]: https://cloud.google.com/pubsub/docs/authenticate-push-subscriptions
[npm-publisher]: https://www.npmjs.com/package/@googleapis/androidpublisher
[npm-auth]: https://www.npmjs.com/package/google-auth-library
