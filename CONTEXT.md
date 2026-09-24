# OpenSubKit

OpenSubKit is an open-source, self-hosted backend for in-app purchases and subscriptions, compatible with the existing SDK. It uses that SDK's vocabulary so migrating developers keep the words they already know.

## Language

**Project**:
A set of Apps that share Customers, Entitlements, and Offerings. One install holds any number of Projects.
_Avoid_: workspace, tenant, organization

**App**:
One app in one store, the App Store or Google Play, belonging to a Project and holding its own public SDK key.
_Avoid_: platform, client

**App User ID**:
The identifier an app uses for one person, either assigned by the app or generated anonymously by the SDK.
_Avoid_: user ID, account ID

**Anonymous App User ID**:
An App User ID the SDK generates when the app hasn't identified the person, shaped `$RCAnonymousID:` plus 32 hex characters.
_Avoid_: guest ID, temporary ID

**Identified App User ID**:
An App User ID the app assigned itself, usually its own account ID.
_Avoid_: custom ID, real ID, logged-in ID

**Customer**:
The record OpenSubKit keeps for one person, holding their purchases and active Entitlements, looked up by any of its App User IDs.
_Avoid_: subscriber, user, account

**Original App User ID**:
The oldest App User ID of a Customer. The others are its Aliases.

**Alias**:
An App User ID that belongs to a Customer alongside its Original App User ID.
_Avoid_: linked ID, secondary ID

**Merge**:
Combining two Customers into one, so every App User ID of either resolves to the same purchases. Never undone.
_Avoid_: link, join, alias (as a verb)

**Transfer**:
Moving a purchase from one Customer to another, so the previous Customer loses access.
_Avoid_: reassign, move

**Restore Behavior**:
A Project's rule for a purchase that already belongs to a different Identified App User ID, either Transfer or keep with the original Customer.
_Avoid_: transfer behavior, restore policy

**Product**:
A purchasable item defined in the App Store or Google Play, identified by its store product ID.
_Avoid_: SKU, item, plan

**Entitlement**:
A level of access, such as "pro", that one or more Products unlock.
_Avoid_: feature, permission, tier

**Access End**:
The moment a Customer's Entitlement stops being active, derived from the store's state for the purchase that grants it. Grace periods extend it, refunds and pauses pull it back.
_Avoid_: expiry, expiration date, end date

**Offering**:
A named set of Packages an app presents to a Customer as a purchase choice.
_Avoid_: paywall, plan set

**Package**:
One slot in an Offering, such as monthly or annual, holding the matching Product for each store.
_Avoid_: option, plan
