# Webhook and server-side customer API formats

Research for [Webhook and server-side customer API formats](https://github.com/LoriKarikari/opensubkit/issues/6). Sources are RevenueCat's public documentation, linked inline. Scope follows [Lifecycle coverage for v0](https://github.com/LoriKarikari/opensubkit/issues/11).

## Findings that shape the spec

1. **The server-side customer check is the SDK's own endpoint.** RevenueCat's REST v1 "Get or Create Customer" is `GET /v1/subscribers/{app_user_id}`. It "gets the latest Customer Info for the customer with the given App User ID, or creates a new customer if it doesn't exist" ([REST v1 Customers][rc-v1-customers]). That's the same path and response the SDK uses ([sdk-wire-contract.md](./sdk-wire-contract.md)). OpenSubKit supports it by accepting a secret key there too.
2. **A secret key adds `subscriber_attributes`.** Secret keys are "prefixed `sk_`", are "project-wide", and only responses to secret-key requests include `subscriber_attributes` ([API keys][rc-keys], [REST v1 reference][rc-v1-transactions]). A public SDK key gets the plain CustomerInfo.
3. **REST v2 is out of v0, and nothing needs it.** The SDK never calls v2, so the SDK wire contract is the only compatibility target. v2 is RevenueCat's newer server API under `https://api.revenuecat.com/v2`, with customer endpoints such as `GET /v2/projects/{project_id}/customers/{customer_id}` that return `active_entitlements` ([REST v2 customer][rc-v2-customer]). But it uses RevenueCat's own object IDs like `proj1ab2c3d4` and `entla1b2c3d4e5`, needs separate v2 keys ("API v1 keys will not work with REST API v2"), and is mostly configuration endpoints ([REST v2 overview][rc-v2]). RevenueCat staff say v1 `/subscribers` "is safe, it won't be deprecated" ([community answer][rc-v1-safe]). v2 is revisited when OpenSubKit's own admin API is designed.
4. **Webhooks are a single signed POST per event, at least once.** Delivery is `POST` with a JSON body. Only a 200 counts as success. Retries happen "up to 5 times" at 5, 10, 20, 40, and 80 minutes, with a 60-second response timeout ([Webhooks][rc-webhooks]). Retries "reuse the same `id` and `event_timestamp_ms`". Receivers are told to dedupe on `id` and to call `GET /subscribers` after any webhook instead of relying on event contents ([Webhooks][rc-webhooks], [Event types and fields][rc-events]).
5. **Webhooks need their own outbox.** Retries over about 2.5 hours, a manual retry button in RevenueCat's dashboard, and at-least-once delivery all imply stored events with delivery attempts. That's a table for the Data model ticket.

## REST v1 customer check

| | |
| --- | --- |
| Request | `GET /v1/subscribers/{app_user_id}`, App User ID URL-encoded |
| Auth | `Authorization: Bearer <key>`. A public SDK key or an `sk_` secret key for the Project. |
| Response | CustomerInfo as in [sdk-wire-contract.md](./sdk-wire-contract.md). With a secret key, `subscriber.subscriber_attributes` is added, keyed by attribute name, each `{"value", "updated_at_ms"}`. |
| Side effect | Creates the Customer if the App User ID is unknown |
| `management_url` | Picks the store matching the `X-Platform` header when the Customer has subscriptions on several stores. Otherwise the store of the subscription with the latest expiry ([REST v1 reference][rc-v1-transactions]). |

Attribute limits the SDK and API enforce on RevenueCat: at most 50 custom attributes per Customer, keys up to 40 characters, values up to 500 characters, and keys starting with `$` reserved ([Customer attributes][rc-attributes]).

## Webhook delivery

- **Configuration per Project.** Several webhook configurations are allowed. Each has an HTTPS URL, an optional `Authorization` header value, an environment filter (production, sandbox, or both), an App filter (one App or all), and an event type filter ([Webhooks][rc-webhooks]).
- **Authentication.** The configured `Authorization` header is sent on every request. Optional HMAC signing adds `X-RevenueCat-Webhook-Signature: t=<unix_timestamp>,v1=<hmac_sha256_hex>`, computed over `"<t>.<raw body>"` with a per-configuration secret. `t` and `v1` are recomputed on every attempt ([Webhooks][rc-webhooks]).
- **Success.** HTTP 200 only. "Any other status code will be considered a failure."
- **Retries.** Up to 5, at 5, 10, 20, 40, and 80 minutes. Then delivery stops. Retries reuse `id` and `event_timestamp_ms`.
- **Timeout.** 60 seconds.
- **Body.** `{"api_version": "1.0", "event": {...}}`.

## Event types for v0

From [Event types and fields][rc-events], limited to what v0 supports:

| Event | When OpenSubKit emits it |
| --- | --- |
| `TEST` | The self-hoster triggers a test delivery |
| `INITIAL_PURCHASE` | A new subscription is first seen |
| `RENEWAL` | A subscription renews or a lapsed Customer resubscribes. `is_trial_conversion` when the previous period was a free trial. |
| `CANCELLATION` | Auto-renew turned off, a billing issue detected (`cancel_reason` `BILLING_ERROR`), or a refund (`CUSTOMER_SUPPORT`) |
| `UNCANCELLATION` | Auto-renew turned back on before expiry |
| `NON_RENEWING_PURCHASE` | A consumable or non-consumable purchase |
| `SUBSCRIPTION_PAUSED` | A Google pause is scheduled. Access is not revoked yet. |
| `EXPIRATION` | Access End passes. `expiration_reason` is `UNSUBSCRIBE`, `BILLING_ERROR`, `DEVELOPER_INITIATED`, `PRICE_INCREASE`, `CUSTOMER_SUPPORT`, `UNKNOWN`, or `SUBSCRIPTION_PAUSED`. |
| `BILLING_ISSUE` | A renewal charge fails. Carries `grace_period_expiration_at_ms`. |
| `PRODUCT_CHANGE` | A plan change. `new_product_id` for Apple and deferred Google changes. |
| `REFUND_REVERSED` | Apple reverses a refund |
| `TRANSFER` | A purchase moves between Customers under Restore Behavior `transfer`. Sent for the destination only, with `transferred_from` and `transferred_to`. |

Not emitted in v0, because their features are out of scope: `SUBSCRIPTION_EXTENDED`, `TEMPORARY_ENTITLEMENT_GRANT`, `VIRTUAL_CURRENCY_TRANSACTION`, `EXPERIMENT_ENROLLMENT`, `PURCHASE_REDEEMED`, `PRICE_INCREASE_CONSENT_REQUIRED`, `PRICE_INCREASE_CONSENT_APPROVED`, and `INVOICE_ISSUANCE`. `SUBSCRIBER_ALIAS` is deprecated and "new projects don't receive this webhook".

## Event fields

Common to every event: `type`, `id`, `event_timestamp_ms`, `app_id`, plus `api_version` at the root.

Customer identity: `app_user_id` (last seen), `original_app_user_id`, `aliases` (all App User IDs), and `subscriber_attributes` when present.

Purchase fields, with what OpenSubKit can fill:

| Field | Source in OpenSubKit |
| --- | --- |
| `product_id` | Store product ID. For Google, RevenueCat uses `<subscription_id>:<base_plan_id>`. |
| `period_type` | Uppercase: `TRIAL`, `INTRO`, `NORMAL`, `PREPAID`. `PROMOTIONAL` is never emitted, since promotional grants are out of scope. |
| `purchased_at_ms`, `expiration_at_ms` | Store data. `expiration_at_ms` is `null` for one-time purchases. |
| `environment` | `SANDBOX` or `PRODUCTION` |
| `entitlement_ids` | From config. `entitlement_id` is deprecated but still sent. |
| `presented_offering_id` | From post receipt's `presented_offering_identifier` |
| `transaction_id`, `original_transaction_id` | Apple transaction IDs. Google order IDs, matching the `GPA.…` values RevenueCat's REST v1 example returns as `store_transaction_id` ([REST v1 reference][rc-v1-transactions]). The webhook samples only show placeholders, so confirm during verification. |
| `is_family_share` | Apple `inAppOwnershipType` |
| `store` | `APP_STORE` or `PLAY_STORE` |
| `country_code` | Apple `storefront` or Google `regionCode` |
| `currency`, `price_in_purchased_currency` | Store price data when available |
| `price` | USD price. `null` in v0, which RevenueCat's docs allow ("This can be `null` if unknown"), since OpenSubKit has no exchange rates. |
| `tax_percentage`, `commission_percentage`, `takehome_percentage` | `null` |
| `offer_code`, `renewal_number` | Store data when available |
| `grace_period_expiration_at_ms`, `auto_resume_at_ms`, `cancel_reason`, `expiration_reason`, `new_product_id`, `is_trial_conversion` | Only on their event types, as above |

Parsers on the receiving side are told to "parse defensively" and to expect new fields ([Event types and fields][rc-events]), so emitting `null` for unknown pricing is compatible.

## Open questions for other tickets

- **Data model.** A webhook configuration per Project (URL, auth header, optional HMAC secret, environment filter, App filter, event type filter). An event outbox with the event `id`, payload, and delivery attempts. Customer attributes with `updated_at_ms`.
- **Config file shape.** Webhook configurations and `sk_` secret keys per Project. Whether HMAC secrets live in the config file or in a secrets reference.
- **Not yet specified.** A way to replay a failed webhook, the equivalent of RevenueCat's dashboard Retry button, without a dashboard. It goes with the dashboard question.

[rc-v1-customers]: https://www.revenuecat.com/docs/api-v1/customers
[rc-v1-transactions]: https://www.revenuecat.com/docs/api-v1/transactions
[rc-v2]: https://www.revenuecat.com/docs/api-v2
[rc-v2-customer]: https://www.revenuecat.com/docs/api-v2/customer
[rc-v1-safe]: https://community.revenuecat.com/general-questions-7/not-getting-subscriptions-associated-to-a-customerid-4861
[rc-keys]: https://www.revenuecat.com/docs/projects/authentication
[rc-attributes]: https://www.revenuecat.com/docs/customers/customer-attributes
[rc-webhooks]: https://www.revenuecat.com/docs/integrations/webhooks
[rc-events]: https://www.revenuecat.com/docs/integrations/webhooks/event-types-and-fields
