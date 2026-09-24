# RevenueCat's data model, reverse-engineered

Input for [Data model](https://github.com/LoriKarikari/opensubkit/issues/13). RevenueCat's database schema isn't public, but four public surfaces describe the same underlying model. This file distills them into one entity model that OpenSubKit can copy.

## Sources

- **REST v2 OpenAPI spec.** RevenueCat publishes it per section at `https://www.revenuecat.com/docs/redocusaurus/openapi-v2-<section>.yaml`. 21 sections, 179 component schemas, downloaded 2026-09-24. Sections: app, audience, audit-log, charts-and-metrics, collaborator, customer, discount, entitlement, iam, integration, invoice, offering, package, paywall, product, project, purchase, subscription, subscription-data-model, subscription-transactions, virtual-currency. Linked from the [v2 reference][rc-v2].
- **CustomerInfo**, the SDK and REST v1 response. See [`sdk-wire-contract.md`](./sdk-wire-contract.md).
- **Webhook events.** See [`webhook-formats.md`](./webhook-formats.md).
- **Scheduled data exports**, one row per transaction ([Scheduled data exports][rc-exports]).

## Entities

### Configuration

| Entity | Key fields in v2 | Notes |
| --- | --- | --- |
| Project | `id`, `name` | |
| App | `id`, `name`, `type` (`app_store`, `play_store`, and others), `project_id`. `app_store`: `bundle_id`, `subscription_key_configured`, `app_store_connect_api_key_configured`. `play_store`: `package_name`, `play_service_account_credentials_configured`. | Credentials are stored per App. The API only reports whether they're configured. |
| Product | `id`, `store_identifier`, `type` (`subscription`, `one_time`, `consumable`, `non_consumable`, `non_renewing_subscription`), `app_id`, `display_name`, `subscription` (`duration`, `grace_period_duration`, `trial_duration` as ISO 8601), `one_time.is_consumable` | Belongs to one App. The consumable flag lives here. |
| Entitlement | `id`, `lookup_key`, `display_name`, `products` | `lookup_key` is the name the SDK shows (`pro`). `id` is internal (`entla1b2c3d4e5`). |
| Offering | `id`, `lookup_key`, `display_name`, `is_current`, `metadata`, `packages` | `is_current` backs `current_offering_id`. |
| Package | `id`, `lookup_key`, `display_name`, `position`, `products` (each with `eligibility_criteria`) | One Product per App. |
| WebhookIntegration | `id`, `name`, `url`, `authorization_header`, `environment`, `event_types`, `app_id`, `signing_secret` | Matches [`webhook-formats.md`](./webhook-formats.md). |

### Customer data

| Entity | Key fields in v2 | Notes |
| --- | --- | --- |
| Customer | `id`, `project_id`, `first_seen_at`, `last_seen_at`, `last_seen_app_version`, `last_seen_country`, `last_seen_platform`, `last_seen_platform_version` | `id` is the Original App User ID. The `last_seen_*` fields come from SDK request headers. |
| CustomerAlias | `id`, `created_at` | Every App User ID of the Customer. |
| CustomerAttribute | `name`, `value`, `updated_at` | |
| Subscription | `id`, `customer_id`, `original_customer_id`, `product_id`, `store`, `store_subscription_identifier`, `environment` (`production`, `sandbox`), `ownership` (`purchased`, `family_shared`), `starts_at`, `current_period_starts_at`, `current_period_ends_at`, `ends_at`, `gives_access`, `status`, `auto_renewal_status`, `pending_changes`, `presented_offering_id`, `country`, `management_url`, `total_revenue_in_usd`, `entitlements` | Auto-renewing only. `original_customer_id` differs from `customer_id` after a Transfer. |
| Purchase | `id`, `customer_id`, `original_customer_id`, `product_id`, `store`, `store_purchase_identifier`, `environment`, `ownership`, `purchased_at`, `quantity`, `status` (`owned`, `refunded`), `presented_offering_id`, `country`, `revenue_in_usd`, `entitlements` | One-time purchases only. |
| SubscriptionTransaction | `id`, `purchased_at`, `product_store_identifier`, `expiration_date`, `effective_expiration_date`, `revenue_in_local_currency`, `revenue_in_usd` | One per billing period. The data export adds `store_transaction_id`, `original_store_transaction_id`, `is_trial_period`, `is_in_intro_offer_period`, `refunded_at`, `unsubscribe_detected_at`, `billing_issues_detected_at`, `renewal_number`, `is_trial_conversion`, `offer`, `offer_type`, `ownership_type`, `grace_period_end_time`, `effective_end_time`, `entitlement_identifiers` ([Scheduled data exports][rc-exports]). |
| AuditLog | `action_type`, `target_type`, `target_identifier`, `actor_type`, `actor_identifier`, `occurred_at`, `additional_data` | Configuration changes, not identity changes. |

### Enums worth copying

- **Subscription `status`:** `trialing`, `active`, `expired`, `in_grace_period`, `in_billing_retry`, `paused`, `unknown`, `incomplete`.
- **Subscription `auto_renewal_status`:** `will_renew`, `will_not_renew`, `will_change_product`, `will_pause`, `requires_price_increase`.
- **Purchase `status`:** `owned`, `refunded`.

## What the model tells us

1. **Subscriptions and one-time purchases are separate entities.** They have different lifecycles and different fields (`status` enums, `quantity`, `current_period_*`).
2. **Transfers are a column, not only an event.** `original_customer_id` stays with the first owner. `customer_id` moves.
3. **Access is stored as a boolean plus an end date.** `gives_access` and `ends_at` on the Subscription, `effective_expiration_date` per transaction. These are what OpenSubKit calls Access End.
4. **Configuration objects have two identifiers.** An internal `id` and the `lookup_key` developers use. OpenSubKit's config keys play the role of `lookup_key`.
5. **Customer ID is the Original App User ID.** Aliases hang off it.

## Out of scope parts of the model

Audiences, charts, collaborators, discounts, IAM, invoices, paywalls, virtual currencies, and Web Billing redemptions. They exist in the spec and are outside v0.

[rc-v2]: https://www.revenuecat.com/docs/api-v2
[rc-exports]: https://www.revenuecat.com/docs/integrations/scheduled-data-exports
