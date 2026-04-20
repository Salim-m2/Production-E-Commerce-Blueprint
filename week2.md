# Week 2 AI Implementation Prompt — Product Catalog, Categories, Search & Cloudinary Images
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## CONTEXT TO GIVE THE AI AT THE START OF EVERY SESSION THIS WEEK

> "I am continuing a production e-commerce build. Week 1 is fully complete and tested.
>
> **What already exists and is working:**
> - Django project with split settings (base.py / dev.py / prod.py), loaded via django-environ
> - MongoDB Atlas connected via mongoengine
> - User Document with set_password() and check_password() methods
> - Full JWT auth in httpOnly cookies: register, login, logout, token refresh, /auth/me/
> - Password reset skeleton (Token Document with TTL, endpoints without emails yet)
> - DRF throttling on all endpoints
> - Django APITestCase tests for all auth endpoints
> - React 18 + Vite frontend with Redux Toolkit, TanStack Query, Axios interceptor
> - authSlice, uiSlice, cartSlice (stub) in Redux store
> - Login and Register pages with Zod validation
> - React Router v6 with Layout, ProtectedRoute, auth routes
> - Navbar reflecting auth state, auth persistence via /auth/me/ on startup
>
> **Stack reminder:**
> - Backend: Django 4.x + DRF + mongoengine (NO Django ORM)
> - Frontend: React 18 + Vite + Redux Toolkit + TanStack Query + Axios
> - All API endpoints prefixed /api/v1/
> - JWT in httpOnly cookies ONLY — never localStorage
> - Images: Cloudinary (NOT S3 this week)
> - Styling: Tailwind CSS
>
> Today we are working on: [paste the specific day below]"

---

## DAY 1 — Product & Category mongoengine Documents + MongoDB Indexes

### Prompt:

> "Build all the backend data models for the product catalog. We are in apps/products/.
>
> **Step 1 — Create apps/products/documents.py** with the following mongoengine Documents:
>
> Variant (EmbeddedDocument) — embedded inside Product, not a separate collection:
>   variant_id   StringField — generate with str(uuid.uuid4()) on creation
>   size         StringField, optional
>   color        StringField, optional
>   sku          StringField, required, unique within the product
>   price        FloatField, required — can differ from base_price
>   stock        IntField, default=0, min_value=0
>   images       ListField of StringField (Cloudinary URLs, optional)
>
> Product (Document) — the main collection:
>   seller_id    ObjectIdField — reference to User._id
>   category_id  ObjectIdField — reference to Category._id
>   name         StringField, required, max_length=200
>   slug         StringField, required, unique — URL-safe version of name
>   description  StringField, required
>   brand        StringField, optional
>   base_price   FloatField, required — the lowest/default price
>   images       ListField of StringField (Cloudinary URLs)
>   tags         ListField of StringField (e.g. ['sneakers', 'basketball'])
>   variants     ListField of EmbeddedDocumentField(Variant)
>   avg_rating   FloatField, default=0.0
>   review_count IntField, default=0
>   is_active    BooleanField, default=True
>   created_at   DateTimeField, default=datetime.utcnow
>   updated_at   DateTimeField, default=datetime.utcnow
>
> Add a save() override that sets updated_at = datetime.utcnow() on every save.
> Add a generate_slug() classmethod that converts a name to a URL slug
> (lowercase, spaces to hyphens, remove special characters).
>
> Category (Document) — separate collection:
>   name       StringField, required, unique
>   slug       StringField, required, unique
>   parent_id  ObjectIdField, null=True (null = top-level category)
>   image_url  StringField, optional (Cloudinary URL)
>   order      IntField, default=0 — for display ordering
>
> **Step 2 — Create apps/products/indexes.py** — a standalone script to create all required
> MongoDB indexes. Write a function create_product_indexes() that uses pymongo's
> collection.create_index() syntax accessed via Product._get_collection():
>   products: slug (unique), category_id + is_active compound, base_price, avg_rating desc,
>             text index on name + description + tags
>   categories: parent_id, slug (unique)
>
> **Step 3 — Call create_product_indexes()** from config/settings/base.py after the
> mongoengine connection block so indexes are always ensured on startup.
>
> **Step 4 — Create a Django management command**
> apps/products/management/commands/seed_products.py that creates:
>   - 5 top-level categories: Electronics, Footwear, Clothing, Home and Kitchen, Sports
>   - 3 subcategories under Footwear: Sneakers, Sandals, Boots
>   - 50 realistic products spread across categories, each with:
>       2-4 variants with different sizes/colors
>       Realistic prices between $15 and $350
>       3-5 tags each
>       Placeholder image: https://res.cloudinary.com/demo/image/upload/sample.jpg
>       avg_rating between 3.5 and 5.0, review_count between 0 and 200
>
> Show all complete file contents."

---

## DAY 2 — Product & Category Serializers + API Views (List, Detail, Category Tree)

### Prompt:

> "Build all the serializers and API views for the product catalog.
>
> **Step 1 — Create apps/products/serializers.py** with:
>
> VariantSerializer (for embedded variants):
>   Fields: variant_id, size, color, sku, price, stock, images
>   Read-only: variant_id
>
> ProductListSerializer (lightweight — for listing pages):
>   Fields: id, name, slug, brand, base_price, images (FIRST image only as a string),
>   avg_rating, review_count, is_active, category_id
>   Add a SerializerMethodField min_price that returns the lowest price across all variants
>
> ProductDetailSerializer (full — for detail pages):
>   All fields from ProductListSerializer PLUS: description, tags, variants (full list
>   using VariantSerializer), all images as a list, created_at
>
> CategorySerializer:
>   Fields: id, name, slug, parent_id, image_url, order
>
> CategoryTreeSerializer:
>   Same as CategorySerializer but adds a children field (list of nested CategorySerializer)
>
> **Step 2 — Create apps/products/views.py** with these views:
>
> ProductListView (APIView, no auth required):
>   GET /api/v1/products/
>   Query params: category, min_price, max_price, rating, search, sort, page, page_size (default 12)
>   Filtering (apply only if param is present):
>     category: filter by category slug — look up the category first, then filter by its _id
>     min_price: base_price >= value
>     max_price: base_price <= value
>     rating: avg_rating >= value
>     search: use mongoengine text search or __icontains on name
>   Always filter is_active=True
>   Sorting options via sort param:
>     newest     -> order_by('-created_at') [default]
>     price_asc  -> order_by('base_price')
>     price_desc -> order_by('-base_price')
>     rating     -> order_by('-avg_rating')
>     popular    -> order_by('-review_count')
>   Pagination response shape:
>     { count, total_pages, current_page, page_size, results: [...] }
>   Use ProductListSerializer for results
>
> ProductDetailView (APIView, no auth required):
>   GET /api/v1/products/{slug}/
>   Return 404 if not found or is_active=False
>   Use ProductDetailSerializer
>
> CategoryListView (APIView, no auth required):
>   GET /api/v1/categories/
>   Fetch all categories, build parent/child tree in Python (match by parent_id)
>   Return only top-level categories with their children array populated
>   Cache result for 1 hour using Django's cache framework:
>     cached = cache.get('category_tree')
>     if cached: return Response(cached)
>     cache.set('category_tree', result, timeout=3600)
>   Note: Redis is set up in Week 8. Use Django's default in-memory cache for now —
>   the cache.get/set calls are identical and will work with any backend.
>
> **Step 3 — Create apps/products/urls.py** wiring all 3 endpoints.
> Include it in config/urls.py under /api/v1/.
>
> Show all complete file contents."

---

## DAY 3 — Cloudinary Integration + Image Upload Endpoint

### Prompt:

> "Integrate Cloudinary for product image storage and build the image upload endpoint.
>
> **Step 1 — Install and configure Cloudinary:**
>   pip install cloudinary  (add to requirements.txt)
>
>   Add to .env.example:
>     CLOUDINARY_CLOUD_NAME=
>     CLOUDINARY_API_KEY=
>     CLOUDINARY_API_SECRET=
>
>   Add to config/settings/base.py:
>     import cloudinary
>     cloudinary.config(
>         cloud_name=env('CLOUDINARY_CLOUD_NAME'),
>         api_key=env('CLOUDINARY_API_KEY'),
>         api_secret=env('CLOUDINARY_API_SECRET'),
>         secure=True
>     )
>
> **Step 2 — Create apps/products/cloudinary_utils.py** with:
>
> upload_product_image(file, product_slug):
>   - Upload to Cloudinary under folder: ecommerce/products/{product_slug}/
>   - Transformations: f_auto, q_auto, max width 1200px
>   - Return the secure URL string
>   - Raise ValueError if content_type is not jpeg/png/webp
>   - Raise ValueError if file size exceeds 5MB
>
> delete_cloudinary_image(public_id):
>   - Delete an image by its public_id
>   - Used when admin removes a product image
>
> get_optimized_url(cloudinary_url, width=400):
>   - Insert w_{width},f_auto,q_auto transformation into an existing Cloudinary URL
>   - Returns the optimized URL string
>   - Used to generate thumbnails for product cards
>
> **Step 3 — Create the upload endpoint:**
>   POST /api/v1/products/upload-image/ (IsAdminOrSeller permission required)
>   Accepts multipart form data, field name: image
>   Validates file type and size via cloudinary_utils
>   Returns: { url, public_id }
>
> Write the IsAdminOrSeller permission class in apps/core/permissions.py:
>   has_permission returns True if user is authenticated AND role in ['admin', 'seller']
>
> **Step 4 — Update the seed command** from Day 1 to accept a --use-cloudinary flag.
>   With the flag: upload real placeholder images to Cloudinary.
>   Without the flag: use the demo URL (for offline dev).
>
> **Step 5 — Explain which page of the Cloudinary dashboard** contains the cloud name,
> API key, and API secret so I know where to find them.
>
> Show all complete file contents."

---

## DAY 4 — Product API Tests + Postman Collection

### Prompt:

> "Write a comprehensive test suite and Postman collection for the product catalog API.
>
> **Step 1 — Create apps/products/tests.py** using DRF's APITestCase.
>
> setUp creates:
>   - 3 categories: Electronics (top-level), Footwear (top-level), Sneakers (child of Footwear)
>   - 10 test products across those categories with varied prices, ratings, and tags
>   - 2 products with is_active=False (must never appear in results)
>
> Product List test cases:
>   test_product_list_returns_only_active
>   test_product_list_pagination           — verify count, total_pages, current_page
>   test_filter_by_category_slug           — ?category=footwear
>   test_filter_by_min_price               — ?min_price=100, all results >= 100
>   test_filter_by_max_price               — ?max_price=50, all results <= 50
>   test_filter_by_price_range             — ?min_price=50&max_price=150
>   test_filter_by_rating                  — ?rating=4, all avg_rating >= 4
>   test_search_by_name                    — ?search=jordan
>   test_sort_price_asc                    — prices ascending in results
>   test_sort_price_desc                   — prices descending in results
>   test_sort_newest                       — created_at descending
>   test_sort_rating                       — avg_rating descending
>   test_combined_filters                  — category + min_price + sort together
>   test_empty_results                     — returns empty list, not 404
>
> Product Detail test cases:
>   test_product_detail_by_slug            — full detail with variants in response
>   test_product_detail_inactive_returns_404
>   test_product_detail_invalid_slug_returns_404
>
> Category test cases:
>   test_category_tree_structure           — top-level items have children array
>   test_subcategory_nested_under_parent   — Sneakers inside Footwear's children
>   test_category_tree_cached             — second call hits cache
>
> **Step 2 — Write a complete Postman collection JSON** for these 10 requests:
>   1. GET /api/v1/products/
>   2. GET /api/v1/products/?category=footwear
>   3. GET /api/v1/products/?min_price=50&max_price=200&sort=price_asc
>   4. GET /api/v1/products/?search=jordan&rating=4
>   5. GET /api/v1/products/?page=2&page_size=6
>   6. GET /api/v1/products/{slug}/ with a real seeded slug
>   7. GET /api/v1/products/nonexistent-slug/ — expect 404
>   8. GET /api/v1/categories/
>   9. POST /api/v1/products/upload-image/ no auth — expect 401
>   10. POST /api/v1/products/upload-image/ with admin auth — expect 200
>
> Show all test file contents and the Postman collection JSON."

---

## DAY 5 — React: productAPI + TanStack Query Hooks + ProductCard Component

### Prompt:

> "Build the React frontend data layer and the core product display components.
>
> **Step 1 — Create src/api/productAPI.js** using the existing axiosInstance:
>   productAPI.list(filters)     GET /products/ — build query string from filters object,
>                                skip null/undefined/empty string values
>   productAPI.getBySlug(slug)   GET /products/{slug}/
>   productAPI.getCategories()   GET /categories/
>   productAPI.uploadImage(file) POST /products/upload-image/ using FormData
>   All functions return response.data directly.
>
> **Step 2 — Create src/hooks/useProducts.js** with TanStack Query hooks:
>   useProducts(filters):
>     queryKey: ['products', filters]
>     queryFn: productAPI.list(filters)
>     keepPreviousData: true  — prevents flash when changing pages or filters
>     staleTime: 5 * 60 * 1000  — 5 minutes, matches backend cache
>
>   useProduct(slug):
>     queryKey: ['product', slug]
>     queryFn: productAPI.getBySlug(slug)
>     enabled: !!slug
>
>   useCategories():
>     queryKey: ['categories']
>     queryFn: productAPI.getCategories()
>     staleTime: 60 * 60 * 1000  — 1 hour, matches backend cache
>
> **Step 3 — Create src/utils/formatters.js** with:
>   formatPrice(amount, currency='USD')  — returns '$120.00'
>   formatRating(rating)                 — returns '4.3' (1 decimal)
>   truncateText(text, maxLength=100)    — truncates with '...'
>   buildCloudinaryUrl(url, width=400)   — inserts w_{width},f_auto,q_auto into URL
>
> **Step 4 — Create src/components/product/StarRating.jsx:**
>   Props: rating (0-5 number), count (optional review count), size (sm/md/lg)
>   Renders filled, half, and empty stars using lucide-react
>   If count provided, shows it next to stars
>
> **Step 5 — Create src/components/product/ProductCard.jsx:**
>   Props: product (object from ProductListSerializer)
>   Shows: lazy-loaded image with explicit width/height, brand, name (2-line clamp),
>          StarRating, formatted price
>   Uses buildCloudinaryUrl(product.images[0], 400) for the thumbnail
>   Shows a gray placeholder div if no image exists
>   Entire card is a React Router Link to /products/{product.slug}
>   Hover: subtle shadow and scale via Tailwind transition classes
>   Shows 'Out of Stock' badge if every variant has stock === 0
>
> **Step 6 — Create src/components/product/ProductGrid.jsx:**
>   Props: products (array), isLoading (bool), skeletonCount (default 12)
>   isLoading=true: render skeletonCount animated gray pulse skeleton cards
>   isLoading=false: render responsive grid of ProductCards
>   Grid columns: 1 (mobile), 2 (tablet), 3 (desktop), 4 (large desktop)
>
> Show all complete file contents."

---

## DAY 6 — React: ProductList Page with Filters, Search & Pagination

### Prompt:

> "Build the full ProductList page — the core browsing experience of the store.
>
> **Step 1 — Create src/components/product/FilterSidebar.jsx:**
>   Reads current filter state from URL search params (useSearchParams)
>   Sections:
>     Categories: clickable list with subcategory expand/collapse, active item highlighted
>     Price Range: Min and Max inputs with an Apply button
>     Minimum Rating: 5 star options (1 star and above through 5 stars)
>     Clear All Filters button — deletes all params except page
>   On mobile: hidden by default, toggled with a Filters button
>
> **Step 2 — Create src/components/product/SortDropdown.jsx:**
>   Props: value (current sort), onChange (callback)
>   Options: Newest First (newest), Price Low to High (price_asc),
>            Price High to Low (price_desc), Highest Rated (rating), Most Popular (popular)
>
> **Step 3 — Create src/components/product/SearchBar.jsx:**
>   Controlled input reading/writing the search URL param
>   Uses useDebounce hook at 400ms — API is not called on every keystroke
>   Clear (x) button when input has text
>   Search icon (lucide-react) on the left side
>
> **Step 4 — Create src/components/ui/Pagination.jsx:**
>   Props: currentPage, totalPages, onPageChange
>   Shows Previous, up to 5 page number buttons with ellipsis, Next
>   Disables Previous on page 1, disables Next on last page
>   On page click: window.scrollTo({ top: 0, behavior: 'smooth' })
>
> **Step 5 — Build src/pages/ProductList.jsx:**
>   Read ALL filter state from URL search params — not component state:
>     category, min_price, max_price, rating, search, sort (default newest), page (default 1)
>   Use useProducts(filters) for data, useCategories() for sidebar
>   Layout: SearchBar at top, FilterSidebar left (toggled on mobile),
>           SortDropdown + result count right of sidebar, ProductGrid main area, Pagination bottom
>   Show 'No products found' + 'Clear filters' link when results array is empty
>   Show result count string: 'Showing 13-24 of 50 results'
>   All filter changes update URL params and reset page to 1:
>     setSearchParams updates the relevant key, sets page to 1, deletes key if value is empty
>
> **Step 6 — Update src/router.jsx:**
>   /products -> ProductList
>   /products/:slug -> placeholder div for now
>
> **Step 7 — Update the Navbar:**
>   Add a search input that navigates to /products?search={term} on Enter or button click.
>
> Show all complete file contents."

---

## DAY 7 — React: ProductDetail Page + Home Page + Week 2 Cleanup & Testing

### Prompt:

> "Build the Product Detail page and Home page, update the router, and run the full
> Week 2 verification checklist.
>
> **Step 1 — Install Swiper:**
>   npm install swiper
>
> **Step 2 — Build src/pages/ProductDetail.jsx:**
>
> Image Gallery (left column desktop, full width mobile):
>   Swiper with thumbnail navigation below the main image
>   If only 1 image, show it without Swiper controls
>   Images use buildCloudinaryUrl(url, 800) for full size
>
> Product Info (right column desktop):
>   Brand name in small muted text above the title
>   Product name as a large heading
>   StarRating component with review count
>   Price: show selectedVariant.price, or base_price if no variant selected
>   Color selector: row of clickable labeled color buttons
>   Size selector: row of clickable size buttons
>   Variant stock=0: show with strikethrough text + 'Out of stock', disable it
>   Quantity input: +/- buttons, min 1, max = selectedVariant.stock
>   Add to Cart button:
>     Shows 'Select options' and is disabled if no variant selected
>     Shows 'Out of Stock' and is disabled if selected variant stock = 0
>     For now: on click show toast 'Cart coming in Week 5!'
>   Add to Wishlist heart button: on click show toast 'Wishlist coming in Week 7!'
>
> Product Tabs (below the two columns):
>   Three tabs: Description | Reviews | Shipping Info
>   Description: render product.description
>   Reviews: placeholder text 'Reviews coming in Week 7'
>   Shipping Info: static text about shipping times and return policy
>
> Related Products (bottom):
>   useProducts({ category: product.category_id, page_size: 4 })
>   Filter out the current product from results
>   Render as a horizontal scrolling row of ProductCards with label 'You may also like'
>
> Loading state: skeleton placeholders
> Error state: 'Product not found' message with a back link
>
> **Step 3 — Build src/pages/Home.jsx:**
>   Hero section: full-width banner, headline, subheadline, 'Shop Now' button to /products
>   Categories section: responsive grid of category cards (image + name),
>     each links to /products?category={slug}, data from useCategories()
>   Featured Products: useProducts({ sort: 'rating', page_size: 8 }) in a ProductGrid
>   New Arrivals: useProducts({ sort: 'newest', page_size: 8 }) in a ProductGrid
>
> **Step 4 — Update src/router.jsx:**
>   / -> Home
>   /products -> ProductList
>   /products/:slug -> ProductDetail
>
> **Step 5 — Run and confirm the full Week 2 verification checklist:**
>
> Backend:
>   1. python manage.py seed_products — runs, creates 50 products and all categories
>   2. python manage.py test apps.products — all tests pass
>   3. GET /api/v1/products/ — paginated response with count, total_pages, results
>   4. GET /api/v1/products/?category=footwear — only footwear products
>   5. GET /api/v1/products/?search=nike — matching products returned
>   6. GET /api/v1/products/?min_price=50&max_price=200&sort=price_asc — filtered and sorted
>   7. GET /api/v1/products/?page=2&page_size=6 — correct page slice
>   8. GET /api/v1/products/{slug}/ — full product with variants
>   9. GET /api/v1/products/nonexistent/ — 404
>   10. GET /api/v1/categories/ — nested tree with children arrays
>   11. Product list response time target: under 500ms
>
> Frontend:
>   12. localhost:5173 — Home page loads with categories and product grids
>   13. Click category card — navigates to /products?category={slug}, filters apply
>   14. Shop Now — navigates to /products, all 50 products visible
>   15. Change sort dropdown — products reorder without page reload
>   16. Apply price filter — results update, URL params update
>   17. Type in search bar — debounced (400ms), results update
>   18. Click product card — navigates to /products/{slug}
>   19. Product Detail — image gallery renders, variants are selectable
>   20. Select variant — price updates to that variant's price
>   21. Select out-of-stock variant — Add to Cart disables with correct message
>   22. Click page 2 — page param updates, new products load, scroll to top
>   23. Refresh /products?category=footwear&sort=price_asc — filters persist from URL
>   24. Navbar search: type 'nike' and press Enter — navigates to /products?search=nike
>
> Fix any issues and confirm all 24 items pass.
>
> **Step 6 — Provide the Week 2 session handoff summary:**
>
>   ## Session Handoff
>
>   What was built:
>   Files created:
>   Files modified:
>   Decisions made:
>   Current working state:
>     Backend:
>     Frontend:
>   Next session starts with:"

---

## WEEK 2 SUMMARY — What You Will Have Built

By the end of Day 7, your project will have:

**Backend:**
- Product Document (with embedded Variant EmbeddedDocument) in mongoengine
- Category Document with parent/child tree structure
- All required MongoDB indexes: slug (unique), category+active compound, price, rating desc, text search on name+description+tags
- Product List API: pagination, filtering by category/price/rating/search, 5 sort options
- Product Detail API: full document with all variants
- Category Tree API: nested structure built in Python, cached for 1 hour
- Cloudinary integration: upload utility, image optimization URL builder, file type + 5MB validation
- Image upload endpoint protected by IsAdminOrSeller custom permission class
- Management command to seed 50 realistic products across 5+ categories
- Django APITestCase suite: 14 product tests + 3 category tests
- Postman collection covering all 10 catalog endpoint scenarios

**Frontend:**
- productAPI.js with all catalog API calls (list, detail, categories, image upload)
- useProducts, useProduct, useCategories TanStack Query hooks with correct staleTime values
- formatters.js: price, rating, text truncation, Cloudinary URL builder
- StarRating component with filled/half/empty stars and optional review count
- ProductCard: lazy image, Cloudinary thumbnail, out-of-stock badge, hover effect, full card link
- ProductGrid: responsive 4-column grid with animated skeleton loading state
- FilterSidebar: categories with expand/collapse, price range, rating filter, clear all, mobile toggle
- SortDropdown with all 5 sort options
- SearchBar with 400ms debounce and clear button
- Pagination with ellipsis, disabled states, and smooth scroll to top
- ProductList page: all filters stored in URL params, making them shareable and browser-history friendly
- ProductDetail page: Swiper gallery, variant selector with stock awareness, quantity input, tabs, related products
- Home page: hero banner, category grid, featured products, new arrivals
- Router: /, /products, /products/:slug all wired

**Week 3 continues with:** Cart system — guest cart via session cookie, logged-in user cart,
cart merge on login, Add to Cart wired from ProductDetail, cart sidebar drawer, and full cart page.
