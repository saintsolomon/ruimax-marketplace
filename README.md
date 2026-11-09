# Ruimax Market — website, seller storefronts and mobile apps

A Ruimax-branded B2B wholesale marketplace with an editable website and matching React Native / Expo Android and iPhone source. Independent of Alibaba. No added watermark.

## Implemented

- Responsive marketplace with searchable categories, countries, price filters, supplier collections, saved products and inquiry basket.
- Advertising hero slider with autoplay, pause/play, dots, previous/next, touch gestures, reduced-motion support and focus/hover pause. A separate featured-business carousel offers advertising placements.
- Public seller storefronts at `/store/<seller-slug>`, seller sign-in, one storefront per account, unique permanent URLs, business profiles, product creation/editing, visibility controls and image uploads.
- Seller workspace at `/seller`, with product management, orders, advertising packages, booking creatives and storefront URL copying.
- Protected administrator dashboard at `/admin`: storefront suspension, paid ad approval/pause/rejection, editable advertising rates, orders, live revenue and a payment/refund ledger.
- MoMo and Visa/card hosted checkout integration through Paystack; payment credentials are not configured in source and payments remain disabled until securely connected.
- Server-owned order prices and whole-pesewa arithmetic. Every paid product sale records a fixed 15% commission and 85% seller allocation. Advertising payments are recorded separately as platform revenue.
- Server verification of currency, amount, reference and live/test environment. Idempotent checkout references and payment ledger entries. Atomic payment updates and proportional refund reversal. Test payments and test adverts are excluded from live earnings / public advertising feeds.
- Database-backed sourcing requests are scoped to signed-in users. Sample catalog entries cannot be purchased.
- Native app source includes both advertising carousels, public live-catalog / advertiser feed with local caching, offline sample catalog, device-local saved products and quote drafts. Store creation, admin, ad booking and checkout open the secure website in the device browser.

## Commission model

Amounts are integer Ghana pesewas. Product subtotal × 15%, rounded to the nearest pesewa, is the platform commission. Seller allocation is subtotal minus commission, preserving the exact collected total. For a GH₵1,000 product subtotal, platform commission is GH₵150 and seller allocation is GH₵850. This release does not add shipping or tax. Gateway fees are borne by the platform; dashboard earnings are before gateway fees. Seller allocations are accounting balances, not automatic payouts. No seller payout transfer integration is implemented.

Advertising is optional and one-time, without automatic renewal. Editable starter rates: main slider 7 days GH₵150, featured carousel 7 days GH₵75, main slider 30 days GH₵500. New bookings use the current rate; existing orders keep their original price. Advert duration starts on first approval after verified payment. Expired, unpaid, suspended-store and refunded adverts are not delivered. Test advertisements are never shown in the public live feed.

## Identity and administrator access

The hosted platform owns ChatGPT sign-in. Seller and buyer API actions require authenticated site identity, and each seller operation checks storefront ownership server-side. Public browsing does not require sign-in. `/api/admin` requires the signed-in email to match `RUIMAX_ADMIN_EMAILS`, a server-side allowlist configured securely for the site owner. A client cannot claim administrator access. Never implement a first-user-becomes-admin rule.

## Payment activation

Add these server settings securely through the hosting environment, not the source or browser:

- `PAYSTACK_SECRET_KEY`: a test key first, then an activated Ghana business live key when ready.
- `PAYSTACK_WEBHOOK_SECRET`: the full `whsec_...` signing secret generated in the Paystack dashboard.
- `RUIMAX_ADMIN_EMAILS`: comma-separated email allowlist for verified administrator identities.

Configure the Paystack webhook to `https://<your-site-origin>/api/paystack/webhook`. The site must be publicly reachable for webhooks and the native catalog feed. Modern Paystack-Signature headers are checked using timestamp tolerance and HMAC-SHA512. If no modern signing secret is configured, the legacy x-paystack-signature HMAC is verified with the API secret. A configured modern secret disables legacy downgrade.

Payment callbacks go to `/payments/return`, which verifies the transaction on the server rather than trusting browser success. Administrators and relevant buyers/sellers can also reconcile pending payments. Successful refund.processed events reverse advertising or sales revenue, with cumulative rounding for partial refunds. Rejected paid advertising bookings are flagged for refund review; rejection does not itself issue a refund. Configure and test the live payment business account before collecting funds.

No card details, MoMo PINs, API keys or payment credentials are collected by Ruimax forms. Checkout occurs on Paystack's hosted checkout. Keys and signing credentials are not embedded in the website, native apps or source downloads.

## Database and files

Cloudflare D1 is bound as `DB`; R2 image storage is bound as `BUCKET` in `.openai/hosting.json`. Schema is in `db/schema.ts`, with immutable migrations in `drizzle/`. Do not replay applied migrations. Build once, then apply any pending local migration using:

```sh
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0002_yellow_mach_iv.sql
```

Production migrations are applied by the hosting publish workflow. Product and ad image uploads accept JPG, PNG or WebP under 5 MB and are served with explicit image content types. Initial product photographs are illustrative Unsplash assets; custom packshot illustrations are included. Replace samples with sellers' authorized product images for a commercial catalog.

## Development and validation

```sh
npm ci
npm run dev
npx tsc --noEmit
node --test tests/*.test.mjs
npm run build
cd mobile
npm ci
npm run typecheck
npm run export:native
```

Tests execute actual route handlers with an isolated SQLite database and test-only auth/payment adapters. They cover seller ownership, administrator access, reserved/taken slugs, immutable price snapshots, amount/currency/environment tampering, 15% rounding, no earnings before payment, duplicate notifications, partial/full refunds, ad approval gates, live/test separation and expiry.

Browser QA uses local fixture-only pages that are removed before publication. No test identities or sample financial records are seeded into the production database. Native Android and iOS JavaScript exports validate bundles; physical device QA, signing, APK/IPA delivery and store releases remain separate release work.

The site's source manifest contains its existing project identity. Do not reuse it to register an unrelated site.

## Source downloads and shared catalog

`shared/catalog.json` and `public/products/` are the canonical sample assets. Run `node scripts/sync-mobile-catalog.mjs` to copy them into the mobile project. Real listings and paid active ads are fetched from `/api/catalog` by the website and app. Cached native ad data expires using each campaign's end timestamp; checkout revalidates current server data.

Run `python scripts/package-source.py` to update both downloadable archives. Build output, dependencies, keys and local runtime state are excluded. See `mobile/README.md` for running and distributing the apps.

## Remaining integrations

Live payment credentials and gateway dashboard setup; automated seller payouts; shipping/tax calculation; real-time supplier messaging and notifications; store policy/dispute handling; native account sessions (current seller/admin/checkout workflows use the device browser); physical-device testing and app store submissions. User storefront profiles and capabilities are seller-provided, not independently verified by Ruimax.

## Social sharing and app access

- Shareable product pages at `/product/<id>` have canonical URLs and server-rendered Open Graph / Twitter image metadata, including an explicit Sample label for illustrative listings. Hidden products and suspended stores cannot be shared as active listings.
- Product cards, detail views, seller product tools and storefronts expose WhatsApp / Facebook link composers, image sharing through supported browser share sheets, image download and copy-caption / copy-link actions. The application never publishes a social post automatically.
- Browser image sharing prepares the file before the click to preserve user activation. Unsupported browsers can save the image and attach it in their social app. External seller image hosts may restrict browser downloads through CORS; the fallback opens the original image for saving.
- `/apps` is the public, shareable app access and download page. It includes Android / iPhone instructions, web-app installation, WhatsApp / Facebook sharing, and clearly labeled developer ZIP downloads. Native APK / IPA and app-store releases remain unavailable.
- A root-scoped service worker supplies a generic offline page only. It does not cache account pages, private API data, checkout or payment content. Browser-supported installation uses the web manifest; iPhone users can add the site from Safari.
- The native app uses Expo Sharing / FileSystem / Asset to share local or downloaded product images, React Native Share for links / app invitations, and social app composers. Users choose the final destination; image captions depend on the chosen social app. Native seller / checkout workflows continue in the browser.

Sharing checks cover canonical link encoding, special-character captions, public app URLs and suspended-product access. Browser QA checked caption copying, image downloads, unsupported-image-share fallback, product / app image metadata and a phone viewport. Android and iOS exports passed; device-specific social-app share sheets still require real-device QA.

## Global interface, categories and optional premium
- Shared English / Simplified Chinese / French UI dictionaries: `shared/translations.json`; copy is included in the standalone mobile project. The language menu persists the preference. `?lang=en`, `?lang=zh`, and `?lang=fr` are shareable language choices. Seller-written product information, business names and messages are retained verbatim; this is UI translation, not automatic message translation.
- Sixteen category families with 48 descriptive subcategory examples and category artwork. Listings use canonical category names so language changes cannot break filtering. Subcategories are discovery descriptions, not separate listing fields in this release.
- 249 ISO country/territory choices for seller registration and store settings. International discovery and shop creation do not imply guaranteed shipping, customs handling, international payment approval or eligibility in every jurisdiction.
- Free plan: no registration or listing charge, seller URL, products, messages, social sharing and orders. 15% commission applies to paid product sales. Premium: GH₵199 for 30 days, one-time checkout with no automatic renewal; branded shop cover, paid-plan badge, paid-sales summary and private CSV export. Advertising is separate. Premium is not an identity-verification badge.
- Premium entitlement starts on the first server-verified payment and is not extended by duplicate notifications. Only live paid upgrades unlock production features. Refunds remove entitlement. Checkout uses a server price snapshot. Commission remains 15% for both plans.

## Admin wallet activation
No live keys, recipient or transfer permission are bundled in source. Do not put credentials into client code, Git or public forms.
1. Activate the owner-controlled registered Ghana Paystack business. Enable international card acceptance if approved by the provider. GHS is the checkout/settlement currency. MoMo coverage is provider-specific, not worldwide.
2. Configure `PAYSTACK_SECRET_KEY` and `PAYSTACK_WEBHOOK_SECRET` through secure hosting secrets. Verify charge and refund webhook delivery to `/api/paystack/webhook` before enabling live collections.
3. In the payment provider account, verify/create a GHS `mobile_money` recipient belonging to the admin. Configure server secret `RUIMAX_MOMO_RECIPIENT_CODE` and a masked human-readable `RUIMAX_MOMO_RECIPIENT_LABEL`. This destination is not editable by a browser request.
4. Configure `RUIMAX_WITHDRAWAL_FEE_CENTS` to an adequate reserve for the current provider transfer fee (minimum 100 pesewas). Review current provider terms before setting it. Provider charges are additional to the withdrawal amount.
5. Confirm transfer permissions and funding/settlement arrangements. Collections do not automatically imply an available Paystack transfer balance. Do not disable provider OTP protections; the admin UI supports OTP finalization when required.
6. Use separate test provider fixtures first, then a small owner-authorized live end-to-end validation through the admin UI. Physical device, OTP, live fees, settlement and transfer tests require owner credentials and have not been performed here.

The admin wallet totals verified live commission + advertising + premium receipts net of refunds, less recorded gateway fees and reserved/completed withdrawal amounts. Payment fees from verified provider data are recorded once; if absent, a conservative 5% reserve is used. Seller allocations (85% of product sales, net of refunds) are excluded. A provider balance check also protects all unpaid seller allocations. Seller payouts are still a separate, nonautomated workflow.

Withdrawal requests atomically reserve platform earnings, use an idempotency key and unique provider reference, and are restricted to the configured admin. Timeout or unknown provider outcomes remain reserved until reference verification; they are never blindly retried. Matching amount, currency, recipient and reference are required before changing a withdrawal’s provider status. Pending/OTP/success reservations remain held; confirmed failed/reversed/abandoned/rejected results release the reserve. Fee reserves are conservative and are not automatically refunded to available earnings. Refunds or provider chargebacks can reduce future available platform earnings; admins should maintain an operating reserve.

Before public financial launch, reconcile all existing production payment fees and refunds, confirm retention/deletion policy and operational refund procedures, and complete seller settlement arrangements. Existing automatic provider settlements are outside this internal wallet; amounts already withdrawn or paid out directly via the provider must be reconciled before exposing withdrawable balances. No actual money movement is performed by the development tests.

## Account security: email confirmation and verification
Hosted ChatGPT sign-in remains the primary identity. `/security` now sends a six-digit confirmation and access code to the signed-in account’s email. There is no browser-supplied recipient override. Native account navigation opens this same hosted flow. Public catalog pages remain available; private buyer, seller and admin actions require an email approval.

Set `RESEND_API_KEY` and `RUIMAX_EMAIL_FROM` as secure runtime secrets. The sender address/domain must be verified with Resend. `RUIMAX_SECURITY_KEY` remains required as a Base64 32-byte signing key. Until configured, no email is sent and protected actions fail closed. Check delivery to real inboxes before announcing activation; API acceptance does not prove inbox delivery.

Codes expire after five minutes, use a cryptographically secure random generator, and are stored only as owner/email/challenge-bound HMAC hashes. Verification has attempt limits and atomic single-use consumption. Sending has a one-minute cooldown, account/destination hourly caps and a platform daily cap. Provider failures invalidate their challenge. Changing the account email invalidates earlier proofs. Ordinary approvals expire after 15 minutes; purchases and admin access require proof less than five minutes old. Withdrawals consume an additional fresh email code. The payment provider may separately require its own OTP.

The migration removes SMS factors, clears authenticator secrets and recovery codes, and invalidates old browser approvals. Legacy verification endpoints are rejected. Email verification can restore a frozen account; revocation and freezing cancel outstanding challenges. ChatGPT account recovery remains with the identity provider. Email access must be secured separately; this does not replace protection of the primary sign-in account.

Tests exercise real API/database logic with mocked primary identity and email delivery, including expiry, replay, concurrent use, attempts, cross-origin/owner isolation, recipient binding, provider failure, freeze/restore, admin freshness and rejection of legacy methods. No live email is sent by tests.


## Reliability and layout review
- Chat, Security & 2FA and language selection now share a coordinated responsive toolbar. It sits at the lower right on desktop and above the measured mobile navigation height on phones. Messaging/security pages use an inline top toolbar instead, with no duplicate self-link. Floating controls hide during text entry. Footer space is reserved so end-of-page actions remain reachable; the chat composer fits the remaining viewport.
- UI requests use a 15-second timeout unless an explicit cancellation signal is already provided. The toolbar consolidates account/unread polling and clears stale unread counts when private access is unavailable. Chat clears private content after sign-out, factor expiry or account freeze.
- Product/ad checkout retries retain their request identifier for the same input. Server-side fingerprints bind an identifier to its exact product/quantity, premium plan or advertisement input. Changed input cannot silently retrieve a different checkout. Previously stored orders without a fingerprint remain viewable in order history but are not automatically reinitialized with an unverified request payload. Excessive order values return a validation error.
- Chat identifiers cannot be reused for different text; edited retry drafts get a fresh identifier and the composer is disabled while its send is in flight.
- Mobile drafts retain a product-name/unit snapshot and remain readable if a live seller listing is later removed. Old drafts without a snapshot display an unavailable-product label while preserving notes and deletion controls.

Verification includes TypeScript checks, isolated API/database regression tests, native JavaScript exports for Android/iOS, and browser UI checks for search, product detail, inquiry basket, app links, translations, narrow phone layouts and toolbar/composer spacing. Live collections, MoMo disbursements, actual user authenticator enrollment, physical-device sharing and signed store-release validation require the owner's external accounts/devices and are not represented as completed by these checks.

