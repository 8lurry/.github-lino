# Privacy Policy for Pronto BD

**Last updated:** 2026-03-17

This Privacy Policy describes how Pronto BD processes personal data when used through the web application and Android app.

## 1) Service scope

Pronto BD is a Lino-based business application used for product, order, inventory, invoicing, accounting, and related workflows.

Primary business user scopes are:

- **Retailer** (`seller`)
- **Wholesaler** (`supplier`)

Operational subtypes (such as `pos_agent` and `supply_staff`) operate within these scopes.

Important record status:

- Company registrations created in Pronto are internal application records for workflow integrity and do not represent government-issued trade licenses or equivalent legal approvals.
- Barcode records in Pronto are primarily internal identifiers used for product workflows. Where users provide barcode values from recognized standards or regulatory systems, Pronto stores and uses those values as entered.

## 2) Platform architecture

- Backend runs on Lino/Django infrastructure.
- React frontend runs in browser sessions.
- Android app uses a WebView shell and selected native bridge features.

Android bridge interactions (for example camera and notifications) return data into the same authenticated session context.

## 3) Personal data categories

Depending on configuration and usage, Pronto may process:

- account/profile data (name, username, email, language)
- business/transaction data entered by users
- product images and uploaded files
- session and authentication metadata
- device/app metadata required for mobile operations
- push notification token data (when enabled)
- operational/security logs

## 4) Automated setup for verified users

For verified active users, integrity/setup workflows may create or update linked records needed for valid business operation, such as:

- person/contact links
- company records
- registration/audit records
- user-to-ledger linkage

## 5) Camera and product images

When camera capture is used on Android:

1. Camera is opened by explicit user action and permission.
2. Image may be processed on-device (e.g., orientation/size adjustments).
3. Image is returned to the active Pronto form flow.
4. Image is used for product picture workflows only.

Product image visibility is limited by configured access rules, including owner/customer scope.

## 6) Push notifications

If enabled, Pronto may register a device token for notifications.
Token/device metadata is used for delivery, and registration can be revoked (for example on logout).

## 7) Legal basis and purpose

Data processing is used to:

- provide and operate the service
- execute requested workflows
- maintain security and prevent abuse/fraud
- deliver operational communications

Applicable legal basis depends on jurisdiction and context (such as contract, legitimate interest, or legal obligation).

## 8) Sharing and processors

Pronto does not sell personal data.

Data may be shared only with:

- authorized users/admins in the relevant deployment scope
- infrastructure/hosting processors
- notification processors (e.g., FCM) when enabled
- authorities when legally required

## 9) Retention and deletion limits

Retention depends on legal duties, organizational policy, and system configuration.

Even after deletion/forget requests, some records may be retained where required for legal, accounting, audit, fraud-prevention, or integrity reasons (for example invoices, vouchers, ledger movements, approvals/registrations, and traceability links).

## 10) Security

Pronto applies technical and organizational safeguards, including access controls and authenticated sessions. No system can guarantee absolute security.

## 11) Data subject rights

Subject to applicable law, users may have rights to access, correction, deletion, restriction, objection, and portability.

These rights may be limited where retention is legally required or necessary to preserve financial/audit integrity.

## 12) International transfers

Where infrastructure or processors operate across regions, data may be processed in other jurisdictions under applicable safeguards.

## 13) Policy updates

This policy may be updated over time. The latest effective date is shown at the top.

## 14) Contact

- **Organization / Data controller:** Pronto BD
- **Email:** sharifmehedi24@gmail.com
