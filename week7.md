# Week 7 AI Implementation Prompt — Admin Dashboard
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## CONTEXT TO GIVE THE AI AT THE START OF EVERY SESSION THIS WEEK

> "I am continuing a production e-commerce build. Weeks 1 through 6 are fully
> complete and tested.
>
> **What already exists and is working:**
> - Django 4.x + DRF + mongoengine (NO Django ORM, NO SQL)
> - MongoDB Atlas connected via mongoengine
> - MongoJWTAuthentication reads httpOnly access_token cookie, looks up
>   mongoengine User (NOT Django's SQL auth.User)
> - Full JWT auth: register, login, logout, refresh, /auth/me/,
>   /auth/verify-email/{token}/
> - Password reset with email (Brevo SMTP via Celery task)
> - DRF throttling on all endpoints
> - Product, Category, Variant (embedded) Documents with all indexes
> - Product list/detail/category tree APIs (pagination, filtering, search)
> - Cloudinary integration: image upload at POST /api/v1/products/upload-image/
>   protected by IsAdminOrSeller permission in apps/core/permissions.py
> - Cart (guest + authenticated, merge on login), full CRUD
> - Order Document with create_from_cart, generate_order_number, add_status
> - Payment flow (IntaSend/dev-confirm), _fulfill_order idempotent helper
> - PATCH /api/v1/orders/{order_number}/status/ — admin status update,
>   dispatches shipped/delivered email tasks
> - Review (verified purchase only), Wishlist (toggle pattern), User Profile
> - Celery + Redis: send_welcome_email, send_password_reset_email,
>   send_order_confirmation_email, send_order_shipped_email,
>   send_review_request_email, check_low_stock (periodic daily 8 AM)
> - Email templates (table-based HTML): all 7 types implemented
> - React 18 + Vite + Tailwind CSS + Redux Toolkit + TanStack Query + Axios
> - All pages: Home, ProductList, ProductDetail, Cart, Checkout,
>   OrderConfirmation, Orders, Wishlist, Profile
> - authSlice, cartSlice, wishlistSlice, uiSlice in Redux store
> - Navbar with user dropdown, wishlist badge, cart badge
> - ProtectedRoute, GuestRoute, React Router v6
>
> **Stack reminder:**
> - Backend: Django 4.x + DRF + mongoengine (NO Django ORM)
> - Frontend: React 18 + Vite + Redux Toolkit + TanStack Query + Axios
> - All API endpoints prefixed /api/v1/
> - JWT in httpOnly cookies ONLY
> - Payments: IntaSend (Kenyan gateway, hosted redirect)
> - Images: Cloudinary
> - Background tasks: Celery + Redis
> - Email: Brevo SMTP
> - Styling: Tailwind CSS
>
> **Critical rules for Week 7:**
> - All admin endpoints live under /api/v1/admin/ prefix and require
>   role == 'admin' — never expose admin actions to customers or sellers
> - Product deletion is SOFT DELETE only — set is_active=False, never remove
>   the document from MongoDB (orders reference product snapshots but the
>   product page should 404)
> - avg_rating and review_count must be preserved when soft-deleting a product
> - Analytics queries use mongoengine's __raw__ aggregation pipeline for
>   performance — never loop over all documents in Python
> - The admin dashboard routes (/admin/*) are protected by an AdminRoute
>   component that checks role == 'admin', redirecting customers to /
> - Recharts is already in the frontend dependencies — import directly
> - Never show real MongoDB ObjectIds in the admin UI — always show
>   human-readable identifiers (order_number, slug, email)
>
> Today we are working on: [paste the specific day below]"

---

## DAY 1 — Admin App Scaffold + Product CRUD + Category CRUD

### Prompt:

> "Create the admin dashboard backend app and build all product and category
> management endpoints. The app is called 'dashboard' to avoid clashing with
> Django's built-in 'admin' app name.
>
> **Step 1 — Scaffold the app:**
> ```
> backend/apps/dashboard/
> ├── __init__.py
> ├── views.py        ← empty placeholder
> ├── urls.py         ← empty placeholder
> └── tests.py        ← empty placeholder
> ```
> Register 'apps.dashboard' in INSTALLED_APPS in config/settings/base.py.
> Add an empty url include to config/urls.py under /api/v1/ to prevent
> import errors.
>
> **Step 2 — Create apps/core/permissions.py** (update the existing file):
>
> Keep the existing IsAdminOrSeller class. Add a new IsAdmin class:
> ```python
> class IsAdmin(BasePermission):
>     '''
>     Allows access only to users with role == 'admin'.
>     Used exclusively for the admin dashboard endpoints.
>     Returning 403 (not 404) on failure so it's clear the endpoint
>     exists but the caller lacks permission.
>     '''
>     def has_permission(self, request, view):
>         return (
>             request.user
>             and request.user.is_authenticated
>             and getattr(request.user, 'role', None) == 'admin'
>         )
> ```
>
> **Step 3 — Create apps/dashboard/serializers.py** with:
>
> AdminVariantSerializer:
>   Fields: variant_id, size, color, sku, price, stock, images
>   All fields writable (admin can edit any variant field)
>
> AdminProductCreateSerializer (for POST /admin/products/):
>   Fields: name, description, brand, base_price, category_id (CharField,
>   validated as ObjectId), tags (ListField), images (ListField),
>   variants (AdminVariantSerializer many=True)
>   validate_name(): check no other active product has the same slug
>   (generate the slug in validate and attach to validated_data)
>   validate_category_id(): verify the Category exists in MongoDB
>   validate_variants(): at least one variant required
>
> AdminProductUpdateSerializer (for PATCH /admin/products/{slug}/):
>   Same fields as Create but all optional.
>   At least one field required — validate in validate().
>
> AdminCategorySerializer (for POST/PATCH /admin/categories/):
>   Fields: name, slug (auto-generated from name if not provided),
>   parent_id (nullable CharField), image_url, order
>   validate_name(): check no other category has the same slug
>
> **Step 4 — Create apps/dashboard/views.py** with:
>
> AdminProductListView (APIView, IsAdmin):
>   GET /api/v1/admin/products/
>   Returns ALL products (including is_active=False) with pagination.
>   Query params: search, category, status (active/inactive), sort,
>   page, page_size (default 20)
>   Response shape: { count, total_pages, current_page, results }
>   Each result: id, name, slug, brand, base_price, category_id,
>   is_active, avg_rating, review_count, stock_total (sum of all variant
>   stocks), created_at
>   WHY include inactive: admins need to see and re-activate soft-deleted
>   products. Customers never see this endpoint.
>
> AdminProductCreateView (APIView, IsAdmin):
>   POST /api/v1/admin/products/
>   Validate with AdminProductCreateSerializer.
>   Set seller_id = request.user.id (admin is the seller for manually
>   created products).
>   Generate slug from name using Product.generate_slug(name).
>   If slug already exists, append a short uuid suffix.
>   Set avg_rating=0.0, review_count=0, is_active=True.
>   Save and return the full product with 201.
>
> AdminProductDetailView (APIView, IsAdmin):
>   GET /api/v1/admin/products/{slug}/
>   Returns full product detail including all variants and is_active status.
>
>   PATCH /api/v1/admin/products/{slug}/
>   Validate with AdminProductUpdateSerializer.
>   Only update the fields that were provided (use .get() for each field).
>   If name changes and slug is not explicitly provided, regenerate the slug.
>   Save and return updated product.
>
>   DELETE /api/v1/admin/products/{slug}/
>   SOFT DELETE ONLY: set product.is_active = False, save.
>   Return 200 { detail: 'Product deactivated.' }
>   WHY soft delete: orders contain snapshots of product data. Hard-deleting
>   would not break orders (snapshots are embedded), but it would break any
>   admin audit trail. Soft delete lets the admin re-activate the product.
>   Also add a separate PATCH endpoint to re-activate:
>   PATCH /api/v1/admin/products/{slug}/activate/ → set is_active=True.
>
> AdminCategoryListView (APIView, IsAdmin):
>   GET /api/v1/admin/categories/
>   Returns flat list of all categories (not the nested tree — admin needs
>   to see and edit individual categories). Sorted by order field.
>
>   POST /api/v1/admin/categories/
>   Validate with AdminCategorySerializer.
>   Auto-generate slug from name if not provided.
>   Save and return 201.
>
> AdminCategoryDetailView (APIView, IsAdmin):
>   PATCH /api/v1/admin/categories/{slug}/
>   Update category fields. If name changes, regenerate slug.
>   Invalidate the category tree cache after any change:
>     from django.core.cache import cache
>     cache.delete('category_tree')
>
>   DELETE /api/v1/admin/categories/{slug}/
>   Only allow if no products are assigned to this category:
>     count = Product.objects(category_id=category.id, is_active=True).count()
>     if count > 0: return 400 'Cannot delete a category with active products.'
>   Delete the category document and return 200.
>
> **Step 5 — Create apps/dashboard/urls.py:**
> ```python
> urlpatterns = [
>     # Products
>     path('admin/products/',
>          AdminProductListView.as_view(),       name='admin-product-list'),
>     path('admin/products/create/',
>          AdminProductCreateView.as_view(),     name='admin-product-create'),
>     path('admin/products/<str:slug>/',
>          AdminProductDetailView.as_view(),     name='admin-product-detail'),
>     path('admin/products/<str:slug>/activate/',
>          AdminProductActivateView.as_view(),   name='admin-product-activate'),
>
>     # Categories
>     path('admin/categories/',
>          AdminCategoryListView.as_view(),      name='admin-category-list'),
>     path('admin/categories/<str:slug>/',
>          AdminCategoryDetailView.as_view(),    name='admin-category-detail'),
> ]
> ```
> Include in config/urls.py.
>
> **Step 6 — Manual verification:**
> ```bash
> python manage.py check --settings=config.settings.dev
> # Expected: System check identified no issues
>
> # Login as admin, then test:
> # POST /api/v1/admin/products/create/ → 201
> # GET /api/v1/admin/products/ → paginated list including inactive products
> # PATCH /api/v1/admin/products/{slug}/ → updated product
> # DELETE /api/v1/admin/products/{slug}/ → 200, product.is_active = False
> # GET /api/v1/products/{slug}/ → 404 (soft deleted products don't appear)
> # PATCH /api/v1/admin/products/{slug}/activate/ → 200, product re-activated
> ```
>
> Show all complete file contents."

---

## DAY 2 — Admin Order Management + User Management + Analytics Endpoints

### Prompt:

> "Build the admin order management, user management, and analytics data
> endpoints that will power the Recharts dashboard on the frontend.
>
> **Step 1 — Add to apps/dashboard/views.py:**
>
> AdminOrderListView (APIView, IsAdmin):
>   GET /api/v1/admin/orders/
>   Returns ALL orders across ALL users with pagination.
>   Query params:
>     status     — filter by order status (pending/paid/processing/etc.)
>     search     — search by order_number or user email (do separate
>                  queries: find users matching email, then find their orders)
>     date_from  — created_at >= date (format: YYYY-MM-DD)
>     date_to    — created_at <= date
>     sort       — newest (default), oldest, total_desc, total_asc
>     page, page_size (default 20)
>   Response includes user_email (look up User by user_id for each order —
>   use a SerializerMethodField).
>   WHY admin sees all orders: customers only see their own orders via
>   /api/v1/orders/. This endpoint is strictly for the admin dashboard.
>
> AdminOrderDetailView (APIView, IsAdmin):
>   GET /api/v1/admin/orders/{order_number}/
>   Returns full order detail including status_history array.
>   Includes user_email, user_first_name, user_last_name (looked up from
>   User document by user_id).
>
> AdminUserListView (APIView, IsAdmin):
>   GET /api/v1/admin/users/
>   Returns paginated list of all users.
>   Query params: search (email, first_name, last_name), role, is_active,
>   page, page_size (default 20)
>   Fields: id, email, first_name, last_name, role, is_active, is_verified,
>   created_at, order_count (number of orders placed by this user — use
>   Order.objects(user_id=user.id).count())
>   Sort: newest registered first.
>
> AdminUserDetailView (APIView, IsAdmin):
>   GET /api/v1/admin/users/{user_id}/
>   Full user profile including addresses, but NOT password_hash.
>
>   PATCH /api/v1/admin/users/{user_id}/
>   Admin can change: role (customer/seller/admin), is_active, is_verified.
>   Admin CANNOT change: email, password (user must do this themselves).
>   WHY limit what admin can change: least-privilege principle. If an admin
>   could change passwords, a compromised admin account could lock out users.
>
> **Step 2 — Analytics endpoints (for Recharts charts):**
>
> AdminRevenueView (APIView, IsAdmin):
>   GET /api/v1/admin/analytics/revenue/
>   Query param: period = 'daily' | 'weekly' | 'monthly' (default: 'daily')
>                days   = 30 (how many periods to return, default 30)
>
>   Implementation:
>   Use mongoengine's __raw__ with MongoDB aggregation to group paid orders
>   by date and sum their totals:
>   ```python
>   from datetime import datetime, timedelta
>   from apps.orders.documents import Order
>
>   cutoff = datetime.utcnow() - timedelta(days=days)
>
>   # Get all paid orders since cutoff
>   orders = Order.objects(
>       status__in=['paid', 'processing', 'shipped', 'delivered'],
>       created_at__gte=cutoff,
>   ).only('created_at', 'total')
>
>   # Group in Python (fast enough for typical store volumes)
>   # For daily: key = date string 'YYYY-MM-DD'
>   # For weekly: key = 'YYYY-WNN'
>   # For monthly: key = 'YYYY-MM'
>   ```
>   Returns: list of { period: 'YYYY-MM-DD', revenue: float, order_count: int }
>   Sorted ascending by period (oldest first — Recharts expects this).
>
> AdminOrderStatsView (APIView, IsAdmin):
>   GET /api/v1/admin/analytics/order-stats/
>   Returns counts of orders grouped by status plus overall totals:
>   ```json
>   {
>     "total_orders":    142,
>     "total_revenue":   28450.00,
>     "avg_order_value": 200.35,
>     "by_status": {
>       "pending":    12,
>       "paid":        8,
>       "processing": 25,
>       "shipped":    47,
>       "delivered":  45,
>       "cancelled":   5
>     }
>   }
>   ```
>   Implementation: loop over statuses with .count() queries.
>   Only count orders from the last 90 days for performance.
>
> AdminTopProductsView (APIView, IsAdmin):
>   GET /api/v1/admin/analytics/top-products/
>   Query param: limit = 10
>   Returns the top N products by review_count * avg_rating score.
>   Simple implementation: Product.objects(is_active=True)
>     .order_by('-review_count').limit(limit)
>   Fields: name, slug, base_price, avg_rating, review_count, images[0]
>
> AdminLowStockView (APIView, IsAdmin):
>   GET /api/v1/admin/analytics/low-stock/
>   Query param: threshold = 5 (default from LOW_STOCK_THRESHOLD setting)
>   Returns all variants across all active products where stock <= threshold.
>   Same logic as the check_low_stock Celery task but as an API endpoint
>   so the admin can see it in real time in the dashboard.
>   Returns: list of {
>     product_name, product_slug, variant_id, sku, color, size,
>     stock, is_critical (stock <= 2)
>   }
>   Sorted ascending by stock (most urgent first).
>
> AdminInventoryUpdateView (APIView, IsAdmin):
>   PATCH /api/v1/admin/inventory/{product_slug}/{variant_id}/
>   Body: { stock: integer (min 0) }
>   Updates the stock of a specific variant atomically:
>   ```python
>   Product.objects(
>       slug=product_slug,
>       variants__variant_id=variant_id,
>   ).update_one(
>       set__variants__S__stock=new_stock
>   )
>   ```
>   Returns: { product_slug, variant_id, new_stock }
>   WHY atomic update: if two admins update the same variant simultaneously,
>   a read-then-write approach would cause a race condition. MongoDB's
>   positional operator ($) with update_one is atomic.
>
> **Step 3 — Add to apps/dashboard/urls.py:**
> ```python
> # Orders
> path('admin/orders/',
>      AdminOrderListView.as_view(),       name='admin-order-list'),
> path('admin/orders/<str:order_number>/',
>      AdminOrderDetailView.as_view(),     name='admin-order-detail'),
>
> # Users
> path('admin/users/',
>      AdminUserListView.as_view(),        name='admin-user-list'),
> path('admin/users/<str:user_id>/',
>      AdminUserDetailView.as_view(),      name='admin-user-detail'),
>
> # Analytics
> path('admin/analytics/revenue/',
>      AdminRevenueView.as_view(),         name='admin-revenue'),
> path('admin/analytics/order-stats/',
>      AdminOrderStatsView.as_view(),      name='admin-order-stats'),
> path('admin/analytics/top-products/',
>      AdminTopProductsView.as_view(),     name='admin-top-products'),
> path('admin/analytics/low-stock/',
>      AdminLowStockView.as_view(),        name='admin-low-stock'),
>
> # Inventory
> path('admin/inventory/<str:product_slug>/<str:variant_id>/',
>      AdminInventoryUpdateView.as_view(), name='admin-inventory-update'),
> ```
>
> **Step 4 — Manual verification with curl:**
> ```bash
> # Login as admin
> curl -s -c cookies.txt -X POST http://localhost:8000/api/v1/auth/login/ \
>   -H 'Content-Type: application/json' \
>   -d '{"email":"seed@example.com","password":"adminpass123"}' | python -m json.tool
>
> # Test admin order list
> curl -s -b cookies.txt http://localhost:8000/api/v1/admin/orders/ | python -m json.tool
>
> # Test analytics
> curl -s -b cookies.txt 'http://localhost:8000/api/v1/admin/analytics/revenue/?period=daily&days=7' \
>   | python -m json.tool
>
> curl -s -b cookies.txt http://localhost:8000/api/v1/admin/analytics/order-stats/ \
>   | python -m json.tool
>
> curl -s -b cookies.txt http://localhost:8000/api/v1/admin/analytics/low-stock/ \
>   | python -m json.tool
>
> # Test that a customer cannot access admin endpoints
> # Login as customer first, then:
> curl -s -b customer_cookies.txt http://localhost:8000/api/v1/admin/orders/ \
>   | python -m json.tool
> # Expected: 403 Forbidden
> ```
>
> Show all complete file contents."

---

## DAY 3 — Admin Backend Tests

### Prompt:

> "Write the Django test suite for the admin dashboard backend.
> All tests use APITestCase with force_authenticate.
>
> **Create apps/dashboard/tests.py** with:
>
> setUp:
>   - Clean all documents
>   - Create admin user (role='admin')
>   - Create customer user (role='customer')
>   - Create seller user (role='seller')
>   - Create 2 categories: Electronics (top-level), Footwear (top-level)
>   - Create 5 products: 3 active, 2 inactive (is_active=False)
>   - Create 3 orders in various statuses for the customer
>
> tearDown:
>   - Delete all documents created in setUp
>
> Permission tests (apply to ALL admin endpoints):
>   test_unauthenticated_request_returns_401
>     — GET /admin/products/ without auth → 401
>   test_customer_cannot_access_admin_endpoints
>     — force_authenticate as customer → GET /admin/products/ → 403
>   test_seller_cannot_access_admin_endpoints
>     — force_authenticate as seller → GET /admin/products/ → 403
>   test_admin_can_access_admin_endpoints
>     — force_authenticate as admin → GET /admin/products/ → 200
>
> Product management tests:
>   test_admin_product_list_includes_inactive_products
>     — GET /admin/products/ as admin → count includes is_active=False products
>   test_admin_product_list_filter_by_status
>     — GET /admin/products/?status=inactive → only inactive products
>   test_admin_create_product_success
>     — POST /admin/products/create/ with valid data → 201, product in DB
>   test_admin_create_product_missing_variants_returns_400
>     — POST without variants → 400
>   test_admin_create_product_invalid_category_returns_400
>     — POST with non-existent category_id → 400
>   test_admin_update_product_name
>     — PATCH /admin/products/{slug}/ with new name → 200, name updated
>   test_admin_soft_delete_product
>     — DELETE /admin/products/{slug}/ → 200, is_active becomes False
>   test_soft_deleted_product_not_in_public_list
>     — soft delete a product → GET /products/ (public) → product not in results
>   test_admin_activate_product
>     — soft delete then PATCH /activate/ → is_active becomes True again
>   test_admin_delete_category_with_active_products_returns_400
>     — try to delete a category that has active products → 400
>
> Order management tests:
>   test_admin_order_list_returns_all_users_orders
>     — GET /admin/orders/ → sees orders from all users
>   test_admin_order_list_filter_by_status
>     — GET /admin/orders/?status=paid → only paid orders
>   test_admin_order_detail_includes_user_email
>     — GET /admin/orders/{order_number}/ → response has user_email field
>
> User management tests:
>   test_admin_user_list_returns_all_users
>     — GET /admin/users/ → all users visible
>   test_admin_can_deactivate_user
>     — PATCH /admin/users/{id}/ { is_active: false } → user deactivated
>   test_admin_cannot_change_user_password
>     — PATCH with { password: 'newpass' } → field ignored or 400
>
> Analytics tests:
>   test_revenue_endpoint_returns_correct_shape
>     — GET /admin/analytics/revenue/ → list of { period, revenue, order_count }
>   test_order_stats_endpoint_returns_by_status
>     — GET /admin/analytics/order-stats/ → has by_status dict
>   test_low_stock_endpoint_returns_only_low_variants
>     — Set one variant stock=2, GET /admin/analytics/low-stock/ → appears in results
>   test_inventory_update_changes_stock
>     — PATCH /admin/inventory/{slug}/{variant_id}/ { stock: 99 } → stock updated
>
> Run:
> ```bash
> python manage.py test apps.dashboard --settings=config.settings.test -v 2
> ```
> All tests must pass before proceeding to Day 4.
>
> Show all complete file contents."

---

## DAY 4 — React Admin Foundation: Routes, Layout, Dashboard Overview

### Prompt:

> "Build the React admin section foundation: role-based route protection,
> the admin sidebar layout, and the main dashboard overview page with
> Recharts charts.
>
> **Step 1 — Create src/api/adminAPI.js:**
> All functions use the shared axiosInstance (withCredentials: true).
>
> ```javascript
> // Products
> adminAPI.listProducts(filters)       GET /admin/products/
> adminAPI.getProduct(slug)            GET /admin/products/{slug}/
> adminAPI.createProduct(data)         POST /admin/products/create/
> adminAPI.updateProduct(slug, data)   PATCH /admin/products/{slug}/
> adminAPI.deleteProduct(slug)         DELETE /admin/products/{slug}/
> adminAPI.activateProduct(slug)       PATCH /admin/products/{slug}/activate/
>
> // Categories
> adminAPI.listCategories()            GET /admin/categories/
> adminAPI.createCategory(data)        POST /admin/categories/
> adminAPI.updateCategory(slug, data)  PATCH /admin/categories/{slug}/
> adminAPI.deleteCategory(slug)        DELETE /admin/categories/{slug}/
>
> // Orders
> adminAPI.listOrders(filters)         GET /admin/orders/
> adminAPI.getOrder(orderNumber)       GET /admin/orders/{orderNumber}/
> adminAPI.updateOrderStatus(orderNumber, data)
>                                      PATCH /orders/{orderNumber}/status/
>
> // Users
> adminAPI.listUsers(filters)          GET /admin/users/
> adminAPI.getUser(userId)             GET /admin/users/{userId}/
> adminAPI.updateUser(userId, data)    PATCH /admin/users/{userId}/
>
> // Analytics
> adminAPI.getRevenue(period, days)    GET /admin/analytics/revenue/
> adminAPI.getOrderStats()             GET /admin/analytics/order-stats/
> adminAPI.getTopProducts(limit)       GET /admin/analytics/top-products/
> adminAPI.getLowStock(threshold)      GET /admin/analytics/low-stock/
> adminAPI.updateInventory(slug, variantId, stock)
>                                      PATCH /admin/inventory/{slug}/{variantId}/
> ```
> All return response.data directly.
>
> **Step 2 — Create src/components/admin/AdminRoute.jsx:**
> A route guard that only lets role === 'admin' users through.
> Reads from Redux authSlice (selectCurrentUser).
> If not authenticated: redirect to /login.
> If authenticated but not admin: redirect to / with a toast
>   'You do not have admin access.'
> If admin: render the outlet.
>
> ```jsx
> const AdminRoute = () => {
>   const user = useSelector(selectCurrentUser);
>   const isAuthenticated = useSelector(selectIsAuthenticated);
>
>   if (!isAuthenticated) return <Navigate to='/login' replace />;
>   if (user?.role !== 'admin') {
>     toast.error('You do not have admin access.');
>     return <Navigate to='/' replace />;
>   }
>   return <Outlet />;
> };
> ```
>
> **Step 3 — Create src/components/admin/AdminLayout.jsx:**
> A two-column layout: fixed sidebar left, content area right.
> The sidebar is separate from the customer-facing Navbar — admin pages
> have their own navigation.
>
> Sidebar links (use NavLink for active styling):
>   ⚡ Store Admin (header/logo — links to /admin)
>   Dashboard → /admin
>   Products  → /admin/products
>   Categories → /admin/categories
>   Orders    → /admin/orders
>   Users     → /admin/users
>   Inventory → /admin/inventory
>   ← Back to Store → / (link back to customer site)
>
> Sidebar footer: logged-in admin's email and a Logout button.
>
> Content area: renders <Outlet /> (the active admin page).
>
> Tailwind classes for sidebar: dark background (bg-slate-900), white text,
> w-64, h-screen, fixed, flex flex-col, overflow-y-auto.
> Content area: ml-64, min-h-screen, bg-slate-50, p-8.
>
> **Step 4 — Create src/hooks/useAdmin.js:**
> TanStack Query hooks for all admin data:
> ```javascript
> useAdminProducts(filters)   queryKey: ['admin', 'products', filters]
> useAdminOrders(filters)     queryKey: ['admin', 'orders', filters]
> useAdminUsers(filters)      queryKey: ['admin', 'users', filters]
> useAdminRevenue(period, days) queryKey: ['admin', 'revenue', period, days]
> useAdminOrderStats()        queryKey: ['admin', 'order-stats']
> useAdminTopProducts(limit)  queryKey: ['admin', 'top-products', limit]
> useAdminLowStock()          queryKey: ['admin', 'low-stock']
> ```
> All staleTime: 2 * 60 * 1000 (2 minutes — admin data can be slightly stale).
>
> **Step 5 — Create src/pages/admin/AdminDashboard.jsx:**
> The main overview page at /admin.
>
> Layout: 4 stat cards at the top, then 2 charts side by side,
> then a low-stock alert table at the bottom.
>
> Stat cards (use useAdminOrderStats()):
>   Total Orders  — total_orders value, violet background
>   Total Revenue — formatted as $XX,XXX.XX, green background
>   Avg Order     — avg_order_value formatted, blue background
>   Pending Orders — by_status.pending count, yellow background
>
> Revenue Line Chart (use useAdminRevenue('daily', 30)):
>   Recharts LineChart with CartesianGrid, XAxis, YAxis, Tooltip, Legend
>   X axis: period dates (show last 7 of the 30-day label for readability)
>   Y axis: revenue in dollars
>   Line: violet color (#7c3aed), smooth curve
>   Title: 'Revenue — Last 30 Days'
>
> Orders by Status Bar Chart (use useAdminOrderStats()):
>   Recharts BarChart using by_status data
>   Convert the by_status object to array: [{ status: 'paid', count: 8 }, ...]
>   Color each bar by status:
>     pending → yellow, paid → blue, processing → indigo,
>     shipped → violet, delivered → green, cancelled → red
>   Title: 'Orders by Status'
>
> Low Stock Alert Table (use useAdminLowStock()):
>   Columns: Product, SKU, Variant, Stock
>   Rows with stock <= 2: red background, stock 3-5: orange background
>   'Update Stock' button on each row — navigates to /admin/inventory
>   Empty state: green checkmark + 'All products are well stocked'
>
> Loading states: skeleton placeholders for all sections.
>
> **Step 6 — Update src/router.jsx:**
> Add admin routes as a nested group under AdminRoute:
> ```jsx
> {
>   element: <AdminRoute />,
>   children: [
>     {
>       element: <AdminLayout />,
>       children: [
>         { path: '/admin',            element: <AdminDashboard /> },
>         { path: '/admin/products',   element: <div>Products coming Day 5</div> },
>         { path: '/admin/categories', element: <div>Categories coming Day 6</div> },
>         { path: '/admin/orders',     element: <div>Orders coming Day 6</div> },
>         { path: '/admin/users',      element: <div>Users coming Day 6</div> },
>         { path: '/admin/inventory',  element: <div>Inventory coming Day 6</div> },
>       ]
>     }
>   ]
> }
> ```
>
> Show all complete file contents."

---

## DAY 5 — React Admin: Product Management

### Prompt:

> "Build the complete product management section of the admin dashboard:
> list page with filters, create form, and edit form.
>
> **Step 1 — Create src/pages/admin/AdminProducts.jsx:**
> The product list page at /admin/products.
>
> Features:
>   Search bar: filters by product name (debounced 400ms)
>   Status filter: All / Active / Inactive dropdown
>   Category filter: dropdown populated from useCategories()
>   Sort: Newest, Name A-Z, Price Low-High, Price High-Low
>   'Add Product' button → navigates to /admin/products/new
>
>   Products table:
>   Columns: Image (40x40 thumbnail), Name, Category, Price, Stock,
>            Status badge (Active/Inactive), Rating, Actions
>   Actions per row:
>     Edit (pencil icon) → /admin/products/{slug}/edit
>     Deactivate/Activate toggle (eye icon)
>       On deactivate: confirm dialog → call adminAPI.deleteProduct(slug)
>       On activate: call adminAPI.activateProduct(slug)
>       Both invalidate ['admin', 'products'] query on success
>   Pagination: same Pagination component from the customer side
>
>   Loading: skeleton rows (10 rows of gray placeholders)
>   Empty: 'No products found. Create your first product.'
>
> **Step 2 — Create src/pages/admin/AdminProductForm.jsx:**
> Used for BOTH creating (/admin/products/new) and
> editing (/admin/products/{slug}/edit).
>
> Detect mode: if useParams().slug exists, fetch the product and
> pre-populate the form. Title changes to 'Edit Product' or 'New Product'.
>
> Form sections (react-hook-form + Zod):
>
>   Basic Info section:
>     name (required, max 200)
>     brand (optional)
>     description (required, min 20, textarea)
>     base_price (required, number > 0)
>     category_id (required, select populated from useCategories())
>     tags (text input — comma-separated, parsed to array on submit)
>
>   Images section:
>     Existing images: thumbnail grid, each with an X button to remove
>     Upload new image:
>       File input (jpeg/png/webp, max 5MB)
>       On select: call adminAPI.uploadImage(file) via the existing
>       POST /products/upload-image/ endpoint
>       Show upload spinner while uploading
>       On success: add the returned URL to the images array
>
>   Variants section:
>     Table of existing variants:
>       Columns: SKU, Color, Size, Price, Stock, Remove
>     'Add Variant' button → adds a new empty row
>     Each row is editable inline (controlled inputs)
>     Stock field: number input, min 0
>     Price field: number input, min 0
>     Remove button: removes the variant row (confirm if it's the last one)
>
>   Submit button: 'Create Product' or 'Save Changes'
>   Cancel link: navigates back to /admin/products
>
>   On submit:
>     Create mode: call adminAPI.createProduct(data) → on success,
>       navigate to /admin/products with toast 'Product created!'
>     Edit mode: call adminAPI.updateProduct(slug, data) → on success,
>       invalidate product queries, show toast 'Product updated!'
>
>   Error handling: show field-level errors from the server response
>   (e.g. if slug already exists).
>
> **Step 3 — Update src/router.jsx:**
> Replace the placeholder admin product routes:
> ```jsx
> { path: '/admin/products',       element: <AdminProducts /> },
> { path: '/admin/products/new',   element: <AdminProductForm /> },
> { path: '/admin/products/:slug/edit', element: <AdminProductForm /> },
> ```
>
> Show all complete file contents."

---

## DAY 6 — React Admin: Orders, Users, Categories, Inventory

### Prompt:

> "Build the remaining admin pages: order management, user management,
> category management, and inventory management.
>
> **Step 1 — Create src/pages/admin/AdminOrders.jsx:**
> Order list at /admin/orders.
>
> Filters: status dropdown, date range (from/to date pickers using
>   standard HTML date inputs), search (by order number or customer email)
> Sort: newest (default), oldest, total_desc, total_asc
>
> Orders table:
>   Columns: Order Number, Customer Email, Date, Items, Total, Status, Actions
>   Status badge: color-coded (same colors as customer Orders page)
>   Actions: 'View' button → opens an inline slide-over panel (or modal)
>     showing the full order detail with a status update dropdown.
>
> Status update inline:
>   Select new status (processing / shipped / delivered / cancelled)
>   'Tracking Number' text field (only shown when shipped is selected)
>   'Update' button → calls adminAPI.updateOrderStatus(orderNumber, data)
>   On success: toast 'Order updated', invalidate admin orders query.
>
> Pagination at the bottom.
>
> **Step 2 — Create src/pages/admin/AdminUsers.jsx:**
> User list at /admin/users.
>
> Filters: role dropdown (all/customer/seller/admin), is_active toggle,
>   search by email or name
>
> Users table:
>   Columns: Email, Name, Role badge, Status (Active/Inactive),
>            Verified badge, Joined, Orders, Actions
>   Actions: 'View' button → opens a side panel with full user info
>     Side panel shows: all profile fields, address list, order history
>     (link to /admin/orders?search=their-email)
>     Toggle active/inactive button: calls adminAPI.updateUser(id, {is_active})
>     Change role dropdown: calls adminAPI.updateUser(id, {role})
>
> **Step 3 — Create src/pages/admin/AdminCategories.jsx:**
> Category management at /admin/categories.
>
> Layout: two-column — category list on the left, edit form on the right.
>
> Category list:
>   All categories, indented to show parent/child relationships
>   (parent categories first, subcategories indented under them)
>   Each item: category name, product count, Edit and Delete buttons
>   'Add Category' button at the top
>
> Edit/Create form (right panel, react-hook-form):
>   Fields: name, parent_id (select — choose from existing categories
>   or 'None' for top-level), image_url (text input), order (number)
>   Name auto-generates slug preview as the user types
>   Save button — calls createCategory or updateCategory
>   On delete: confirm dialog → if server returns 400 (has products),
>   show that error message to the admin
>   On success: invalidate ['admin', 'categories'] and ['categories'] queries
>   (the second one refreshes the customer-facing category tree too)
>
> **Step 4 — Create src/pages/admin/AdminInventory.jsx:**
> Inventory management at /admin/inventory.
>
> Shows the same low-stock data as the dashboard alert table,
> but with ALL products (not just low-stock ones).
>
> Filter: search by product name, filter by stock level
>   (all / low stock ≤5 / critical ≤2 / out of stock = 0)
>
> Inventory table:
>   Columns: Product, SKU, Color, Size, Current Stock, Update
>   Stock column: color-coded number
>     0     → red badge 'Out of Stock'
>     1-2   → red text (critical)
>     3-5   → orange text (low)
>     6+    → normal text
>   Update column: inline number input + 'Save' button per row
>     On Save: calls adminAPI.updateInventory(slug, variantId, newStock)
>     Show spinner on the button while saving
>     On success: green flash on the row, invalidate inventory queries
>
> **Step 5 — Update src/router.jsx:**
> Replace all admin route placeholders:
> ```jsx
> { path: '/admin/categories', element: <AdminCategories /> },
> { path: '/admin/orders',     element: <AdminOrders />    },
> { path: '/admin/users',      element: <AdminUsers />     },
> { path: '/admin/inventory',  element: <AdminInventory /> },
> ```
>
> **Step 6 — Add Admin link to Navbar:**
> In the user dropdown (Navbar.jsx), if user.role === 'admin',
> add an 'Admin Dashboard' link to /admin:
> ```jsx
> {user?.role === 'admin' && (
>   <Link to='/admin'
>     className='block px-4 py-2.5 text-sm text-violet-400
>                hover:bg-slate-700 rounded-xl'>
>     Admin Dashboard
>   </Link>
> )}
> ```
>
> Show all complete file contents."

---

## DAY 7 — Integration Testing + Week 7 Cleanup + Handoff

### Prompt:

> "Run the full Week 7 verification checklist, fix any issues, and confirm
> the admin dashboard works end-to-end.
>
> **Step 1 — Run all backend tests:**
> ```bash
> python manage.py test apps.dashboard apps.orders apps.products \
>   apps.authentication apps.cart apps.reviews apps.wishlist \
>   --settings=config.settings.test -v 2
> ```
> All tests must pass.
>
> **Step 2 — Full end-to-end verification checklist:**
>
> Backend (curl as admin):
>   1.  GET /api/v1/admin/products/ → all products including inactive
>   2.  POST /api/v1/admin/products/create/ → new product created
>   3.  PATCH /api/v1/admin/products/{slug}/ → name updated
>   4.  DELETE /api/v1/admin/products/{slug}/ → is_active=False (soft delete)
>   5.  GET /api/v1/products/{slug}/ → 404 (soft deleted)
>   6.  PATCH /api/v1/admin/products/{slug}/activate/ → is_active=True again
>   7.  POST /api/v1/admin/categories/ → category created
>   8.  DELETE /api/v1/admin/categories/{slug}/ with products → 400
>   9.  GET /api/v1/admin/orders/ → all orders, all users
>   10. GET /api/v1/admin/orders/?status=paid → only paid orders
>   11. GET /api/v1/admin/users/ → all users
>   12. PATCH /api/v1/admin/users/{id}/ {is_active: false} → user deactivated
>   13. GET /api/v1/admin/analytics/revenue/?period=daily&days=7 → list of 7 items
>   14. GET /api/v1/admin/analytics/order-stats/ → by_status object
>   15. GET /api/v1/admin/analytics/low-stock/ → low stock items
>   16. PATCH /api/v1/admin/inventory/{slug}/{variant_id}/ {stock:99} → updated
>   17. Customer trying any admin endpoint → 403
>   18. Unauthenticated request → 401
>
> Frontend:
>   19. Login as admin → Navbar shows 'Admin Dashboard' link
>   20. Navigate to /admin → Dashboard overview loads
>   21. Stat cards show correct numbers (Total Orders, Revenue, etc.)
>   22. Revenue line chart renders with data points for last 30 days
>   23. Orders by Status bar chart renders with correct colors
>   24. Low stock table shows items with color-coded stock levels
>   25. Navigate to /admin/products → product list with all products
>   26. Filter by status=inactive → only inactive products shown
>   27. Search by name → matching products filtered
>   28. Click 'Add Product' → /admin/products/new form loads
>   29. Fill form, upload image → product created, redirected to list
>   30. Click Edit on a product → form pre-populated with existing data
>   31. Save changes → product updated, toast confirmation
>   32. Deactivate a product → status badge changes to Inactive
>   33. Navigate to /admin/orders → all orders from all customers
>   34. Filter by status → correctly filtered
>   35. Click View on order → side panel opens with full order detail
>   36. Update status to shipped + tracking number → order status updates
>   37. Navigate to /admin/users → all users listed
>   38. Toggle a user to inactive → Status badge changes
>   39. Navigate to /admin/categories → category list + edit form
>   40. Create a new subcategory → appears under parent in list
>   41. Navigate to /admin/inventory → all variants with stock levels
>   42. Update a variant stock → number updates instantly
>   43. Low stock items show color-coded badges
>   44. Login as customer → navigate to /admin → redirected to / with toast
>
> **Step 3 — Provide the Week 7 session handoff summary:**
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

## WEEK 7 SUMMARY — What You Will Have Built

By the end of Day 7, your project will have:

**Backend:**
- `apps/dashboard/` app with all admin endpoints under `/api/v1/admin/`
- `IsAdmin` permission class — blocks everyone except role='admin'
- Product CRUD: list (including inactive), create, update, soft-delete,
  re-activate — all admin-only
- Category CRUD: create, update, delete (blocked if active products exist)
- Admin order list (all users, all orders) with status/date/search filters
- Admin user list with role and active status management
- Analytics endpoints: revenue by period (daily/weekly/monthly),
  order stats by status, top products, low stock (real-time)
- Inventory update endpoint (atomic MongoDB positional update)
- Django APITestCase suite: permission tests, product CRUD, order/user
  management, analytics shape validation

**Frontend:**
- `adminAPI.js` — all admin API calls
- `useAdmin.js` — TanStack Query hooks for all admin data (2-min staleTime)
- `AdminRoute.jsx` — role-based route guard (redirects non-admins to /)
- `AdminLayout.jsx` — fixed sidebar with navigation, separate from customer Navbar
- `AdminDashboard.jsx` — stat cards + Recharts LineChart (revenue) +
  Recharts BarChart (orders by status) + low stock alert table
- `AdminProducts.jsx` — searchable, filterable product list with
  activate/deactivate toggle and pagination
- `AdminProductForm.jsx` — dual-mode create/edit form with image upload
  and inline variant management
- `AdminOrders.jsx` — all-orders list with inline status update panel
- `AdminUsers.jsx` — user list with role management and activity toggle
- `AdminCategories.jsx` — two-column category manager with parent/child display
- `AdminInventory.jsx` — per-variant stock table with inline update
- Router updated: /admin/* routes nested under AdminRoute + AdminLayout
- Navbar: Admin Dashboard link visible only to role='admin' users

**Week 8 continues with:** Coupons system (coupon Document, validation
endpoint, apply at checkout, discount calculation), User Dashboard
enhancements (order tracking timeline, re-order button, account deletion),
and final polish on the customer-facing UI.