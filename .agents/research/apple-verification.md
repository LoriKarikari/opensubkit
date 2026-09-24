# Apple purchase verification and server notifications

Research for [Apple purchase verification and server notifications](https://github.com/LoriKarikari/opensubkit/issues/3). It builds on [`sdk-wire-contract.md`](./sdk-wire-contract.md) and [`subscription-lifecycle.md`](./subscription-lifecycle.md).

Sources, abbreviated below:

- **Apple lib**: `apple/app-store-server-library-node` v3.1.0 at [`bb0c0f8`][lib-tree], Apple's official Node library, MIT-licensed.
- **Apple docs**: the App Store Server API and App Store Server Notifications reference, linked inline.
- **SDK-I**: `RevenueCat/purchases-ios` at [`9b8a4ef`][i-tree].

## Findings that shape the spec

1. **Apple's Node library covers everything v0 needs.** It has the API client, JWS verification against Apple's root certificates, notification decoding, and app receipt parsing. OpenSubKit should depend on it instead of reimplementing any of it ([Apple lib `index.ts`][lib-index], [`jws_verification.ts`][lib-jws], [`receipt_utility.ts`][lib-receipt]).
2. **One ID is enough to rebuild a Customer's history.** Get Transaction History and Get All Subscription Statuses accept "any `originalTransactionId`, `transactionId` or `appTransactionId` that belongs to the customer for your app" ([Get Transaction History][doc-history]). So the StoreKit 2 restore case from the wire contract works. If the SDK posts only `app_transaction`, its `appTransactionId` still returns the full history.
3. **Never accept Xcode-environment data in production.** Apple's verifier skips signature checks for the `Xcode` and local-testing environments, because "Data is not signed by the App Store" ([Apple lib `jws_verification.ts#L212`][lib-xcode]). The API client refuses the Xcode environment outright ([Apple lib `index.ts#L221`][lib-client-env]). The SDK sends Xcode purchases as its own JSON receipt anyway ([sdk-wire-contract.md](./sdk-wire-contract.md)). Accepting them makes forged purchases trivial.
4. **Grace period end is separate from `expiresDate`.** `expiresDate` "is a static value that applies for each transaction" and renewal creates a new transaction ([expiresDate][doc-expires]). The grace end is only in the renewal info's `gracePeriodExpiresDate` ([gracePeriodExpiresDate][doc-grace]). Access end is the later of the two, as [`subscription-lifecycle.md`](./subscription-lifecycle.md) assumed.
5. **The notification endpoint must return non-2xx on failure.** Apple treats 200 to 206 as success and retries on 40x and 50x, "five times, at 1, 12, 24, 48, and 72 hours". Sandbox gets one attempt and no retries ([Responding to notifications][doc-responding]). As with post receipt, failures on OpenSubKit's side must not return 2xx.
6. **The SDK sets `appAccountToken` only for UUID App User IDs.** iOS attaches `appAccountToken(uuid)` when the App User ID parses as a UUID ([SDK-I `PurchasesOrchestrator.swift#L844`][i-app-account-token]). A notification for a purchase OpenSubKit hasn't seen yet can be matched to a Customer that way, but only for those Customers.

## What the SDK posts, and how to verify each form

From [`sdk-wire-contract.md`](./sdk-wire-contract.md), `fetch_token` holds one of four things.

| SDK sends | When | Verification |
| --- | --- | --- |
| Signed transaction JWS | StoreKit 2, the iOS 15+ default | `SignedDataVerifier.verifyAndDecodeTransaction`. Then use its `transactionId` for history and statuses. |
| `app_transaction` JWS only | StoreKit 2 restore with no transactions | `verifyAndDecodeAppTransaction`. Then query history with its `appTransactionId`. |
| Base64 app receipt | StoreKit 1 path | `ReceiptUtility.extractTransactionIdFromAppReceipt` pulls one transaction ID with "NO validation". The API call that follows is the verification. |
| Base64 JSON receipt | Xcode StoreKit testing only | Unsigned. Reject unless the Project explicitly allows local testing (finding 3). |

The legacy `verifyReceipt` endpoint isn't needed. Its documentation page no longer resolves in Apple's docs API.

## App Store Server API

- **Base URLs.** `https://api.storekit.apple.com` for production, `https://api.storekit-sandbox.apple.com` for sandbox ([Apple lib `index.ts#L195`][lib-urls]).
- **Auth.** A bearer JWT signed ES256 with the In-App Purchase key. Header `kid` is the key ID. Claims are `iss` (issuer ID), `aud` `appstoreconnect-v1`, `bid` (bundle ID), and a short expiry. The library uses 5 minutes ([Apple lib `index.ts#L716`][lib-jwt]).
- **Endpoints v0 uses** ([Apple lib `index.ts`][lib-index]):

| Endpoint | Path | Use |
| --- | --- | --- |
| Get Transaction Info | `GET /inApps/v1/transactions/{transactionId}` | Verify one transaction from Apple's side. |
| Get Transaction History v2 | `GET /inApps/v2/history/{anyTransactionId}` | Rebuild a Customer's purchases, 20 per page with a `revision` token. |
| Get All Subscription Statuses | `GET /inApps/v1/subscriptions/{anyTransactionId}` | Current status and latest renewal info per subscription group. |
| Get App Transaction Info | `GET /inApps/v1/transactions/appTransactions/{anyTransactionId}` | The app-level transaction, which carries `appTransactionId`. |
| Get Refund History | `GET /inApps/v2/refund/lookup/{anyTransactionId}` | Recover refunds missed during an outage. |
| Get Notification History | `POST /inApps/v1/notifications/history` | Replay notifications missed during an outage. |
| Request a Test Notification | `POST /inApps/v1/notifications/test` | Setup check for self-hosters. |

- **Rate limits.** Per app, enforced hourly. 50 requests per second for transaction info, history, statuses, app transaction info, and finish transaction. A `429` carries `Retry-After` as a UNIX time in milliseconds ([Identifying rate limits][doc-rate-limits]).
- **Environment routing.** Each signed payload carries `environment` (`Sandbox`, `Production`, `Xcode`). The verifier rejects a payload whose environment doesn't match its own ([Apple lib `jws_verification.ts#L95`][lib-env-check]). OpenSubKit needs one verifier and one API client per environment, and picks by the decoded `environment` field. TestFlight builds use sandbox.

## Signed data verification

- The library's `SignedDataVerifier` takes Apple's root certificates, a flag for online checks, the environment, the bundle ID, and the app Apple ID ([Apple lib `jws_verification.ts#L70`][lib-verifier]).
- It checks the `x5c` chain of 3 certificates up to Apple's root, the bundle ID, the environment, and in production the app Apple ID.
- **Root certificates** come from the Apple Root Certificates section of [Apple PKI](https://www.apple.com/certificateauthority/) ([Apple lib `README.md`][lib-readme]). They are public. OpenSubKit can ship them in the image.
- **Online checks** enable revocation checking and check expiry against the current date. Turn them on in production.
- **`appAppleId`** is required when the environment is production and ignored in sandbox.

## App Store Server Notifications v2

- **Setup.** An HTTPS URL per environment, configured in App Store Connect. The same URL may serve both. Version 2 must be selected. TLS 1.2 or later ([Enabling notifications][doc-enabling]).
- **Payload.** A JSON body `{"signedPayload": "<JWS>"}`. Decoded with `verifyAndDecodeNotification`, it carries `notificationType`, `subtype`, `notificationUUID`, `signedDate`, and `data` with `appAppleId`, `bundleId`, `environment`, `signedTransactionInfo`, `signedRenewalInfo`, and `status` ([Apple lib `jws_verification.ts#L126`][lib-notification]).
- **Types that change access** are mapped in [`subscription-lifecycle.md`](./subscription-lifecycle.md). The full list is in [Apple lib `NotificationTypeV2.ts`][lib-types].
- **Idempotency.** Retries resend the same `notificationUUID`. Process each one once.
- **Ordering.** Delivery order isn't guaranteed. After a notification, reload the subscription's current status from the API instead of applying the notification as a diff.
- **Missed notifications.** Sandbox never retries, and an outage in production can outlast the 72-hour tail. Get Notification History and on-demand refresh at post receipt cover both.

## Credentials a self-hoster provides per App Store App

| Credential | Where it comes from | Used for |
| --- | --- | --- |
| Bundle ID | the app | JWT `bid` claim, payload checks |
| App Apple ID | App Store Connect, App Information | production payload checks |
| Issuer ID | App Store Connect, Users and Access, Integrations, In-App Purchase keys | JWT `iss` claim |
| Key ID and private key (`.p8`) | same page, downloaded once | JWT signing |
| Notification URLs | set by the self-hoster in App Store Connect, pointing at OpenSubKit | receiving notifications |

The `.p8` key is a secret. The rest are identifiers. No App Store Connect API key or shared secret is needed.

## Sandbox testing

- Sandbox Apple accounts, the sandbox API base URL, and payloads with `environment` `Sandbox`.
- Time compression and the 12-renewal cap are covered in [`subscription-lifecycle.md`](./subscription-lifecycle.md).
- Request a Test Notification confirms the notification URL works.

## Open questions for other tickets

- **Config file shape.** Per App Store App: bundle ID, app Apple ID, issuer ID, key ID, and a path or secret reference for the `.p8`. A flag to allow Xcode local testing, off by default.
- **Data model.** Key App Store purchases by `originalTransactionId`, with `appTransactionId` stored per Customer. Keep processed `notificationUUID`s for idempotency.
- **Identity rules follow-up.** `appAccountToken` equals the App User ID when that ID is a UUID. This partly answers the "not yet specified" item on matching notification-first purchases.
- **Verification strategy.** Test with a sandbox account, a TestFlight build, and Request a Test Notification. Include a restore on a fresh install to exercise the `appTransactionId` path.

[lib-tree]: https://github.com/apple/app-store-server-library-node/tree/bb0c0f874494321ea2d005329c3dc2188e893d41
[lib-index]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/index.ts
[lib-urls]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/index.ts#L195
[lib-client-env]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/index.ts#L221
[lib-jwt]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/index.ts#L716
[lib-jws]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/jws_verification.ts
[lib-verifier]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/jws_verification.ts#L70
[lib-env-check]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/jws_verification.ts#L95
[lib-notification]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/jws_verification.ts#L126
[lib-xcode]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/jws_verification.ts#L212
[lib-receipt]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/receipt_utility.ts
[lib-readme]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/README.md
[lib-types]: https://github.com/apple/app-store-server-library-node/blob/bb0c0f874494321ea2d005329c3dc2188e893d41/models/NotificationTypeV2.ts
[i-tree]: https://github.com/RevenueCat/purchases-ios/tree/9b8a4efb7f36f6173329653845b7942ebfb234e0
[i-app-account-token]: https://github.com/RevenueCat/purchases-ios/blob/9b8a4efb7f36f6173329653845b7942ebfb234e0/Sources/Purchasing/Purchases/PurchasesOrchestrator.swift#L844
[doc-history]: https://developer.apple.com/documentation/appstoreserverapi/get-transaction-history
[doc-expires]: https://developer.apple.com/documentation/appstoreserverapi/expiresdate
[doc-grace]: https://developer.apple.com/documentation/appstoreserverapi/graceperiodexpiresdate
[doc-responding]: https://developer.apple.com/documentation/appstoreservernotifications/responding-to-app-store-server-notifications
[doc-enabling]: https://developer.apple.com/documentation/appstoreservernotifications/enabling-app-store-server-notifications
[doc-rate-limits]: https://developer.apple.com/documentation/appstoreserverapi/identifying-rate-limits
