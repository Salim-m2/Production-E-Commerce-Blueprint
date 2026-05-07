# Week 5 Handoff Summary + Week 6 Implementation Prompt — Celery + Email System
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## WEEK 5 HANDOFF SUMMARY

### What Was Built

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
- `OrderListSerializer` updated with `item_product_ids` for frontend hasPurchased check
- Full Django test suites for reviews and wishlist
- `config/settings/test.py` updated to use mongomock (no Atlas connection in tests)

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
- `ProductDetail.jsx` updated — ReviewSection in Reviews tab, WishlistButton next to Add to Cart
- `Wishlist.jsx` page — grid with instant Redux-filtered removal, clear all, empty state
- `Profile.jsx` page — 3-tab layout: personal info + avatar upload, change password, addresses CRUD
- `wishlistSlice.js` — Redux store for wishlist IDs, fetchWishlist + toggleWishlist thunks
- Navbar — wishlist heart icon with red count badge, profile link active
- Router — /wishlist and /profile protected routes added
- 6 bug fixes: Navbar badge, hasPurchased check, auto-login after logout, addToCart import,
  rating summary visibility, small-screen overflow

### Files Created (Week 5)
**Backend:** `apps/reviews/documents.py`, `apps/reviews/utils.py`, `apps/reviews/serializers.py`,
`apps/reviews/views.py`, `apps/reviews/urls.py`, `apps/reviews/tests.py`,
`apps/wishlist/documents.py`, `apps/wishlist/serializers.py`, `apps/wishlist/views.py`,
`apps/wishlist/urls.py`, `apps/wishlist/tests.py`, `apps/users/serializers.py`,
`apps/users/views.py`, `apps/users/urls.py`

**Frontend:** `src/api/reviewAPI.js`, `src/api/wishlistAPI.js`, `src/api/userAPI.js`,
`src/hooks/useReviews.js`, `src/hooks/useProfile.js`, `src/hooks/useWishlist.js`,
`src/store/wishlistSlice.js`, `src/components/review/StarInput.jsx`,
`src/components/review/ReviewCard.jsx`, `src/components/review/ReviewForm.jsx`,
`src/components/review/ReviewSection.jsx`, `src/components/product/WishlistButton.jsx`,
`src/pages/user/Wishlist.jsx`, `src/pages/user/Profile.jsx`

### Files Modified (Week 5)
**Backend:** `apps/orders/serializers.py` (item_product_ids), `config/settings/test.py` (mongomock),
`config/urls.py` (users prefix fix), `config/settings/base.py` (reviews + wishlist apps)

**Frontend:** `src/api/axiosInstance.js` (skip refresh on /auth/me/), `src/store/authSlice.js`
(setUser action, clearWishlist on logout), `src/store/index.js` (wishlistReducer),
`src/main.jsx` (fetchWishlist on startup), `src/router.jsx` (/wishlist + /profile routes),
`src/components/layout/Navbar.jsx` (wishlist badge), `src/components/product/ProductCard.jsx`
(WishlistButton overlay), `src/pages/ProductDetail.jsx` (ReviewSection + WishlistButton + fixes)

### Key Decisions Made
- Wishlist IDs stored in Redux (for fast per-card lookups), full data in TanStack Query (for the page)
- hasPurchased check uses `item_product_ids` from OrderListSerializer — no N+1 fetches
- mongomock replaces Atlas for all tests — no network dependency, tests run in seconds
- axiosInstance skips silent refresh for `/auth/me/` to prevent auto-login after logout
- Review rating breakdown bars calculated from fetched page (not all reviews — acceptable tradeoff)

### Current Working State
- Backend: Django on port 8000, all test suites passing with mongomock
- Frontend: React on port 5173, all Week 5 features working end-to-end
- MongoDB Atlas: reviews, wishlists, user profiles populated from testing

### Next Session Starts With
Week 6, Day 1 — Celery + Redis setup and configuration

---

---

# Week 6 AI Implementation Prompt — Celery + Email System
### Based on: Production E-Commerce Blueprint (Django REST + React + MongoDB)

---

## CONTEXT TO GIVE THE AI AT THE START OF EVERY SESSION THIS WEEK

> "I am continuing a production e-commerce build. Weeks 1 through 5 are fully complete and tested.
>
> **What already exists and is working:**
> - Django 4.x + DRF + mongoengine (NO Django ORM, NO SQL)
> - MongoDB Atlas connected via mongoengine
> - Full JWT auth in httpOnly cookies (register, login, logout, refresh, /auth/me/)
> - Password reset SKELETON already exists — `Token` Document with TTL, endpoints
>   `POST /auth/password/reset/` and `POST /auth/password/reset/confirm/` — but
>   they do NOT send emails yet. Week 6 Day 3 will wire the actual email sending.
> - Email verification SKELETON: User has `is_verified` BooleanField (default False).
>   Registration sets it to False. Week 6 Day 3 will send the verification email.
> - Product catalog, cart, checkout, Stripe/IntaSend payments, orders
> - Reviews, wishlist, user profile (Week 5)
> - React 18 + Vite frontend, Redux Toolkit, TanStack Query, Axios
> - All API endpoints prefixed /api/v1/
> - JWT in httpOnly cookies ONLY — never localStorage
> - config/settings/ has base.py, dev.py, prod.py, test.py (mongomock)
> - `config/celery.py` exists as a PLACEHOLDER from Week 1 — it is empty/stub.
>   Week 6 Day 1 will fill it in completely.
>
> **Stack reminder:**
> - Background tasks: Celery + Redis
> - Email provider: SendGrid (via Django SMTP backend)
> - Email templates: HTML with inline CSS (table-based for email client compatibility)
> - Celery Beat: for periodic tasks (low stock alerts)
> - Styling: Tailwind CSS (frontend)
>
> **Critical rules for Week 6:**
> - Celery tasks MUST use `bind=True` and `max_retries=3` with `countdown=60`
>   so failed email sends retry automatically after 1 minute
> - NEVER call `.send()` or any email function directly from a view —
>   always dispatch a Celery task and return immediately
> - Email templates use inline CSS and table-based layout (not Tailwind) for
>   compatibility with Gmail, Outlook, and Apple Mail
> - The Stripe/IntaSend webhook handler already exists and marks orders as paid.
>   Week 6 will add `.delay()` calls inside that handler to trigger emails.
> - Celery Beat runs as a SEPARATE process from the Celery worker.
>   Both run alongside Django — three processes total in development.
> - All Celery tasks must be idempotent — running them twice must not send
>   two emails. Use a simple flag on the model or check task state.
>
> Today we are working on: [paste the specific day below]"

---

## DAY 1 — Celery + Redis Setup + Worker Verification

### Prompt:

> "Set up Celery and Redis from scratch. The `config/celery.py` file currently
> exists as a placeholder stub. Replace it completely and configure everything.
>
> **Step 1 — Install required packages:**
> ```
> celery>=5.3
> redis>=5.0
> django-celery-beat
> django-celery-results
> flower          ← Celery monitoring dashboard (dev only)
> ```
> Add all to requirements.txt.
>
> **Step 2 — Write `config/celery.py` (replace the stub completely):**
> ```python
> import os
> from celery import Celery
>
> # Tell Celery which Django settings module to use.
> # We set this here so Celery workers can bootstrap Django correctly
> # when started as separate processes (not through manage.py).
> os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.dev')
>
> app = Celery('ecommerce')
>
> # Read all CELERY_* settings from Django settings automatically.
> # namespace='CELERY' means only settings starting with CELERY_ are read.
> app.config_from_object('django.conf:settings', namespace='CELERY')
>
> # Auto-discover tasks in all INSTALLED_APPS.
> # Celery looks for a tasks.py file in each app directory.
> app.autodiscover_tasks()
>
> @app.task(bind=True, ignore_result=True)
> def debug_task(self):
>     print(f'Request: {self.request!r}')
> ```
>
> **Step 3 — Update `config/__init__.py`:**
> ```python
> # This makes the Celery app available when Django starts,
> # so @shared_task decorators in any app can find it.
> from .celery import app as celery_app
> __all__ = ('celery_app',)
> ```
>
> **Step 4 — Add Celery settings to `config/settings/base.py`:**
> ```python
> # ── Celery ──────────────────────────────────────────────────────────
> # Redis is used as BOTH the message broker (task queue) and the result
> # backend (stores task return values). Using the same Redis instance
> # for both is fine in development. In production use separate DBs:
> # redis://redis:6379/0 for broker, redis://redis:6379/1 for results.
>
> CELERY_BROKER_URL         = env('REDIS_URL', default='redis://localhost:6379/0')
> CELERY_RESULT_BACKEND     = env('REDIS_URL', default='redis://localhost:6379/0')
> CELERY_ACCEPT_CONTENT     = ['json']
> CELERY_TASK_SERIALIZER    = 'json'
> CELERY_RESULT_SERIALIZER  = 'json'
> CELERY_TIMEZONE           = 'Africa/Nairobi'
>
> # Store task results in Django's DB so we can query them from the admin
> CELERY_RESULT_EXTENDED    = True
>
> # Beat scheduler stores the schedule in the database (via django-celery-beat)
> # so schedules survive server restarts and can be edited in the admin.
> CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
>
> # Maximum time a task is allowed to run before it is killed.
> # Prevents runaway email tasks from blocking the worker indefinitely.
> CELERY_TASK_SOFT_TIME_LIMIT = 300   # 5 minutes
> CELERY_TASK_TIME_LIMIT      = 600   # 10 minutes hard limit
> ```
>
> **Step 5 — Add `django_celery_beat` and `django_celery_results` to INSTALLED_APPS
> in `config/settings/base.py`:**
> ```python
> INSTALLED_APPS = [
>     ...
>     'django_celery_beat',
>     'django_celery_results',
>     ...
> ]
> ```
>
> **Step 6 — Add `REDIS_URL` to `.env.example`:**
> ```
> REDIS_URL=redis://localhost:6379/0
> ```
>
> **Step 7 — Run Django migrations** (django-celery-beat creates its own tables):
> ```bash
> python manage.py migrate --settings=config.settings.dev
> ```
>
> **Step 8 — Write a smoke-test task in `apps/core/tasks.py`:**
> ```python
> from celery import shared_task
>
> @shared_task(bind=True, max_retries=3)
> def smoke_test(self):
>     '''Simple task to verify Celery worker is running and connected.'''
>     print('Celery smoke test: worker is alive!')
>     return 'ok'
> ```
>
> **Step 9 — Verify the full setup:**
>
> Open THREE terminal windows. In each run:
>
> Terminal 1 — Django dev server:
> ```bash
> python manage.py runserver --settings=config.settings.dev
> ```
>
> Terminal 2 — Celery worker:
> ```bash
> celery -A config worker --loglevel=info
> ```
> You should see: `[tasks]` lists `apps.core.tasks.smoke_test` and `config.celery.debug_task`.
> You should see: `[2024-xx-xx] celery@hostname ready.`
>
> Terminal 3 — Send the smoke test task:
> ```bash
> python manage.py shell --settings=config.settings.dev
> ```
> Then inside the shell:
> ```python
> from apps.core.tasks import smoke_test
> result = smoke_test.delay()
> print('Task ID:', result.id)
> print('Task status:', result.status)
> import time; time.sleep(2)
> print('Task result:', result.result)
> ```
>
> In Terminal 2 (worker), you should see:
> `Celery smoke test: worker is alive!`
>
> **Step 10 — Start Flower (optional but useful for monitoring):**
> ```bash
> celery -A config flower --port=5555
> ```
> Open http://localhost:5555 — you should see the worker and the completed smoke_test task.
>
> Show all complete file contents and confirm the three-terminal setup works."

---

## DAY 2 — SendGrid + Django Email Backend + Base HTML Email Template

### Prompt:

> "Set up SendGrid as the email provider and build the base HTML email template
> that all email types will extend.
>
> **Step 1 — Install SendGrid:**
> ```bash
> pip install sendgrid
> ```
> Add to requirements.txt.
>
> **Step 2 — Add email settings to `config/settings/base.py`:**
> ```python
> # ── Email (SendGrid via SMTP) ────────────────────────────────────────
> # We use Django's built-in SMTP backend pointed at SendGrid's SMTP relay.
> # Why SMTP instead of SendGrid's API directly?
> #   Django's send_mail() and EmailMessage work out of the box with SMTP.
> #   No extra SDK calls needed in tasks — just standard Django email code.
>
> EMAIL_BACKEND       = 'django.core.mail.backends.smtp.EmailBackend'
> EMAIL_HOST          = 'smtp.sendgrid.net'
> EMAIL_PORT          = 587
> EMAIL_USE_TLS       = True
> EMAIL_HOST_USER     = 'apikey'                          # SendGrid requires this exact string
> EMAIL_HOST_PASSWORD = env('SENDGRID_API_KEY')           # your actual API key
> DEFAULT_FROM_EMAIL  = env('DEFAULT_FROM_EMAIL', default='noreply@yourdomain.com')
> FRONTEND_URL        = env('FRONTEND_URL', default='http://localhost:5173')
> ```
>
> **Step 3 — Add to `.env` and `.env.example`:**
> ```
> SENDGRID_API_KEY=SG.xxxxxxxxxxxx
> DEFAULT_FROM_EMAIL=orders@yourdomain.com
> FRONTEND_URL=http://localhost:5173
> ```
>
> **Step 4 — Create a development email backend override in `config/settings/dev.py`:**
> ```python
> from config.settings.base import *
>
> DEBUG = True
>
> # In development, print emails to the console instead of sending them.
> # Switch to the real SMTP backend when you are ready to test with SendGrid.
> # To test real sending: comment out this line and set SENDGRID_API_KEY in .env
> EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
> ```
>
> **Step 5 — Create `backend/templates/emails/base.html`:**
>
> This is the master template all emails extend. It uses table-based layout
> and inline CSS because email clients (especially Outlook) do not support
> modern CSS or external stylesheets.
>
> ```html
> <!DOCTYPE html>
> <html lang="en">
> <head>
>   <meta charset="UTF-8"/>
>   <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
>   <title>{% block subject %}Email{% endblock %}</title>
>   <style>
>     body { margin: 0; padding: 0; background-color: #f1f5f9; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; }
>     .wrapper { width: 100%; background-color: #f1f5f9; padding: 32px 0; }
>     .container { max-width: 600px; margin: 0 auto; background-color: #ffffff; border-radius: 12px; overflow: hidden; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
>     .header { background-color: #0f172a; padding: 24px 32px; text-align: center; }
>     .header-logo { color: #a78bfa; font-size: 22px; font-weight: 700; text-decoration: none; letter-spacing: -0.5px; }
>     .body { padding: 32px; }
>     .h1 { color: #0f172a; font-size: 24px; font-weight: 700; margin: 0 0 16px 0; }
>     .p { color: #475569; font-size: 15px; line-height: 1.6; margin: 0 0 16px 0; }
>     .btn { display: inline-block; background-color: #7c3aed; color: #ffffff !important; text-decoration: none; padding: 12px 28px; border-radius: 8px; font-size: 15px; font-weight: 600; margin: 8px 0; }
>     .divider { border: none; border-top: 1px solid #e2e8f0; margin: 24px 0; }
>     .footer { background-color: #f8fafc; padding: 20px 32px; text-align: center; }
>     .footer p { color: #94a3b8; font-size: 12px; margin: 0; line-height: 1.6; }
>     .order-table { width: 100%; border-collapse: collapse; margin: 16px 0; }
>     .order-table th { background-color: #f8fafc; color: #64748b; font-size: 12px; text-transform: uppercase; letter-spacing: 0.05em; padding: 10px 12px; text-align: left; border-bottom: 1px solid #e2e8f0; }
>     .order-table td { padding: 10px 12px; color: #374151; font-size: 14px; border-bottom: 1px solid #f1f5f9; vertical-align: top; }
>     .badge { display: inline-block; padding: 4px 10px; border-radius: 20px; font-size: 12px; font-weight: 600; }
>     .badge-green { background-color: #dcfce7; color: #166534; }
>     .badge-blue  { background-color: #dbeafe; color: #1e40af; }
>     .badge-violet { background-color: #ede9fe; color: #6d28d9; }
>     .summary-row { display: flex; justify-content: space-between; padding: 6px 0; color: #374151; font-size: 14px; }
>     .summary-total { font-weight: 700; font-size: 16px; color: #0f172a; border-top: 2px solid #0f172a; padding-top: 10px; margin-top: 4px; }
>   </style>
> </head>
> <body>
>   <div class="wrapper">
>     <div class="container">
>       <!-- Header -->
>       <div class="header">
>         <a href="{{ frontend_url }}" class="header-logo">⚡ Your Store</a>
>       </div>
>
>       <!-- Body -->
>       <div class="body">
>         {% block content %}{% endblock %}
>       </div>
>
>       <hr class="divider">
>
>       <!-- Footer -->
>       <div class="footer">
>         <p>© 2024 Your Store. All rights reserved.</p>
>         <p style="margin-top:6px;">
>           You received this email because you have an account with us.<br/>
>           <a href="{{ frontend_url }}" style="color: #7c3aed;">Visit our store</a>
>         </p>
>       </div>
>     </div>
>   </div>
> </body>
> </html>
> ```
>
> **Step 6 — Configure Django template loader to find the templates directory:**
>
> In `config/settings/base.py`, update the TEMPLATES setting:
> ```python
> TEMPLATES = [
>     {
>         'BACKEND': 'django.template.backends.django.DjangoTemplates',
>         'DIRS': [BASE_DIR / 'templates'],    # ← add this line
>         'APP_DIRS': True,
>         'OPTIONS': {
>             'context_processors': [
>                 'django.template.context_processors.debug',
>                 'django.template.context_processors.request',
>                 'django.contrib.auth.context_processors.auth',
>                 'django.contrib.messages.context_processors.messages',
>             ],
>         },
>     },
> ]
> ```
>
> **Step 7 — Create `apps/core/email_utils.py`** — shared email sending helper:
> ```python
> from django.core.mail import EmailMultiAlternatives
> from django.template.loader import render_to_string
> from django.conf import settings
>
>
> def send_html_email(subject, template_name, context, recipient_email):
>     '''
>     Renders an HTML email template and sends it via the configured backend.
>
>     Why EmailMultiAlternatives instead of send_mail?
>       EmailMultiAlternatives lets us attach both a plain-text fallback and
>       the HTML version. Email clients that cannot render HTML will show the
>       plain-text version automatically — this is considered best practice.
>
>     Args:
>         subject:        Email subject line string
>         template_name:  Path relative to templates/ e.g. 'emails/order_confirmation.html'
>         context:        Dict of variables available in the template
>         recipient_email: Single email address string
>     '''
>     # Always inject FRONTEND_URL and store name into every email template
>     context['frontend_url']  = settings.FRONTEND_URL
>     context['store_name']    = 'Your Store'
>
>     html_content  = render_to_string(template_name, context)
>
>     # Plain-text fallback — strip the HTML tags for a basic readable version
>     import re
>     text_content = re.sub(r'<[^>]+>', ' ', html_content)
>     text_content = re.sub(r'\s+', ' ', text_content).strip()
>
>     email = EmailMultiAlternatives(
>         subject=subject,
>         body=text_content,
>         from_email=settings.DEFAULT_FROM_EMAIL,
>         to=[recipient_email],
>     )
>     email.attach_alternative(html_content, 'text/html')
>     email.send(fail_silently=False)
> ```
>
> **Step 8 — Verify the template system works:**
> ```bash
> python manage.py shell --settings=config.settings.dev
> ```
> ```python
> from django.template.loader import render_to_string
> html = render_to_string('emails/base.html', {'frontend_url': 'http://localhost:5173'})
> print('Template loaded OK, length:', len(html))
> ```
>
> Show all complete file contents and confirm the shell test passes."

---

## DAY 3 — Registration + Email Verification + Password Reset Emails

### Prompt:

> "Today we complete the email skeleton that was left unfinished in Week 1.
> The endpoints already exist — they just don't send emails yet. Wire up
> the actual sending using Celery tasks.
>
> **Background — what already exists from Week 1:**
> - `Token` Document with `create_for_user(user, token_type, hours_valid)` classmethod
>   that stores a hashed token and returns the plain version
> - `POST /auth/password/reset/` — creates a Token, returns a placeholder response
> - `POST /auth/password/reset/confirm/` — verifies token, updates password
> - `POST /auth/register/` — creates user with is_verified=False, returns user data
>
> **Step 1 — Create `backend/templates/emails/welcome.html`:**
> Extends base.html. Shows:
> - 'Welcome to Your Store, {first_name}!' heading
> - Short welcome message
> - A prominent 'Verify Your Email' button linking to:
>   `{frontend_url}/verify-email/{token}`
> - Note that the link expires in 24 hours
> - If they didn't register: 'ignore this email' message
>
> **Step 2 — Create `backend/templates/emails/email_verification.html`:**
> Same as welcome.html but sent standalone when a user requests a new
> verification link. Heading: 'Verify your email address'.
>
> **Step 3 — Create `backend/templates/emails/password_reset.html`:**
> Extends base.html. Shows:
> - 'Reset your password' heading
> - 'We received a request to reset your password' message
> - A prominent 'Reset Password' button linking to:
>   `{frontend_url}/reset-password/{token}`
> - 'This link expires in 1 hour'
> - 'If you did not request this, ignore this email — your account is safe.'
>
> **Step 4 — Create `apps/authentication/tasks.py`:**
> ```python
> from celery import shared_task
> from apps.core.email_utils import send_html_email
>
>
> @shared_task(bind=True, max_retries=3, default_retry_delay=60)
> def send_welcome_email(self, user_id, plain_token):
>     '''
>     Sends a welcome + email verification email to a newly registered user.
>
>     Why pass user_id instead of the full user object?
>       Celery serializes task arguments to JSON. mongoengine Documents are
>       not JSON-serializable, so we pass the string ID and re-fetch inside
>       the task. This is the correct pattern for all Celery tasks.
>
>     Why max_retries=3 + default_retry_delay=60?
>       SendGrid occasionally has transient failures. Retrying up to 3 times
>       with a 60-second gap handles these without losing the email entirely.
>     '''
>     try:
>         from apps.users.documents import User
>         from bson import ObjectId
>         user = User.objects(id=ObjectId(user_id)).first()
>         if not user:
>             return  # user deleted between task dispatch and execution
>
>         send_html_email(
>             subject=f'Welcome to Your Store, {user.first_name}! Please verify your email.',
>             template_name='emails/welcome.html',
>             context={
>                 'user':  user,
>                 'token': plain_token,
>             },
>             recipient_email=user.email,
>         )
>     except Exception as exc:
>         raise self.retry(exc=exc)
>
>
> @shared_task(bind=True, max_retries=3, default_retry_delay=60)
> def send_password_reset_email(self, user_id, plain_token):
>     '''Sends a password reset link to the user.'''
>     try:
>         from apps.users.documents import User
>         from bson import ObjectId
>         user = User.objects(id=ObjectId(user_id)).first()
>         if not user:
>             return
>
>         send_html_email(
>             subject='Reset your password — Your Store',
>             template_name='emails/password_reset.html',
>             context={
>                 'user':  user,
>                 'token': plain_token,
>             },
>             recipient_email=user.email,
>         )
>     except Exception as exc:
>         raise self.retry(exc=exc)
> ```
>
> **Step 5 — Update `apps/authentication/views.py` — RegisterView:**
> After creating the user, dispatch the welcome email task:
> ```python
> from apps.authentication.tasks import send_welcome_email
> from apps.authentication.documents import Token
>
> # Inside RegisterView.post(), after user.save():
> plain_token = Token.create_for_user(user, 'email_verify', hours_valid=24)
> send_welcome_email.delay(str(user.id), plain_token)
> ```
>
> **Step 6 — Update `apps/authentication/views.py` — PasswordResetView:**
> Replace the placeholder response with real token creation + email dispatch:
> ```python
> from apps.authentication.tasks import send_password_reset_email
>
> # Inside PasswordResetView.post():
> user = User.objects(email=request.data.get('email')).first()
> if user:  # intentionally vague response regardless of whether user exists
>     plain_token = Token.create_for_user(user, 'password_reset', hours_valid=1)
>     send_password_reset_email.delay(str(user.id), plain_token)
> return Response({'detail': 'If this email exists, a reset link has been sent.'})
> ```
>
> **Step 7 — Add email verification endpoint:**
> ```
> GET /api/v1/auth/verify-email/{token}/
> ```
> ```python
> class VerifyEmailView(APIView):
>     permission_classes = [AllowAny]
>
>     def get(self, request, token):
>         import hashlib
>         from apps.authentication.documents import Token
>         hashed = hashlib.sha256(token.encode()).hexdigest()
>         token_doc = Token.objects(
>             token=hashed,
>             type='email_verify'
>         ).first()
>
>         if not token_doc:
>             return Response({'detail': 'Invalid or expired link.'}, status=400)
>
>         from datetime import datetime
>         if token_doc.expires_at < datetime.utcnow():
>             token_doc.delete()
>             return Response({'detail': 'This verification link has expired.'}, status=400)
>
>         user = User.objects(id=token_doc.user_id).first()
>         if user:
>             user.is_verified = True
>             user.save()
>         token_doc.delete()
>         return Response({'detail': 'Email verified successfully.'})
> ```
> Add to `apps/authentication/urls.py`:
> ```python
> path('auth/verify-email/<str:token>/', VerifyEmailView.as_view(), name='verify-email'),
> ```
>
> **Step 8 — Verification:**
> ```bash
> # With console email backend (dev.py), register a new user:
> curl -s -X POST http://localhost:8000/api/v1/auth/register/ \
>   -H 'Content-Type: application/json' \
>   -d '{\"email\":\"new@test.com\",\"password\":\"testpass123\",\"first_name\":\"New\",\"last_name\":\"User\"}' \
>   | python -m json.tool
>
> # Check the console (Django terminal) — you should see the full HTML email printed
> # The Celery worker terminal should show the task was received and executed
>
> # Test password reset:
> curl -s -X POST http://localhost:8000/api/v1/auth/password/reset/ \
>   -H 'Content-Type: application/json' \
>   -d '{\"email\":\"new@test.com\"}' | python -m json.tool
> # Again, check the console for the reset email output
> ```
>
> Show all complete file contents."

---

## DAY 4 — Order Confirmation + Shipping Notification Email Tasks

### Prompt:

> "Build the two most important customer-facing emails: the order confirmation
> (sent when payment succeeds) and the shipping notification (sent when an
> admin marks an order as shipped).
>
> **Step 1 — Create `backend/templates/emails/order_confirmation.html`:**
> Extends base.html. Sections:
>
> Header section:
> - Green checkmark emoji or icon
> - 'Your order is confirmed!' heading
> - 'Hi {user.first_name}, thank you for your order.' subtext
> - Order number in a highlighted pill: ORD-2024-00142
>
> Order items table (use .order-table CSS class from base):
> - Columns: Product | Qty | Price
> - Each row: product snapshot name + variant (color/size if present) | quantity | subtotal
>
> Order summary block:
> - Subtotal, shipping cost, discount (if any), total — each on its own row
> - Total row in bold
>
> Shipping address block:
> - 'Delivering to:' label
> - full_name, street, city, country
>
> Next steps block:
> - 'We'll email you when your order ships.'
> - 'Track your order' link to {frontend_url}/orders/{order_number}
>
> **Step 2 — Create `backend/templates/emails/order_shipped.html`:**
> Extends base.html. Sections:
> - '📦 Your order has shipped!' heading
> - 'Hi {user.first_name}, great news — your order is on its way.' message
> - Order number pill
> - Tracking number (if set): 'Track your shipment: {tracking_number}'
> - Estimated delivery note (based on shipping_method: standard = 5-7 days, express = 1-2 days)
> - Link to view order: {frontend_url}/orders/{order_number}
> - Items summary (compact — just names and quantities, no prices)
>
> **Step 3 — Create `apps/orders/tasks.py`:**
> ```python
> from celery import shared_task
> from apps.core.email_utils import send_html_email
>
>
> @shared_task(bind=True, max_retries=3, default_retry_delay=60)
> def send_order_confirmation_email(self, order_id):
>     '''
>     Sends the order confirmation email after payment is verified.
>
>     Called by: the payment webhook handler (Stripe/IntaSend), NOT by the
>     order creation view. The order is only confirmed after money is received.
>
>     Idempotency: The webhook handler already guards against duplicate
>     processing using stripe_event_id / intasend_invoice_id. This task
>     will therefore only ever be dispatched once per order.
>     '''
>     try:
>         from apps.orders.documents import Order
>         from apps.users.documents import User
>         from bson import ObjectId
>
>         order = Order.objects(id=ObjectId(order_id)).first()
>         if not order:
>             return
>
>         user = User.objects(id=order.user_id).only('email', 'first_name').first()
>         if not user:
>             return
>
>         send_html_email(
>             subject=f'Order {order.order_number} confirmed! ✓',
>             template_name='emails/order_confirmation.html',
>             context={
>                 'order': order,
>                 'user':  user,
>             },
>             recipient_email=user.email,
>         )
>     except Exception as exc:
>         raise self.retry(exc=exc)
>
>
> @shared_task(bind=True, max_retries=3, default_retry_delay=60)
> def send_order_shipped_email(self, order_id):
>     '''
>     Sends a shipping notification when admin marks order as shipped.
>     Called by the order status update view after status changes to 'shipped'.
>     '''
>     try:
>         from apps.orders.documents import Order
>         from apps.users.documents import User
>         from bson import ObjectId
>
>         order = Order.objects(id=ObjectId(order_id)).first()
>         if not order:
>             return
>
>         user = User.objects(id=order.user_id).only('email', 'first_name').first()
>         if not user:
>             return
>
>         delivery_days = '5-7 business days' if order.shipping_method == 'standard' else '1-2 business days'
>
>         send_html_email(
>             subject=f'Your order {order.order_number} has shipped! 📦',
>             template_name='emails/order_shipped.html',
>             context={
>                 'order':         order,
>                 'user':          user,
>                 'delivery_days': delivery_days,
>             },
>             recipient_email=user.email,
>         )
>     except Exception as exc:
>         raise self.retry(exc=exc)
>
>
> @shared_task(bind=True, max_retries=3, default_retry_delay=60)
> def send_review_request_email(self, order_id):
>     '''
>     Asks the customer to leave a review after delivery.
>     Called when order status changes to delivered.
>     '''
>     try:
>         from apps.orders.documents import Order
>         from apps.users.documents import User
>         from bson import ObjectId
>
>         order = Order.objects(id=ObjectId(order_id)).first()
>         if not order:
>             return
>
>         user = User.objects(id=order.user_id).only('email', 'first_name').first()
>         if not user:
>             return
>
>         send_html_email(
>             subject=f'How was your order? Leave a review — Your Store',
>             template_name='emails/review_request.html',
>             context={
>                 'order': order,
>                 'user':  user,
>             },
>             recipient_email=user.email,
>         )
>     except Exception as exc:
>         raise self.retry(exc=exc)
> ```
>
> **Step 4 — Create `backend/templates/emails/review_request.html`:**
> Extends base.html. Sections:
> - '⭐ How did we do?' heading
> - 'Hi {user.first_name}, your order has been delivered!' message
> - Compact items list (name only, no prices)
> - 'Leave a Review' button linking to {frontend_url}/orders/{order.order_number}
> - 'Your feedback helps other shoppers.' closing note
>
> **Step 5 — Wire tasks into the payment webhook handler:**
>
> Open your existing payments webhook view (`apps/payments/views.py`).
> In the `_fulfill_order()` helper (or wherever `order.add_status('paid')` is called),
> add the task dispatch:
>
> ```python
> from apps.orders.tasks import send_order_confirmation_email
>
> # After marking the order as paid and saving:
> send_order_confirmation_email.delay(str(order.id))
> ```
>
> **Step 6 — Wire tasks into the order status update endpoint:**
>
> Open `apps/orders/views.py`. Find (or create) the admin status update view.
> Add task dispatch based on the new status:
>
> ```python
> from apps.orders.tasks import send_order_shipped_email, send_review_request_email
>
> # Inside the status update view, after order.add_status(new_status):
> if new_status == 'shipped':
>     send_order_shipped_email.delay(str(order.id))
> elif new_status == 'delivered':
>     send_review_request_email.delay(str(order.id))
> ```
>
> If a `PATCH /api/v1/orders/{order_number}/status/` endpoint does not yet exist,
> create it now:
> ```python
> class OrderStatusUpdateView(APIView):
>     permission_classes = [IsAuthenticated]  # add IsAdminUser check
>
>     def patch(self, request, order_number):
>         # Only admins can update status
>         if request.user.role != 'admin':
>             return Response({'detail': 'Admin access required.'}, status=403)
>
>         new_status = request.data.get('status')
>         valid_statuses = ['processing', 'shipped', 'delivered', 'cancelled', 'refunded']
>         if new_status not in valid_statuses:
>             return Response({'detail': f'Invalid status. Choose from: {valid_statuses}'}, status=400)
>
>         note = request.data.get('note', '')
>         tracking = request.data.get('tracking_number')
>
>         order = Order.objects(order_number=order_number).first()
>         if not order:
>             return Response({'detail': 'Order not found.'}, status=404)
>
>         if tracking and new_status == 'shipped':
>             order.tracking_number = tracking
>
>         order.add_status(new_status, by=request.user.email, note=note)
>
>         if new_status == 'shipped':
>             send_order_shipped_email.delay(str(order.id))
>         elif new_status == 'delivered':
>             send_review_request_email.delay(str(order.id))
>
>         return Response(OrderSerializer(order).data)
> ```
> Add to `apps/orders/urls.py`:
> ```python
> path('orders/<str:order_number>/status/', OrderStatusUpdateView.as_view(), name='order-status-update'),
> ```
>
> **Step 7 — Verify:**
> ```bash
> # Place a test order through the frontend, complete payment.
> # In the Celery worker terminal you should see:
> # Task apps.orders.tasks.send_order_confirmation_email received
> # Task apps.orders.tasks.send_order_confirmation_email succeeded
>
> # Check the Django console for the email HTML output.
>
> # Test shipping notification:
> curl -s -X PATCH http://localhost:8000/api/v1/orders/ORD-2024-00001/status/ \
>   -H 'Content-Type: application/json' \
>   --cookie 'access_token=<admin_token>' \
>   -d '{\"status\": \"shipped\", \"tracking_number\": \"KQ12345KE\"}' \
>   | python -m json.tool
> # Celery worker should show the shipped email task executing
> ```
>
> Show all complete file contents."

---

## DAY 5 — Celery Beat + Low Stock Alert + Admin Email

### Prompt:

> "Set up Celery Beat for periodic tasks and build the low-stock alert
> that emails the admin when any product variant drops below 5 units.
>
> **Step 1 — Create `backend/templates/emails/low_stock_alert.html`:**
> Extends base.html. This email goes to the store admin, not customers.
> Sections:
> - '⚠️ Low Stock Alert' heading (red/warning tone)
> - 'The following products are running low and need restocking.' message
> - Table of low-stock items:
>   Columns: Product Name | SKU | Variant | Current Stock | Action
>   Each row: product name | sku | color + size | stock count (red if ≤ 2, orange if ≤ 5) | 'View Product' link
> - 'Log in to your admin dashboard to update inventory.' CTA
>
> **Step 2 — Create `apps/products/tasks.py`:**
> ```python
> from celery import shared_task
> from django.conf import settings
> from apps.core.email_utils import send_html_email
>
>
> @shared_task(bind=True, name='apps.products.tasks.check_low_stock')
> def check_low_stock(self):
>     '''
>     Periodic task: finds all product variants with stock <= LOW_STOCK_THRESHOLD
>     and emails the admin with a summary.
>
>     Why a threshold of 5?
>       It gives the admin enough lead time to reorder before stock runs out.
>       Adjust LOW_STOCK_THRESHOLD in settings if your reorder time differs.
>
>     Run schedule: every day at 8:00 AM Nairobi time (set in Step 3).
>     '''
>     from apps.products.documents import Product
>
>     LOW_STOCK_THRESHOLD = getattr(settings, 'LOW_STOCK_THRESHOLD', 5)
>     ADMIN_EMAIL         = getattr(settings, 'ADMIN_EMAIL', settings.DEFAULT_FROM_EMAIL)
>
>     low_stock_items = []
>
>     for product in Product.objects(is_active=True):
>         for variant in product.variants:
>             if variant.stock <= LOW_STOCK_THRESHOLD:
>                 low_stock_items.append({
>                     'product_name': product.name,
>                     'product_slug': product.slug,
>                     'sku':          variant.sku,
>                     'color':        variant.color or '-',
>                     'size':         variant.size  or '-',
>                     'stock':        variant.stock,
>                     'is_critical':  variant.stock <= 2,
>                 })
>
>     if not low_stock_items:
>         # No low-stock items today — skip sending the email
>         return f'No low-stock items found. Threshold: {LOW_STOCK_THRESHOLD}'
>
>     send_html_email(
>         subject=f'⚠️ Low Stock Alert — {len(low_stock_items)} item(s) need restocking',
>         template_name='emails/low_stock_alert.html',
>         context={
>             'items':     low_stock_items,
>             'threshold': LOW_STOCK_THRESHOLD,
>         },
>         recipient_email=ADMIN_EMAIL,
>     )
>
>     return f'Low stock alert sent for {len(low_stock_items)} items.'
> ```
>
> **Step 3 — Add `LOW_STOCK_THRESHOLD` and `ADMIN_EMAIL` to `config/settings/base.py`:**
> ```python
> LOW_STOCK_THRESHOLD = int(env('LOW_STOCK_THRESHOLD', default='5'))
> ADMIN_EMAIL         = env('ADMIN_EMAIL', default='admin@yourdomain.com')
> ```
> Add to `.env.example`:
> ```
> LOW_STOCK_THRESHOLD=5
> ADMIN_EMAIL=admin@yourdomain.com
> ```
>
> **Step 4 — Register the periodic schedule using django-celery-beat.**
>
> Create a management command `apps/core/management/commands/setup_periodic_tasks.py`
> that programmatically creates the beat schedule in the database:
> ```python
> from django.core.management.base import BaseCommand
> from django_celery_beat.models import PeriodicTask, CrontabSchedule
> import json
>
>
> class Command(BaseCommand):
>     help = 'Create Celery Beat periodic task schedules in the database.'
>
>     def handle(self, *args, **options):
>         # Every day at 8:00 AM Nairobi time
>         schedule, created = CrontabSchedule.objects.get_or_create(
>             minute='0',
>             hour='8',
>             day_of_week='*',
>             day_of_month='*',
>             month_of_year='*',
>             timezone='Africa/Nairobi',
>         )
>
>         task, created = PeriodicTask.objects.update_or_create(
>             name='Daily low-stock alert',
>             defaults={
>                 'task':     'apps.products.tasks.check_low_stock',
>                 'crontab':  schedule,
>                 'args':     json.dumps([]),
>                 'enabled':  True,
>             },
>         )
>
>         action = 'Created' if created else 'Updated'
>         self.stdout.write(self.style.SUCCESS(f'{action} periodic task: {task.name}'))
> ```
>
> Run it:
> ```bash
> python manage.py setup_periodic_tasks --settings=config.settings.dev
> ```
>
> **Step 5 — Start Celery Beat:**
> In a FOURTH terminal window:
> ```bash
> celery -A config beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler
> ```
> You should see: `beat: Starting...` and the schedule listed.
>
> **Step 6 — Test the low-stock task manually:**
> ```bash
> # Trigger immediately without waiting for the schedule
> python manage.py shell --settings=config.settings.dev
> ```
> ```python
> from apps.products.tasks import check_low_stock
> result = check_low_stock.delay()
> import time; time.sleep(3)
> print('Result:', result.result)
> ```
> Check the Django console for the email output (or Celery worker logs).
>
> **Step 7 — Verify the four-process setup:**
> ```
> Terminal 1: Django server      — python manage.py runserver
> Terminal 2: Celery worker      — celery -A config worker --loglevel=info
> Terminal 3: Celery beat        — celery -A config beat --loglevel=info
> Terminal 4: Flower (optional)  — celery -A config flower --port=5555
> ```
>
> Show all complete file contents."

---

## DAY 6 — Docker Compose for Redis + Test Real Email Sending + Week Cleanup

### Prompt:

> "Today we run Redis via Docker so you don't need a local Redis install,
> test real email delivery via SendGrid, and clean up any remaining wiring.
>
> **Step 1 — Run Redis via Docker (simplest local setup):**
> ```bash
> docker run -d --name ecommerce-redis -p 6379:6379 redis:7-alpine
> ```
> Verify it's running:
> ```bash
> docker ps | grep redis
> ```
> Verify Celery connects to it — restart the worker and check for:
> `Connected to redis://localhost:6379/0`
>
> **Step 2 — Test real email via SendGrid:**
> In `config/settings/dev.py`, temporarily comment out the console backend:
> ```python
> # EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
> # (Remove this comment to use real sending)
> ```
> With `EMAIL_BACKEND` removed from dev.py, the base.py SMTP backend is used.
>
> Set real values in `.env`:
> ```
> SENDGRID_API_KEY=SG.your-real-key-here
> DEFAULT_FROM_EMAIL=you@yourdomain.com
> ```
>
> Send a test:
> ```bash
> python manage.py shell --settings=config.settings.dev
> ```
> ```python
> from apps.core.email_utils import send_html_email
> send_html_email(
>     subject='Test email from Celery setup',
>     template_name='emails/welcome.html',
>     context={'user': type('U', (), {'first_name': 'Test'})(), 'token': 'abc123'},
>     recipient_email='your-personal-email@gmail.com',
> )
> print('Sent!')
> ```
> Check your inbox — the email should arrive within 30 seconds.
>
> **Step 3 — SendGrid dashboard setup (if not done already):**
> - Go to https://app.sendgrid.com → Settings → API Keys → Create API Key
>   (Full Access or restricted to Mail Send only)
> - Go to Settings → Sender Authentication → verify your sender domain or
>   at minimum verify a single sender email address
> - Without sender verification, SendGrid will reject outbound emails
>
> **Step 4 — Add Celery worker startup to `config/settings/prod.py`:**
> ```python
> # In production, NEVER use the console email backend
> # EMAIL_BACKEND is inherited from base.py (SMTP via SendGrid)
>
> # Use separate Redis DBs for broker and results in production
> CELERY_BROKER_URL     = env('REDIS_BROKER_URL', default='redis://redis:6379/0')
> CELERY_RESULT_BACKEND = env('REDIS_RESULT_URL', default='redis://redis:6379/1')
> ```
>
> **Step 5 — Wire any missing task dispatches.**
>
> Go through this checklist and confirm each trigger exists in the codebase:
>
> | Event | Task | Where called |
> |---|---|---|
> | User registers | send_welcome_email | RegisterView.post() |
> | Password reset requested | send_password_reset_email | PasswordResetView.post() |
> | Payment confirmed | send_order_confirmation_email | _fulfill_order() or webhook handler |
> | Order marked shipped | send_order_shipped_email | OrderStatusUpdateView.patch() |
> | Order marked delivered | send_review_request_email | OrderStatusUpdateView.patch() |
> | Daily 8 AM | check_low_stock | Celery Beat schedule |
>
> For any row where the trigger is missing, add the `.delay()` call now.
>
> **Step 6 — Test the complete email flow end-to-end:**
> 1. Register a new user → check email (welcome + verify)
> 2. Click the verify link → `GET /api/v1/auth/verify-email/{token}/` → 200
> 3. Request password reset → check email → follow link → reset works
> 4. Place an order + complete payment → check email (order confirmation)
> 5. PATCH order to shipped → check email (shipping notification with tracking)
> 6. PATCH order to delivered → check email (review request)
> 7. Trigger check_low_stock manually → if low-stock items exist, check admin email
>
> Show any missing files and confirm all 7 steps work."

---

## DAY 7 — Django Tests + Full Week 6 Verification + Handoff

### Prompt:

> "Write tests for the email tasks and run the full Week 6 verification
> checklist. Since Celery tasks call external services, we mock the email
> sending and test the task logic in isolation.
>
> **Step 1 — Create `apps/authentication/tests_email.py`:**
> ```python
> from unittest.mock import patch
> from rest_framework.test import APITestCase
> from rest_framework import status
> from apps.users.documents import User
> from apps.authentication.documents import Token
>
>
> def _make_user(email='emailtest@test.com'):
>     u = User(email=email, first_name='Email', last_name='Test',
>               role='customer', is_active=True, is_verified=False)
>     u.set_password('testpassword123')
>     u.save()
>     return u
>
>
> class RegistrationEmailTests(APITestCase):
>
>     def setUp(self):
>         Token.objects.delete()
>         User.objects.delete()
>
>     def tearDown(self):
>         Token.objects.delete()
>         User.objects.delete()
>
>     @patch('apps.authentication.tasks.send_welcome_email.delay')
>     def test_register_dispatches_welcome_email_task(self, mock_delay):
>         '''Registration must dispatch send_welcome_email as a Celery task.'''
>         resp = self.client.post('/api/v1/auth/register/', {
>             'email': 'new@test.com', 'password': 'testpass123',
>             'first_name': 'New', 'last_name': 'User',
>         }, format='json')
>         self.assertEqual(resp.status_code, status.HTTP_201_CREATED)
>         mock_delay.assert_called_once()
>         args = mock_delay.call_args[0]
>         self.assertIsNotNone(args[0])  # user_id
>         self.assertIsNotNone(args[1])  # plain_token
>
>     @patch('apps.authentication.tasks.send_password_reset_email.delay')
>     def test_password_reset_dispatches_email_task(self, mock_delay):
>         '''Password reset request must dispatch the email task.'''
>         _make_user()
>         resp = self.client.post('/api/v1/auth/password/reset/', {
>             'email': 'emailtest@test.com',
>         }, format='json')
>         self.assertEqual(resp.status_code, status.HTTP_200_OK)
>         mock_delay.assert_called_once()
>
>     @patch('apps.authentication.tasks.send_password_reset_email.delay')
>     def test_password_reset_nonexistent_email_returns_200(self, mock_delay):
>         '''Must return 200 even for non-existent emails — prevents user enumeration.'''
>         resp = self.client.post('/api/v1/auth/password/reset/', {
>             'email': 'nobody@test.com',
>         }, format='json')
>         self.assertEqual(resp.status_code, status.HTTP_200_OK)
>         mock_delay.assert_not_called()
>
>     def test_verify_email_with_valid_token(self):
>         '''Valid token verifies the user and sets is_verified=True.'''
>         user = _make_user(email='verify@test.com')
>         plain_token = Token.create_for_user(user, 'email_verify', hours_valid=24)
>         resp = self.client.get(f'/api/v1/auth/verify-email/{plain_token}/')
>         self.assertEqual(resp.status_code, status.HTTP_200_OK)
>         user.reload()
>         self.assertTrue(user.is_verified)
>
>     def test_verify_email_with_invalid_token_returns_400(self):
>         resp = self.client.get('/api/v1/auth/verify-email/totally-fake-token/')
>         self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)
>
>     def test_verify_email_token_deleted_after_use(self):
>         '''Token document must be deleted after successful verification.'''
>         user = _make_user(email='token_cleanup@test.com')
>         plain_token = Token.create_for_user(user, 'email_verify', hours_valid=24)
>         self.client.get(f'/api/v1/auth/verify-email/{plain_token}/')
>         # Token should be gone
>         import hashlib
>         hashed = hashlib.sha256(plain_token.encode()).hexdigest()
>         self.assertIsNone(Token.objects(token=hashed).first())
>
>
> class OrderEmailTests(APITestCase):
>     '''Tests that order tasks dispatch correctly when status changes.'''
>
>     def setUp(self):
>         User.objects.delete()
>         from apps.orders.documents import Order
>         Order.objects.delete()
>         self.admin = User(email='admin@test.com', first_name='Admin',
>                           last_name='User', role='admin', is_active=True,
>                           is_verified=True)
>         self.admin.set_password('adminpass123')
>         self.admin.save()
>
>     def tearDown(self):
>         User.objects.delete()
>         from apps.orders.documents import Order
>         Order.objects.delete()
>
>     def _login_admin(self):
>         resp = self.client.post('/api/v1/auth/login/',
>             {'email': 'admin@test.com', 'password': 'adminpass123'}, format='json')
>         self.assertEqual(resp.status_code, 200)
>
>     @patch('apps.orders.tasks.send_order_shipped_email.delay')
>     def test_status_update_to_shipped_dispatches_email(self, mock_delay):
>         '''PATCH status=shipped must dispatch send_order_shipped_email.'''
>         from apps.orders.documents import Order, OrderItem, ShippingAddress, StatusHistory
>         from bson import ObjectId
>         item = OrderItem(product_id=ObjectId(), product_name='Test', product_slug='test',
>                          variant_id='v1', variant_sku='S1', quantity=1,
>                          unit_price=50.0, subtotal=50.0)
>         addr = ShippingAddress(full_name='Test', phone='+254700000000',
>                                street='123 St', city='Nairobi', country='Kenya')
>         order = Order(order_number='ORD-TEST-99999', user_id=self.admin.id,
>                       status='paid', items=[item], shipping_address=addr,
>                       shipping_method='standard', shipping_cost=5.0,
>                       subtotal=50.0, total=55.0,
>                       status_history=[StatusHistory(status='paid', by='system')])
>         order.save()
>
>         self._login_admin()
>         resp = self.client.patch(
>             f'/api/v1/orders/{order.order_number}/status/',
>             {'status': 'shipped', 'tracking_number': 'KQ99999'},
>             format='json',
>         )
>         self.assertEqual(resp.status_code, status.HTTP_200_OK)
>         mock_delay.assert_called_once_with(str(order.id))
>
>
> class LowStockTaskTests(APITestCase):
>
>     def setUp(self):
>         from apps.products.documents import Product
>         Product.objects.delete()
>
>     def tearDown(self):
>         from apps.products.documents import Product
>         Product.objects.delete()
>
>     @patch('apps.products.tasks.send_html_email')
>     def test_low_stock_task_sends_email_when_items_found(self, mock_send):
>         '''check_low_stock must call send_html_email when variants are low.'''
>         from apps.products.documents import Product, Variant
>         from bson import ObjectId
>         v = Variant(variant_id=str(ObjectId()), sku='LO-01', price=50.0, stock=2)
>         p = Product(seller_id=ObjectId(), category_id=ObjectId(),
>                     name='Low Product', slug='low-product', description='...',
>                     base_price=50.0, variants=[v], is_active=True)
>         p.save()
>
>         from apps.products.tasks import check_low_stock
>         check_low_stock()   # call synchronously (no .delay() in tests)
>         mock_send.assert_called_once()
>
>     @patch('apps.products.tasks.send_html_email')
>     def test_low_stock_task_skips_email_when_all_stocked(self, mock_send):
>         '''check_low_stock must NOT email when all variants are above threshold.'''
>         from apps.products.documents import Product, Variant
>         from bson import ObjectId
>         v = Variant(variant_id=str(ObjectId()), sku='HI-01', price=50.0, stock=100)
>         p = Product(seller_id=ObjectId(), category_id=ObjectId(),
>                     name='Stocked Product', slug='stocked-product', description='...',
>                     base_price=50.0, variants=[v], is_active=True)
>         p.save()
>
>         from apps.products.tasks import check_low_stock
>         check_low_stock()
>         mock_send.assert_not_called()
> ```
>
> **Step 2 — Run all Week 6 tests:**
> ```bash
> python manage.py test apps.authentication.tests_email apps.orders apps.products \
>   --settings=config.settings.test -v 2
> ```
> All tests must pass.
>
> **Step 3 — Full Week 6 end-to-end verification checklist:**
>
> Celery infrastructure:
> 1. Redis is running (docker ps shows ecommerce-redis)
> 2. Celery worker starts without errors, lists all tasks in [tasks]
> 3. Celery Beat starts without errors, shows the daily schedule
> 4. smoke_test.delay() executes and returns 'ok' in under 3 seconds
> 5. Flower dashboard at http://localhost:5555 shows the worker and completed tasks
>
> Email sending (console backend in dev):
> 6. Register new user → Celery worker processes task → email HTML printed to console
> 7. Email contains the user's first name and a verify link with the token
> 8. GET /auth/verify-email/{token}/ → user.is_verified becomes True
> 9. POST /auth/password/reset/ with valid email → email with reset link printed
> 10. POST /auth/password/reset/confirm/ with token → password updated, token deleted
> 11. Complete a payment → order_confirmation email task fires → HTML printed to console
>     Email contains order number, item list, totals, shipping address
> 12. PATCH order to shipped → shipping email task fires → HTML printed to console
>     Email contains tracking number and estimated delivery
> 13. PATCH order to delivered → review_request email task fires
> 14. check_low_stock.delay() with low-stock products → admin alert email printed
> 15. check_low_stock.delay() with all products well-stocked → no email sent
>
> Real email (switch to SMTP backend temporarily):
> 16. send_html_email() with your real email address → arrives in inbox within 30s
> 17. Email renders correctly in Gmail (check for broken layout or missing styles)
>
> **Step 4 — Provide the Week 6 session handoff summary:**
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

## WEEK 6 SUMMARY — What You Will Have Built

By the end of Day 7, your project will have:

**Backend:**
- Celery fully configured with `config/celery.py` and `config/__init__.py` import
- Redis as message broker and result backend (`CELERY_BROKER_URL` + `CELERY_RESULT_BACKEND`)
- `django-celery-beat` for persistent periodic task schedules
- `django-celery-results` for queryable task results
- Base HTML email template (`templates/emails/base.html`) with table layout and inline CSS
- `send_html_email()` helper in `apps/core/email_utils.py` — renders + sends with plain-text fallback
- `apps/authentication/tasks.py`: `send_welcome_email`, `send_password_reset_email`
- `apps/orders/tasks.py`: `send_order_confirmation_email`, `send_order_shipped_email`, `send_review_request_email`
- `apps/products/tasks.py`: `check_low_stock` periodic task
- `GET /api/v1/auth/verify-email/{token}/` endpoint — verifies email and deletes the token
- `PATCH /api/v1/orders/{order_number}/status/` endpoint — admin status update with email triggers
- All 6 email types wired to their triggers (registration, password reset, order confirmation, shipping, delivery, low stock)
- `apps/core/management/commands/setup_periodic_tasks.py` — creates Beat schedules in DB
- Django tests with `unittest.mock.patch` — tests task dispatch without hitting SendGrid
- Email templates: welcome.html, email_verification.html, password_reset.html, order_confirmation.html, order_shipped.html, review_request.html, low_stock_alert.html

**Infrastructure:**
- Redis running via Docker (`docker run -d --name ecommerce-redis -p 6379:6379 redis:7-alpine`)
- Four-process development setup: Django server + Celery worker + Celery Beat + Flower
- SendGrid configured via SMTP relay in base.py, console backend in dev.py

**Week 7 continues with:** Admin Dashboard — React admin pages for product CRUD,
order management with inline status updates, inventory view with low-stock alerts,
coupon management, and a Recharts revenue dashboard. Role-based route guards.
