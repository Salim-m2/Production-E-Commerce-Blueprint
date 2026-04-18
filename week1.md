# Week 1 AI Implementation Prompt — Foundation: Project Setup + Auth
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## CONTEXT TO GIVE THE AI BEFORE STARTING

> "I am building a production e-commerce platform using the following stack:
> - **Backend**: Django 4.x + Django REST Framework + mongoengine (MongoDB ODM)
> - **Database**: MongoDB Atlas
> - **Frontend**: React 18 + Vite + Redux Toolkit + TanStack Query + Axios
> - **Auth**: JWT stored in httpOnly cookies (NOT localStorage) using djangorestframework-simplejwt
> - **Background tasks**: Celery + Redis (set up later in Week 8)
> - **Deployment target**: Docker + Nginx + VPS (set up in Week 12)
>
> The project folder structure will be:
> ```
> ecommerce/
> ├── backend/       ← Django project lives here
> └── frontend/      ← React/Vite project lives here
> ```
> Help me implement exactly what I describe below. Write complete, working code — no placeholders."

---

## DAY 1 — Django Project Scaffolding + Split Settings + MongoDB Connection

### Prompt:

> "Set up the Django backend project from scratch. Do the following steps in order:
>
> **Step 1 — Create the project structure:**
> ```
> backend/
> ├── config/
> │   ├── __init__.py
> │   ├── settings/
> │   │   ├── __init__.py
> │   │   ├── base.py       ← shared settings
> │   │   ├── dev.py        ← development overrides
> │   │   └── prod.py       ← production overrides
> │   ├── urls.py
> │   ├── wsgi.py
> │   └── celery.py         ← placeholder, we'll fill this in Week 8
> ├── apps/
> │   ├── authentication/
> │   ├── users/
> │   └── core/             ← shared: custom permissions, pagination, utils
> ├── manage.py
> ├── requirements.txt
> └── .env.example
> ```
>
> **Step 2 — Install these packages and produce requirements.txt:**
> ```
> django>=4.2
> djangorestframework
> djangorestframework-simplejwt
> mongoengine
> django-environ
> django-cors-headers
> Pillow
> gunicorn
> celery
> redis
> ```
>
> **Step 3 — Write `config/settings/base.py` with:**
> - `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` loaded via django-environ from a `.env` file
> - `INSTALLED_APPS` including `rest_framework`, `corsheaders`, and all apps under `apps/`
> - `CORS_ALLOWED_ORIGINS` set from env variable
> - REST_FRAMEWORK config with:
>   - Default authentication: `JWTAuthentication`
>   - Default permission: `IsAuthenticatedOrReadOnly`
>   - Default throttle: AnonRateThrottle (100/hour), UserRateThrottle (1000/hour), and a tight 'login' throttle (10/minute)
> - mongoengine connection block using `MONGO_URI` and `MONGO_DB` from env:
>   ```python
>   import mongoengine
>   mongoengine.connect(db=env('MONGO_DB'), host=env('MONGO_URI'), alias='default')
>   ```
> - Do NOT configure Django's built-in `DATABASES` dict for SQL — we are using mongoengine exclusively.
> - Simple JWT settings:
>   - Access token lifetime: 15 minutes
>   - Refresh token lifetime: 7 days
>   - `JWT_AUTH_COOKIE = 'access_token'`
>   - `JWT_AUTH_REFRESH_COOKIE = 'refresh_token'`
>   - `JWT_AUTH_COOKIE_HTTP_ONLY = True`
>   - `JWT_AUTH_COOKIE_SAMESITE = 'Lax'`
>   - `JWT_AUTH_COOKIE_SECURE = False` (dev only; True in prod)
>
> **Step 4 — Write `config/settings/dev.py`** that imports everything from base and sets `DEBUG=True`.
>
> **Step 5 — Write `config/settings/prod.py`** that imports from base and sets `DEBUG=False`, `JWT_AUTH_COOKIE_SECURE=True`.
>
> **Step 6 — Write `.env.example`** with all required variables:
> ```
> SECRET_KEY=
> DEBUG=True
> MONGO_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net/
> MONGO_DB=ecommerce_db
> CORS_ALLOWED_ORIGINS=http://localhost:5173
> ```
>
> **Step 7 — Write `config/urls.py`** with a base router that prefixes all routes under `/api/v1/`. Wire in `apps.authentication.urls` and `apps.users.urls` as placeholders.
>
> After writing all files, show me how to run `python manage.py check` successfully and confirm the mongoengine connection works."

---

## DAY 2 — User Document (mongoengine) + Custom Auth Backend

### Prompt:

> "Now build the User model and authentication foundation. We are NOT using Django's built-in User model — we use mongoengine Documents instead.
>
> **Step 1 — Create `apps/users/documents.py`** with a mongoengine `User` Document that has these exact fields:
> ```python
> email         # StringField, unique, required — this is the login identifier
> password_hash # StringField (we store bcrypt hash, not plain text)
> role          # StringField, choices=['customer', 'seller', 'admin'], default='customer'
> first_name    # StringField, required
> last_name     # StringField, required
> phone         # StringField, optional
> avatar_url    # StringField, optional
> is_verified   # BooleanField, default=False (email verification)
> is_active     # BooleanField, default=True
> addresses     # ListField of EmbeddedDocumentField (define an Address EmbeddedDocument with: label, street, city, country, is_default)
> created_at    # DateTimeField, default=datetime.utcnow
> last_login    # DateTimeField, optional
> ```
> Add a method `set_password(raw_password)` that hashes using Django's `make_password`.
> Add a method `check_password(raw_password)` that verifies using Django's `check_password`.
>
> **Step 2 — Create `apps/authentication/backends.py`** with a custom authentication backend that:
> - Looks up users by email using mongoengine
> - Verifies password with `check_password`
> - Returns the user object or None
>
> **Step 3 — Create a custom JWT token utility in `apps/authentication/utils.py`** that:
> - Generates an access token and refresh token for a given user (using SimpleJWT's `RefreshToken`)
> - Sets both tokens as httpOnly cookies on the Django response object
> - Access cookie: name=`access_token`, httponly=True, samesite=`Lax`, max_age=900 (15 min)
> - Refresh cookie: name=`refresh_token`, httponly=True, samesite=`Lax`, max_age=604800 (7 days)
>
> **Step 4 — Create `apps/authentication/serializers.py`** with:
> - `RegisterSerializer`: validates email (unique check against MongoDB), password (min 8 chars), first_name, last_name
> - `LoginSerializer`: validates email + password fields
> - `UserSerializer`: safe read-only representation of the user (id, email, role, first_name, last_name, avatar_url, is_verified)
>
> Show me the complete code for all files."

---

## DAY 3 — Auth API Endpoints (Register, Login, Logout, Refresh)

### Prompt:

> "Implement the authentication API views and wire up all URLs. These are the exact endpoints we need:
>
> | Method | Endpoint | Description | Auth Required |
> |--------|----------|-------------|---------------|
> | POST | /api/v1/auth/register/ | Register new user | No |
> | POST | /api/v1/auth/login/ | Login, set httpOnly JWT cookies | No |
> | POST | /api/v1/auth/token/refresh/ | Silently refresh access token from refresh cookie | No |
> | POST | /api/v1/auth/logout/ | Clear both JWT cookies server-side | Yes |
>
> **For each view, write the full implementation:**
>
> `RegisterView` (APIView):
> - Validate with RegisterSerializer
> - Check email is not already taken (query MongoDB via mongoengine)
> - Create user with `set_password()` — never store raw passwords
> - Return 201 with user data; do NOT set cookies yet (user must verify email first — but for now just return the user data and we'll add email verification in Week 8)
>
> `LoginView` (APIView):
> - Validate email + password
> - Use the custom auth backend to verify credentials
> - Check `user.is_active` is True
> - On success: call the token utility to set httpOnly cookies on the response
> - Update `user.last_login = datetime.utcnow()` and save
> - Return 200 with user data (id, email, role, first_name, last_name)
>
> `TokenRefreshView` (APIView):
> - Read the refresh token from `request.COOKIES.get('refresh_token')`
> - If missing or invalid, return 401
> - Generate new access token using SimpleJWT
> - Set new access cookie on the response
> - Return 200
>
> `LogoutView` (APIView, IsAuthenticated):
> - Delete both cookies from the response (set them with max_age=0)
> - Return 200
>
> **Wire everything in `apps/authentication/urls.py`** and include in `config/urls.py`.
>
> **Apply the tight login throttle** to LoginView using DRF's throttle_classes.
>
> After writing all code, show me the exact Postman requests to test all 4 endpoints, including what response body and cookies to expect."

---

## DAY 4 — Tokens Collection (Email Verify + Password Reset Groundwork) + Postman Testing

### Prompt:

> "Add two more things today:
>
> **Part 1 — Token Document for password reset and email verification:**
> Create `apps/authentication/documents.py` with a mongoengine `Token` Document:
> ```
> user_id    # ObjectIdField (reference to User._id)
> token      # StringField (store a hashed version of the token, not plain)
> type       # StringField, choices=['email_verify', 'password_reset']
> expires_at # DateTimeField
> created_at # DateTimeField, default=datetime.utcnow
> ```
> Add a MongoDB TTL index on `expires_at` so expired tokens auto-delete.
> Add a classmethod `create_for_user(user, token_type, hours_valid)` that:
> - Generates a secure random token using `secrets.token_urlsafe(32)`
> - Stores a hashed version (using `hashlib.sha256`) in MongoDB
> - Returns the plain token (to be sent in email)
>
> **Part 2 — Password reset endpoints (skeleton, emails come in Week 8):**
> Add these two endpoints:
>
> `POST /api/v1/auth/password/reset/` — accepts `{email}`, creates a Token document, for now just returns `{detail: 'If this email exists, a reset link will be sent'}` (we add actual email sending in Week 8).
>
> `POST /api/v1/auth/password/reset/confirm/` — accepts `{token, new_password}`, finds matching token by hashing and comparing, checks expiry, updates the user's password with `set_password()`, deletes the Token document, returns 200.
>
> **Part 3 — Write a complete Postman collection** (as a JSON file I can import) that tests:
> 1. Register a new user → expect 201
> 2. Try to register with same email → expect 400
> 3. Login with correct credentials → expect 200 + cookies set
> 4. Login with wrong password → expect 401
> 5. Call a protected endpoint (logout) without cookie → expect 401
> 6. Logout with cookie → expect 200 + cookies cleared
> 7. Attempt token refresh with no cookie → expect 401
>
> Show the Postman collection JSON and all view code."

---

## DAY 5 — React + Vite Frontend Setup + Axios + Redux Auth Slice

### Prompt:

> "Now set up the React frontend. Do everything below:
>
> **Step 1 — Scaffold the Vite project:**
> ```bash
> npm create vite@latest storefront -- --template react
> cd storefront
> npm install react-router-dom @reduxjs/toolkit react-redux \
>   axios @tanstack/react-query \
>   react-hook-form zod @hookform/resolvers \
>   lucide-react react-hot-toast
> ```
>
> **Step 2 — Create this exact folder structure inside `src/`:**
> ```
> src/
> ├── api/
> │   ├── axiosInstance.js
> │   └── authAPI.js
> ├── components/
> │   ├── layout/
> │   │   ├── Navbar.jsx
> │   │   └── Footer.jsx
> │   └── ui/
> │       ├── Button.jsx
> │       ├── Spinner.jsx
> │       └── Input.jsx
> ├── pages/
> │   └── auth/
> │       ├── Login.jsx
> │       └── Register.jsx
> ├── store/
> │   ├── index.js
> │   └── authSlice.js
> ├── hooks/
> │   └── useAuth.js
> ├── router.jsx
> └── main.jsx
> ```
>
> **Step 3 — Write `src/api/axiosInstance.js`:**
> ```javascript
> const api = axios.create({
>   baseURL: '/api/v1',
>   withCredentials: true,  // sends httpOnly cookies automatically
> });
> // Response interceptor: on 401, attempt silent token refresh once, then retry
> // If refresh also fails, dispatch logout action and redirect to /login
> ```
> Implement the full interceptor logic including the retry-once pattern using `_retry` flag.
>
> **Step 4 — Write `src/api/authAPI.js`** with these functions using the axios instance:
> - `register(data)` → POST /auth/register/
> - `login(data)` → POST /auth/login/
> - `logout()` → POST /auth/logout/
> - `refreshToken()` → POST /auth/token/refresh/
>
> **Step 5 — Write `src/store/authSlice.js`** (Redux Toolkit) with:
> - State: `{ user: null, isAuthenticated: false, isLoading: false, error: null }`
> - Async thunks: `loginUser(credentials)`, `registerUser(data)`, `logoutUser()`
> - Reducers: handle pending/fulfilled/rejected for each thunk
> - Selector: `selectCurrentUser`, `selectIsAuthenticated`
>
> **Step 6 — Write `src/store/index.js`** combining reducers.
>
> **Step 7 — Write `src/hooks/useAuth.js`** that returns `{ user, isAuthenticated, login, logout, register }` from the Redux store + dispatches.
>
> Show all complete file contents."

---

## DAY 6 — Login & Register Pages + React Router + Protected Routes

### Prompt:

> "Build the Login and Register pages with form validation and connect them to the Redux auth slice.
>
> **Step 1 — Write `src/pages/auth/Login.jsx`:**
> - Use `react-hook-form` + `zod` for validation
> - Zod schema: `{ email: z.string().email(), password: z.string().min(1) }`
> - On submit: dispatch `loginUser` thunk
> - Show a spinner while `isLoading` is true
> - Show API error message if the thunk rejects
> - On success: navigate to `/` using useNavigate
> - Include a 'Don't have an account? Register' link to `/register`
>
> **Step 2 — Write `src/pages/auth/Register.jsx`:**
> - Zod schema: `{ first_name, last_name, email, password (min 8), confirm_password (must match) }`
> - On submit: dispatch `registerUser` thunk
> - On success: show a success toast ('Account created! Please log in') and navigate to `/login`
> - Include a 'Already have an account? Login' link
>
> **Step 3 — Write `src/components/layout/Navbar.jsx`:**
> - Show the site name/logo on the left
> - If authenticated: show user's first name + a Logout button
> - If not authenticated: show Login and Register links
> - Use `useAuth` hook to read state
>
> **Step 4 — Write `src/router.jsx`** with React Router v6:
> - `/` → Home (just a placeholder for now: `<h1>Home</h1>`)
> - `/login` → Login page
> - `/register` → Register page
> - Wrap all routes in a Layout that renders `<Navbar />` + `<Outlet />`
> - Add a `ProtectedRoute` wrapper component that redirects to `/login` if not authenticated
>
> **Step 5 — Update `src/main.jsx`** to:
> - Wrap the app in `<Provider store={store}>` (Redux)
> - Wrap in `<QueryClientProvider>` (TanStack Query)
> - Wrap in `<BrowserRouter>` (React Router)
> - Add `<Toaster />` from react-hot-toast
>
> **Step 6 — Configure Vite's dev proxy** in `vite.config.js` so that `/api` requests proxy to `http://localhost:8000`:
> ```javascript
> server: {
>   proxy: {
>     '/api': 'http://localhost:8000'
>   }
> }
> ```
> This means Axios calls to `/api/v1/...` in React will hit Django during development.
>
> Show all complete file contents."

---

## DAY 7 — Integration Testing + Auth Persistence + Week 1 Cleanup

### Prompt:

> "This is the final day of Week 1. We need to wire everything together, handle auth persistence on page reload, and verify everything works end-to-end.
>
> **Step 1 — Handle auth persistence on page refresh:**
> Add a `GET /api/v1/auth/me/` endpoint to Django that:
> - Requires authentication (reads the httpOnly access cookie automatically)
> - Returns the current user's data: `{ id, email, role, first_name, last_name, avatar_url, is_verified }`
> - Returns 401 if no valid cookie
>
> In React, add an `initializeAuth` async thunk in `authSlice.js` that calls `GET /auth/me/` on app startup. Dispatch it in `src/main.jsx` or `App.jsx` on component mount. This ensures that if a user refreshes the page, they remain logged in as long as their access cookie is valid.
>
> **Step 2 — Add a `uiSlice.js` to Redux store** with:
> - State: `{ cartDrawerOpen: false }`
> - Actions: `openCartDrawer()`, `closeCartDrawer()`, `toggleCartDrawer()`
> (We'll use this when we build the cart in Week 5.)
>
> **Step 3 — Write Django API tests** using DRF's `APITestCase`:
> Create `apps/authentication/tests.py` with test cases for:
> - `test_register_success` → POST /auth/register/ with valid data, expect 201
> - `test_register_duplicate_email` → Register same email twice, expect 400 on second
> - `test_login_success` → Login with valid creds, expect 200 + cookies in response
> - `test_login_wrong_password` → Expect 401
> - `test_logout_clears_cookies` → Login then logout, verify cookie is cleared
> - `test_refresh_token` → Login, then call /auth/token/refresh/, expect new access cookie
> - `test_me_endpoint_authenticated` → Call /auth/me/ with valid session, expect user data
> - `test_me_endpoint_unauthenticated` → Call /auth/me/ without cookies, expect 401
>
> **Step 4 — CORS final check:**
> Verify `django-cors-headers` is correctly configured in `base.py`:
> ```python
> CORS_ALLOW_CREDENTIALS = True  # Required for cookies to be sent cross-origin in dev
> CORS_ALLOWED_ORIGINS = ['http://localhost:5173']
> ```
>
> **Step 5 — Run the full verification checklist:**
> Help me confirm each of the following works:
> 1. `python manage.py check` passes with no errors
> 2. Django server starts: `python manage.py runserver --settings=config.settings.dev`
> 3. React dev server starts: `npm run dev`
> 4. Postman: Register a new user → 201
> 5. Postman: Login → 200 + see `access_token` and `refresh_token` cookies
> 6. Postman: Call `/auth/me/` with cookies → see user data
> 7. Postman: Refresh token → new access cookie issued
> 8. Postman: Logout → cookies cleared
> 9. Browser: Open http://localhost:5173/register → fill form → success toast → redirected to login
> 10. Browser: Login with those credentials → redirected to home, Navbar shows user's name
> 11. Browser: Refresh the page → user is still logged in (auth persistence working)
> 12. Run Django tests: `python manage.py test apps.authentication --settings=config.settings.dev`
>
> Fix any issues you find and confirm the full checklist passes."

---

## WEEK 1 SUMMARY — What You Will Have Built

By the end of Day 7, your project will have:

**Backend:**
- Django project with split settings (dev/prod), loaded via django-environ
- MongoDB Atlas connected via mongoengine
- User Document with secure password hashing
- Full JWT auth in httpOnly cookies (register, login, logout, refresh, /me)
- Password reset skeleton (emails added in Week 8)
- Token Document with TTL auto-expiry
- DRF throttling on all endpoints, tightest on login
- Django API tests for all auth endpoints

**Frontend:**
- Vite + React 18 project with full folder structure
- Axios instance with automatic silent token refresh interceptor
- Redux Toolkit store with authSlice and uiSlice
- TanStack Query client wired up
- Login and Register pages with Zod validation and loading states
- React Router v6 with Layout, ProtectedRoute, and all auth routes
- Navbar that reflects auth state
- Auth persistence on page reload via `/auth/me/` on startup

**Week 2 continues with:** Product Catalog, Category models, seeding 50 test products, and building the Home and Product List pages.
