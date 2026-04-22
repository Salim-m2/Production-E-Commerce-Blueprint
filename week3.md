# Week 3 AI Implementation Prompt — Cart System (Guest + Authenticated + Merge)
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## CONTEXT TO GIVE THE AI AT THE START OF EVERY SESSION THIS WEEK

> "I am continuing a production e-commerce build. Weeks 1 and 2 are fully complete and tested.
>
> **What already exists and is working:**
> - Django project with split settings (base.py / dev.py / prod.py), loaded via django-environ
> - MongoDB Atlas connected via mongoengine
> - User Document with set_password() and check_password() methods
> - Full JWT auth in httpOnly cookies: register, login, logout, token refresh, /auth/me/
> - Password reset skeleton (Token Document with TTL, endpoints without emails yet)
> - DRF throttling on all endpoints
> - Django APITestCase tests for all auth and product endpoints (49 tests passing)
> - Product, Category, Variant (embedded) mongoengine Documents
> - Product list API: pagination, filtering by category/price/rating/search, 5 sort options
> - Product detail API: full document with all variants
> - Category tree API: nested structure, cached 1 hour
> - Cloudinary integration: image upload endpoint with IsAdminOrSeller permission
> - MongoDB indexes: slug (unique), category+active compound, price, rating, text search
> - Management command: seed_products (38 products, 8 categories)
> - React 18 + Vite frontend with Tailwind CSS
> - Redux Toolkit store: authSlice, uiSlice (cartDrawerOpen), cartSlice (stub)
> - TanStack Query wired up for all server data
> - Axios instance with silent JWT refresh interceptor (withCredentials: true)
> - productAPI.js, useProducts/useProduct/useCategories hooks
> - formatters.js: formatPrice, formatRating, buildCloudinaryUrl, truncateText
> - StarRating, ProductCard, ProductGrid components
> - FilterSidebar, SortDropdown, SearchBar, Pagination components
> - Home page, ProductList page (all filters in URL params), ProductDetail page
> - React Router v6: /, /products, /products/:slug, /login, /register all wired
> - Navbar with search form
>
> **Stack reminder:**
> - Backend: Django 4.x + DRF + mongoengine (NO Django ORM, NO SQL)
> - Frontend: React 18 + Vite + Redux Toolkit + TanStack Query + Axios
> - All API endpoints prefixed /api/v1/
> - JWT in httpOnly cookies ONLY — never localStorage
> - Styling: Tailwind CSS
>
> **Critical architecture rules for the cart:**
> - Guest carts use a session_key stored in a browser cookie (UUID generated client-side and sent as a header or cookie)
> - Logged-in carts use the user's ObjectId
> - On login, the guest cart must be merged into the user's cart
> - price_at_add must be stored in every cart item at the moment of addition — never recalculate from the current product price
> - Stock validation must happen atomically using MongoDB's findOneAndUpdate pattern
>
> Today we are working on: [paste the specific day below]"

---

## DAY 1 — Cart Document (mongoengine) + App Scaffold + MongoDB Indexes

### Prompt:

> "Create the cart app and its mongoengine Documents. Do the following steps in order:
>
> **Step 1 — Scaffold the app:**
> Create `backend/apps/cart/` with these files:
> ```
> apps/cart/
> ├── __init__.py
> ├── documents.py
> ├── serializers.py    ← empty for now, fill tomorrow
> ├── views.py          ← empty for now, fill tomorrow
> ├── urls.py           ← empty for now, fill tomorrow
> └── tests.py          ← empty for now, fill Day 3
> ```
> Register `apps.cart` in `INSTALLED_APPS` in `config/settings/base.py`.
>
> **Step 2 — Create `apps/cart/documents.py`** with these mongoengine Documents:
>
> `CartItem` (EmbeddedDocument) — embedded inside Cart, NOT a separate collection.
> Why embedded? Cart items are always read with their cart, never independently.
> Fields:
> ```
> product_id    ObjectIdField, required    — ref to Product._id
> variant_id    StringField, required      — the variant_id string (UUID) from Product.variants
> product_name  StringField, required      — SNAPSHOT of name at add time (never changes even if product is edited)
> variant_sku   StringField, required      — SNAPSHOT of sku
> color         StringField               — SNAPSHOT of variant color (optional)
> size          StringField               — SNAPSHOT of variant size (optional)
> image_url     StringField               — SNAPSHOT of first product image (for cart display)
> price_at_add  FloatField, required      — SNAPSHOT of price when item was added
>                                            (NEVER recalculate from current product price)
> quantity      IntField, required, min_value=1
> ```
>
> `Cart` (Document) — the main cart collection.
> Fields:
> ```
> user_id      ObjectIdField, null=True   — null for guest carts
> session_key  StringField, null=True     — UUID string, set for guest carts, null for user carts
> items        ListField of EmbeddedDocumentField(CartItem)
> coupon_code  StringField, null=True     — applied coupon code
> updated_at   DateTimeField, default=datetime.utcnow
> created_at   DateTimeField, default=datetime.utcnow
> ```
> Add a save() override that stamps updated_at = datetime.utcnow() on every save.
>
> Add these classmethods to Cart:
>
> `get_for_user(user_id)`:
>   - Returns the Cart for this user_id, or None if not found
>   - Use Cart.objects(user_id=user_id).first()
>
> `get_for_session(session_key)`:
>   - Returns the Cart for this session_key, or None if not found
>   - Use Cart.objects(session_key=session_key, user_id=None).first()
>
> `get_or_create_for_user(user_id)`:
>   - Returns (cart, created) — gets existing or creates a new empty cart
>
> `get_or_create_for_session(session_key)`:
>   - Returns (cart, created) — gets existing or creates a new empty cart
>
> Add these instance methods to Cart:
>
> `get_subtotal()`:
>   - Returns sum of item.price_at_add * item.quantity for all items
>   - Returns 0.0 if cart is empty
>
> `get_item_count()`:
>   - Returns total number of individual items (sum of quantities)
>
> `find_item(product_id, variant_id)`:
>   - Returns the CartItem where item.product_id matches AND item.variant_id matches
>   - Returns None if not found
>   - Note: product_id is an ObjectId, compare with str() conversion if needed
>
> **Step 3 — Create `apps/cart/indexes.py`** with a function `create_cart_indexes()`:
> Create these indexes on the `carts` collection:
> - user_id (sparse=True — many carts have null user_id, sparse skips those)
> - session_key (sparse=True — null for logged-in carts)
> - updated_at descending — for cleanup of old guest carts
>
> Call `create_cart_indexes()` from `config/settings/base.py` after the product indexes call.
>
> **Step 4 — Wire the cart URLs placeholder in `config/urls.py`:**
> Add `path('', include('apps.cart.urls')),` inside the /api/v1/ block.
> The urls.py is empty for now — this prevents import errors.
>
> **Step 5 — Verify setup:**
> ```bash
> python manage.py check --settings=config.settings.dev
> ```
> Expected: System check identified no issues (0 silenced).
>
> Open a Django shell and confirm the Document imports work:
> ```python
> python manage.py shell --settings=config.settings.dev
> from apps.cart.documents import Cart, CartItem
> print('Cart Document OK:', Cart._meta)
> ```
>
> Show all complete file contents."

---

## DAY 2 — Cart API Endpoints (GET, POST, PATCH, DELETE)

### Prompt:

> "Build all cart API views and serializers. The cart must work for BOTH guest users
> (identified by a session_key cookie/header) and logged-in users (identified by JWT cookie).
>
> **How guest cart identification works:**
> The frontend generates a UUID v4 on first visit and stores it in localStorage as
> 'guest_session_key'. It sends this value in every API request as the header:
>   X-Session-Key: <uuid>
> The backend reads request.META.get('HTTP_X_SESSION_KEY') to get it.
> For logged-in users, the session key is ignored — we use request.user instead.
>
> **Step 1 — Create a cart resolution helper in `apps/cart/views.py`:**
>
> Write a function `get_cart_for_request(request)` that:
> - If request.user is authenticated: return Cart.get_or_create_for_user(request.user.id)
> - Otherwise: read the session key from request.META.get('HTTP_X_SESSION_KEY')
>   - If session key exists: return Cart.get_or_create_for_session(session_key)
>   - If no session key: return (None, False) — can't create a cart without identification
> Returns a (cart, created) tuple.
>
> **Step 2 — Create `apps/cart/serializers.py`** with:
>
> `CartItemSerializer`:
> Fields: product_id (SerializerMethodField → str), variant_id, product_name,
>         variant_sku, color, size, image_url, price_at_add, quantity, line_total
> `line_total` is a SerializerMethodField returning item.price_at_add * item.quantity
> `product_id` SerializerMethodField returns str(obj.product_id)
>
> `CartSerializer`:
> Fields: id (SerializerMethodField → str(obj.id)), items (CartItemSerializer many=True),
>         item_count (SerializerMethodField → cart.get_item_count()),
>         subtotal (SerializerMethodField → cart.get_subtotal()),
>         coupon_code, updated_at
>
> `AddToCartSerializer` (for validating POST /cart/items/ input):
> Fields:
>   product_id   CharField, required   — will be cast to ObjectId in the view
>   variant_id   CharField, required   — the variant_id UUID string
>   quantity     IntegerField, default=1, min_value=1, max_value=100
>
> `UpdateCartItemSerializer` (for validating PATCH /cart/items/{item_id}/):
> Fields:
>   quantity     IntegerField, required, min_value=1, max_value=100
>
> **Step 3 — Build `apps/cart/views.py`** with these views:
>
> `CartDetailView` (APIView, permission: AllowAny):
>   GET /api/v1/cart/
>   - Resolve cart using get_cart_for_request()
>   - If no cart (no session key and not authenticated): return empty cart response:
>     { id: null, items: [], item_count: 0, subtotal: 0.0, coupon_code: null }
>   - Return serialized cart with CartSerializer
>
> `CartItemAddView` (APIView, permission: AllowAny):
>   POST /api/v1/cart/items/
>   - Resolve or create cart using get_cart_for_request()
>   - If no session key and not authenticated: return 400 'X-Session-Key header required for guest cart'
>   - Validate request body with AddToCartSerializer
>   - Look up the Product by product_id — return 404 if not found or inactive
>   - Find the specific variant in product.variants by variant_id — return 404 if not found
>   - STOCK VALIDATION (atomic):
>     Use the atomic pattern to check stock BEFORE modifying the cart:
>     ```python
>     from apps.products.documents import Product
>     from bson import ObjectId
>     # Check stock atomically — never read stock and then write separately
>     product_with_stock = Product.objects(
>         id=product_id,
>         variants__variant_id=variant_id,
>         variants__stock__gte=quantity
>     ).first()
>     if not product_with_stock:
>         return Response({'detail': 'Insufficient stock.'}, status=400)
>     ```
>   - Find the variant data from the product document for snapshotting
>   - Check if this product+variant already exists in the cart:
>     - If yes: update the quantity (add to existing), re-validate total quantity against stock
>     - If no: append a new CartItem with ALL snapshot fields set:
>       product_name, variant_sku, color, size, image_url, price_at_add (from variant.price)
>   - Save cart and return CartSerializer(cart).data with status 200
>
> `CartItemUpdateView` (APIView, permission: AllowAny):
>   PATCH /api/v1/cart/items/{item_index}/
>   - item_index is the 0-based position of the item in cart.items list
>   - Resolve cart — return 404 if cart not found
>   - Validate with UpdateCartItemSerializer
>   - Validate the new quantity against current stock atomically (same pattern as above)
>   - Update cart.items[item_index].quantity
>   - Save and return CartSerializer
>
> `CartItemDeleteView` (APIView, permission: AllowAny):
>   DELETE /api/v1/cart/items/{item_index}/
>   - item_index is the 0-based position of the item in cart.items list
>   - Resolve cart — return 404 if not found
>   - Remove item at that index: cart.items.pop(item_index)
>   - Save and return CartSerializer with 200
>
> **Step 4 — Create `apps/cart/urls.py`:**
> ```python
> urlpatterns = [
>     path('cart/',                       CartDetailView.as_view(),     name='cart-detail'),
>     path('cart/items/',                 CartItemAddView.as_view(),    name='cart-item-add'),
>     path('cart/items/<int:item_index>/', CartItemUpdateView.as_view(), name='cart-item-update'),
>     path('cart/items/<int:item_index>/', CartItemDeleteView.as_view(), name='cart-item-delete'),
> ]
> ```
> Note: PATCH and DELETE share the same URL pattern — DRF dispatches by HTTP method.
> Use a single class-based view for the item detail and override both patch() and delete().
> Rename CartItemUpdateView to CartItemDetailView and add both methods to it.
>
> **Step 5 — Manual verification with curl:**
>
> Start the server and run these tests:
>
> ```bash
> # 1. Get empty cart as guest (no session key) — should return empty cart shape
> curl http://localhost:8000/api/v1/cart/
>
> # 2. Get cart with session key — should return empty cart (newly created)
> curl http://localhost:8000/api/v1/cart/ -H 'X-Session-Key: test-session-123'
>
> # 3. Add an item as guest (use a real slug from your seeded products)
> curl -X POST http://localhost:8000/api/v1/cart/items/ \
>   -H 'Content-Type: application/json' \
>   -H 'X-Session-Key: test-session-123' \
>   -d '{\"product_id\": \"<real-product-id>\", \"variant_id\": \"<real-variant-id>\", \"quantity\": 1}'
>
> # 4. Get cart again — should show the item with all snapshot fields
> curl http://localhost:8000/api/v1/cart/ -H 'X-Session-Key: test-session-123'
>
> # 5. Try adding more than available stock — should return 400
> curl -X POST http://localhost:8000/api/v1/cart/items/ \
>   -H 'Content-Type: application/json' \
>   -H 'X-Session-Key: test-session-123' \
>   -d '{\"product_id\": \"<id>\", \"variant_id\": \"<id>\", \"quantity\": 9999}'
> ```
>
> Show all complete file contents."

---

## DAY 3 — Cart Merge on Login + Coupon Endpoint + Django Tests

### Prompt:

> "Add the cart merge endpoint (called on login), a coupon validation endpoint,
> and write the full Django test suite for the cart.
>
> **Step 1 — Cart merge endpoint:**
>
> `CartMergeView` (APIView, permission: IsAuthenticated):
>   POST /api/v1/cart/merge/
>   Called by the frontend immediately after a successful login.
>   Request body: { session_key: 'uuid-of-guest-cart' }
>
>   Logic:
>   - Find the guest cart by session_key (Cart.objects(session_key=..., user_id=None).first())
>   - If no guest cart exists: return 200 with the user's current cart (nothing to merge)
>   - Get or create the user's cart
>   - For each item in the guest cart:
>     - Check if the same product+variant already exists in the user's cart
>     - If yes: add quantities together (validate combined quantity against stock)
>     - If no: append the guest cart item directly to user cart items
>   - Save the user's cart
>   - Delete the guest cart (it has been absorbed)
>   - Return 200 with the merged CartSerializer(user_cart).data
>
> Add this endpoint to `apps/cart/urls.py`:
>   path('cart/merge/', CartMergeView.as_view(), name='cart-merge'),
>
> **Step 2 — Coupon validation endpoint (skeleton):**
> Full coupon logic is built in Week 10. For now, create the endpoint so the
> frontend can wire up the input field without errors.
>
> `CartCouponView` (APIView, permission: AllowAny):
>   POST /api/v1/cart/coupon/
>   Request body: { coupon_code: 'SAVE10', session_key: 'uuid' }
>
>   For now, always return:
>   { detail: 'Coupon support coming in Week 10.' } with status 200
>   (We will replace this with real logic in Week 10)
>
> Add to urls.py: path('cart/coupon/', CartCouponView.as_view(), name='cart-coupon'),
>
> **Step 3 — Write `apps/cart/tests.py`** using DRF's APITestCase.
>
> setUp:
>   - Clean all Cart and Product documents before each test
>   - Create a seller user
>   - Create a category
>   - Create 3 test products, each with 2 variants:
>     - product_a: variant_a1 (stock=10, price=50.00), variant_a2 (stock=0, price=60.00)
>     - product_b: variant_b1 (stock=5, price=100.00), variant_b2 (stock=3, price=120.00)
>     - product_c: variant_c1 (stock=1, price=25.00)
>   - Create a customer User for authenticated cart tests
>   - SESSION_KEY = 'test-session-abc123'
>
> tearDown:
>   - Delete all Cart, Product, Category, User documents created in setUp
>
> Guest cart tests:
>   test_get_empty_cart_no_session_key — GET /cart/ with no header, expect empty cart shape
>   test_get_cart_with_session_key — GET /cart/ with X-Session-Key, expect 200 empty cart
>   test_add_item_as_guest — POST /cart/items/ with session key and valid product, expect 200, item in cart
>   test_add_item_snapshot_fields — verify price_at_add, product_name, variant_sku all saved correctly
>   test_add_same_item_twice_merges_quantity — add same variant twice, quantity should combine
>   test_add_item_exceeding_stock_returns_400 — quantity > stock, expect 400
>   test_add_out_of_stock_variant_returns_400 — variant with stock=0, expect 400
>   test_update_item_quantity — PATCH /cart/items/0/ with new quantity, verify change
>   test_update_item_quantity_exceeds_stock_returns_400
>   test_delete_item — DELETE /cart/items/0/, item removed from cart
>   test_delete_invalid_index_returns_404 — item_index out of range
>   test_cart_subtotal_calculation — add 2 items, verify subtotal = sum of price_at_add * quantity
>   test_item_count — add 3 items with quantity 2 each, item_count should be 6
>
> Authenticated cart tests:
>   test_get_cart_authenticated — authenticated user gets their own cart
>   test_add_item_authenticated — POST with JWT cookie, no session key needed
>   test_guest_and_user_carts_are_separate — same product added to both, carts stay separate
>
> Cart merge tests:
>   test_merge_guest_cart_into_user_cart — add items to guest cart, login, POST /cart/merge/, verify items in user cart
>   test_merge_combines_duplicate_items — same product in both carts, quantities combine after merge
>   test_merge_with_no_guest_cart — POST /cart/merge/ with invalid session_key, returns user cart unchanged
>   test_merge_deletes_guest_cart — after merge, guest cart no longer exists in DB
>   test_merge_requires_authentication — unauthenticated POST /cart/merge/ returns 401
>
> Run with:
> ```bash
> python manage.py test apps.cart --settings=config.settings.test -v 2
> ```
> All tests must pass before proceeding to Day 4.
>
> Show all complete file contents."

---

## DAY 4 — React: cartAPI + cartSlice + useCart Hook

### Prompt:

> "Build the React data layer for the cart. This covers the API calls,
> Redux state, and the custom hook.
>
> **Step 1 — Create `frontend/src/api/cartAPI.js`:**
>
> All functions use the shared axiosInstance (withCredentials: true).
> The session key is read from localStorage and sent as the X-Session-Key header.
>
> Write a helper at the top of the file:
> ```javascript
> const getSessionKey = () => localStorage.getItem('guest_session_key');
> const sessionHeader = () => {
>   const key = getSessionKey();
>   return key ? { 'X-Session-Key': key } : {};
> };
> ```
>
> Functions to implement:
>
> `cartAPI.getCart()`:
>   GET /cart/
>   Headers: sessionHeader()
>   Returns response.data
>
> `cartAPI.addItem({ product_id, variant_id, quantity })`:
>   POST /cart/items/
>   Headers: sessionHeader()
>   Body: { product_id, variant_id, quantity }
>   Returns response.data
>
> `cartAPI.updateItem(itemIndex, quantity)`:
>   PATCH /cart/items/{itemIndex}/
>   Headers: sessionHeader()
>   Body: { quantity }
>   Returns response.data
>
> `cartAPI.removeItem(itemIndex)`:
>   DELETE /cart/items/{itemIndex}/
>   Headers: sessionHeader()
>   Returns response.data
>
> `cartAPI.mergeCart(sessionKey)`:
>   POST /cart/merge/
>   No session header needed (user is logged in)
>   Body: { session_key: sessionKey }
>   Returns response.data
>
> `cartAPI.applyCoupon(couponCode)`:
>   POST /cart/coupon/
>   Headers: sessionHeader()
>   Body: { coupon_code: couponCode }
>   Returns response.data
>
> Also export a utility function:
> `ensureSessionKey()`:
>   - Reads 'guest_session_key' from localStorage
>   - If it doesn't exist, generates a new UUID v4 and saves it to localStorage
>   - Returns the key
>   Use this pattern to generate UUID:
>   ```javascript
>   const uuid = () => crypto.randomUUID();
>   ```
>
> **Step 2 — Replace the stub `frontend/src/store/cartSlice.js`** with a full implementation:
>
> State shape:
> ```javascript
> {
>   cart: null,          // the full cart object from the API (or null if not loaded)
>   isLoading: false,
>   isOpen: false,       // controls the cart drawer (sidebar)
>   error: null,
> }
> ```
>
> Async thunks (use createAsyncThunk):
>
> `fetchCart`:
>   - Calls cartAPI.getCart()
>   - Dispatched on app startup and after any mutation
>
> `addToCart({ product_id, variant_id, quantity })`:
>   - Calls ensureSessionKey() first (creates key if not exists)
>   - Calls cartAPI.addItem(...)
>   - On fulfilled: update state.cart with the response
>   - On rejected: set state.error
>
> `updateCartItem({ itemIndex, quantity })`:
>   - Calls cartAPI.updateItem(itemIndex, quantity)
>   - On fulfilled: update state.cart
>
> `removeFromCart(itemIndex)`:
>   - Calls cartAPI.removeItem(itemIndex)
>   - On fulfilled: update state.cart
>
> `mergeCart(sessionKey)`:
>   - Calls cartAPI.mergeCart(sessionKey)
>   - On fulfilled: update state.cart, then clear localStorage 'guest_session_key'
>
> Synchronous actions (reducers):
>   openCartDrawer()
>   closeCartDrawer()
>   toggleCartDrawer()
>   clearCart()  — sets state.cart = null (called on logout)
>
> Selectors to export:
>   selectCart          — state.cart.cart (the cart object)
>   selectCartItems     — state.cart.cart?.items ?? []
>   selectCartItemCount — state.cart.cart?.item_count ?? 0
>   selectCartSubtotal  — state.cart.cart?.subtotal ?? 0
>   selectCartIsOpen    — state.cart.isOpen
>   selectCartIsLoading — state.cart.isLoading
>
> **Step 3 — Create `frontend/src/hooks/useCart.js`:**
>
> Custom hook that wraps Redux cart state and actions for use in components.
> Returns:
> ```javascript
> {
>   cart,           // full cart object
>   items,          // cart.items array
>   itemCount,      // total item count (for navbar badge)
>   subtotal,       // cart subtotal float
>   isLoading,
>   isOpen,         // drawer state
>   addToCart,      // async function(product_id, variant_id, quantity)
>   removeFromCart, // async function(itemIndex)
>   updateItem,     // async function(itemIndex, quantity)
>   openDrawer,
>   closeDrawer,
>   toggleDrawer,
> }
> ```
>
> `addToCart` should:
>   1. Call ensureSessionKey() so the guest key exists before the API call
>   2. Dispatch addToCart thunk
>   3. Dispatch openCartDrawer() on success (opens the drawer to show what was added)
>   4. Return { success: true } or { success: false, error: message }
>
> **Step 4 — Wire cart initialization in `frontend/src/main.jsx`:**
> After the `initializeAuth` dispatch, also dispatch `fetchCart` so the cart
> badge count in the Navbar is correct on page load:
> ```javascript
> store.dispatch(initializeAuth());
> store.dispatch(fetchCart());
> ```
>
> **Step 5 — Update `frontend/src/store/index.js`:**
> Make sure cartSlice is included in the root reducer.
> The cart reducer key should be 'cart'.
>
> **Step 6 — Update the Navbar cart icon** to show the item count badge:
> Import selectCartItemCount from cartSlice.
> Show a red badge with the count when itemCount > 0:
> ```jsx
> <div className='relative'>
>   <ShoppingCart size={20} />
>   {itemCount > 0 && (
>     <span className='absolute -top-1.5 -right-1.5 bg-red-500 text-white text-xs
>                      w-4 h-4 rounded-full flex items-center justify-center font-bold'>
>       {itemCount > 9 ? '9+' : itemCount}
>     </span>
>   )}
> </div>
> ```
>
> Show all complete file contents."

---

## DAY 5 — React: CartItem + CartSummary + Cart Drawer

### Prompt:

> "Build the CartItem component, CartSummary component, and the cart drawer
> that slides in from the right side of the screen.
>
> **Step 1 — Create `frontend/src/components/cart/CartItem.jsx`:**
>
> Props: item (CartItem object from API), itemIndex (number), onRemove, onUpdateQuantity
>
> Display:
> - Product image (item.image_url, built with buildCloudinaryUrl(url, 80) for small thumbnail)
> - If no image: gray placeholder box with a package emoji
> - Product name (item.product_name) — bold, link to /products/{slug}
>   Note: the cart item doesn't have slug. Show name as plain text for now
>   (we'll add slug to the cart response in a later iteration if needed)
> - Variant details: show color and/or size if they exist (e.g. 'Black / Size 42')
> - Price per item: formatPrice(item.price_at_add)
> - Quantity control: minus button, quantity number, plus button
>   - Minus: calls onUpdateQuantity(itemIndex, item.quantity - 1)
>   - If quantity would go to 0: call onRemove(itemIndex) instead
>   - Plus: calls onUpdateQuantity(itemIndex, item.quantity + 1)
> - Line total: formatPrice(item.price_at_add * item.quantity) — right aligned, bold
> - Remove button (X icon, lucide-react Trash2) — calls onRemove(itemIndex)
>
> Loading state: accept an `isUpdating` prop. When true, show a subtle opacity
> reduction on the item to indicate it's being updated.
>
> **Step 2 — Create `frontend/src/components/cart/CartSummary.jsx`:**
>
> Props: subtotal (number), itemCount (number), onCheckout (function)
>
> Display:
> - 'Order Summary' heading
> - Subtotal row: 'Subtotal ({itemCount} item{s})' — right: formatPrice(subtotal)
> - Shipping row: 'Shipping' — right: 'Calculated at checkout'
> - Divider line
> - Total row (bold, larger): 'Total' — right: formatPrice(subtotal)
>   (no tax for now — displayed total equals subtotal until Week 6 checkout)
> - 'Proceed to Checkout' button — full width, indigo, calls onCheckout
>   - For now onCheckout shows a toast: 'Checkout coming in Week 6!'
> - 'Continue Shopping' link below the button — navigates to /products
>
> **Step 3 — Create `frontend/src/components/cart/CartDrawer.jsx`:**
>
> This is the slide-in cart sidebar shown when items are added or the cart icon is clicked.
> It reads from the Redux store via useCart().
>
> Structure:
> - A semi-transparent overlay covering the whole screen when open
>   Clicking the overlay calls closeDrawer()
> - A panel sliding in from the RIGHT side (w-full max-w-md)
>   Transform: translateX(100%) when closed, translateX(0) when open
>   Transition: duration-300 ease-in-out
>
> Panel contents:
>
> Header:
>   - 'Your Cart' title with item count badge
>   - X button (X icon from lucide-react) that calls closeDrawer()
>
> Body (scrollable):
>   If cart is empty:
>     - Empty state: shopping bag icon, 'Your cart is empty', link to /products
>   If cart has items:
>     - List of CartItem components
>     - Each CartItem receives the item, its index, onRemove, onUpdateQuantity handlers
>     - Handlers dispatch removeFromCart and updateCartItem thunks
>     - Show a subtle spinner or opacity change on the item being updated
>
> Footer (sticky at bottom of drawer):
>   - CartSummary component with subtotal and item count
>   - Only show if cart has items
>
> Loading state:
>   If isLoading is true and cart is null: show 3 skeleton CartItem placeholders
>
> **Step 4 — Add CartDrawer to `frontend/src/main.jsx` or `App.jsx`:**
> Import CartDrawer and render it once at the app root level, outside the router
> (so it persists across all page navigations):
> ```jsx
> <Provider store={store}>
>   <QueryClientProvider client={queryClient}>
>     <BrowserRouter>
>       <CartDrawer />   {/* renders once, controlled by Redux isOpen state */}
>       <AppRouter />
>     </BrowserRouter>
>   </QueryClientProvider>
> </Provider>
> ```
> Actually: keep CartDrawer inside AppRouter's Layout component so it's always mounted.
> Add it to the Layout in router.jsx:
> ```jsx
> const Layout = () => (
>   <div className='min-h-screen bg-gray-50'>
>     <Navbar />
>     <CartDrawer />
>     <Outlet />
>   </div>
> );
> ```
>
> **Step 5 — Update the Navbar cart icon** to dispatch toggleCartDrawer on click:
> Replace the Link wrapping the cart icon with a button:
> ```jsx
> <button
>   onClick={() => dispatch(toggleCartDrawer())}
>   className='relative p-1.5 text-gray-600 hover:text-indigo-600 transition-colors'
>   aria-label='Open cart'
> >
>   {/* cart icon + badge */}
> </button>
> ```
>
> **Step 6 — Create barrel export `frontend/src/components/cart/index.js`:**
> ```javascript
> export { default as CartItem    } from './CartItem';
> export { default as CartSummary } from './CartSummary';
> export { default as CartDrawer  } from './CartDrawer';
> ```
>
> Show all complete file contents."

---

## DAY 6 — Full Cart Page + Wire Add to Cart on ProductDetail

### Prompt:

> "Build the full /cart page and wire the 'Add to Cart' button on the
> ProductDetail page to the real cart system.
>
> **Step 1 — Create `frontend/src/pages/Cart.jsx`:**
>
> This is a standalone full-page cart (not the drawer).
> Accessed at /cart.
>
> Layout:
> - Page title: 'Shopping Cart'
> - Two-column layout on desktop (lg+): cart items left (2/3 width), order summary right (1/3 width)
> - Single column on mobile
>
> Left column — Cart items:
> - If cart is empty: large empty state (bag icon, message, 'Start Shopping' button to /products)
> - If loading and no cart: show 3 skeleton item rows
> - List of CartItem components (same component used in the drawer)
> - Each item has full remove and quantity update handlers
> - Show a 'Clear cart' button at the bottom of the items list (removes all items one by one
>   or you can add a clearCart action to the backend — for now loop through and remove each)
>
> Right column — Order summary:
> - CartSummary component
> - Below CartSummary: show a coupon input field
>   - Input + 'Apply' button
>   - On Apply: call cartAPI.applyCoupon(code) and show the response message
>   - For now the backend returns 'Coupon support coming in Week 10.'
>   - Show the message as a small info toast
>
> Back to shopping:
> - A '← Continue Shopping' link at the top left that navigates to /products
>
> **Step 2 — Update `frontend/src/router.jsx`:**
> Replace the protected /cart placeholder with the real Cart page:
> ```jsx
> import Cart from './pages/Cart';
> // ...
> <Route path='/cart' element={<Cart />} />
> ```
> Note: Cart should NOT require authentication — guests can view their cart too.
> Remove the ProtectedRoute wrapper from the /cart route.
>
> **Step 3 — Wire 'Add to Cart' on `frontend/src/pages/ProductDetail.jsx`:**
>
> Import useCart from hooks/useCart.
> Replace the toast placeholder in handleAddToCart with the real implementation:
>
> ```javascript
> const { addToCart } = useCart();
>
> const handleAddToCart = async () => {
>   if (!canAddToCart) return;
>
>   // addToCart also opens the drawer automatically on success
>   const result = await addToCart(product.id, matchedVariant.variant_id, quantity);
>
>   if (result.success) {
>     toast.success(`${product.name} added to cart!`);
>   } else {
>     toast.error(result.error || 'Failed to add to cart. Please try again.');
>   }
> };
> ```
>
> Also update the Add to Cart button to show a loading spinner while the cart
> is being updated. Import selectCartIsLoading from cartSlice and use it:
> ```jsx
> <button
>   onClick={handleAddToCart}
>   disabled={!canAddToCart || cartIsLoading}
>   className={`... ${cartIsLoading ? 'opacity-75 cursor-wait' : ''}`}
> >
>   {cartIsLoading ? (
>     <span className='animate-spin inline-block w-4 h-4 border-2 border-white border-t-transparent rounded-full' />
>   ) : (
>     <ShoppingCart size={16} />
>   )}
>   {/* button text */}
> </button>
> ```
>
> **Step 4 — Wire cart merge on login in `frontend/src/store/authSlice.js`:**
>
> In the `loginUser` fulfilled case, after updating the auth state, dispatch the
> mergeCart thunk with the current guest session key:
>
> In the loginUser thunk (after the API call succeeds):
> ```javascript
> import { mergeCart } from './cartSlice';
> import { getSessionKey } from '../api/cartAPI';
>
> export const loginUser = createAsyncThunk('auth/login', async (credentials, { dispatch }) => {
>   const response = await authAPI.login(credentials);
>   // Merge guest cart into user cart after login
>   const sessionKey = getSessionKey();  // exported utility from cartAPI.js
>   if (sessionKey) {
>     await dispatch(mergeCart(sessionKey));
>   }
>   return response.data;
> });
> ```
> Export `getSessionKey` from cartAPI.js so authSlice can import it.
>
> **Step 5 — Wire cart clear on logout in `frontend/src/store/authSlice.js`:**
> In the logoutUser fulfilled reducer, also dispatch clearCart:
> ```javascript
> // In extraReducers, logoutUser fulfilled case:
> .addCase(logoutUser.fulfilled, (state) => {
>   state.user = null;
>   state.isAuthenticated = false;
>   // Note: dispatch clearCart from the logout thunk, not here
> })
> ```
> In the logoutUser thunk:
> ```javascript
> export const logoutUser = createAsyncThunk('auth/logout', async (_, { dispatch }) => {
>   await authAPI.logout();
>   dispatch(clearCart());
>   // Also clear guest session key on logout
>   localStorage.removeItem('guest_session_key');
> });
> ```
>
> Show all complete file contents."

---

## DAY 7 — Integration Testing + Stock Edge Cases + Week 3 Cleanup

### Prompt:

> "This is the final day of Week 3. Run the full verification checklist, fix any
> issues, and confirm the complete cart system works end-to-end.
>
> **Step 1 — Add missing cart test cases to `apps/cart/tests.py`:**
>
> If not already covered, add these edge case tests:
>
> test_price_at_add_does_not_change_with_product_price:
>   - Add item to cart at price $50
>   - Update product variant price to $99 in DB
>   - Fetch cart again
>   - Verify cart item still shows price_at_add = $50 (snapshot preserved)
>
> test_add_item_with_invalid_product_id_returns_404:
>   - POST /cart/items/ with a random non-existent ObjectId
>   - Expect 404
>
> test_add_item_with_invalid_variant_id_returns_404:
>   - POST /cart/items/ with valid product_id but fake variant_id
>   - Expect 404
>
> test_item_index_out_of_range_returns_404:
>   - PATCH /cart/items/999/ when cart has 1 item
>   - Expect 404
>
> test_cart_merge_respects_stock_limit:
>   - Guest cart has product_a variant_a1 quantity=8 (stock=10)
>   - User cart has same product_a variant_a1 quantity=5
>   - Merge: combined would be 13, stock is 10
>   - After merge: quantity should be capped at 10 (available stock)
>   - No 400 error — merge should handle this gracefully by capping
>
> Run all tests:
> ```bash
> python manage.py test apps.cart apps.products apps.authentication --settings=config.settings.test -v 2
> ```
> All tests must pass.
>
> **Step 2 — Full end-to-end verification checklist:**
>
> Backend:
> 1. GET /api/v1/cart/ with no header → { id: null, items: [], item_count: 0, subtotal: 0.0 }
> 2. GET /api/v1/cart/ with X-Session-Key → empty cart created, 200
> 3. POST /api/v1/cart/items/ (guest) → item added with all snapshot fields
> 4. GET /api/v1/cart/ (guest) → item visible, subtotal correct
> 5. PATCH /api/v1/cart/items/0/ quantity=3 → quantity updated
> 6. PATCH /api/v1/cart/items/0/ quantity=9999 → 400 insufficient stock
> 7. DELETE /api/v1/cart/items/0/ → item removed, cart empty
> 8. POST /api/v1/cart/items/ quantity > stock → 400
> 9. POST /api/v1/cart/items/ with out-of-stock variant → 400
> 10. POST /api/v1/auth/login/ → logs in user
> 11. POST /api/v1/cart/merge/ with guest session_key → items merged
> 12. Guest cart deleted from DB after merge (check Atlas)
>
> Frontend:
> 13. localhost:5173 → Navbar shows cart icon (no badge if empty)
> 14. Navigate to a product detail page
> 15. Select a variant → 'Add to Cart' button enables
> 16. Click 'Add to Cart' → success toast, cart drawer slides in from right
> 17. Drawer shows the item with correct name, color/size, price, quantity
> 18. Navbar badge shows '1'
> 19. Click + in drawer → quantity increases
> 20. Click - until 0 → item removed
> 21. Click cart icon in Navbar → drawer toggles
> 22. Click overlay → drawer closes
> 23. Navigate to /cart → full cart page renders
> 24. Cart page shows items and order summary side by side (desktop)
> 25. Update quantity on cart page → total updates
> 26. Remove item on cart page → item disappears
> 27. Empty cart state shows correctly with 'Start Shopping' link
> 28. Add item as guest, log in → cart merges (guest items appear in user cart)
> 29. Log out → cart clears from UI
> 30. Navbar badge clears on logout
>
> **Step 3 — Provide the Week 3 session handoff summary in this exact format:**
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

## WEEK 3 SUMMARY — What You Will Have Built

By the end of Day 7, your project will have:

**Backend:**
- `Cart` Document with embedded `CartItem` EmbeddedDocument
- `CartItem` snapshots: product_name, variant_sku, color, size, image_url, price_at_add
  (these NEVER change even if the product is edited later)
- Atomic stock validation using MongoDB's query filter pattern (never read-then-write)
- Guest cart identification via `X-Session-Key` request header
- `Cart.get_for_user()`, `Cart.get_for_session()`, `get_or_create_*()` classmethods
- `get_subtotal()`, `get_item_count()`, `find_item()` instance methods
- MongoDB indexes: user_id (sparse), session_key (sparse), updated_at desc
- `GET /api/v1/cart/` — works for both guests and authenticated users
- `POST /api/v1/cart/items/` — add item with full snapshot fields
- `PATCH /api/v1/cart/items/{index}/` — update quantity with stock validation
- `DELETE /api/v1/cart/items/{index}/` — remove item
- `POST /api/v1/cart/merge/` — merge guest cart into user cart on login
- `POST /api/v1/cart/coupon/` — skeleton (real logic in Week 10)
- Full Django APITestCase suite covering guest carts, authenticated carts, merge,
  edge cases (price snapshot, stock limits, merge capping, invalid IDs)

**Frontend:**
- `cartAPI.js` — all cart API calls with automatic X-Session-Key header injection
- `ensureSessionKey()` — creates and persists a UUID v4 for guest identification
- `getSessionKey()` — exported for use by authSlice merge on login
- Full `cartSlice.js` — fetchCart, addToCart, updateCartItem, removeFromCart, mergeCart thunks
  plus openCartDrawer, closeCartDrawer, toggleCartDrawer, clearCart synchronous actions
- Selectors: selectCart, selectCartItems, selectCartItemCount, selectCartSubtotal,
  selectCartIsOpen, selectCartIsLoading
- `useCart.js` hook — wraps all cart state and actions, auto-opens drawer on addToCart
- `CartItem.jsx` — image, name, variant details, quantity control, line total, remove button
- `CartSummary.jsx` — subtotal, shipping note, total, checkout button (stub), continue shopping
- `CartDrawer.jsx` — slide-in panel from right, overlay, empty state, loading skeletons
- `Cart.jsx` (full page) — two-column layout, same CartItem components, coupon input
- ProductDetail → 'Add to Cart' wired to real cartAPI, loading spinner on button
- Cart merge dispatched automatically after loginUser thunk succeeds
- Cart cleared automatically after logoutUser thunk
- Navbar cart icon triggers drawer toggle
- Navbar badge shows live item count

**Week 4 continues with:** Checkout flow — multi-step form (address → shipping → payment),
Stripe Elements integration, PaymentIntent creation endpoint, Stripe webhook handler,
order creation on payment success, and cart clearing after successful checkout.
