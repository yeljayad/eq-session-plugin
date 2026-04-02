---
name: Firestore Database Schema
description: Complete Firestore collection hierarchy with all fields and types — use when building any feature that reads/writes Firestore data (business-crud, transactions, integrations, users, etc.)
type: reference
---

Raw schema file: `docs/firestore-schema.json` (18K lines, exported from Firestore).

## Collection Hierarchy

```
owner/{ownerId}                                          — Root tenant
├── account                                              — Owner-level accounts
├── alert                                                — Security alerts
├── app_version                                          — App version tracking
├── card                                                 — Owner-level cards
├── customer                                             — Owner-level customers
├── retailer                                             — Owner-level retailers (flattened view)
├── service                                              — Owner-level services
├── transaction                                          — Owner-level transactions (flattened view)
├── user                                                 — Users (Firebase Auth UID as doc ID)
│   └── {userId}/
│       ├── customer                                     — User-customer assignments
│       ├── space                                        — User-space assignments
│       └── workspace                                    — User-workspace assignments
├── user_profile                                         — Permission profiles (menu-1, menu-2, etc.)
│   └── {profileId}/users_profils                        — Profile membership
└── workspace/{workspaceId}                              — Workspaces
    ├── account                                          — Workspace-level accounts
    ├── activity                                         — Audit log
    ├── card                                             — Workspace-level cards
    ├── customer                                         — Workspace-level customers
    ├── integration1/{integrationId}                     — Payment integrations
    │   └── partner/{partnerId}
    │       └── service                                  — Integration services
    ├── retailer                                         — Workspace-level retailers
    ├── routing_table                                    — Transaction routing rules
    ├── service                                          — Workspace-level services
    ├── task                                             — Async tasks
    ├── transaction                                      — Workspace-level transactions
    ├── user                                             — Workspace-level users
    └── space/{spaceId}                                  — Spaces
        ├── account                                      — Space-level accounts
        ├── card                                         — Space-level cards
        ├── customer                                     — Space-level customers
        │   └── {customerId}/
        │       ├── account                              — Customer accounts
        │       └── card                                 — Customer cards
        ├── retailer/{retailerId}
        │   ├── account/{accountId}/
        │   │   ├── reconciliation                       — Settlement reconciliation
        │   │   ├── statement                            — Bank statements
        │   │   ├── statement_transaction                — Statement line items
        │   │   └── transaction                          — Account transactions
        │   ├── apple-merchant-identity-certificate      — Apple Pay merchant cert
        │   ├── apple-processing-certificate             — Apple Pay processing cert
        │   ├── customer                                 — Retailer customers
        │   ├── integration1/{integrationId}             — Retailer integrations
        │   │   └── partner/{partnerId}/
        │   │       ├── mop/{mopName}/                   — Methods of payment
        │   │       │   ├── scheme/{schemeName}/service  — Card scheme services
        │   │       │   ├── service                      — MOP services
        │   │       │   └── supported_currencies         — Crypto currencies
        │   │       └── service                          — Partner services
        │   ├── routing_rule1                            — Retailer routing rules
        │   ├── service                                  — Retailer services
        │   ├── transaction                              — Retailer transactions
        │   └── user                                     — Retailer users
        ├── service                                      — Space-level services
        ├── transaction                                  — Space-level transactions
        └── user                                         — Space-level users
```

## Key Design Patterns

1. **Multi-level scoping**: Collections (account, card, customer, retailer, transaction, service, user) exist at MULTIPLE levels — owner, workspace, space, and nested under retailer/customer. The same schema repeats at each level.

2. **Flattened owner-level copies**: Owner-level `transaction`, `retailer`, `customer`, etc. are denormalized copies for cross-workspace queries.

3. **Reference fields**: `workspace_ref`, `space_ref`, `retailer_ref`, `customer_ref`, `original_*_ref` are Firestore document references linking entities across the hierarchy.

4. **Common field patterns across entities**:
   - Identity: `id` (business ID), `name`, `name_lower_case`
   - Status: `status_name`, `validation_status_name`, `archived`
   - Currency: `currency_name`, `currency_alphabetic_code`, `currency_numeric_code`, `currency_minor_unit`
   - Country: `country_name`, `country_alpha2_code`, `country_alpha3_code`, `country_numeric_code`
   - Capacity: `max_number_of_users`, `current_number_of_users`

5. **Integration hierarchy**: `integration1/{name}/partner/{name}/mop/{type}/scheme/{brand}/service` — deep nesting for payment provider config (MPGS, BVNK, D24, Equiti).

## Entity Field Details

### owner/{ownerId}
application_name, city, company_name, country(ref), currency_*, email, key_index, logo, pbl_url, postal_code, price(ref), state_province_region, street_address, template_ref(ref), user_ref(ref), web_site

### account (all levels)
account_owner, account_type_name, archived, balance(number), currency_*, customer_ref(ref), id, is_customer_user_account(bool), original_account_ref(ref), space_ref(ref), status_name, validation_status_name, workspace_ref(ref)

### alert
alert_status_name, date(number), integration_id, mid_count(number), mids(array), retailer_id, retailer_name, retailer_ref(ref), security_alert_*, space_*, transaction_count(number), transaction_*_date, workspace_*

### card (all levels)
archived, customer_ref(ref), encrypted_pan, expiry_date, expiry_date_mm, expiry_date_yy(number|string), gateway_merchant_id, id, name_on_card, original_card_ref(ref), scheme, space_ref(ref), status_name, token, uuid, validation_status_name, workspace_ref(ref)

### customer (all levels)
archived, avatar(null), city_name, country_*, currency_*, current_number_of_users(number), id, location, max_number_of_users(number), name, name_lower_case, original_customer_ref(ref), retailer_id, retailer_ref(ref), space_ref(ref), status_name, validation_status_name, workspace_ref(ref), zip_code

### retailer (workspace & space level)
apple_merchant_id, archived, avatar, business_phone, card_acceptor_business_code_mcc, card_acceptor_business_code_name, cart_activated(bool), commercial_name, company_name, country_*, css_file_path, currency_*, current_number_of_users(number), default_terminal_reader_*, directors, disable_status_page(bool), geolocation, google_merchant_id, google_public_key, id, license_number, location, max_number_of_users(number), name, name_lower_case, original_retailer_ref(ref), postal_code, shareholders, soft_pos_activated(bool), space_ref(ref), status_name, support_multi_currency_change(bool), supported_currencies(array), tax_id, transaction_category_code_tcc, transaction_category_code_transaction_type, validation_status_name, workspace_ref(ref)

### transaction (all levels — ~100 fields)
**Core**: id, date(number), amount(number), original_amount(number), order_id, order_reference, processing_flow, source, api_version
**Currency**: currency_*, original_currency_*, settlement_currency_*, settlement_amount(number), settlement_conversion_rate(number), conversion_rate(number)
**Card/BIN**: masked_pan, masked_pan_with_bin, bin_*(brand, country, currency, issuer, level, scheme, type, valid), card_details(object), card_expiry_*, card_saved(bool), save_card(bool), scheme_type, name_on_card, hashed_name_on_card, expiry_date
**Gateway**: gateway_code, gateway_merchant_id, gateway_order_id, gateway_order_reference, gateway_purchase_id, gateway_recommendation, gateway_transaction_id, gateway_card_token
**Acquirer**: acquirer_code, acquirer_id, acquirer_merchant_id, acquirer_message, acquirer_response_*, acquirer_transaction_*
**Customer**: customer(object), customer_id, customer_name, customer_email, customer_phone, customer_ip, customer_account_*, customer_ref(ref), customer_original_transaction_ref(ref), customer_country_*, customer_city_name, customer_location, customer_zip_code, customer_time_zone(number), customer_avatar, is_customer_transaction(bool)
**Retailer**: retailer(object), retailer_id, retailer_name, retailer_avatar, retailer_location, retailer_account_*, retailer_ref(ref)
**Space/Workspace**: space(object), space_name, space_ref(ref), workspace(object), workspace_name, workspace_ref(ref), owner(object)
**Status**: transaction_status_name, reconciliation_status_name, statement_status_name, response_code
**Integration**: integration_id, integration_fees(object), service_id, service_name, service_mean, psp_id, merchant_id, mcc, terminal_id, transaction_channel_*, transaction_source_name
**Other**: ai_risk_assessment(object), dynamic_currency_conversion(object), is_3ds(bool), is_international_card(bool), clearing_date(number), authorization_code, rrn, stan, routing_history(array), synced_with_crm(bool)

### user
archived, avatar, birth_date(object), email, end_date, first_name, first_name_lower_case, gender_name, identity_*, interface_appearance, language(ref), language_iso_639_1_code, language_name, last_name, last_name_lower_case, mobile, multi_factor_auth_mobile, nick_name, relationship_name, screening_status_name, start_date, status_name, time_zone(ref), time_zone_utc_offset, user_name, user_name_lower_case, user_profile_name, validation_status_name, working_day(object), working_time(object)

### user_profile
menu-1(object), menu-2(object), menu-3(object), menu-4(object), name

### workspace
archived, automatic_id(object), avatar, background_image, currency_*, id, menu_background_color, menu_text_color, name, status_name, validation_max_level(number), validation_status_name

### space
account_id_pattern_name, archived, avatar, currency_*, customer_account_id_pattern_*, id, name, name_lower_case, retailer_account_id_pattern_*, status_name, validation_status_name

### activity (workspace level)
activity_source_name, activity_status_name, collection, collection*_ref(ref), collection_id_*, customer_email, customer_name, customer_ref(ref), date(number), log_execution_id, masked_pan, previous_collection_ref, retailer_id, retailer_ref(ref), service(ref), service_id, service_name, space_*, transaction_amount(number), transaction_currency_*, user_email, user_mobile, user_ref(ref), validators(object)

### integration1
method_of_payment(array), name, status_name, type, utility(array)

### integration partner
credential(object), settlement_partner(object), status_name, technical_partner(object)

### service (all levels)
account_ref(ref), currency_*(nullable at partner level), dynamic_credentials(object, partner level), fee(object|null), markup(null), service_id, service_name, status_name

### routing_table (workspace level)
currency_alphabetic_code, integration_id, rules(array), service_id, service_name

### routing_rule1 (retailer level)
bin_scheme, card_type, currency_alphabetic_code, integration_name, is_international_card(bool), mop_name, rule_priority(number), scheme_name, service_name, status_name

### reconciliation
currency_*, date(number), end_date(number), end_date_isostring, final_balance(number), id, initial_balance(number), ratio_limit(number), reconciled_balance(number), reconciled_transaction_count(number), services(object), status_name

### statement
date(number), file_hash, id, validation_status_name

### statement_transaction
amount(number), authorization_code, currency_*, date(number), gateway_merchant_*, integration_fees(object), integration_id, invoice_reconciliation_date(number), masked_pan*, net_settlement_amount(number), post_date(number), retailer_*, scheme_type, settlement_amount(number), statement_id, transaction_ref(ref), txn_sign, unique_txn_code, verification_status_name

### MOP/scheme/service (deep integration nesting)
**mop**: card_type, has_currency_conversion(bool), method_of_payment_category, method_of_payment_name, status_name
**scheme**: card_scheme, scheme_id, status_name
**service** (at all integration levels): account_ref(ref), currency_*, dynamic_credentials(object), fee(object), markup, service_id, service_name, status_name
**supported_currencies** (crypto): status_name, supported_currency_name

### Apple Pay certificates
**merchant-identity**: apple_merchant_id, creation_date(number), csr, expiry_date(number), merchant_identity_certificate, private_key_encrypted, status_name
**processing**: creation_date(number), csr, expiry_date(number), payment_processing_certificate, private_key_encrypted, status_name
