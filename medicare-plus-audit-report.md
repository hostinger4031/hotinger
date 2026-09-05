# MediCare+ (hotinger) — Complete Read-Only Audit Report
**Repository:** https://github.com/hamza4031/hotinger  
**Theme:** Astra 4.13.2 (Parent Theme — directly modified)  
**Audit Date:** 2026-09-05  
**Status:** READ-ONLY — Zero changes made

---

## REPOSITORY STRUCTURE OVERVIEW

The repository contains a single `astra.zip` (21.7 MB) which extracts to a **directly modified Astra parent theme** — not a child theme. Custom MediCare+ code has been appended to:

- `functions.php` — Astra's core functions file, with ~400 lines of custom code appended at the bottom
- `header.php` — Fully replaced with a 1,687-line custom header
- `footer.php` — Fully replaced with a 906-line custom footer
- `template-medicare-homepage-safe.php` — 844-line custom page template
- `style.css` — Unmodified Astra parent theme CSS

**Referenced but missing files:**
- `assets/css/medicare-custom.css` — **Does NOT exist** (404 on every page)
- `assets/js/medicare-custom.js` — **Does NOT exist** (404 on every page)

---

## A. CRITICAL BUGS (P0)

---

### A1. `medicare-custom.css` and `medicare-custom.js` do not exist
- **File:** `functions.php`, lines 249 & 260
- **What:** `mc_enqueue_assets()` registers and enqueues `get_stylesheet_directory_uri() . '/assets/css/medicare-custom.css'` and `/assets/js/medicare-custom.js`. Neither file exists anywhere in the extracted zip.
- **Why it's a problem:** WordPress will attempt to load these on every front-page load and generate 404 HTTP errors. The `mc-custom-js` script handle is used as a dependency anchor for `wp_localize_script('mc-custom-js', 'mc_vars', [...])`, so the `mc_vars` JavaScript object (containing `cart_url`, `ajax_url`, `nonce`, `whatsapp`, `currency_symbol`) is never injected. Any JS that attempts to use `mc_vars` will throw `ReferenceError: mc_vars is not defined`.
- **Severity:** P0 — causes JS errors and missing PHP-to-JS data bridge on homepage
- **Recommended fix:** Create `assets/css/medicare-custom.css` (can be empty initially) and `assets/js/medicare-custom.js` (can be empty initially), OR move the `wp_localize_script` call to attach to a script handle that actually exists (e.g., `jquery`).
- **Risk:** Low — creating empty placeholder files resolves 404s without breaking anything
- **What it affects:** `mc_vars` JS object; any code referencing `mc_vars.ajax_url` or `mc_vars.nonce`

---

### A2. Entire Cart System is Fake — Completely Bypasses WooCommerce
- **File:** `template-medicare-homepage-safe.php`, lines 584–795
- **What:** The homepage uses a JavaScript object `mcAppCartState = {}` as the sole cart. When a user clicks "Add to Basket", `mcInlineBasketPush(id)` pushes the product into this local JS object. The cart drawer, cart count, cart subtotal, and checkout form all read only from `mcAppCartState`. This state is **ephemeral** — it is wiped on every page refresh.
- **Why it's a problem:**
  - WooCommerce `WC()->cart` is never touched during any "Add to Cart" action on the homepage
  - The header cart badge (`id="cartBadge"`) reads from `WC()->cart->get_cart_contents_count()` at PHP render time; it is never updated by `mcAppCartState` changes
  - The floating badge (`id="mcStickyBadgeCount"`) counts `mcAppCartState` items, but the header badge counts real WC cart items — they will always show different numbers
  - Refreshing the page drops all items silently, with no WooCommerce session
  - If a user navigates to `/cart` or `/checkout`, both will be empty despite the homepage showing items
  - WooCommerce stock is never decremented; orders can be placed on out-of-stock items
- **Severity:** P0 — the core commerce function is broken
- **Recommended fix:** Replace `mcInlineBasketPush` with a real WooCommerce AJAX add-to-cart call using `?add-to-cart=ID` or `wc-ajax=add_to_cart`. Let WooCommerce manage cart state. Update the cart drawer to render WC cart fragments.
- **Risk:** High — requires coordinated change to JS, PHP AJAX handlers, and cart drawer template. Must be done incrementally.
- **What it affects:** Cart count, mini-cart, checkout, WC cart page, stock management, order history

---

### A3. Header Cart Button is a Dead Stub
- **File:** `header.php`, lines 1056 & 1647
- **What:** The header cart button has `onclick="mcpOpenCart()"`. The function `mcpOpenCart` is defined only as a fallback stub: `window.mcpOpenCart = window.mcpOpenCart || function() { mcpShowToast('Cart opening...'); }`. No real implementation overrides this stub.
- **Why it's a problem:** Clicking the cart icon in the header shows a toast that says "Cart opening..." but opens nothing. Users cannot access their WooCommerce cart from the header on any page other than the homepage (where `mcOpenCartDrawer()` is defined instead).
- **Severity:** P0 — the header cart is completely non-functional on all non-homepage pages
- **Recommended fix:** Point `mcpOpenCart` to `wc_get_cart_url()` (redirect) or implement a proper WooCommerce mini-cart drawer that reads WC fragments.
- **Risk:** Low risk to redirect; medium risk if implementing a drawer
- **What it affects:** Cart access on all non-homepage pages

---

### A4. Prescription Upload UI Does Nothing
- **File:** `template-medicare-homepage-safe.php`, lines 554–563
- **What:** The Rx upload modal (`id="mcHtmlRxModal"`) contains an upload-zone div that only shows a toast; there is no `<input type="file">`, no form, and no server request. The submit button only closes the modal and shows a success toast.
- **Why it's a problem:** The user sees a successful upload confirmation, but nothing is uploaded, stored, or sent anywhere. This is completely fake functionality that deceives the user.
- **Severity:** P0 — core business feature does not function
- **Note:** A real PHP handler `mc_handle_prescription_upload()` exists in `functions.php` with nonce validation, file size/type checks, WordPress media upload, and admin email notification. It is not connected to the UI.
- **Recommended fix:** Replace the fake upload zone with an actual multipart form pointing to `admin-post.php` with nonce and file input.
- **Risk:** Low — the existing handler can be wired to the UI, with additional MIME validation recommended.

---

### A5. `mc_ajax_place_order` Has No Stock or Purchasability Validation
- **File:** `functions.php`, lines 528–596
- **What:** The handler receives client-supplied product IDs and quantities and directly adds products to a raw WooCommerce order without checking purchasability, stock, variations, or quantity limits.
- **Why it's a problem:** A malicious client can submit arbitrary product IDs/quantities and bypass normal WooCommerce cart/checkout validation and stock handling.
- **Severity:** P0 — security and inventory integrity risk
- **Recommended fix:** Ultimately retire this custom order path in favor of the standard WooCommerce cart/checkout pipeline. As an interim measure, add server-side purchasability and stock validation.
- **Risk:** Medium — affects order creation and checkout flow

---

## B. FUNCTIONAL BUGS (P1)

### B1. `mcp_live_search` AJAX Action Never Registered
The header calls `mcp_live_search`, but no corresponding WordPress AJAX handler exists. The live search dropdown therefore cannot work. Add authenticated and unauthenticated handlers with a nonce, sanitized search/category input, and a limited product query.

### B2. `mcp_newsletter_subscribe` AJAX Action Never Registered
The footer calls `mcp_newsletter_subscribe`, but no handler exists. The current fallback opens a `mailto:` link, which is poor UX. Implement a real server-side handler with nonce and validation, or remove the feature until a real destination is available.

### B3. Authentication Button Is a Stub
`mcpOpenAuth()` only shows "Auth modal coming soon". Route users to the WooCommerce My Account page unless a custom authentication UI is actually implemented.

### B4. Mobile Upload-Rx Button Is a Stub
`mcpOpenRx()` only shows "Upload Rx coming soon". It should route to a real upload page or a globally available upload modal.

### B5. `mcpAjax` JavaScript Object Is Missing
The header expects `mcpAjax.ajax_url` and `mcpAjax.nonce`, but the object is not localized anywhere. Add a proper localized data bridge to an actually loaded script handle.

### B6. Header Cart Badge Does Not React to Homepage Cart State
The header renders a WooCommerce count while the homepage uses its own JavaScript cart state. This creates inconsistent counts and must be resolved by unifying the cart around WooCommerce.

### B7. Undefined `mcShowToast()` Calls
`functions.php` calls `mcShowToast()` while the defined global toast function is `mcpShowToast()` / homepage `mcAppToast()`. These calls should be corrected before the related events are used.

### B8. Custom Checkout Bypasses WooCommerce
The homepage custom checkout modal creates orders directly through AJAX rather than using the normal WooCommerce cart/checkout pipeline. This can bypass stock, customer/order hooks, emails, tax/shipping/payment integrations, and third-party extensions.

---

## C. UI / CONTENT ISSUES

- Hardcoded fake "FLAT 20% OFF" messaging should only be displayed when a real promotion exists.
- Hardcoded "FREE LOGISTICS" should be verified against actual shipping policy.
- Hardcoded fake countdown resets on every page load and should be removed unless backed by a real campaign end time.
- Fake live-order notifications should be removed unless based on real order data with appropriate privacy handling.
- Product cards currently show fallback ratings/review counts (`4.5` / `24`) that may not represent real product data.
- App store buttons with `href="#"` should be hidden until real URLs are configured.
- Fake/default license and certification values must never be displayed.
- India/New Delhi defaults, Indian currency/payment claims, and default phone/WhatsApp values should be replaced with verified business information before production.
- Footer claims such as 24×7 support, seven-day returns, same-day delivery, payment methods, and SSL badges should match the actual service configuration.

---

## D. CSS / DESIGN SYSTEM ISSUES

- Large amounts of inline CSS are spread across header, footer, and homepage.
- Multiple independent variable systems exist (`--mcp-*`, `--ft-*`, `--blue`, `--green`).
- Multiple overlay/toast systems exist.
- Very high and inconsistent z-index values create stacking-context risk.
- Font loading is duplicated between theme markup and enqueue logic.
- Font Awesome loading is conditional on the front page and should be verified on all pages before changing it.
- Responsive CSS has limited breakpoint coverage; the homepage mainly relies on a 640px breakpoint.
- Many `!important` declarations make future maintenance harder.
- The global reset in `header.php` can affect Astra/WooCommerce markup on every page.

---

## E. ACCESSIBILITY ISSUES

- Many interactive controls are `<div onclick>` instead of semantic buttons/links.
- Quantity controls are not keyboard-friendly buttons.
- Homepage modals need dialog semantics and focus management.
- Focus should be trapped while modal dialogs are open and restored on close.
- Some muted text colors may not meet WCAG contrast requirements for small text.
- `user-scalable=no` prevents/limits user zooming and should be removed.

---

## F. PERFORMANCE ISSUES

- Multiple WooCommerce product queries run during homepage rendering without transient caching.
- Header performs repeated taxonomy queries and a trending-products query.
- Large inline CSS/JS increases page source size and maintenance cost.
- Google Fonts are inserted directly before `wp_head()` instead of being managed through WordPress enqueue APIs.
- Font Awesome should be loaded only where needed and without duplicate copies.

---

## G. SECURITY / DATA-INTEGRITY ISSUES

- Client-supplied cart data must never be trusted for price, stock, purchasability, variation, or quantity validation.
- Custom order creation should be replaced by the WooCommerce cart/checkout pipeline.
- AJAX endpoints need nonces and strict input validation.
- Prescription uploads should use WordPress's file validation APIs; checking a client-supplied MIME type alone is insufficient.
- Newsletter submission should have nonce, email validation, and appropriate abuse/rate-limit protection.
- Do not expose raw exception messages to end users.

---

## H. ARCHITECTURE ISSUES

### H1. Customizations Are in the Astra Parent Theme
All MediCare+ custom code is mixed directly into Astra. Astra updates can overwrite the custom implementation. A child theme should be introduced incrementally.

### H2. `mc_` and `mcp_` Option Prefixes Are Inconsistent
The admin settings page registers/saves some `mc_*` keys while header/footer read `mcp_*` keys. A single prefix should be standardized, with a safe one-time migration for existing values.

### H3. Global Functions
Custom functions are in the global namespace. A future cleanup can group code into a class/namespace, but this should not be combined with the initial migration.

### H4. Fake Cart Is Ephemeral
`mcAppCartState` exists only in browser memory. It should be retired only after WooCommerce cart functionality is proven.

---

## I. UNNECESSARY CODE / UI THAT CAN BE REMOVED SAFELY AFTER DEPENDENCY CHECKS

| Item | Reason |
|---|---|
| `mc_render_product_cards()` | Appears unused by the homepage template; verify with a final repository-wide search before deletion. |
| Fake live-order popup engine | Not real transaction data; misleading UI. |
| Fake countdown timer | Resets on page load and has no real campaign source. |
| Undefined-toast feedback hook | Incorrect function reference; remove or correct after verifying dependencies. |
| Hardcoded fake offer banners | Misleading without a real promotion. |
| App-store buttons with `href="#"` | Dead navigation. |
| Fake license defaults | Must not display unverified credentials. |
| Auth/cart/Rx stubs | Replace with real navigation/features first. |
| Repeated `get_terms` calls | Candidate for caching/reuse after behavior is verified. |
| Header trending query | Candidate for transient caching. |

---

## J. THINGS THAT SHOULD NOT BE TOUCHED

- Astra core files under `inc/`, `assets/`, and `admin/` should remain stock.
- Existing prescription backend should be preserved while wiring the UI, but it should receive a stronger WordPress MIME/ext validation review.
- Existing nonce checks should be preserved.
- Astra WooCommerce compatibility code should not be modified.
- Useful product-query fallback logic should be preserved while adding caching.
- Mobile-menu behavior should be preserved unless a regression is proven.
- Voice-search implementation should not be rewritten without a concrete requirement.

---

# SAFE IMPLEMENTATION PLAN

## Phase 0 — Baseline / Rollback
1. Download and hash the current `astra.zip` as the immutable baseline.
2. Commit the exact current state and tag it `v0-baseline`.
3. Do not make implementation changes until the baseline is verified.

## Phase 1 — Low-Risk Fixes
1. Create missing custom CSS/JS files or move localization to a valid script handle.
2. Remove `user-scalable=no`.
3. Correct undefined toast references.
4. Standardize option prefixes with a safe migration.
5. Hide unverified license/app-store UI.
6. Remove fake countdown/live-order UI.
7. Remove fake rating/review fallbacks.

## Phase 2 — Child Theme Migration
Create an Astra child theme and migrate customizations one logical unit at a time. Test after each migration. Do not modify Astra core files under `inc/`, `assets/`, or `admin/`. Preserve the current front-end before cleanup.

## Phase 3 — WooCommerce Cart Unification
1. Test real WC AJAX add-to-cart on one homepage product.
2. Migrate all homepage cards.
3. Connect cart badge/fragments.
4. Rebuild the cart drawer around `WC()->cart`.
5. Point checkout to the real WooCommerce checkout page.
6. Only after all tests pass, remove `mcAppCartState` and its dependent UI code.

## Phase 4 — Custom Order Handler
Harden the legacy handler while it remains registered, verify the standard WooCommerce checkout end-to-end, monitor for unexpected calls, then remove the legacy `mc_ajax_place_order` handler only after zero legitimate callers are confirmed.

## Phase 5 — Prescription Upload
Wire the existing backend handler to a real multipart upload form with nonce, file input, phone field, server response, and truthful success/error messages. Strengthen MIME validation as part of this work.

## Phase 6 — Functional Stubs
Implement live search and its nonce/data bridge. Route authentication to WooCommerce My Account. Make Upload Rx accessible from global navigation. Implement newsletter only when a real storage/delivery strategy is chosen.

## Phase 7 — Performance
Add carefully invalidated transient caching to product queries. Extract CSS incrementally. Move font loading to WordPress enqueue APIs and check for duplicate font requests.

## Phase 8 — Design System
Unify CSS variables, toast handling, overlays, breakpoints, z-index scale, typography, spacing, buttons, radius, and shadows. Do not mix large design changes with high-risk commerce changes.

## Phase 9 — Accessibility
Convert clickable divs to semantic controls, add proper dialog semantics, keyboard/focus management, accessible quantity controls, and verify contrast.

---

## ZERO-BREAKAGE RULES

- One logical change per commit.
- Backup/tag before implementation.
- Test after every commit.
- Keep `mcAppCartState` until the real WooCommerce cart is verified.
- Keep the legacy order handler until standard WooCommerce checkout is verified and callers are confirmed absent.
- Do not perform a giant Astra-to-child-theme rewrite in one commit.
- Do not modify Astra core internals.
- Do not remove working behavior based only on static assumptions; verify dependencies first.
- Do not show fake ratings, discounts, orders, countdowns, licenses, or service claims.
- Every high-risk change must have a documented rollback path.

**Status:** Planning/read-only report. No website code changes are authorized by this document.
