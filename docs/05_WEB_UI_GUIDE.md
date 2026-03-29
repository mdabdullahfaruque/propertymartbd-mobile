# 🏠 My Property Mart
### Web UI Development Guide

**Document Version:** 1.1  
**Created:** March 20, 2026  
**Last Updated:** March 29, 2026  
**Audience:** Backend + Web Developer (You)  
**Status:** Phase 1 – Complete, Phase 2 – Planning

---

## Table of Contents

- [1. Git Repository Strategy](#1-git-repository-strategy)
- [2. Angular Project Setup](#2-angular-project-setup)
- [3. Folder Structure](#3-folder-structure)
- [4. All Pages/Screens by Phase](#4-all-pagesscreens-by-phase)
- [5. Routing Configuration](#5-routing-configuration)
- [6. Shared Components](#6-shared-components)
- [7. i18n (Bangla/English/Malay)](#7-i18n-banglaenglishmalay)
- [8. Google Analytics 4 Setup](#8-google-analytics-4-setup)
- [9. GitHub Pages Hosting](#9-github-pages-hosting)
- [10. CI/CD Pipeline](#10-cicd-pipeline)
- [11. Domain Strategy](#11-domain-strategy)

---

## 1. Git Repository Strategy

### Recommended: 3 Repositories

| Repository | Contents | Deployed To | Maintainer |
|-----------|----------|-------------|------------|
| `mypropertymart-api` | .NET API + docs | Azure VM | Backend Dev (You) |
| `mypropertymart-web` | Angular web app | GitHub Pages | Backend Dev (You) |
| `mypropertymart-mobile` | Flutter app | Play Store / App Store | Mobile Dev |

**Why 3 repos instead of monorepo:**
- GitHub Pages deploys cleanly from its own repo (`gh-pages` branch)
- Mobile dev only clones what they need
- Independent CI/CD pipelines — web deploy doesn't trigger API deploy
- Cleaner git history per project

**Alternative — 2 repos** (if you prefer managing fewer repos):

| Repository | Contents | Notes |
|-----------|----------|-------|
| `mypropertymart` | .NET API + Angular web + docs | Use GitHub Actions to deploy web subfolder to `gh-pages` branch |
| `mypropertymart-mobile` | Flutter app | Mobile dev's repo |

Both work well. Choose 3 repos for cleaner separation, 2 repos for convenience.

### Shared Documentation

Keep all `docs/` (01-06 markdown files) in the API repo. The mobile dev references the API contract doc (`06_MOBILE_API_CONTRACT.md`) directly from that repo.

---

## 2. Angular Project Setup

```bash
# Create Angular project
ng new mypropertymart-web --routing --style=scss --ssr=false
cd mypropertymart-web

# Install dependencies
npm install leaflet @types/leaflet                      # OpenStreetMap
npm install @angular/fire firebase                      # Firebase Auth
npm install @ngx-translate/core @ngx-translate/http-loader  # i18n
npm install tailwindcss @tailwindcss/forms              # Styling

# Initialize Tailwind
npx tailwindcss init
```

### Environment Configuration

```typescript
// src/environments/environment.ts  (development)
export const environment = {
  production: false,
  apiBaseUrl: 'https://localhost:5001/api/v1',
  firebase: {
    apiKey: 'AIzaSyB6RQG1l92K6bLj8cec-XzPH5w0n4ARTbQ',
    authDomain: 'propertymartbd-8ab7c.firebaseapp.com',
    projectId: 'propertymartbd-8ab7c',
    storageBucket: 'propertymartbd-8ab7c.firebasestorage.app',
    messagingSenderId: '1028971662568',
    appId: '1:1028971662568:web:b0c8ede732ffe0432c8d5d'
  },
  googleAnalytics: {
    measurementId: ''  // Empty in dev — no tracking locally
  }
};

// src/environments/environment.prod.ts  (production - Bangladesh)
export const environment = {
  production: true,
  apiBaseUrl: 'https://api.mypropertymart.com/api/v1',
  isMalaysia: false,
  countryCode: 'BD',
  defaultLanguage: 'bn',
  supportedLanguages: ['bn', 'en'],
  firebase: {
    apiKey: 'AIzaSyB6RQG1l92K6bLj8cec-XzPH5w0n4ARTbQ',
    authDomain: 'propertymartbd-8ab7c.firebaseapp.com',
    projectId: 'propertymartbd-8ab7c',
    storageBucket: 'propertymartbd-8ab7c.firebasestorage.app',
    messagingSenderId: '1028971662568',
    appId: '1:1028971662568:web:b0c8ede732ffe0432c8d5d',
    measurementId: 'G-BEBVW1YC5C'
  },
  googleAnalytics: {
    measurementId: 'G-BEBVW1YC5C'
  }
};

// src/environments/environment.my.ts (production - Malaysia)
export const environment = {
  production: true,
  apiBaseUrl: 'https://api.mypropertymart.com/api/v1',   // or separate MY API URL
  isMalaysia: true,
  countryCode: 'MY',
  defaultLanguage: 'en',
  supportedLanguages: ['en', 'ms'],
  firebase: {
    // ... same firebase config
  },
  googleAnalytics: {
    measurementId: 'G-BEBVW1YC5C'   // or separate MY measurement
  }
};
```

> ℹ️ **Note:** The Firebase Web API Key is intentionally public — it identifies your project to Firebase servers. Security is enforced server-side via Firebase Authentication rules and your own .NET API authorization. It is safe to commit these values to your Git repository.

---

## 3. Folder Structure

```
src/
├── app/
│   ├── core/
│   │   ├── services/
│   │   │   ├── api.service.ts              # HTTP client wrapper
│   │   │   ├── auth.service.ts             # Firebase auth
│   │   │   ├── listing.service.ts
│   │   │   ├── user.service.ts
│   │   │   ├── location.service.ts
│   │   │   └── analytics.service.ts        # GA4 event tracking
│   │   ├── interceptors/
│   │   │   └── auth.interceptor.ts         # Adds Bearer token
│   │   ├── guards/
│   │   │   └── auth.guard.ts
│   │   └── models/
│   │       ├── listing.model.ts
│   │       ├── user.model.ts
│   │       └── api-response.model.ts
│   │
│   ├── features/
│   │   ├── home/                           # Landing page
│   │   ├── search/                         # Search results + filters
│   │   ├── listing-detail/                 # Full listing view
│   │   ├── create-listing/                 # Multi-step form
│   │   ├── edit-listing/                   # Edit existing listing
│   │   ├── my-listings/                    # Seller dashboard
│   │   ├── profile/                        # Public user profile
│   │   ├── edit-profile/                   # Edit own profile
│   │   ├── auth/                           # Login page
│   │   ├── company/                        # [P2A] Company profile
│   │   ├── verification/                   # [P2A] NID verification
│   │   ├── admin/                          # [P2A] Admin dashboard
│   │   ├── materials/                      # [P2A] Construction materials marketplace
│   │   ├── materials-dealer/               # [P2A] Dealer dashboard
│   │   ├── wishlist/                       # [P2A] Property & materials wishlist
│   │   ├── cost-calculator/                # [P2A] Construction cost calculator
│   │   ├── construction-portal/            # [P2B] Construction company portal
│   │   ├── construction-client/            # [P2B] Client portal
│   │   ├── flat-calculator/                # [P2B] Flat price calculator
│   │   ├── map-view/                       # [P3] Map search
│   │   └── blog/                           # [P4] Blog articles
│   │
│   ├── shared/
│   │   ├── components/
│   │   │   ├── listing-card/               # Reusable listing card
│   │   │   ├── map/                        # Leaflet.js wrapper
│   │   │   ├── image-gallery/              # Photo carousel
│   │   │   ├── image-upload/               # Multi-image upload
│   │   │   ├── unit-converter/             # কাঠা/বিঘা converter
│   │   │   ├── rating-stars/               # Star rating display/input
│   │   │   ├── location-selector/          # Division → District → Upazila
│   │   │   ├── language-toggle/            # Bangla/English switcher
│   │   │   └── navbar/                     # Top navigation bar
│   │   └── pipes/
│   │       ├── bdt.pipe.ts                 # ৳ formatting with লাখ/কোটি
│   │       └── land-unit.pipe.ts           # কাঠা display
│   │
│   ├── i18n/
│   │   ├── bn.json                         # Bangla translations
│   │   ├── en.json                         # English translations
│   │   └── ms.json                         # Bahasa Malay translations [P2A]
│   │
│   └── app.routes.ts
│
├── assets/
│   ├── images/
│   └── icons/
├── environments/
└── styles/
    └── styles.scss                          # Tailwind + custom styles
```

---

## 4. All Pages/Screens by Phase

### ✅ Phase 1 — MVP Pages (COMPLETE)

| # | Page | Route | Auth | Description |
|---|------|-------|------|-------------|
| 1 | **Home** | `/` | Public | Hero search bar, location selector, recent listings grid, "নতুন জমি" badge on new listings |
| 2 | **Search Results** | `/search` | Public | Listing cards, filter sidebar (location/price/size), sort dropdown, infinite scroll |
| 3 | **Listing Detail** | `/listings/:id` | Public | Image gallery carousel, all property data, mini map with pin, seller info card, contact information (Listing contact, Owner name/phone), WhatsApp & Direct Call buttons, share, report |
| 4 | **Create Listing** | `/listings/new` | Required | Multi-step form: ① Details → ② "Is Owner" toggle (if No: contact number required, optional owner name & contact) → ③ Location + Map Pin → ④ Photos (up to 10) → ⑤ Review & Publish |
| 5 | **Edit Listing** | `/listings/:id/edit` | Owner | Same form as create, pre-filled |
| 6 | **My Listings** | `/my-listings` | Required | Two tabs: **"As Seller"** (`?isOwner=true`) and **"As Agent"** (`?isOwner=false`). Listing cards with status badges, view counts, edit/delete actions |
| 7 | **User Profile** | `/users/:id` | Public | Photo, name, user type, verified badge, average rating, bio, listing history |
| 8 | **Edit Profile** | `/profile/edit` | Required | Edit name, phone, bio, **user type (Seller/Agent dropdown)** form |
| 9 | **Login** | `/login` | Public | Google sign-in + Phone OTP options, brief platform intro |
| 10 | **Phone Login** | `/login/phone` | Public | Phone number input (+880), OTP verification (6 digits), countdown timer |
| 11 | **About** | `/about` | Public | Platform info, contact, FAQ |
| 12 | **404 / Error** | `**` | Public | Not found page with search redirect |

### 🟡 Phase 2A — Additional Pages

| # | Page | Route | Auth | Description |
|---|------|-------|------|-------------|
| 13 | **NID Verification** | `/verification` | Required | Upload NID front/back, status tracking |
| 14 | **Company Profile** | `/companies/:id` | Public | Company logo, info, all company listings |
| 15 | **Create Company** | `/companies/new` | Company | Company registration form |
| 16 | **Admin Dashboard** | `/admin` | Admin | Pending verifications, reported listings, platform stats |
| 17 | **Admin Verification Review** | `/admin/verifications/:id` | Admin | Review NID images, approve/reject |
| 18 | **Materials Marketplace** | `/materials` | Public | Browse construction materials, filter by category, compare prices, sort by price/dealer/location |
| 19 | **Material Detail** | `/materials/:id` | Public | Material item detail, price comparison across dealers, dealers providing this material |
| 20 | **Dealer Dashboard** | `/dealer/dashboard` | Dealer | Manage material prices, add/edit items, view analytics |
| 21 | **Dealer Registration** | `/dealer/register` | Required | Register as construction material dealer |
| 22 | **Property Wishlist** | `/wishlist` | Required | Wishlisted properties and materials, price change alerts |
| 23 | **Construction Cost Calculator** | `/calculator` | Public | Select materials, enter quantities, calculate total construction cost |

### 🟠 Phase 2B — Additional Pages

| # | Page | Route | Auth | Description |
|---|------|-------|------|-------------|
| 24 | **Construction Portal Dashboard** | `/construction` | Constructor | Company admin panel — projects, clients overview |
| 25 | **Construction Project** | `/construction/projects/:id` | Constructor | Project details, manage clients, progress, notices |
| 26 | **Flat Price Calculator** | `/construction/calculator` | Constructor | Calculate per sqft price, try different values, profit margin analysis |
| 27 | **Client Portal** | `/client` | ConstrClient | Client dashboard — project progress, payments, installments, notices |
| 28 | **Client Payment Detail** | `/client/payments` | ConstrClient | Payment history, installment schedule, payment proof upload |

### 🟠 Phase 3 — Additional Pages

| # | Page | Route | Auth | Description |
|---|------|-------|------|-------------|
| 29 | **Search Alerts** | `/alerts` | Required | Manage saved search alerts |
| 30 | **Map View** | `/search/map` | Public | Full-screen map with listing markers |
| 31 | **Notifications** | `/notifications` | Required | Notification center |

### 🔴 Phase 4 — Additional Pages

| # | Page | Route | Auth | Description |
|---|------|-------|------|-------------|
| 32 | **Feature Listing** | `/listings/:id/feature` | Owner | Purchase featured placement |
| 33 | **Subscription Plans** | `/subscriptions` | Required | Agent/Company plan selection |
| 34 | **Blog Listing** | `/blog` | Public | Property guides, market reports |
| 35 | **Blog Detail** | `/blog/:slug` | Public | Full blog article |
| 36 | **AI Chat** | `/chat` | Public | AI property assistant |

---

## 5. Routing Configuration

```typescript
// app.routes.ts
export const routes: Routes = [
  // Phase 1
  { path: '', loadComponent: () => import('./features/home/home.component') },
  { path: 'search', loadComponent: () => import('./features/search/search.component') },
  { path: 'listings/new', loadComponent: () => import('./features/create-listing/create-listing.component'), canActivate: [authGuard] },
  { path: 'listings/:id', loadComponent: () => import('./features/listing-detail/listing-detail.component') },
  { path: 'listings/:id/edit', loadComponent: () => import('./features/edit-listing/edit-listing.component'), canActivate: [authGuard] },
  { path: 'my-listings', loadComponent: () => import('./features/my-listings/my-listings.component'), canActivate: [authGuard] },
  { path: 'users/:id', loadComponent: () => import('./features/profile/profile.component') },
  { path: 'profile/edit', loadComponent: () => import('./features/edit-profile/edit-profile.component'), canActivate: [authGuard] },
  { path: 'login', loadComponent: () => import('./features/auth/login.component') },
  { path: 'login/phone', loadComponent: () => import('./features/auth/phone-login.component') },
  { path: 'about', loadComponent: () => import('./features/about/about.component') },

  // Phase 2A (lazy loaded when built)
  { path: 'verification', loadComponent: () => import('./features/verification/verification.component'), canActivate: [authGuard] },
  { path: 'companies/:id', loadComponent: () => import('./features/company/company.component') },
  { path: 'companies/new', loadComponent: () => import('./features/company/create-company.component'), canActivate: [authGuard] },
  { path: 'admin', loadChildren: () => import('./features/admin/admin.routes'), canActivate: [adminGuard] },
  { path: 'materials', loadComponent: () => import('./features/materials/materials.component') },
  { path: 'materials/:id', loadComponent: () => import('./features/materials/material-detail.component') },
  { path: 'dealer', loadChildren: () => import('./features/materials-dealer/dealer.routes'), canActivate: [authGuard] },
  { path: 'wishlist', loadComponent: () => import('./features/wishlist/wishlist.component'), canActivate: [authGuard] },
  { path: 'calculator', loadComponent: () => import('./features/cost-calculator/cost-calculator.component') },

  // Phase 2B
  { path: 'construction', loadChildren: () => import('./features/construction-portal/construction.routes'), canActivate: [authGuard] },
  { path: 'client', loadChildren: () => import('./features/construction-client/client.routes'), canActivate: [authGuard] },

  // Phase 3
  { path: 'alerts', loadComponent: () => import('./features/search-alerts/search-alerts.component'), canActivate: [authGuard] },
  { path: 'search/map', loadComponent: () => import('./features/map-view/map-view.component') },

  // Catch-all
  { path: '**', loadComponent: () => import('./features/not-found/not-found.component') }
];
```

All routes use **lazy loading** (`loadComponent`) for faster initial load.

---

## 6. Shared Components

| Component | Purpose | Used In |
|-----------|---------|---------|
| `listing-card` | Property card with image, price, size, location | Home, Search, My Listings, Profile, Wishlist |
| `map` | Leaflet.js wrapper (tap-to-pin + display) | Create Listing, Listing Detail, Map View |
| `image-gallery` | Photo carousel with swipe | Listing Detail |
| `image-upload` | Multi-image upload with preview, reorder, delete | Create/Edit Listing |
| `unit-converter` | কাঠা ↔ বিঘা ↔ শতাংশ ↔ বর্গফুট | Create Listing, Listing Detail |
| `rating-stars` | Star display + input (1-5) | Profile, Listing Detail |
| `location-selector` | Division → District → Upazila cascading dropdowns. Supports pre-filling via IDs or string names (for API name-only responses) | Create Listing, Search, Search Alerts, Edit Listing |
| `language-toggle` | Bangla / English switcher (BD), English / Bahasa Malay switcher (MY) — based on `IsMalaysia` config | Navbar |
| `navbar` | Top navigation bar (responsive). Logged-in users see **profile picture dropdown** with "Profile", "My Listings", and "Logout" options | All pages |
| `price-comparison` | Displays price comparison table for materials | Materials Marketplace, Material Detail |
| `material-card` | Reusable material item card for marketplace | Materials Marketplace, Wishlist, Cost Calculator |
| `progress-chart` | Construction progress visualization (bars/charts) [P2B] | Construction Portal, Client Portal |
| `installment-table` | Installment schedule display [P2B] | Client Portal, Client Payment Detail |
| `currency-display` | Handles BDT/MYR display based on country config | All pages with prices |

### Navigation Structure

**Desktop:** Top bar — Logo | Home | Search | + Add Listing | My Listings | [Profile Picture Dropdown: Profile, My Listings, Logout]  
**Mobile:** Responsive hamburger menu with same items + profile dropdown

---

## 7. i18n (Bangla/English/Malay)

Using **@ngx-translate** for runtime language switching.

> **`IsMalaysia` Config:**
> - When `IsMalaysia = false` (Bangladesh): Show Bangla + English. Default: Bangla
> - When `IsMalaysia = true` (Malaysia): Show English + Bahasa Malay. Default: English

```json
// src/app/i18n/bn.json (sample)
{
  "nav.home": "হোম",
  "nav.search": "খুঁজুন",
  "nav.addListing": "জমি যোগ করুন",
  "nav.myListings": "আমার লিস্টিং",
  "nav.profile": "প্রোফাইল",
  "nav.login": "লগইন",
  "listing.price": "মূল্য",
  "listing.size": "আয়তন",
  "listing.perKatha": "প্রতি কাঠা",
  "listing.contact": "WhatsApp-এ যোগাযোগ করুন",
  "listing.share": "শেয়ার করুন",
  "listing.report": "রিপোর্ট",
  "listing.contactInfo": "যোগাযোগের তথ্য",
  "listing.agentContact": "এজেন্ট",
  "listing.propertyOwner": "মালিক",
  "listing.ownerPhone": "মালিকের ফোন",
  "listing.callNow": "এখনই কল করুন",
  "search.placeholder": "জমি খুঁজুন...",
  "search.filter": "ফিল্টার",
  "search.sort": "সাজান"
}
```

```json
// src/app/i18n/ms.json (sample — Bahasa Malay)
{
  "nav.home": "Laman Utama",
  "nav.search": "Cari",
  "nav.addListing": "Tambah Hartanah",
  "nav.myListings": "Senarai Saya",
  "nav.profile": "Profil",
  "nav.login": "Log Masuk",
  "nav.materials": "Bahan Binaan",
  "nav.wishlist": "Senarai Hajat",
  "listing.price": "Harga",
  "listing.size": "Saiz",
  "listing.contact": "Hubungi melalui WhatsApp",
  "materials.compare": "Bandingkan Harga",
  "materials.calculator": "Kalkulator Kos"
}
```

```json
// Additional keys for Phase 2 in bn.json:
{
  "nav.materials": "নির্মাণ সামগ্রী",
  "nav.wishlist": "পছন্দ তালিকা",
  "materials.compare": "দাম তুলনা করুন",
  "materials.calculator": "খরচ ক্যালকুলেটর",
  "materials.dealer": "বিক্রেতা",
  "construction.portal": "নির্মাণ পোর্টাল",
  "construction.progress": "অগ্রগতি",
  "construction.installment": "কিস্তি"
}
```

**Default language:** Bangla (`bn`) for Bangladesh, English (`en`) for Malaysia  
**Font:** Noto Sans Bengali (Google Fonts) — add to `index.html`

---

## 8. Google Analytics 4 Setup

### Why GA4
- **Completely free** — no data limits for standard GA4
- Track: page views, button clicks, user flows, bounce rate, user engagement
- Real-time dashboard for monitoring user activity
- No paid version needed (GA360 starts at ~$50K/year — not for us)

### Step 1: GA4 Property

> ✅ **Already done!** When you created the Firebase project with Google Analytics enabled, a GA4 property was automatically created and linked.
>
> Your **Measurement ID: `G-BEBVW1YC5C`** — this was in your Firebase web config as `measurementId`.
>
> You can view it at [analytics.google.com](https://analytics.google.com) → Admin → Data Streams → your web stream.

### Step 2: Install gtag.js

Add to `src/index.html` before `</head>`:

```html
<!-- Google Analytics 4 — only in production build (injected by Angular build config) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-BEBVW1YC5C"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-BEBVW1YC5C');
</script>
```

> ⚠️ Add this only to `src/index.html`. The `AnalyticsService` already guards against tracking in dev (`if (!environment.production) return`), so no double-tracking occurs.

### Step 3: Create Analytics Service

```typescript
// core/services/analytics.service.ts
import { Injectable } from '@angular/core';
import { environment } from '../../../environments/environment';

declare let gtag: Function;

@Injectable({ providedIn: 'root' })
export class AnalyticsService {

  trackPageView(url: string, title: string): void {
    if (!environment.production) return;
    gtag('config', environment.googleAnalytics.measurementId, {
      page_path: url,
      page_title: title
    });
  }

  trackEvent(action: string, category: string, label?: string, value?: number): void {
    if (!environment.production) return;
    gtag('event', action, {
      event_category: category,
      event_label: label,
      value: value
    });
  }
}
```

### Step 4: Track Page Views on Route Change

```typescript
// app.component.ts
constructor(private router: Router, private analytics: AnalyticsService) {
  this.router.events.pipe(
    filter(event => event instanceof NavigationEnd)
  ).subscribe((event: NavigationEnd) => {
    this.analytics.trackPageView(event.urlAfterRedirects, document.title);
  });
}
```

### Key Events to Track from Day 1

| Event | Category | When | Why |
|-------|----------|------|-----|
| `page_view` | Navigation | Every route change | Track which pages users visit |
| `listing_view` | Listing | Open listing detail | Most popular listings |
| `listing_create` | Listing | Publish new listing | Seller engagement |
| `search` | Search | Perform search | Popular search terms, locations |
| `whatsapp_click` | Contact | Click WhatsApp button | Buyer-seller connection rate |
| `share_click` | Social | Share listing | Viral growth tracking |
| `login` | Auth | User logs in | Auth conversion rate |
| `language_switch` | UI | Toggle Bangla/English | Language preference data |
| `filter_use` | Search | Apply any filter | Which filters matter most |

---

## 9. GitHub Pages Hosting

### Why GitHub Pages
- **Free** for public repositories
- 100 GB bandwidth/month (more than enough for Phase 1-3)
- Custom domain support with HTTPS (via Let's Encrypt)
- Deploy via `gh-pages` branch or GitHub Actions
- No additional platform to manage

### GitHub Pages Limits (verified from GitHub docs)

| Limit | Value |
|-------|-------|
| Repository size | 1 GB recommended |
| Site size | 1 GB max |
| Bandwidth | 100 GB/month |
| Builds | 10 per hour |
| Custom domains | Supported with HTTPS |
| SPA routing | Needs `404.html` workaround |

### SPA Routing Fix

GitHub Pages doesn't support client-side routing natively. Add a `404.html` that redirects to `index.html`:

```bash
# In your build output, copy index.html as 404.html
cp dist/mypropertymart-web/browser/index.html dist/mypropertymart-web/browser/404.html
```

Or use the hash location strategy in Angular:

```typescript
// app.config.ts — Option A: Hash routing (simpler)
providers: [
  provideRouter(routes, withHashLocation())
]

// Option B: Keep path routing + 404.html redirect (recommended)
// The CI/CD script handles copying index.html → 404.html
```

### Deploy Steps (Manual)

```bash
# Build for production
ng build --configuration production --base-href /mypropertymart-web/

# Deploy to gh-pages branch
npx angular-cli-ghpages --dir=dist/mypropertymart-web/browser
```

### With Custom Domain

```bash
# Build without base-href (custom domain serves from root)
ng build --configuration production

# Add CNAME file to output
echo "www.mypropertymart.com" > dist/mypropertymart-web/browser/CNAME
```

---

## 10. CI/CD Pipeline

### GitHub Actions — Deploy to GitHub Pages

```yaml
# .github/workflows/deploy.yml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build -- --configuration production

      - name: Copy 404.html for SPA routing
        run: cp dist/mypropertymart-web/browser/index.html dist/mypropertymart-web/browser/404.html

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: dist/mypropertymart-web/browser

      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

> **Malaysia Build:** To build for Malaysia deployment, create a separate build configuration (`environment.my.ts`) and deploy to `app.mypropertymart.com`. Use `ng build --configuration my` with an `angular.json` configuration that replaces `environment.ts` with `environment.my.ts`.

### API Deploy (in the API repo)

```yaml
# .github/workflows/api-deploy.yml
name: Deploy API

on:
  push:
    branches: [main]
    paths: ['src/**']

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - run: dotnet restore
      - run: dotnet build --configuration Release --no-restore
      - run: dotnet test --configuration Release --no-build
      - run: dotnet publish src/MyPropertyMart.API -c Release -o ./publish

      - name: Deploy to VM via SSH
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USERNAME }}
          key: ${{ secrets.VM_SSH_KEY }}
          source: "publish/*"
          target: "/opt/mypropertymart/"

      - name: Restart API
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USERNAME }}
          key: ${{ secrets.VM_SSH_KEY }}
          script: sudo systemctl restart mypropertymart-api
```

---

## 11. Domain Strategy

### Phase 1: Development (No Custom Domain)

| Service | URL |
|---------|-----|
| API (local) | `https://localhost:5001` |
| Web (local) | `http://localhost:4200` |
| API (deployed) | `http://<vm-ip>:5000` or via Cloudflare tunnel |
| Web (deployed) | `https://<username>.github.io/mypropertymart-web` |

### Phase 1+: Custom Domain

Purchase `mypropertymart.com` (~$10-15/year from Namecheap or Cloudflare Registrar).

| Service | URL | DNS |
|---------|-----|-----|
| **Web** | `www.mypropertymart.com` | CNAME → `<username>.github.io` |
| **API** | `api.mypropertymart.com` | A record → Azure VM IP |
| **Malaysia Web** | `app.mypropertymart.com` | A record → VM IP |
| **Construction Portal** | `construction.mypropertymart.com` | A record → VM IP (P2B) |

### Cloudflare DNS Setup

```
Type    Name            Content                    Proxy
A       api             <vm-ip>                    Proxied ☁️
A       app             <vm-ip>                    Proxied ☁️   (Malaysia)
A       construction    <vm-ip>                    Proxied ☁️   (Phase 2B)
CNAME   www             <username>.github.io       DNS only
CNAME   @               www.mypropertymart.com     Proxied ☁️
```

> **Note:** GitHub Pages custom domains require the CNAME to be DNS-only (not Cloudflare proxied) OR you use Cloudflare's "Full" SSL mode. Follow [GitHub's custom domain docs](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site).

### When to Migrate Away from GitHub Pages

| Signal | Action |
|--------|--------|
| Need server-side rendering (SSR) | Move to Cloudflare Pages or Azure Static Web Apps |
| Bandwidth exceeding 100 GB/month | Move to Cloudflare Pages (unlimited bandwidth) |
| Need edge functions / middleware | Move to Cloudflare Pages or Vercel |

For Phase 1-3, GitHub Pages is more than sufficient.

---

*This document is the web UI development reference. For API & database see `04_API_AND_DATABASE.md`. For mobile API contract see `06_MOBILE_API_CONTRACT.md`.*
