# Week 5 AI Implementation Prompt — Reviews, Wishlist & User Profile
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## WEEK 4 HANDOFF SUMMARY

### What Was Built

**Backend:**
- `Order` Document with embedded `OrderItem`, `ShippingAddress`, `StatusHistory` (mongoengine)
- `Order.create_from_cart()` — builds order from cart with full item snapshots
- `Order.generate_order_number()` — ORD-YEAR-XXXXX format, always unique
- `Order.add_status()` — appends to status_history and updates order status atomically
- `Payment` Document with IntaSend fields: `flw_tx_ref` replaced by `intasend_checkout_id`, `intasend_invoice_id`
- IntaSend configured via django-environ (INTASEND_PUBLISHABLE_KEY, INTASEND_API_TOKEN, INTASEND_WEBHOOK_SECRET)
- `POST /api/v1/orders/` — creates order from cart, validates stock, clears cart, returns 201
- `GET /api/v1/orders/` — user's own orders list with pagination
- `GET /api/v1/orders/{order_number}/` — full order detail (owner only, 404 for others)
- `POST /api/v1/payments/initialize/` — creates IntaSend hosted checkout session, returns payment_url
- `POST /api/v1/payments/verify/` — server-side verification after IntaSend redirect
- `POST /api/v1/payments/webhook/` — CSRF exempt, secret hash verified, idempotent via `_fulfill_order()`
- `POST /api/v1/payments/dev-confirm/` — DEV ONLY bypass (DEBUG=True gate) — **must be removed before Week 12**
- `_fulfill_order()` shared helper — idempotent, marks Payment succeeded, marks Order paid, decrements stock atomically
- 16 Django APITestCase tests for orders — all passing

**Frontend:**
- `orderAPI.js` — createOrder, listOrders, getOrder, initializePayment, verifyPayment, devConfirmPayment
- `useOrders(page)`, `useOrder(orderNumber)` TanStack Query hooks
- `Checkout.jsx` — Step 1 (address + Zod validation) + Step 2 (shipping method selection + order summary sidebar)
- `CheckoutPayment.jsx` — Step 3 (IntaSend redirect page with DEV bypass yellow button)
- `OrderConfirmation.jsx` — server-side verification on mount, cleans URL params, full order detail display
- `Orders.jsx` — order history with status filter tabs, skeleton loading, pagination, colored status badges
- `CartSummary.jsx` — Proceed to Checkout navigates to /checkout and closes drawer
- `Navbar.jsx` — user dropdown with email preview, My Orders, Profile (stub), Logout; mobile menu

### Files Created (Week 4)
**Backend:** `apps/orders/documents.py`, `apps/orders/serializers.py`, `apps/orders/views.py`,
`apps/orders/urls.py`, `apps/orders/tests.py`, `apps/payments/documents.py`,
`apps/payments/views.py`, `apps/payments/urls.py`

**Frontend:** `src/api/orderAPI.js`, `src/hooks/useOrders.js`, `src/pages/Checkout.jsx`,
`src/pages/CheckoutPayment.jsx`, `src/pages/OrderConfirmation.jsx`, `src/pages/user/Orders.jsx`

### Files Modified (Week 4)
**Backend:** `config/settings/base.py` (IntaSend block), `config/urls.py` (orders + payments included)

**Frontend:** `src/components/cart/CartSummary.jsx`, `src/components/layout/Navbar.jsx`,
`src/router.jsx` (/checkout, /checkout/payment/:orderId, /order-confirmation/:orderNumber, /orders)

### Key Decisions Made
- Stripe replaced with IntaSend (Kenyan payment gateway — Flutterwave rejected, Stripe unsupported in Kenya)
- IntaSend uses hosted redirect flow — no card form on our pages, PCI compliance handled by IntaSend
- DEV bypass endpoint added for sandbox testing (IntaSend STK push unreliable in sandbox)
- Two-path payment fulfillment: VerifyPaymentView (redirect) + WebhookView (fallback)
- `_fulfill_order()` is idempotent — both paths can call it without double-processing

### Current Working State
- Backend: Django on port 8000, all tests passing (auth + products + cart + orders)
- Frontend: React on port 5173, full cart → checkout → payment → confirmation flow working
- MongoDB Atlas: orders and payments collections populated from Week 4 testing
- Stock decrement confirmed working via dev bypass flow

### Next Session Starts With
Week 5, Day 1 — Review Document + App Scaffold

---

---

# Week 5 AI Implementation Prompt — Reviews, Wishlist & User Profile
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## CONTEXT TO GIVE THE AI AT THE START OF EVERY SESSION THIS WEEK

> "I am continuing a production e-commerce build. Weeks 1 through 4 are fully complete and tested.
>
> **What already exists and is working:**
> - Django project with split settings (base.py / dev.py / prod.py / test.py), loaded via django-environ
> - MongoDB Atlas connected via mongoengine
> - User Document with set_password() and check_password() methods
> - Full JWT auth in httpOnly cookies: register, login, logout, token refresh, /auth/me/
> - Password reset skeleton (Token Document with TTL, endpoints without emails yet)
> - DRF throttling on all endpoints
> - Django APITestCase tests for all auth, product, cart, and order endpoints (all passing)
> - Product, Category, Variant (embedded) mongoengine Documents with all indexes
> - Product list API: pagination, filtering by category/price/rating/search, 5 sort options
> - Product detail API: full document with all variants
> - Category tree API: nested structure, cached 1 hour
> - Cloudinary integration: image upload with IsAdminOrSeller permission
> - Cart Document with embedded CartItem (guest + authenticated, merge on login)
> - Full cart API: GET, POST, PATCH, DELETE, merge, coupon (skeleton)
> - Order Document with embedded OrderItem, ShippingAddress, StatusHistory
> - Order create (from cart), list, detail APIs
> - Payment Document with IntaSend fields (intasend_checkout_id, intasend_invoice_id)
> - IntaSend payment flow: initialize, verify, webhook (with DEV bypass for testing)
> - React 18 + Vite frontend with Tailwind CSS
> - Redux Toolkit store: authSlice, cartSlice, uiSlice
> - TanStack Query for all server data fetching
> - Axios instance with silent JWT refresh interceptor (withCredentials: true)
> - All product pages (Home, ProductList, ProductDetail), Cart, Checkout, Payment, OrderConfirmation, Orders
> - Navbar with user dropdown (My Orders, Profile stub, Logout) + mobile menu
> - React Router v6 with ProtectedRoute, GuestRoute, all routes wired
>
> **Stack reminder:**
> - Backend: Django 4.x + DRF + mongoengine (NO Django ORM, NO SQL)
> - Frontend: React 18 + Vite + Redux Toolkit + TanStack Query + Axios
> - All API endpoints prefixed /api/v1/
> - JWT in httpOnly cookies ONLY — never localStorage
> - Images: Cloudinary
> - Payments: IntaSend (Kenyan payment gateway, hosted redirect flow)
> - Styling: Tailwind CSS
>
> **Critical rules for Week 5:**
> - Reviews are ONLY allowed for verified purchases — the user must have a paid/delivered
>   order containing the product before they can review it
> - avg_rating and review_count on the Product document must be recalculated atomically
>   every time a review is created, updated, or deleted
> - Wishlist uses a toggle pattern — one endpoint handles both add and remove
>   (if the item exists, remove it; if it does not, add it)
> - User profile updates must NEVER allow the user to change their email to one
>   already taken by another user
> - Addresses are embedded in the User document — never a separate collection
>
> Today we are working on: [paste the specific day below]"

---

## DAY 1 — Review + Wishlist Documents + App Scaffold

### Prompt:

> "Create the reviews and wishlist apps with their mongoengine Documents.
> Do NOT build any views or serializers yet — just the data layer and scaffolding.
>
> **Step 1 — Scaffold both apps:**
> ```
> backend/apps/reviews/
> ├── __init__.py
> ├── documents.py
> ├── serializers.py    ← empty placeholder
> ├── views.py          ← empty placeholder
> ├── urls.py           ← empty placeholder (empty urlpatterns = [])
> └── tests.py          ← empty placeholder
>
> backend/apps/wishlist/
> ├── __init__.py
> ├── documents.py
> ├── serializers.py    ← empty placeholder
> ├── views.py          ← empty placeholder
> ├── urls.py           ← empty placeholder (empty urlpatterns = [])
> └── tests.py          ← empty placeholder
> ```
> Register both in INSTALLED_APPS in config/settings/base.py.
> Add empty url includes to config/urls.py to prevent import errors.
>
> **Step 2 — Create apps/reviews/documents.py** with:
>
> `Review` (Document):
> ```
> product_id         ObjectIdField, required      — ref to Product._id
> user_id            ObjectIdField, required      — ref to User._id
> order_id           ObjectIdField, required      — ref to Order._id (proves purchase)
> rating             IntField, required, min=1, max=5
> title              StringField, required, max_length=120
> body               StringField, required, min_length=10
> images             ListField of StringField     — Cloudinary URLs (optional, max 3)
> is_verified_purchase BooleanField, default=True — always True when created via our API
> helpful_votes      IntField, default=0          — for future 'Was this helpful?' feature
> created_at         DateTimeField, default=datetime.utcnow
> updated_at         DateTimeField, default=datetime.utcnow
> ```
> meta indexes:
>   - product_id + created_at desc (for product review list, newest first)
>   - user_id + product_id (unique=True — one review per user per product)
>   - order_id (for lookup during verified purchase check)
>
> Add save() override that stamps updated_at on every save.
>
> Add a classmethod `has_purchased(user_id, product_id)`:
>   - Queries the Order collection for any order where:
>     - order.user_id == user_id
>     - order.status in ['paid', 'processing', 'shipped', 'delivered']
>     - order.items contains an item with product_id == product_id
>   - Returns the first matching Order or None
>   - WHY: This is the verified purchase check. We only call it here, never inline in views.
>
> Add a classmethod `has_reviewed(user_id, product_id)`:
>   - Returns the existing Review for this user+product, or None
>   - Used to prevent duplicate reviews
>
> **Step 3 — Create `apps/reviews/utils.py`** with:
>
> `recalculate_product_rating(product_id)`:
>   - Queries all reviews for this product_id
>   - Calculates new avg_rating (round to 1 decimal) and review_count
>   - Updates the Product document atomically:
>     ```python
>     Product.objects(id=product_id).update_one(
>         set__avg_rating=new_avg,
>         set__review_count=new_count,
>     )
>     ```
>   - WHY atomic update instead of fetching and saving:
>     If two reviews are submitted simultaneously, a read-then-write approach would
>     lose one update. The atomic set__ ensures both are reflected.
>   - Called after every review create, update, and delete.
>
> **Step 4 — Create apps/wishlist/documents.py** with:
>
> `WishlistItem` (EmbeddedDocument) — embedded inside Wishlist:
> ```
> product_id   ObjectIdField, required
> added_at     DateTimeField, default=datetime.utcnow
> ```
>
> `Wishlist` (Document):
> ```
> user_id   ObjectIdField, required, unique   — one wishlist per user
> items     ListField of EmbeddedDocumentField(WishlistItem)
> updated_at DateTimeField, default=datetime.utcnow
> ```
> meta indexes:
>   - user_id (unique=True — enforces one wishlist per user)
>
> Add save() override that stamps updated_at on every save.
>
> Add classmethods to Wishlist:
>
> `get_or_create_for_user(user_id)`:
>   Returns (wishlist, created) — gets existing or creates empty wishlist.
>
> Add instance methods to Wishlist:
>
> `has_product(product_id)`:
>   Returns True if any item.product_id matches (cast both to str for comparison).
>
> `add_product(product_id)`:
>   Appends a new WishlistItem if not already present. Saves.
>
> `remove_product(product_id)`:
>   Removes the item with matching product_id. Saves.
>
> `toggle(product_id)`:
>   Calls remove_product if has_product, else add_product.
>   Returns ('added', product_id) or ('removed', product_id).
>
> **Step 5 — Verify setup:**
> ```bash
> python manage.py check --settings=config.settings.dev
> ```
> Then in Django shell:
> ```python
> from apps.reviews.documents import Review
> from apps.wishlist.documents import Wishlist, WishlistItem
> from apps.reviews.utils import recalculate_product_rating
> print('All imports OK')
> ```
>
> Show all complete file contents."

---

## DAY 2 — Review API Endpoints (Create, List, Update, Delete)

### Prompt:

> "Build the full reviews API. Reviews require verified purchase — meaning the user
> must have a paid order containing the product before they can submit a review.
>
> **Step 1 — Create apps/reviews/serializers.py** with:
>
> `ReviewSerializer` (read-only — for responses):
>   Fields: id (SerializerMethodField → str), product_id (→ str), user_id (→ str),
>           rating, title, body, images, is_verified_purchase,
>           helpful_votes, created_at, updated_at
>   Add a SerializerMethodField `user_name` that returns the reviewer's
>   first_name + last_name initial (e.g. 'John K.') — look up from User document.
>   Add a SerializerMethodField `user_avatar` that returns user.avatar_url or None.
>
> `CreateReviewSerializer` (for POST input validation):
>   Fields:
>     product_id   CharField, required   — cast to ObjectId in the view
>     rating       IntegerField, required, min=1, max=5
>     title        CharField, required, max_length=120
>     body         CharField, required, min_length=10
>     images       ListField of CharField, default=[], max 3 items
>   Custom validate() method:
>     - Verify product exists and is active
>     - Verify user has not already reviewed this product (Review.has_reviewed)
>     - Verify user has a qualifying order (Review.has_purchased)
>     - If any check fails: raise serializers.ValidationError with a clear message
>     - If all pass: attach order to validated_data so the view can use it
>
> `UpdateReviewSerializer` (for PATCH input validation):
>   Fields: rating (optional), title (optional), body (optional, min_length=10)
>   At least one field must be provided — validate this in validate().
>
> **Step 2 — Create apps/reviews/views.py** with:
>
> `ProductReviewListView` (APIView, AllowAny):
>   GET /api/v1/products/{product_id}/reviews/
>   Returns all reviews for a product, newest first.
>   Pagination: page and page_size (default 10).
>   Response shape: { count, total_pages, current_page, results: [...] }
>   Use ReviewSerializer for each result.
>
> `ReviewCreateView` (APIView, IsAuthenticated):
>   POST /api/v1/reviews/
>   Validate with CreateReviewSerializer (which handles all business rule checks).
>   Create the Review document.
>   Call recalculate_product_rating(product_id) after saving.
>   Return 201 with ReviewSerializer(review).data.
>
> `ReviewDetailView` (APIView, IsAuthenticated):
>   PATCH /api/v1/reviews/{review_id}/
>   Validate with UpdateReviewSerializer.
>   Verify the review belongs to request.user — return 403 if not.
>   Update only the fields provided (rating, title, body).
>   Call recalculate_product_rating(product_id) if rating was updated.
>   Return 200 with ReviewSerializer(review).data.
>
>   DELETE /api/v1/reviews/{review_id}/
>   Verify the review belongs to request.user — return 403 if not.
>   Delete the review document.
>   Call recalculate_product_rating(product_id) after deleting.
>   Return 204.
>
> **Step 3 — Create apps/reviews/urls.py:**
> ```python
> urlpatterns = [
>     path('products/<str:product_id>/reviews/', ProductReviewListView.as_view(), name='product-reviews'),
>     path('reviews/',                           ReviewCreateView.as_view(),      name='review-create'),
>     path('reviews/<str:review_id>/',           ReviewDetailView.as_view(),      name='review-detail'),
> ]
> ```
> Include in config/urls.py.
>
> **Step 4 — Manual curl tests:**
>
> Make sure you have a paid order in your database from Week 4 testing.
>
> ```bash
> # 1. List reviews for a product (should be empty initially)
> curl http://localhost:8000/api/v1/products/{product_id}/reviews/
>
> # 2. Create a review without auth → expect 401
> curl -X POST http://localhost:8000/api/v1/reviews/ \
>   -H 'Content-Type: application/json' \
>   -d '{\"product_id\": \"{id}\", \"rating\": 5, \"title\": \"Great!\", \"body\": \"Really loved this product.\"}'
>
> # 3. Create a review with auth + valid paid order → expect 201
> curl -X POST http://localhost:8000/api/v1/reviews/ \
>   -H 'Content-Type: application/json' \
>   --cookie 'access_token=YOUR_TOKEN' \
>   -d '{\"product_id\": \"{id}\", \"rating\": 5, \"title\": \"Great!\", \"body\": \"Really loved this product.\"}'
>
> # 4. Try to review the same product again → expect 400
>
> # 5. Try to review a product not in any paid order → expect 400
>
> # 6. Check the product's avg_rating and review_count updated in Atlas
>
> # 7. PATCH the review (change rating to 4) → expect 200, avg_rating recalculated
>
> # 8. DELETE the review → expect 204, review_count back to 0
> ```
>
> Show all complete file contents."

---

## DAY 3 — Wishlist API Endpoints + Django Tests for Reviews

### Prompt:

> "Build the full wishlist API and write the Django test suite for reviews.
>
> **Part 1 — Wishlist API:**
>
> **Step 1 — Create apps/wishlist/serializers.py** with:
>
> `WishlistItemSerializer` (read-only):
>   Fields: product_id (SerializerMethodField → str), added_at
>   Add SerializerMethodFields for product data so the frontend does not
>   need a second API call:
>     product_name       — Product.objects(id=product_id).first().name or ''
>     product_slug       — product.slug or ''
>     product_image      — product.images[0] if product.images else None
>     product_price      — product.base_price
>     product_avg_rating — product.avg_rating
>     in_stock           — any(v.stock > 0 for v in product.variants) if variants else False
>   WHY inline product data: The wishlist page shows product cards. Fetching
>   product data inline avoids N+1 queries by loading it all in one serializer pass.
>   Use .only() to fetch just the needed fields: name, slug, images, base_price,
>   avg_rating, variants.
>
> `WishlistSerializer` (read-only):
>   Fields: id (→ str), user_id (→ str),
>           items (WishlistItemSerializer many=True),
>           item_count (SerializerMethodField → len(obj.items)),
>           updated_at
>
> **Step 2 — Create apps/wishlist/views.py** with:
>
> `WishlistView` (APIView, IsAuthenticated):
>   GET /api/v1/wishlist/
>   Get or create wishlist for request.user.
>   Return WishlistSerializer(wishlist).data with 200.
>
> `WishlistToggleView` (APIView, IsAuthenticated):
>   POST /api/v1/wishlist/toggle/
>   Request body: { product_id: 'string' }
>   Validate product exists and is active — return 404 if not.
>   Call wishlist.toggle(product_id).
>   Returns: { action: 'added' or 'removed', product_id: str, item_count: int }
>   WHY toggle pattern: A single endpoint avoids the race condition of a
>   client checking 'is it wishlisted?' and then calling add/remove separately.
>
> `WishlistClearView` (APIView, IsAuthenticated):
>   DELETE /api/v1/wishlist/
>   Clears all items from the user's wishlist (wishlist.items = [], save).
>   Returns 200 with { detail: 'Wishlist cleared.' }
>
> **Step 3 — Create apps/wishlist/urls.py:**
> ```python
> urlpatterns = [
>     path('wishlist/',        WishlistView.as_view(),       name='wishlist'),
>     path('wishlist/toggle/', WishlistToggleView.as_view(), name='wishlist-toggle'),
>     path('wishlist/clear/',  WishlistClearView.as_view(),  name='wishlist-clear'),
> ]
> ```
> Include in config/urls.py.
>
> **Part 2 — Django tests for reviews:**
>
> **Step 4 — Create apps/reviews/tests.py** using DRF's APITestCase.
>
> setUp:
>   - Clean Review, Wishlist, Order, Cart, Product, Category, User documents
>   - Create seller, customer, customer2 users
>   - Create category and 2 products with in-stock variants
>   - Create a paid order for customer containing product_a
>     (status='paid', with correct embedded OrderItem snapshot)
>
> Review creation tests:
>   test_create_review_requires_authentication
>     — POST /reviews/ without JWT → 401
>
>   test_create_review_verified_purchase_success
>     — customer has a paid order with product_a → POST → 201
>     — Response contains rating, title, body, user_name
>
>   test_create_review_updates_product_avg_rating
>     — After creating a 4-star review, fetch product → avg_rating == 4.0, review_count == 1
>
>   test_create_review_without_purchase_returns_400
>     — customer2 has no order with product_a → POST → 400
>     — Error message mentions verified purchase
>
>   test_create_duplicate_review_returns_400
>     — customer reviews product_a twice → second attempt → 400
>
>   test_create_review_invalid_rating_returns_400
>     — rating=6 → 400
>
>   test_create_review_body_too_short_returns_400
>     — body='ok' (less than 10 chars) → 400
>
> Review list tests:
>   test_list_reviews_for_product
>     — GET /products/{product_id}/reviews/ → 200, results array
>
>   test_list_reviews_paginated
>     — Create 5 reviews (5 users each with a paid order), GET ?page_size=3 → count=5, results length=3
>
>   test_list_reviews_no_auth_required
>     — GET /products/{id}/reviews/ without token → 200
>
> Review update/delete tests:
>   test_update_review_by_owner
>     — PATCH /reviews/{id}/ with new rating → 200, avg_rating recalculated
>
>   test_update_review_by_non_owner_returns_403
>     — customer2 tries to PATCH customer's review → 403
>
>   test_delete_review_by_owner
>     — DELETE /reviews/{id}/ → 204, review_count drops to 0
>
>   test_delete_review_by_non_owner_returns_403
>     — customer2 tries to DELETE → 403
>
>   test_delete_review_recalculates_rating
>     — 2 reviews (ratings 4 and 2) → delete the 4-star review → avg_rating == 2.0
>
> Run tests:
> ```bash
> python manage.py test apps.reviews apps.wishlist apps.orders apps.cart \
>   apps.products apps.authentication --settings=config.settings.test -v 2
> ```
>
> Show all complete file contents."

---

## DAY 4 — User Profile API (View + Update + Addresses)

### Prompt:

> "Build the user profile API — allowing users to view and update their own profile
> and manage their saved addresses. All endpoints live under /api/v1/users/.
>
> **Step 1 — Create apps/users/serializers.py** with:
>
> `AddressSerializer`:
>   Fields: label, street, city, country, is_default
>   All required except is_default (default False)
>
> `UserProfileSerializer` (read-only — for GET /users/me/):
>   Fields: id (→ str), email, first_name, last_name, phone, avatar_url,
>           role, is_verified, addresses (AddressSerializer many=True), created_at
>
> `UpdateProfileSerializer` (for PATCH /users/me/):
>   Fields: first_name, last_name, phone — all optional
>   Validate that at least one field is provided.
>   WHY no email here: changing email requires re-verification (Week 8).
>   WHY no password here: password change has its own endpoint.
>
> `ChangePasswordSerializer` (for POST /users/me/change-password/):
>   Fields: current_password (required), new_password (required, min 8 chars),
>           confirm_password (required)
>   validate(): verify new_password == confirm_password
>   WHY validate confirm_password here: catches typos before hitting the DB.
>
> `AddAddressSerializer` (for POST /users/me/addresses/):
>   Fields: label (required, e.g. 'Home', 'Work'), street, city, country, is_default
>   validate(): if is_default is True, warn but allow — the view will clear
>   other default flags before saving.
>
> **Step 2 — Create apps/users/views.py** with:
>
> `UserProfileView` (APIView, IsAuthenticated):
>   GET /api/v1/users/me/
>   Look up User document by request.user.id.
>   Return UserProfileSerializer(user).data.
>
>   PATCH /api/v1/users/me/
>   Validate with UpdateProfileSerializer.
>   Update only the fields that were provided (use .get() to skip missing fields).
>   Save and return updated UserProfileSerializer(user).data.
>
> `ChangePasswordView` (APIView, IsAuthenticated):
>   POST /api/v1/users/me/change-password/
>   Validate with ChangePasswordSerializer.
>   Verify current_password using user.check_password() — return 400 if wrong.
>   Call user.set_password(new_password) and save.
>   Return 200 { detail: 'Password updated successfully.' }
>   WHY not return a new token: the user's existing JWT cookies remain valid.
>   Password changes do not invalidate JWTs in our current setup
>   (token blacklisting is a Week 11 security hardening task).
>
> `AvatarUploadView` (APIView, IsAuthenticated):
>   POST /api/v1/users/me/avatar/
>   Accepts multipart: field name 'avatar'
>   Validate file type (jpeg/png/webp only) and size (max 2MB).
>   Upload to Cloudinary under folder: ecommerce/avatars/{user_id}/
>   Update user.avatar_url and save.
>   Return 200 { avatar_url: 'https://...' }
>
> `AddressListView` (APIView, IsAuthenticated):
>   GET /api/v1/users/me/addresses/
>   Returns user.addresses list serialized with AddressSerializer.
>
>   POST /api/v1/users/me/addresses/
>   Validate with AddAddressSerializer.
>   If is_default is True: loop through existing addresses and set is_default=False.
>   Append new Address EmbeddedDocument to user.addresses.
>   Enforce max 5 addresses — return 400 if already at 5.
>   Save and return updated address list.
>
> `AddressDetailView` (APIView, IsAuthenticated):
>   DELETE /api/v1/users/me/addresses/{index}/
>   Remove address at given index from user.addresses.
>   Return 200 with updated address list.
>
>   PATCH /api/v1/users/me/addresses/{index}/
>   Update specific fields of the address at index.
>   If is_default is being set to True: clear other defaults first.
>   Return 200 with updated address list.
>
> **Step 3 — Create apps/users/urls.py:**
> ```python
> urlpatterns = [
>     path('users/me/',                    UserProfileView.as_view(),    name='user-profile'),
>     path('users/me/change-password/',    ChangePasswordView.as_view(), name='change-password'),
>     path('users/me/avatar/',             AvatarUploadView.as_view(),   name='avatar-upload'),
>     path('users/me/addresses/',          AddressListView.as_view(),    name='address-list'),
>     path('users/me/addresses/<int:index>/', AddressDetailView.as_view(), name='address-detail'),
> ]
> ```
> Include in config/urls.py.
>
> **Step 4 — Manual curl tests:**
> ```bash
> # 1. GET /users/me/ → full profile with addresses
> # 2. PATCH /users/me/ with { first_name: 'New' } → updated profile returned
> # 3. PATCH /users/me/ with no fields → 400
> # 4. POST /users/me/change-password/ with wrong current_password → 400
> # 5. POST /users/me/change-password/ with mismatched new/confirm → 400
> # 6. POST /users/me/change-password/ correctly → 200
> # 7. POST /users/me/addresses/ with valid address → address added
> # 8. POST /users/me/addresses/ when already 5 addresses → 400
> # 9. PATCH /users/me/addresses/0/ set is_default=True → other addresses cleared
> # 10. DELETE /users/me/addresses/0/ → address removed
> ```
>
> Show all complete file contents."

---

## DAY 5 — React: reviewAPI + wishlistAPI + Hooks

### Prompt:

> "Build the React data layer for reviews and wishlist — API files, TanStack Query
> hooks, and the Redux wishlist slice.
>
> **Step 1 — Create src/api/reviewAPI.js:**
> ```javascript
> reviewAPI.listForProduct(productId, page)
>   GET /products/{productId}/reviews/?page={page}
>   Returns { count, total_pages, current_page, results }
>
> reviewAPI.create(data)
>   POST /reviews/
>   Body: { product_id, rating, title, body, images }
>   Returns the created Review object
>
> reviewAPI.update(reviewId, data)
>   PATCH /reviews/{reviewId}/
>   Body: { rating?, title?, body? }
>   Returns the updated Review object
>
> reviewAPI.delete(reviewId)
>   DELETE /reviews/{reviewId}/
>   Returns nothing (204)
> ```
> All use axiosInstance (withCredentials: true). All return response.data.
>
> **Step 2 — Create src/api/wishlistAPI.js:**
> ```javascript
> wishlistAPI.get()
>   GET /wishlist/
>   Returns full Wishlist object
>
> wishlistAPI.toggle(productId)
>   POST /wishlist/toggle/
>   Body: { product_id: productId }
>   Returns { action, product_id, item_count }
>
> wishlistAPI.clear()
>   DELETE /wishlist/
>   Returns { detail }
> ```
>
> **Step 3 — Create src/api/userAPI.js:**
> ```javascript
> userAPI.getProfile()
>   GET /users/me/
>
> userAPI.updateProfile(data)
>   PATCH /users/me/
>
> userAPI.changePassword(data)
>   POST /users/me/change-password/
>
> userAPI.uploadAvatar(file)
>   POST /users/me/avatar/ using FormData
>
> userAPI.getAddresses()
>   GET /users/me/addresses/
>
> userAPI.addAddress(data)
>   POST /users/me/addresses/
>
> userAPI.updateAddress(index, data)
>   PATCH /users/me/addresses/{index}/
>
> userAPI.deleteAddress(index)
>   DELETE /users/me/addresses/{index}/
> ```
>
> **Step 4 — Create src/hooks/useReviews.js:**
> ```javascript
> useProductReviews(productId, page):
>   queryKey: ['reviews', productId, page]
>   queryFn: reviewAPI.listForProduct(productId, page)
>   enabled: !!productId
>   staleTime: 2 * 60 * 1000
>   keepPreviousData: true
>
> useCreateReview():
>   useMutation wrapping reviewAPI.create()
>   onSuccess: invalidate ['reviews', productId] and ['product', slug]
>   (invalidating product refetches avg_rating and review_count)
>
> useUpdateReview():
>   useMutation wrapping reviewAPI.update(reviewId, data)
>   onSuccess: invalidate reviews and product queries
>
> useDeleteReview():
>   useMutation wrapping reviewAPI.delete(reviewId)
>   onSuccess: invalidate reviews and product queries
> ```
>
> **Step 5 — Create src/store/wishlistSlice.js** (Redux Toolkit):
> State: { items: [], isLoading: false, error: null }
>   — items is the array of wishlist product_ids (strings) for quick lookup
>   — the full wishlist object lives in TanStack Query cache, not Redux
>   — Redux stores only the IDs because every component needs to check
>     'is this product wishlisted?' without re-fetching the full list
>
> Async thunks:
>   fetchWishlist → calls wishlistAPI.get() → sets state.items to array of product_id strings
>   toggleWishlist(productId) → calls wishlistAPI.toggle() → updates state.items optimistically
>
> Selectors:
>   selectWishlistIds       — state.wishlist.items (array of product_id strings)
>   selectIsWishlisted(id)  — selector factory: (state) => state.wishlist.items.includes(id)
>   selectWishlistCount     — state.wishlist.items.length
>
> **Step 6 — Create src/hooks/useWishlist.js:**
> Returns: { isWishlisted(productId), toggle(productId), wishlistCount, isLoading }
> isWishlisted uses selectWishlistIds from Redux — no extra API calls.
> toggle dispatches toggleWishlist thunk and returns { action: 'added' | 'removed' }.
>
> **Step 7 — Create src/hooks/useProfile.js** with TanStack Query hooks:
> ```javascript
> useProfile():
>   queryKey: ['profile']
>   queryFn: userAPI.getProfile()
>   staleTime: 5 * 60 * 1000
>   enabled: isAuthenticated (import from Redux auth selector)
>
> useUpdateProfile():
>   useMutation wrapping userAPI.updateProfile()
>   onSuccess: invalidate ['profile'] and update authSlice user state
>
> useChangePassword():
>   useMutation wrapping userAPI.changePassword()
>
> useAddresses():
>   queryKey: ['addresses']
>   queryFn: userAPI.getAddresses()
>   staleTime: 10 * 60 * 1000
> ```
>
> **Step 8 — Wire wishlist initialization:**
> In src/main.jsx (or wherever initializeAuth is dispatched),
> also dispatch fetchWishlist after initializeAuth resolves:
> ```javascript
> store.dispatch(initializeAuth()).then(() => {
>   store.dispatch(fetchCart());
>   store.dispatch(fetchWishlist());
> });
> ```
>
> Show all complete file contents."

---

## DAY 6 — React: Review Components + ProductDetail Wiring + Wishlist Button

### Prompt:

> "Build the review UI components, wire them into the ProductDetail page,
> and add the wishlist heart button throughout the app.
>
> **Step 1 — Create src/components/review/StarInput.jsx:**
> An interactive 5-star rating input for the review form.
> Props: value (number 1-5), onChange (callback)
> Shows 5 star icons — filled for value and below, empty above.
> Stars are clickable and hoverable (hover preview before clicking).
> Accessible: use role='radio' or button with aria-label.
>
> **Step 2 — Create src/components/review/ReviewCard.jsx:**
> Props: review (object from ReviewSerializer)
> Shows:
>   - User avatar (or initials fallback circle) + user_name
>   - StarRating (read-only, from existing StarRating component)
>   - review.title in bold
>   - review.body
>   - review.images (if any) as a small thumbnail row, lightbox on click
>   - created_at formatted as 'Mar 12, 2025'
>   - 'Verified Purchase' badge (green checkmark + text) if is_verified_purchase
> If the review belongs to the authenticated user (review.user_id == auth user id):
>   Show Edit (pencil) and Delete (trash) icon buttons.
>   Edit opens an inline edit form (same fields as create form, pre-populated).
>   Delete shows a confirm dialog (window.confirm or a small inline confirmation).
>
> **Step 3 — Create src/components/review/ReviewForm.jsx:**
> Used for both creating and editing reviews.
> Props: productId, existingReview (null for create, review object for edit), onSuccess
> Fields (react-hook-form + Zod):
>   rating: required, 1-5 (use StarInput component)
>   title: required, max 120 chars
>   body: required, min 10 chars
> On submit (create): call useCreateReview() mutation.
>   Show 'Only verified purchasers can review this product.' error if 400.
> On submit (edit): call useUpdateReview(existingReview.id) mutation.
> On success: call onSuccess() callback + show toast.
> Loading state: disable submit button and show spinner.
>
> **Step 4 — Create src/components/review/ReviewSection.jsx:**
> This is the complete reviews tab content rendered inside ProductDetail.
> Props: product (full product object), userOrderIds (array of order ids containing this product)
>
> Layout:
>   Rating summary at top:
>     Large avg_rating number (e.g. 4.3) + stars + review_count text
>     5-to-1 star breakdown bars (percentage of each rating) — calculate from fetched reviews
>   'Write a Review' button:
>     If not authenticated: shows 'Login to write a review' → link to /login
>     If authenticated but hasn't purchased: shows 'Buy this product to leave a review' (disabled)
>     If authenticated and has purchased but already reviewed: shows 'You reviewed this' (disabled)
>     If authenticated and has purchased and has not reviewed: shows the ReviewForm
>   Reviews list:
>     Paginated list of ReviewCard components using useProductReviews hook
>     'Load more' button at the bottom (or page number buttons)
>     Empty state: 'No reviews yet. Be the first to review!'
>
> **Step 5 — Update src/pages/ProductDetail.jsx:**
>   Replace the Reviews tab placeholder with ReviewSection component.
>   Pass product.id as productId to ReviewSection.
>   To determine if the current user has purchased this product:
>     Use useOrders(1) to get recent orders, check if any contain this product.
>     Pass the matching order ids to ReviewSection.
>   When useDeleteReview() succeeds: show toast 'Review deleted'.
>
> **Step 6 — Create src/components/product/WishlistButton.jsx:**
> Props: productId (string), size ('sm' | 'md'), className (optional)
> Uses useWishlist() hook.
> Shows a heart icon (lucide-react Heart).
>   Filled red heart: product is wishlisted
>   Outline heart: not wishlisted
> On click:
>   If not authenticated: show toast 'Login to save items to your wishlist'
>   If authenticated: call toggle(productId)
>     On toggle success: show toast 'Added to wishlist' or 'Removed from wishlist'
> Include a subtle scale animation on click (Tailwind: active:scale-110 transition-transform).
>
> **Step 7 — Add WishlistButton to ProductCard.jsx:**
> Add the WishlistButton (size='sm') as an overlay on the product image:
> ```jsx
> <div className='relative'>
>   <img ... />
>   <div className='absolute top-2 right-2'>
>     <WishlistButton productId={product.id} size='sm' />
>   </div>
> </div>
> ```
>
> **Step 8 — Add WishlistButton to ProductDetail.jsx:**
> Place the WishlistButton (size='md') next to the 'Add to Cart' button:
> ```jsx
> <div className='flex gap-3'>
>   <button ...>Add to Cart</button>
>   <WishlistButton productId={product.id} size='md' />
> </div>
> ```
>
> Show all complete file contents."

---

## DAY 7 — Wishlist Page + User Profile Page + Week 5 Cleanup + Full Verification

### Prompt:

> "Build the Wishlist page and User Profile page, add Django tests for wishlist
> and profile, then run the full Week 5 verification checklist.
>
> **Step 1 — Create src/pages/user/Wishlist.jsx:**
>
> Fetches the wishlist with useQuery(['wishlist'], wishlistAPI.get).
>
> Layout:
>   Page title: 'My Wishlist' with item count badge.
>   Empty state: heart icon + 'Your wishlist is empty' + 'Discover Products' button to /products.
>   Loading state: grid of skeleton ProductCard shapes (same grid as ProductList).
>   Wishlist items grid: responsive (1/2/3/4 columns) using ProductCard components.
>     Each ProductCard already has WishlistButton overlay from Day 6.
>     When WishlistButton is toggled to remove, invalidate ['wishlist'] query so the
>     item disappears from the grid immediately.
>   'Clear Wishlist' button (top right, only shown if items > 0):
>     Calls wishlistAPI.clear() then invalidates the wishlist query.
>     Shows a confirm dialog first.
>
> **Step 2 — Create src/pages/user/Profile.jsx:**
>
> Two-column layout on desktop. Left: sidebar nav. Right: active section content.
>
> Sidebar tabs (use URL hash or local state to switch):
>   Personal Info | Change Password | Saved Addresses
>
> Personal Info section:
>   Avatar: round image (or initials circle if no avatar).
>     'Change Photo' button → file input → calls userAPI.uploadAvatar() → updates avatar_url.
>     Show upload progress indicator.
>   Form (react-hook-form + Zod):
>     first_name (required), last_name (required), phone (optional)
>   'Save Changes' button → calls useUpdateProfile() mutation.
>   Show success toast on save.
>   Display email in a disabled read-only field with note '(Email cannot be changed here)'.
>
> Change Password section:
>   Form: current_password, new_password (min 8), confirm_password.
>   All fields are password type with show/hide toggle.
>   Submit → calls useChangePassword() mutation.
>   On success: clear the form fields, show toast 'Password updated successfully.'
>   On 400 (wrong current password): show inline error under current_password field.
>
> Saved Addresses section:
>   List of address cards (from user.addresses).
>   Each card shows: label badge, street, city, country, 'Default' badge if is_default.
>   Each card has Edit (inline form, same fields) and Delete (confirm first) buttons.
>   'Set as Default' button on non-default addresses.
>   'Add New Address' button at bottom → shows an inline add form.
>   Enforce max 5 addresses — hide 'Add New Address' when at limit.
>
> **Step 3 — Update src/router.jsx:**
> Add these protected routes:
> ```jsx
> { path: '/wishlist', element: <Wishlist /> },
> { path: '/profile',  element: <Profile />  },
> ```
>
> **Step 4 — Update the Navbar:**
> Add wishlist icon (Heart from lucide-react) between the search bar and cart icon.
> Shows a count badge from selectWishlistCount Redux selector.
> On click: navigates to /wishlist.
> The Profile link in the user dropdown now points to /profile (was a stub in Week 4).
>
> **Step 5 — Write Django tests for wishlist:**
> Create apps/wishlist/tests.py with:
>
>   test_get_wishlist_requires_authentication → 401
>   test_get_wishlist_creates_if_not_exists → GET /wishlist/ → 200, empty items
>   test_toggle_add_product → POST /wishlist/toggle/ → { action: 'added' }, item_count=1
>   test_toggle_remove_product → toggle same product twice → action='removed', item_count=0
>   test_toggle_nonexistent_product → 404
>   test_wishlist_item_includes_product_data → response items include product_name, product_price
>   test_clear_wishlist → DELETE /wishlist/ → 200, items=[]
>   test_clear_empty_wishlist → 200 (not an error)
>
> **Step 6 — Full Week 5 verification checklist:**
>
> Backend:
>   1. POST /reviews/ without auth → 401
>   2. POST /reviews/ with auth but no purchase → 400 with clear error
>   3. POST /reviews/ with auth and paid order → 201, avg_rating updated on product
>   4. POST /reviews/ duplicate → 400
>   5. PATCH /reviews/{id}/ by owner → 200, avg_rating recalculated
>   6. DELETE /reviews/{id}/ by owner → 204, review_count decremented
>   7. PATCH or DELETE by non-owner → 403
>   8. GET /wishlist/ → 200, empty items for new user
>   9. POST /wishlist/toggle/ → adds then removes product correctly
>   10. GET /users/me/ → full profile with addresses
>   11. PATCH /users/me/ → updates first_name, saves correctly
>   12. POST /users/me/change-password/ with wrong password → 400
>   13. POST /users/me/change-password/ correctly → 200
>   14. POST /users/me/addresses/ → address added, max 5 enforced
>   15. Run all tests: python manage.py test apps.reviews apps.wishlist apps.users apps.orders apps.cart apps.products apps.authentication --settings=config.settings.test -v 2
>
> Frontend:
>   16. ProductDetail reviews tab → shows rating summary, review count, star breakdown
>   17. As logged-in user with a paid order → 'Write a Review' form appears
>   18. Submit a review → appears in the list, avg_rating updates on the page
>   19. Edit own review inline → updated review appears
>   20. Delete own review → disappears from list, count decrements
>   21. As user without a purchase → 'Buy this product' message (no form shown)
>   22. ProductCard hover → wishlist heart appears in top-right corner of image
>   23. Click heart when not logged in → toast 'Login to save items'
>   24. Click heart when logged in → toggles, filled/outline state changes
>   25. Navbar wishlist icon shows count badge after adding items
>   26. Navigate to /wishlist → wishlist page shows saved products as product cards
>   27. Remove from wishlist on wishlist page → item disappears immediately
>   28. 'Clear Wishlist' confirms and clears all items
>   29. Navigate to /profile → profile page loads with personal info pre-filled
>   30. Update first name → saves and shows toast
>   31. Upload avatar → avatar image updates in the profile and navbar
>   32. Change password flow works end to end
>   33. Add a new address → appears in address list
>   34. Set an address as default → other addresses lose default badge
>   35. Delete an address → removed from list
>
> **Step 7 — Provide the Week 5 session handoff summary:**
>
> ## Session Handoff
>
> What was built:
> Files created:
> Files modified:
> Decisions made:
> Current working state:
>   Backend:
>   Frontend:
> Next session starts with:"

---

## WEEK 5 SUMMARY — What You Will Have Built

By the end of Day 7, your project will have:

**Backend:**
- `Review` Document with verified-purchase enforcement via `has_purchased()` classmethod
- `recalculate_product_rating()` utility — atomic avg_rating + review_count update after every change
- One-review-per-user-per-product enforced at the database index level
- `GET /api/v1/products/{product_id}/reviews/` — paginated, no auth required
- `POST /api/v1/reviews/` — verified purchase check, duplicate check, rating recalculation
- `PATCH /api/v1/reviews/{id}/` — owner-only, recalculates rating if changed
- `DELETE /api/v1/reviews/{id}/` — owner-only, recalculates rating after deletion
- `Wishlist` Document with embedded `WishlistItem` — one per user, toggle pattern
- `GET /api/v1/wishlist/` — returns full wishlist with inline product data
- `POST /api/v1/wishlist/toggle/` — adds or removes in one endpoint
- `DELETE /api/v1/wishlist/` — clears all items
- `GET /PATCH /api/v1/users/me/` — view and update own profile
- `POST /api/v1/users/me/change-password/` — validates current password before updating
- `POST /api/v1/users/me/avatar/` — Cloudinary upload, updates avatar_url
- `GET /POST /api/v1/users/me/addresses/` — list and add embedded addresses (max 5)
- `PATCH /DELETE /api/v1/users/me/addresses/{index}/` — update or remove by index
- Full Django test suites for reviews, wishlist, and user profile

**Frontend:**
- `reviewAPI.js`, `wishlistAPI.js`, `userAPI.js` — all API calls
- `useProductReviews`, `useCreateReview`, `useUpdateReview`, `useDeleteReview` hooks
- `useWishlist` hook backed by `wishlistSlice` (stores only IDs for fast lookups)
- `useProfile`, `useUpdateProfile`, `useChangePassword`, `useAddresses` hooks
- `StarInput.jsx` — interactive 5-star input with hover preview
- `ReviewCard.jsx` — shows reviewer info, rating, body, verified badge, owner edit/delete
- `ReviewForm.jsx` — create and edit reviews with Zod validation
- `ReviewSection.jsx` — full reviews tab: rating summary, star breakdown bars, write form, paginated list
- `WishlistButton.jsx` — heart icon toggle, auth-aware, toast feedback
- `ProductCard.jsx` updated — WishlistButton overlay on image
- `ProductDetail.jsx` updated — ReviewSection wired into Reviews tab, WishlistButton next to Add to Cart
- `Wishlist.jsx` page — grid of wishlist products, clear wishlist, empty state
- `Profile.jsx` page — 3 sections: personal info + avatar upload, change password, addresses CRUD
- `wishlistSlice.js` — Redux store for wishlist IDs, fetchWishlist + toggleWishlist thunks
- Navbar — wishlist heart icon with count badge, /profile link active
- Router — /wishlist and /profile protected routes added

**Week 6 continues with:** Celery + Redis setup, all email types via SendGrid
(registration verification, password reset, order confirmation, shipping notification),
and background task queue for heavy operations.
