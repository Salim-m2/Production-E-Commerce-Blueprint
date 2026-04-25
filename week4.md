# Week 3 Handoff Summary + Week 4 Implementation Prompt — Checkout + Stripe Payments
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## WEEK 3 HANDOFF SUMMARY

### What Was Built

**Backend:**
- `Cart` Document with embedded `CartItem` EmbeddedDocument (mongoengine)
- `CartItem` snapshots: product_name, variant_sku, color, size, image_url, price_at_add (frozen at add-time, never recalculated)
- Atomic stock validation using MongoDB `$elemMatch` query pattern (never read-then-write)
- Guest cart identification via `X-Session-Key` request header
- `Cart.get_for_user()`, `Cart.get_for_session()`, `get_or_create_*()` classmethods
- `get_subtotal()`, `get_item_count()`, `find_item()` instance methods
- `GET /api/v1/cart/` — works for guests and authenticated users
- `POST /api/v1/cart/items/` — add item with full snapshot fields
- `PATCH /api/v1/cart/items/{index}/` — update quantity with stock validation
- `DELETE /api/v1/cart/items/{index}/` — remove item
- `POST /api/v1/cart/merge/` — merge guest cart into user cart on login
- `POST /api/v1/cart/coupon/` — skeleton (real logic in Week 10)
- Full Django APITestCase suite: 25 tests, all passing
- `config/settings/test.py` isolated to `ecommerce_test_db`

**Frontend:**
- `cartAPI.js` — all cart API calls with automatic X-Session-Key header injection
- `ensureSessionKey()` / `getSessionKey()` — UUID v4 guest identification
- Full `cartSlice.js` — fetchCart, addToCart, updateCartItem, removeFromCart, mergeCart thunks + drawer actions
- Selectors: selectCart, selectCartItems, selectCartItemCount, selectCartSubtotal, selectCartIsOpen, selectCartIsLoading
- `useCart.js` hook — wraps all cart state and actions
- `CartItem.jsx` — image, name, variant details, quantity control, line total, remove button
- `CartSummary.jsx` — subtotal, shipping note, total, checkout button (stub), continue shopping
- `CartDrawer.jsx` — slide-in panel from right, overlay, empty state, loading skeletons
- `Cart.jsx` (full page) — two-column layout, coupon input, price breakdown
- ProductDetail → "Add to Cart" wired to real cartAPI using `matchedVariant`
- Cart merge dispatched automatically after `loginUser` thunk succeeds
- Cart cleared automatically after `logoutUser` thunk
- Navbar cart icon dispatches `toggleCartDrawer`
- Navbar badge shows live item count

### Files Created
**Backend:** `apps/cart/documents.py`, `apps/cart/serializers.py`, `apps/cart/views.py`, `apps/cart/urls.py`, `apps/cart/tests.py`, `apps/cart/indexes.py`, `config/settings/test.py`

**Frontend:** `src/api/cartAPI.js`, `src/store/cartSlice.js` (replaced stub), `src/hooks/useCart.js`, `src/components/cart/CartItem.jsx`, `src/components/cart/CartSummary.jsx`, `src/components/cart/CartDrawer.jsx`, `src/pages/Cart.jsx`

### Files Modified
**Backend:** `config/settings/base.py`, `config/urls.py`, `apps/products/documents.py` (Variant auto uuid)

**Frontend:** `src/store/authSlice.js` (merge + clear on login/logout), `src/store/index.js`, `src/main.jsx` (RouterProvider + fetchCart dispatch), `src/router.jsx` (CartDrawer in RootLayout, /cart route), `src/pages/ProductDetail.jsx` (real addToCart), `src/components/layout/Navbar.jsx` (cart badge + drawer toggle)

### Current Working State
- Backend: Django running on port 8000, 50 seeded products, 8 categories, MongoDB Atlas connected
- Frontend: React running on port 5173, full cart flow working end-to-end
- Tests: 25 cart tests + all auth + product tests passing
- Real DB: 50 products, 8 categories, 0 carts, 0 leaked test data

### Next Session Starts With
Week 4, Day 1 — Order Document + Payment Document (mongoengine) + Stripe account setup

---

---

# Week 4 AI Implementation Prompt — Checkout + Stripe Payments
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## CONTEXT TO GIVE THE AI AT THE START OF EVERY SESSION THIS WEEK

> "I am continuing a production e-commerce build. Weeks 1, 2, and 3 are fully complete and tested.
>
> **What already exists and is working:**
> - Django project with split settings (base.py / dev.py / prod.py / test.py), loaded via django-environ
> - MongoDB Atlas connected via mongoengine
> - User Document with set_password() and check_password() methods
> - Full JWT auth in httpOnly cookies: register, login, logout, token refresh, /auth/me/
> - Password reset skeleton (Token Document with TTL, endpoints without emails yet)
> - DRF throttling on all endpoints
> - Django APITestCase tests for all auth, product, and cart endpoints (all passing)
> - Product, Category, Variant (embedded) mongoengine Documents with all indexes
> - Product list API: pagination, filtering, search, 5 sort options
> - Product detail API: full document with all variants
> - Category tree API: nested structure, cached 1 hour
> - Cloudinary integration: image upload with IsAdminOrSeller permission
> - Cart Document with embedded CartItem EmbeddedDocument
> - CartItem snapshots: product_name, variant_sku, color, size, image_url, price_at_add
> - Atomic stock validation using MongoDB $elemMatch
> - Full cart API: GET, POST, PATCH, DELETE, merge, coupon (skeleton)
> - React 18 + Vite frontend with Tailwind CSS
> - Redux Toolkit store: authSlice (with isInitialized), cartSlice, uiSlice
> - TanStack Query for all server data
> - Axios instance with silent JWT refresh interceptor (withCredentials: true)
> - cartAPI.js, useCart hook, CartItem, CartSummary, CartDrawer, Cart page
> - ProductDetail with real Add to Cart wired to cartSlice
> - Home page, ProductList page (URL params), ProductDetail page, Cart page
> - React Router v6 with createBrowserRouter, RootLayout, ProtectedRoute, GuestRoute
>
> **Stack reminder:**
> - Backend: Django 4.x + DRF + mongoengine (NO Django ORM, NO SQL)
> - Frontend: React 18 + Vite + Redux Toolkit + TanStack Query + Axios
> - All API endpoints prefixed /api/v1/
> - JWT in httpOnly cookies ONLY — never localStorage
> - Payments: Stripe (PaymentIntents + Webhooks)
> - Styling: Tailwind CSS
>
> **Critical payment rules:**
> - NEVER store card details — Stripe Elements handles all card data in an iframe
> - ALWAYS verify Stripe webhook signatures — reject anything without a valid signature
> - ALWAYS use idempotency keys on PaymentIntent creation to prevent double charges
> - ALWAYS check if a Stripe event has already been processed before acting on it
> - The webhook endpoint is the ONLY endpoint exempt from CSRF — verified by Stripe signature instead
> - Orders are created as 'pending' before payment — only marked 'paid' after webhook confirms
>
> Today we are working on: [paste the specific day below]"

---

## DAY 1 — Order + Payment mongoengine Documents + App Scaffold

### Prompt:

> "Create the orders and payments apps with their mongoengine Documents.
>
> **Step 1 — Scaffold both apps:**
> ```
> backend/apps/orders/
> ├── __init__.py
> ├── documents.py
> ├── serializers.py    ← empty placeholder
> ├── views.py          ← empty placeholder
> ├── urls.py           ← empty placeholder
> └── tests.py          ← empty placeholder
>
> backend/apps/payments/
> ├── __init__.py
> ├── documents.py
> ├── serializers.py    ← empty placeholder
> ├── views.py          ← empty placeholder
> └── urls.py           ← empty placeholder
> ```
> Register both in INSTALLED_APPS in config/settings/base.py.
> Add empty url includes to config/urls.py to prevent import errors.
>
> **Step 2 — Create apps/orders/documents.py** with these mongoengine Documents:
>
> OrderItem (EmbeddedDocument) — embedded inside Order, never a separate collection.
> Fields:
> ```
> product_id    ObjectIdField, required
> product_name  StringField, required   — SNAPSHOT at order time
> product_slug  StringField            — SNAPSHOT for linking back to product
> variant_id    StringField, required
> variant_sku   StringField, required   — SNAPSHOT
> color         StringField            — SNAPSHOT, optional
> size          StringField            — SNAPSHOT, optional
> image_url     StringField            — SNAPSHOT, first product image
> quantity      IntField, required, min_value=1
> unit_price    FloatField, required   — SNAPSHOT of price_at_add from cart
> subtotal      FloatField, required   — unit_price * quantity
> ```
>
> StatusHistory (EmbeddedDocument) — tracks every status change.
> Fields:
> ```
> status      StringField, required
> changed_at  DateTimeField, default=datetime.utcnow
> by          StringField, default='system'   — 'system', 'admin', or user email
> note        StringField                     — optional admin note
> ```
>
> Order (Document) — the main orders collection.
> Fields:
> ```
> order_number     StringField, required, unique   — human-readable e.g. ORD-2024-00142
> user_id          ObjectIdField, required
> status           StringField, default='pending'
>                  choices: pending, paid, processing, shipped, delivered, cancelled, refunded
> items            ListField of EmbeddedDocumentField(OrderItem)
> shipping_address EmbeddedDocumentField(ShippingAddress) — define below
> shipping_method  StringField, default='standard'   — 'standard' or 'express'
> shipping_cost    FloatField, default=0.0
> subtotal         FloatField, required   — sum of all item subtotals
> discount         FloatField, default=0.0
> tax              FloatField, default=0.0
> total            FloatField, required   — subtotal + shipping_cost - discount + tax
> coupon_code      StringField            — applied coupon if any
> payment_id       ObjectIdField          — ref to Payment._id (set after payment)
> tracking_number  StringField            — set when shipped
> notes            StringField            — customer notes at checkout
> status_history   ListField of EmbeddedDocumentField(StatusHistory)
> created_at       DateTimeField, default=datetime.utcnow
> updated_at       DateTimeField, default=datetime.utcnow
> ```
>
> ShippingAddress (EmbeddedDocument):
> ```
> full_name   StringField, required
> phone       StringField, required
> street      StringField, required
> city        StringField, required
> country     StringField, default='Kenya'
> postal_code StringField
> ```
>
> Add to Order:
> - save() override that stamps updated_at = datetime.utcnow() on every save
> - classmethod generate_order_number() that produces ORD-{YEAR}-{5-digit-zero-padded-count}
>   Example: ORD-2024-00001, ORD-2024-00142
>   Implementation: count existing orders for current year + 1, zero-pad to 5 digits
> - classmethod create_from_cart(cart, user, shipping_address, shipping_method, notes='')
>   that: builds OrderItems from cart items (copying all snapshot fields from CartItem),
>   calculates subtotal, applies shipping cost (standard=$5, express=$15),
>   generates order_number, saves and returns the Order
> - instance method add_status(status, by='system', note='') that appends to status_history and updates status field
>
> Add indexes to Order meta:
>   user_id + created_at desc, order_number (unique)
>
> **Step 3 — Create apps/payments/documents.py** with:
>
> Payment (Document):
> ```
> order_id                  ObjectIdField, required
> stripe_payment_intent_id  StringField, required, unique
> stripe_event_id           StringField        — stored after webhook to prevent duplicate processing
> amount                    FloatField, required   — in dollars (NOT cents)
> currency                  StringField, default='usd'
> status                    StringField, default='pending'
>                           choices: pending, succeeded, failed, refunded
> metadata                  DictField              — store any extra Stripe data
> created_at                DateTimeField, default=datetime.utcnow
> updated_at                DateTimeField, default=datetime.utcnow
> ```
> Add save() override for updated_at.
> Add index on stripe_payment_intent_id (unique) and stripe_event_id.
>
> **Step 4 — Verify:**
> ```bash
> python manage.py check --settings=config.settings.dev
> ```
> Then in Django shell confirm imports work:
> ```python
> from apps.orders.documents import Order, OrderItem, ShippingAddress
> from apps.payments.documents import Payment
> print('All documents import OK')
> ```
>
> Show all complete file contents."

---

## DAY 2 — Stripe Setup + PaymentIntent Endpoint + Webhook Handler

### Prompt:

> "Set up Stripe and build the two payment endpoints: create-intent and webhook.
>
> **Step 1 — Install Stripe:**
> ```bash
> pip install stripe
> ```
> Add to requirements.txt.
>
> Add to .env:
> ```
> STRIPE_SECRET_KEY=sk_test_...
> STRIPE_WEBHOOK_SECRET=whsec_...
> STRIPE_PUBLISHABLE_KEY=pk_test_...
> ```
>
> Add to config/settings/base.py:
> ```python
> import stripe
> STRIPE_SECRET_KEY       = env('STRIPE_SECRET_KEY')
> STRIPE_WEBHOOK_SECRET   = env('STRIPE_WEBHOOK_SECRET')
> STRIPE_PUBLISHABLE_KEY  = env('STRIPE_PUBLISHABLE_KEY')
> stripe.api_key = STRIPE_SECRET_KEY
> ```
>
> **Step 2 — Create apps/payments/views.py** with two views:
>
> CreatePaymentIntentView (APIView, IsAuthenticated):
>   POST /api/v1/payments/create-intent/
>   Request body: { order_id: 'string' }
>
>   Logic:
>   1. Look up the Order by order_id — verify it belongs to request.user
>   2. Verify order.status == 'pending' — reject if already paid
>   3. Create a Stripe PaymentIntent:
>      ```python
>      intent = stripe.PaymentIntent.create(
>          amount=int(order.total * 100),   # Stripe uses cents
>          currency='usd',
>          metadata={'order_id': str(order.id)},
>          idempotency_key=str(order.id),   # prevents duplicate charges on retry
>      )
>      ```
>   4. Create a Payment document with status='pending', stripe_payment_intent_id=intent.id
>   5. Link payment to order: order.payment_id = payment.id, order.save()
>   6. Return { client_secret: intent.client_secret, publishable_key: settings.STRIPE_PUBLISHABLE_KEY }
>
> StripeWebhookView (APIView):
>   POST /api/v1/payments/webhook/
>   Permission: AllowAny (verified by Stripe signature instead)
>
>   Logic:
>   1. Read raw request body and Stripe-Signature header
>   2. Verify signature:
>      ```python
>      event = stripe.Webhook.construct_event(
>          payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
>      )
>      ```
>      If verification fails: return 400
>   3. Handle payment_intent.succeeded:
>      - Extract order_id from event.data.object.metadata
>      - Check if this stripe_event_id already exists in Payment collection
>        If yes: return 200 immediately (idempotency — already processed)
>      - Find the Payment by stripe_payment_intent_id
>      - Update Payment: status='succeeded', stripe_event_id=event.id
>      - Find the Order, call order.add_status('paid', by='stripe')
>      - Decrement stock for each order item atomically using $elemMatch
>      - ALWAYS return HttpResponse(status=200) — Stripe will retry if it gets anything else
>   4. Handle payment_intent.payment_failed:
>      - Update Payment status to 'failed'
>      - Return 200
>   5. All other event types: return 200 (ignore gracefully)
>
>   IMPORTANT: This endpoint must be CSRF exempt because Stripe cannot send a CSRF token.
>   Add @csrf_exempt decorator or use DRF's APIView which already bypasses Django CSRF
>   for JSON requests. Also exempt from JWT authentication.
>
> **Step 3 — Create apps/payments/urls.py:**
> ```python
> urlpatterns = [
>     path('payments/create-intent/', CreatePaymentIntentView.as_view(), name='payment-create-intent'),
>     path('payments/webhook/',       StripeWebhookView.as_view(),        name='stripe-webhook'),
> ]
> ```
> Include in config/urls.py.
>
> **Step 4 — Stock decrement on payment success:**
> In the webhook handler after marking the order paid, decrement stock for each item:
> ```python
> from apps.products.documents import Product
>
> for item in order.items:
>     Product.objects(__raw__={
>         '_id': item.product_id,
>         'variants': {'$elemMatch': {
>             'variant_id': item.variant_id,
>             'stock': {'$gte': item.quantity}
>         }}
>     }).update_one(__raw__={
>         '$inc': {'variants.$.stock': -item.quantity}
>     })
> ```
>
> **Step 5 — Explain how to get Stripe test keys:**
> - Go to https://dashboard.stripe.com → register a free account
> - In the dashboard, toggle 'Test mode' ON
> - Go to Developers → API keys
> - Copy the Publishable key (pk_test_...) and Secret key (sk_test_...)
> - For the webhook secret: install Stripe CLI, run:
>   stripe listen --forward-to localhost:8000/api/v1/payments/webhook/
>   The CLI will print the webhook signing secret (whsec_...)
>
> Show all complete file contents."

---

## DAY 3 — Order Creation Endpoint + Order Serializers

### Prompt:

> "Build the order creation endpoint and all order serializers.
>
> **Step 1 — Create apps/orders/serializers.py** with:
>
> ShippingAddressSerializer:
>   Fields: full_name, phone, street, city, country, postal_code
>   All required except postal_code
>
> OrderItemSerializer (read-only):
>   Fields: product_id (SerializerMethodField → str), product_name, product_slug,
>           variant_id, variant_sku, color, size, image_url, quantity, unit_price, subtotal
>
> OrderSerializer (read-only — for responses):
>   Fields: id (SerializerMethodField → str(obj.id)), order_number, status,
>           items (OrderItemSerializer many=True), shipping_address (ShippingAddressSerializer),
>           shipping_method, shipping_cost, subtotal, discount, tax, total,
>           coupon_code, tracking_number, notes, created_at, updated_at
>
> OrderListSerializer (lightweight — for list views):
>   Fields: id, order_number, status, total, item_count (SerializerMethodField → len(obj.items)),
>           created_at, first_item_image (SerializerMethodField → obj.items[0].image_url if items else None)
>
> CreateOrderSerializer (for validating POST /orders/ input):
>   Fields:
>     shipping_address  DictField, required   — validated against ShippingAddressSerializer
>     shipping_method   ChoiceField(['standard', 'express']), default='standard'
>     notes             CharField, allow_blank=True, default=''
>   Validate that shipping_address contains all required fields in validate()
>
> **Step 2 — Create apps/orders/views.py** with:
>
> OrderCreateView (APIView, IsAuthenticated):
>   POST /api/v1/orders/
>
>   Logic:
>   1. Validate request body with CreateOrderSerializer
>   2. Get the user's cart — return 400 if cart is empty
>   3. Validate stock for every cart item before creating the order:
>      For each item in cart.items, verify stock >= quantity using $elemMatch
>      If ANY item fails stock check: return 400 with detail of which item is unavailable
>      DO NOT decrement stock here — stock is decremented only when payment succeeds (webhook)
>   4. Build ShippingAddress EmbeddedDocument from validated data
>   5. Call Order.create_from_cart(cart, request.user, shipping_address, shipping_method, notes)
>   6. Clear the cart items (cart.items = [], cart.save()) — cart is now converted to an order
>   7. Return 201 with OrderSerializer(order).data
>
> OrderListView (APIView, IsAuthenticated):
>   GET /api/v1/orders/
>   Returns the current user's orders, newest first
>   Use OrderListSerializer for the response
>   Simple pagination: page and page_size query params, default page_size=10
>
> OrderDetailView (APIView, IsAuthenticated):
>   GET /api/v1/orders/{order_number}/
>   Verify the order belongs to request.user — return 404 if not found or wrong user
>   Use OrderSerializer for the full response
>
> **Step 3 — Create apps/orders/urls.py:**
> ```python
> urlpatterns = [
>     path('orders/',                  OrderCreateView.as_view(),  name='order-create'),
>     path('orders/',                  OrderListView.as_view(),    name='order-list'),
>     path('orders/<str:order_number>/', OrderDetailView.as_view(), name='order-detail'),
> ]
> ```
> Note: POST and GET on /orders/ use the same URL — dispatch by HTTP method.
> Combine into a single class-based view:
>   OrderListCreateView — override get() for list, post() for create.
>
> Include in config/urls.py.
>
> **Step 4 — Write manual test with curl:**
>
> First login to get cookies, then:
> ```bash
> # 1. Create an order (replace with real shipping data)
> curl -s -X POST http://localhost:8000/api/v1/orders/ \
>   -H 'Content-Type: application/json' \
>   --cookie 'access_token=YOUR_TOKEN' \
>   -d '{
>     \"shipping_address\": {
>       \"full_name\": \"Test User\",
>       \"phone\": \"+254712345678\",
>       \"street\": \"123 Main St\",
>       \"city\": \"Nairobi\",
>       \"country\": \"Kenya\"
>     },
>     \"shipping_method\": \"standard\",
>     \"notes\": \"\"
>   }' | python -m json.tool
>
> # 2. List orders
> curl -s http://localhost:8000/api/v1/orders/ \
>   --cookie 'access_token=YOUR_TOKEN' | python -m json.tool
>
> # 3. Get order detail
> curl -s http://localhost:8000/api/v1/orders/ORD-2024-00001/ \
>   --cookie 'access_token=YOUR_TOKEN' | python -m json.tool
> ```
>
> Show all complete file contents."

---

## DAY 4 — React: orderAPI + Checkout Form (Step 1 — Address + Shipping)

### Prompt:

> "Build the React data layer for orders and the first two steps of the checkout form.
>
> **Step 1 — Create src/api/orderAPI.js:**
> ```javascript
> orderAPI.createOrder(data)        POST /orders/
> orderAPI.listOrders(page)         GET /orders/?page=1
> orderAPI.getOrder(orderNumber)    GET /orders/{order_number}/
> orderAPI.createPaymentIntent(orderId)  POST /payments/create-intent/
> ```
> All use the shared axiosInstance (withCredentials: true).
> All return response.data directly.
>
> **Step 2 — Create src/hooks/useOrders.js** with TanStack Query hooks:
> ```javascript
> useOrders(page)         — queryKey: ['orders', page], queryFn: orderAPI.listOrders
> useOrder(orderNumber)   — queryKey: ['order', orderNumber], queryFn: orderAPI.getOrder
> ```
>
> **Step 3 — Create src/pages/Checkout.jsx** — a multi-step form with 3 steps:
>   Step 1: Shipping Address
>   Step 2: Shipping Method
>   Step 3: Payment (Day 5)
>
> Step indicator at the top — shows 3 numbered circles with labels,
> completed steps shown in violet, current step highlighted, future steps grayed out.
>
> Step 1 — Shipping Address:
>   Form fields using react-hook-form + Zod:
>   - full_name (required)
>   - phone (required)
>   - street (required)
>   - city (required)
>   - country (default 'Kenya', required)
>   - postal_code (optional)
>
>   Zod schema:
>   ```javascript
>   const addressSchema = z.object({
>     full_name:   z.string().min(2, 'Full name is required'),
>     phone:       z.string().min(9, 'Valid phone number required'),
>     street:      z.string().min(3, 'Street address is required'),
>     city:        z.string().min(2, 'City is required'),
>     country:     z.string().default('Kenya'),
>     postal_code: z.string().optional(),
>   })
>   ```
>   'Next: Shipping Method' button — validates the form and advances to Step 2.
>   Store the validated address data in component state (not Redux — it's ephemeral).
>
> Step 2 — Shipping Method:
>   Two option cards, each showing:
>   - Method name and description
>   - Estimated delivery time
>   - Price
>   ```
>   Standard Shipping  — 5-7 business days  — $5.00
>   Express Shipping   — 1-2 business days  — $15.00
>   ```
>   Selected card has violet border and check icon.
>   Order summary panel on the right showing cart items and running total
>   (subtotal + selected shipping cost).
>   'Back' button returns to Step 1.
>   'Next: Payment' button advances to Step 3 and calls orderAPI.createOrder():
>   ```javascript
>   const order = await orderAPI.createOrder({
>     shipping_address: addressData,
>     shipping_method: selectedMethod,
>     notes: '',
>   })
>   // Store order in component state — needed for Step 3
>   setCreatedOrder(order)
>   setStep(3)
>   ```
>   If createOrder fails: show error toast and stay on Step 2.
>
> **Step 4 — Add /checkout to the router:**
>   Protect with ProtectedRoute — only logged-in users can checkout.
>   Import Checkout in router.jsx and add:
>   ```jsx
>   { path: '/checkout', element: <Checkout /> }
>   ```
>   inside the ProtectedRoute children block.
>
> **Step 5 — Update CartSummary.jsx:**
>   Change the 'Proceed to Checkout' button's onCheckout handler
>   so it navigates to /checkout instead of showing a toast.
>   In CartSummary, import useNavigate and dispatch closeCartDrawer before navigating:
>   ```javascript
>   const navigate  = useNavigate()
>   const dispatch  = useDispatch()
>   const handleCheckout = () => {
>     dispatch(closeCartDrawer())
>     navigate('/checkout')
>   }
>   ```
>   Also update the Cart page's handleCheckout the same way.
>
> Show all complete file contents."

---

## DAY 5 — Stripe Elements Integration (Step 3 — Payment)

### Prompt:

> "Integrate Stripe Elements for the payment step of checkout.
>
> **Step 1 — Install Stripe React library:**
> ```bash
> npm install @stripe/react-stripe-js @stripe/stripe-js
> ```
>
> **Step 2 — Create src/lib/stripe.js:**
> ```javascript
> import { loadStripe } from '@stripe/stripe-js'
>
> // Publishable key is safe to expose in frontend code — it's not secret.
> // We load it from the Vite environment variable.
> const stripePromise = loadStripe(import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY)
>
> export default stripePromise
> ```
>
> Add to frontend/.env (create this file if it doesn't exist):
> ```
> VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...
> ```
> Note: Vite only exposes env variables prefixed with VITE_ to the browser.
>
> **Step 3 — Create src/components/checkout/PaymentForm.jsx:**
>
> This component is rendered inside an Elements provider and handles Step 3.
>
> Props: order (the created Order object from Step 2), onSuccess (callback)
>
> On mount:
>   Call orderAPI.createPaymentIntent(order.id) to get the client_secret.
>   Show a spinner while this is loading.
>   If it fails: show error and a 'Try Again' button.
>
> Once client_secret is available:
>   Render the Stripe CardElement inside a styled container.
>   Show order total clearly: 'Total to pay: $X.XX'
>
> On submit ('Pay Now' button):
>   ```javascript
>   const { error, paymentIntent } = await stripe.confirmCardPayment(clientSecret, {
>     payment_method: { card: cardElement }
>   })
>   ```
>   If error: show error.message in red below the card field
>   If paymentIntent.status === 'succeeded':
>     Show a success state (green checkmark, 'Payment successful!')
>     Call onSuccess(order.order_number) after 1.5 seconds
>
>   While payment is processing: disable the Pay Now button and show spinner.
>
> CardElement styling — match the dark theme:
> ```javascript
> const cardElementOptions = {
>   style: {
>     base: {
>       fontSize: '16px',
>       color: '#e2e8f0',
>       '::placeholder': { color: '#64748b' },
>       backgroundColor: 'transparent',
>     },
>     invalid: { color: '#f87171' },
>   },
>   hidePostalCode: true,
> }
> ```
>
> **Step 4 — Update src/pages/Checkout.jsx Step 3:**
> Wrap Step 3 content in Stripe's Elements provider:
> ```jsx
> import { Elements } from '@stripe/react-stripe-js'
> import stripePromise from '../lib/stripe'
> import PaymentForm from '../components/checkout/PaymentForm'
>
> // In the Step 3 render:
> {step === 3 && createdOrder && (
>   <Elements stripe={stripePromise}>
>     <PaymentForm
>       order={createdOrder}
>       onSuccess={(orderNumber) => navigate(`/order-confirmation/${orderNumber}`)}
>     />
>   </Elements>
> )}
> ```
>
> **Step 5 — Add /order-confirmation/:orderNumber to the router:**
> Create a placeholder page src/pages/OrderConfirmation.jsx (Day 6 will flesh it out):
> ```jsx
> import { useParams, Link } from 'react-router-dom'
>
> const OrderConfirmation = () => {
>   const { orderNumber } = useParams()
>   return (
>     <div className='min-h-screen bg-slate-950 flex flex-col items-center
>                     justify-center px-4 text-center'>
>       <div className='text-6xl mb-4'>✅</div>
>       <h1 className='text-3xl font-bold text-white mb-2'>Order Confirmed!</h1>
>       <p className='text-slate-400 mb-2'>Your order number is:</p>
>       <p className='text-violet-400 font-mono text-xl font-bold mb-6'>{orderNumber}</p>
>       <p className='text-slate-400 text-sm mb-8'>
>         A confirmation email will be sent to you shortly (Week 8).
>       </p>
>       <Link to='/'
>         className='px-6 py-3 bg-violet-600 hover:bg-violet-700 text-white
>                    font-semibold rounded-xl transition-colors'>
>         Continue Shopping
>       </Link>
>     </div>
>   )
> }
>
> export default OrderConfirmation
> ```
>
> Add to router.jsx (public route — no auth needed to view confirmation):
> ```jsx
> { path: '/order-confirmation/:orderNumber', element: <OrderConfirmation /> }
> ```
>
> **Step 6 — Test with Stripe test card:**
> Use card number: 4242 4242 4242 4242
> Any future expiry date (e.g. 12/26)
> Any 3-digit CVC (e.g. 123)
> Any postal code
>
> Show all complete file contents."

---

## DAY 6 — OrderConfirmation Page + Order History Page

### Prompt:

> "Build the full Order Confirmation page and the Order History page for users.
>
> **Step 1 — Build the full src/pages/OrderConfirmation.jsx:**
>
> Fetches the order using useOrder(orderNumber) from TanStack Query.
>
> Layout:
>   Success banner:
>     Large green checkmark icon (lucide-react CheckCircle2)
>     'Order Confirmed!' heading
>     Order number in a violet monospace badge
>     'We received your order and it is being processed.' subtext
>
>   Order summary card:
>     List of ordered items (image, name, variant, qty, subtotal)
>     Shipping address block
>     Shipping method + cost
>     Subtotal, discount (if any), total rows
>     Payment status badge (green 'Paid' if status is paid)
>
>   Next steps section:
>     'What happens next?' block with 3 steps:
>     1. Order Confirmed → 2. Processing → 3. Shipped
>     Each step with an icon and short description
>
>   Action buttons:
>     'View Order History' → /orders
>     'Continue Shopping' → /products
>
>   Loading state: skeleton cards while order is fetching
>   Error state: 'Order not found' with a link to /orders
>
> **Step 2 — Create src/pages/user/Orders.jsx** — the user's order history page.
>
> Fetches orders with useOrders(page).
> Layout:
>   Page title: 'My Orders'
>   Filter tabs: All | Pending | Paid | Processing | Shipped | Delivered
>   (clicking a tab filters the displayed orders — filter client-side on the fetched results)
>
>   Order card for each order:
>     Order number (monospace, violet) + created date
>     Status badge — color coded:
>       pending     → yellow
>       paid        → blue
>       processing  → indigo
>       shipped     → violet
>       delivered   → green
>       cancelled   → red
>       refunded    → orange
>     First item image + product name + 'and X more items' if multiple
>     Total amount
>     'View Details' button → /order-confirmation/{order_number}
>
>   Empty state: 'No orders yet' with a 'Start Shopping' button
>   Loading state: skeleton cards
>   Pagination at the bottom if more than 10 orders
>
> **Step 3 — Add /orders to the router:**
> Inside the ProtectedRoute children:
> ```jsx
> import Orders from './pages/user/Orders'
> { path: '/orders', element: <Orders /> }
> ```
>
> **Step 4 — Update the Navbar:**
> If the user is authenticated, add a dropdown or simple links:
>   'My Orders' → /orders
>   'Profile' → /profile (Week 10)
>   'Logout' button
>
> Currently the Navbar shows just the user's first name + Logout button.
> Wrap the user's name in a simple dropdown:
> ```jsx
> // Show user menu when authenticated
> {isAuthenticated && (
>   <div className='relative group'>
>     <button className='flex items-center gap-1.5 text-sm text-slate-300
>                        hover:text-white transition-colors'>
>       {user?.first_name}
>       <ChevronDown size={14} />
>     </button>
>     <div className='absolute right-0 top-full mt-1 w-40 bg-slate-800
>                     border border-slate-700 rounded-xl shadow-xl
>                     opacity-0 invisible group-hover:opacity-100
>                     group-hover:visible transition-all z-50'>
>       <Link to='/orders'
>         className='block px-4 py-2.5 text-sm text-slate-300
>                    hover:text-white hover:bg-slate-700 rounded-t-xl'>
>         My Orders
>       </Link>
>       <button onClick={() => dispatch(logoutUser())}
>         className='w-full text-left px-4 py-2.5 text-sm text-red-400
>                    hover:text-red-300 hover:bg-slate-700 rounded-b-xl'>
>         Logout
>       </button>
>     </div>
>   </div>
> )}
> ```
>
> Show all complete file contents."

---

## DAY 7 — Django Tests + Full Integration Verification + Week 4 Cleanup

### Prompt:

> "Write the Django test suite for orders and payments, then run the full
> Week 4 verification checklist.
>
> **Step 1 — Create apps/orders/tests.py** using DRF's APITestCase.
>
> setUp:
>   - Clean Order, Payment, Cart, Product, Category, User documents
>   - Create seller, customer users
>   - Create category and 2 products with in-stock variants
>   - Create a cart for the customer with 2 items (using Cart.get_or_create_for_user)
>   - Define valid shipping_address dict
>
> Order creation tests:
>   test_create_order_success
>     — POST /orders/ as authenticated user with valid data
>     — Expect 201, order_number starts with 'ORD-', status='pending'
>     — Cart items are cleared after order creation
>
>   test_create_order_requires_authentication
>     — POST /orders/ without auth → 401
>
>   test_create_order_empty_cart_returns_400
>     — Clear cart items, then POST /orders/ → 400
>
>   test_create_order_insufficient_stock_returns_400
>     — Set a variant stock to 0, then POST /orders/ → 400
>
>   test_order_items_are_snapshots
>     — Create order, then edit product name
>     — Fetch order — order item still has original name
>
>   test_order_number_is_unique
>     — Create two orders — they have different order numbers
>
>   test_shipping_cost_standard
>     — Create order with shipping_method='standard' — shipping_cost == 5.0
>
>   test_shipping_cost_express
>     — Create order with shipping_method='express' — shipping_cost == 15.0
>
>   test_order_total_calculation
>     — Verify total = subtotal + shipping_cost
>
> Order list/detail tests:
>   test_list_orders_returns_only_own_orders
>     — Create order for customer A and customer B
>     — Customer A only sees their own order
>
>   test_get_order_detail_by_order_number
>     — GET /orders/ORD-XXXX/ → full order with items
>
>   test_get_other_users_order_returns_404
>     — Try to fetch another user's order → 404
>
> **Step 2 — Run all tests:**
> ```bash
> python manage.py test apps.orders apps.cart apps.products apps.authentication \
>   --settings=config.settings.test -v 2
> ```
> All tests must pass.
>
> **Step 3 — Full end-to-end verification checklist:**
>
> Backend:
>   1. POST /api/v1/orders/ with items in cart → 201, order created
>   2. Cart is empty after order creation
>   3. GET /api/v1/orders/ → list of user's orders
>   4. GET /api/v1/orders/{order_number}/ → full order detail
>   5. POST /api/v1/payments/create-intent/ → returns client_secret
>   6. Stripe CLI receives the webhook and forwards to localhost
>   7. After test payment, order status changes to 'paid' in MongoDB
>   8. Product stock decremented after successful payment
>
> Frontend:
>   9.  Add items to cart, click 'Proceed to Checkout'
>   10. Redirected to /checkout (if logged in) or /login (if guest)
>   11. Step 1: Fill in shipping address, click Next
>   12. Step 2: Select shipping method, see order summary with correct total
>   13. Click 'Next: Payment', order is created, step 3 appears
>   14. Stripe CardElement renders correctly in the dark theme
>   15. Enter test card 4242 4242 4242 4242, any expiry, any CVC
>   16. Click 'Pay Now' — spinner shows, payment processes
>   17. Redirected to /order-confirmation/{order_number}
>   18. Order confirmation page shows items, shipping address, total, 'Paid' badge
>   19. Navigate to /orders — order appears in history with correct status
>   20. Status badge is colored correctly
>   21. Click 'View Details' — navigates back to order confirmation
>   22. Navbar user dropdown shows 'My Orders' link
>   23. Try checkout without being logged in — redirected to /login
>   24. After login, redirected back to /checkout
>
> **Step 4 — Provide the Week 4 session handoff summary:**
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

## WEEK 4 SUMMARY — What You Will Have Built

By the end of Day 7, your project will have:

**Backend:**
- `Order` Document with embedded `OrderItem`, `ShippingAddress`, `StatusHistory`
- `Order.create_from_cart()` — builds order from cart with full item snapshots
- `Order.generate_order_number()` — ORD-YEAR-XXXXX format, always unique
- `Order.add_status()` — appends to status_history and updates status field
- `Payment` Document with Stripe fields: payment_intent_id, event_id (for idempotency)
- Stripe configured via django-environ, stripe.api_key set at startup
- `POST /api/v1/orders/` — creates order from cart, validates stock, clears cart
- `GET /api/v1/orders/` — user's own orders list with pagination
- `GET /api/v1/orders/{order_number}/` — full order detail (owner only)
- `POST /api/v1/payments/create-intent/` — creates Stripe PaymentIntent, returns client_secret
- `POST /api/v1/payments/webhook/` — CSRF exempt, signature verified, idempotent
- Webhook handles: payment_intent.succeeded → mark paid, decrement stock
- Webhook handles: payment_intent.payment_failed → mark failed
- Django APITestCase suite covering all order creation + list/detail scenarios

**Frontend:**
- `orderAPI.js` — createOrder, listOrders, getOrder, createPaymentIntent
- `useOrders`, `useOrder` TanStack Query hooks
- `Checkout.jsx` — 3-step form: Address → Shipping → Payment
  - Step indicator component
  - Step 1: react-hook-form + Zod address validation
  - Step 2: shipping method selection cards with live order summary
  - Step 3: Stripe Elements CardElement
- `src/lib/stripe.js` — loadStripe singleton with publishable key from VITE_ env var
- `PaymentForm.jsx` — fetches client_secret, confirms card payment, handles errors
- `OrderConfirmation.jsx` — full page: success banner, item list, address, next steps
- `Orders.jsx` — order history with status filter tabs and colored status badges
- Navbar user dropdown with My Orders link
- CartSummary + Cart page checkout button navigate to /checkout
- /checkout protected by ProtectedRoute
- /order-confirmation/:orderNumber public route
- /orders protected route

**Week 5 continues with:** Reviews system (verified purchase only, updates avg_rating),
Wishlist toggle endpoint and page, and User Profile page (edit info, manage addresses).
