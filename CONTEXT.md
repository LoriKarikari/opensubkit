# OpenSubKit

OpenSubKit is an open-source, self-hosted backend for in-app purchases and subscriptions that speaks the RevenueCat SDK's protocol. It uses RevenueCat's vocabulary so migrating developers keep the words they already know.

## Language

**App User ID**:
The identifier an app uses for one person, either assigned by the app or generated anonymously by the SDK.
_Avoid_: user ID, account ID

**Customer**:
The record OpenSubKit keeps for one person, holding their purchases and active Entitlements, looked up by App User ID.
_Avoid_: subscriber, user, account

**Product**:
A purchasable item defined in the App Store or Google Play, identified by its store product ID.
_Avoid_: SKU, item, plan

**Entitlement**:
A level of access, such as "pro", that one or more Products unlock.
_Avoid_: feature, permission, tier

**Offering**:
A named set of Packages an app presents to a Customer as a purchase choice.
_Avoid_: paywall, plan set

**Package**:
One slot in an Offering, such as monthly or annual, holding the matching Product for each store.
_Avoid_: option, plan
