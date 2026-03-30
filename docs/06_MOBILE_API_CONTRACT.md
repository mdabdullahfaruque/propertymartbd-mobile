# 🏠 My Property Mart
### Mobile API Contract & Screens Plan

**Document Version:** 1.2  
**Created:** March 20, 2026  
**Last Updated:** March 30, 2026  
**Audience:** Mobile Developer (Flutter)  
**Status:** Phase 1 – Complete, Phase 2 – Planning

---

## Table of Contents

- [1. API Overview](#1-api-overview)
- [2. Authentication Flow](#2-authentication-flow)
- [3. API Endpoints – Phase 1](#3-api-endpoints--phase-1)
- [4. API Endpoints – Phase 2-4](#4-api-endpoints--phase-2-4)
- [5. Mobile Screens Plan – All Phases](#5-mobile-screens-plan--all-phases)
- [6. Navigation Structure](#6-navigation-structure)
- [7. Flutter Project Hints](#7-flutter-project-hints)
- [8. Offline & Performance Strategy](#8-offline--performance-strategy)
- [9. Google Analytics 4 (Mobile)](#9-google-analytics-4-mobile)

---

## 1. API Overview

| Item | Value |
|------|-------|
| **Base URL (dev)** | `http://<vm-ip>:5000/api/v1` |
| **Base URL (prod)** | `https://api.mypropertymart.com/api/v1` |
| **Format** | JSON (camelCase properties) |
| **Auth** | Firebase ID Token in `Authorization: Bearer <token>` header |
| **Pagination** | `?page=1&pageSize=20` |
| **Sorting** | `?sortBy=price&sortOrder=asc` |
| **Swagger** | `https://api.mypropertymart.com/swagger` |
| **Dates** | ISO 8601 (`2026-03-20T10:00:00Z`) |
| **Enums** | Strings (e.g., `"Land"`, `"Active"`) |

### Standard Response Envelope

```json
{
  "success": true,
  "data": { ... },
  "errors": [],
  "message": null,
  "meta": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 150,
    "totalPages": 8,
    "hasPrevious": false,
    "hasNext": true
  }
}
```

### Error Response

```json
{
  "success": false,
  "data": null,
  "errors": ["Listing not found"],
  "message": null
}
```

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 400 | Bad Request (validation errors) |
| 401 | Unauthorized (missing/invalid token) |
| 403 | Forbidden (not the owner) |
| 404 | Not Found |
| 429 | Rate Limited |
| 500 | Server Error |

---

## 2. Authentication Flow

```
1. User taps "Sign in with Google" or "Sign in with Email/Password"
2. Firebase SDK handles Google OAuth / Email-Password auth → returns Firebase ID Token
3. App calls any authenticated endpoint (e.g. GET /api/v1/auth/me)
   Headers: { Authorization: "Bearer <firebase-id-token>" }
4. API validates token → auto-creates user in DB on first request using Firebase profile (name, email, photo). **Default role: Seller**
5. App stores Firebase ID token (SDK auto-refreshes ~every 1 hour)
6. All subsequent API calls include: Authorization: Bearer <token>
7. On 401 response → call user.getIdToken(true) to force refresh
8. Optionally call POST /api/v1/auth/login with { "userType": "Agent" } to change user type
9. User can also change role via PUT /api/v1/users/me with { "userType": "Agent" }
```

> **Auth Note (v1.2):** Phase 1-2 uses **Email/Password + Google Sign-In** only ($0 cost). Phone OTP login is deferred to Phase 2C (construction clients only) and Phase 3+ (public users) due to Firebase SMS costs (~$0.06/SMS to Bangladesh).

### Flutter Packages
- `firebase_auth` — Firebase Authentication (Email/Password + Google)
- `google_sign_in` — Google Sign-In
- Phone OTP packages deferred to Phase 3

---

## 3. API Endpoints – Phase 1

### Authentication

**POST** `/auth/login` — Login / Update User Type
```
Headers: Authorization: Bearer <firebase-id-token>
Body: { "userType": "Seller" }  // Optional — updates user type

Response 200:
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@gmail.com",
    "fullName": "Ahmed Khan",
    "fullNameBn": "আহমেদ খান",
    "profileImageUrl": "https://lh3.googleusercontent.com/...",
    "userType": "Seller",
    "isVerified": false,
    "bio": null,
    "preferredLanguage": "bn",
    "createdAt": "2026-03-20T10:00:00Z"
  }
}
```

> **Note:** User is auto-created by the middleware on first authenticated request. This endpoint is for retrieving the auth response and optionally updating `userType`.

**GET** `/auth/me` — Get current user (Auth required)

---

#### ~~POST `/auth/phone-login` — Phone OTP Login~~ [Deferred to Phase 3]

> **Moved to Phase 3 (v1.2).** Phone OTP login deferred due to Firebase SMS costs. Same behavior and response as `/auth/login` — Firebase Phone Auth produces the same type of ID token. See Phase 3 endpoints.

#### ~~POST `/auth/link-phone` — Link Phone to Existing Account~~ [Deferred to Phase 3]

> **Moved to Phase 3 (v1.2).** Link phone number to existing Google/Email account. Deferred along with phone auth.

---

### Listings

**GET** `/listings` — Search / Browse
```
Query: ?divisionId=2&districtId=10&upazilaId=85&minPrice=500000&maxPrice=2000000
       &minSize=3&maxSize=10&sizeUnit=Katha&sortBy=price&sortOrder=asc
       &page=1&pageSize=20

Response 200:
{
  "success": true,
  "data": [
    {
      "id": "guid",
      "title": "৫ কাঠা জমি বিক্রয়",
      "titleBn": "৫ কাঠা জমি বিক্রয়",
      "price": 5000000,
      "pricePerUnit": 1000000,
      "landSize": 5.0,
      "landSizeUnit": "Katha",
      "propertyType": "Land",
      "listingType": "Sale",
      "status": "Active",
      "divisionName": "Chattogram",
      "divisionNameBn": "চট্টগ্রাম",
      "districtName": "Cumilla",
      "districtNameBn": "কুমিল্লা",
      "upazilaName": "Cumilla Sadar",
      "upazilaNameBn": "কুমিল্লা সদর",
      "areaName": "Kandirpar",
      "primaryImageUrl": "https://api.mypropertymart.com/files/listing-images/guid/photo.jpg",
      "thumbnailUrl": "https://api.mypropertymart.com/files/listing-images/guid/photo_thumb.jpg",
      "viewCount": 45,
      "createdAt": "2026-03-15T08:00:00Z",
      "seller": {
        "id": "guid",
        "fullName": "Ahmed Khan",
        "profileImageUrl": "https://...",
        "isVerified": false,
        "averageRating": 4.5
      }
    }
  ],
  "meta": { "page": 1, "pageSize": 20, "totalCount": 150, "totalPages": 8 }
}
```

**GET** `/listings/{id}` — Listing Detail
```
Response includes all fields from search PLUS:
- description, descriptionBn
- latitude, longitude
- mouza, khatianNumber, dagNumber
- fullAddress, fullAddressBn
- images: [{ id, imageUrl, thumbnailUrl, displayOrder, isPrimary }]
- contactCount, viewCount
- isOwner, contactNumber (required when isOwner=false), ownerName, ownerContactNumber
- primaryImageUrl, thumbnailUrl (root level, repeat for stability)
- seller: { full profile with ratings }
```

**POST** `/listings` — Create Listing (Auth: Required)
```
Body:
{
  "title": "৫ কাঠা জমি বিক্রয়",
  "titleBn": "৫ কাঠা জমি বিক্রয়",
  "description": "সুন্দর জায়গায় ৫ কাঠা জমি...",
  "propertyType": "Land",
  "listingType": "Sale",
  "price": 5000000,
  "landSize": 5.0,
  "landSizeUnit": "Katha",
  "divisionId": 2,
  "districtId": 10,
  "upazilaId": 85,
  "areaName": "Kandirpar",
  "mouza": "কান্দিরপাড়",
  "latitude": 23.4607,
  "longitude": 91.1809,
  "isOwner": true,              // default: true. Set false if listing as agent
  "contactNumber": null,        // required when isOwner=false (agent's own number)
  "ownerName": null,            // optional, for agent-posted listings
  "ownerContactNumber": null    // optional, for agent-posted listings
}

Response 201: { "success": true, "data": { "id": "new-guid", ... } }
```

**PUT** `/listings/{id}` — Update Listing (Auth: Owner only)

**DELETE** `/listings/{id}` — Soft Delete (Auth: Owner only)

**PATCH** `/listings/{id}/status` — Change Status
```
Body: { "status": "Sold" }  // Active, Sold, Taken, Draft
```

**GET** `/listings/nearby?lat=23.46&lng=91.18&radiusKm=10` — Nearby Listings

---

### Image Upload (2-Step Flow)

**Step 1:** Upload file directly to API
```
POST /uploads
Auth: Required
Content-Type: multipart/form-data
Fields:
  - file: <binary image data>
  - purpose: "listing-image"   // listing-image | nid-front | nid-back | profile-photo

Response: {
  "success": true,
  "data": {
    "filePath": "listing-images/2024/06/abc123-photo1.jpg",
    "url": "https://api.mypropertymart.com/files/listing-images/2024/06/abc123-photo1.jpg",
    "sizeBytes": 245000
  }
}
```

**Step 2:** Attach image to listing
```
POST /listings/{listingId}/images
Auth: Owner only
Body: { "filePath": "listing-images/2024/06/abc123-photo1.jpg", "displayOrder": 0, "isPrimary": true }

Response 201: Image metadata saved
```

> **Note:** Files are stored on the VM disk (Phase 1-2) and served by Nginx as static files.
> URLs will remain stable after migration to Azure Blob Storage (Phase 3+) — the API handles URL generation.

**DELETE** `/listings/{listingId}/images/{imageId}` — Delete Image

**PATCH** `/listings/{listingId}/images/reorder` — Reorder Images

> **Important:** Compress/resize images on the client before upload. Max 1920px width, 80% JPEG quality, max 5 MB per image, max 10 images per listing.

---

### Users

**GET** `/users/{id}` — Public Profile

**PUT** `/users/me` — Update Own Profile (Auth required)
```
Body: {
  "fullName": "Updated Name",
  "bio": "Property agent in Cumilla",
  "phoneNumber": "+8801712345678",
  "userType": "Agent"           // optional — switch between "Seller" and "Agent"
}
```

**GET** `/users/{id}/listings?page=1&pageSize=20&isOwner=true` — User's Listings
> Optional `isOwner` filter: `true` = listings as seller/owner, `false` = listings as agent. Omit for all.

**GET** `/users/{id}/ratings` — User's Ratings

---

### Ratings

**POST** `/ratings` — Create Rating (P1: Public, P3+: Auth required)
```
Body: {
  "ratedUserId": "seller-guid",
  "listingId": "listing-guid",
  "score": 4,
  "comment": "Very responsive seller",
  "raterName": "Anonymous Buyer"
}
```

---

### Locations

**GET** `/locations/divisions` — All 8 divisions  
**GET** `/locations/divisions/{divId}/districts` — Districts in division  
**GET** `/locations/districts/{distId}/upazilas` — Upazilas in district

```
Response: {
  "success": true,
  "data": [
    { "id": 85, "name": "Cumilla Sadar", "nameBn": "কুমিল্লা সদর" },
    { "id": 86, "name": "Cumilla Sadar Dakshin", "nameBn": "কুমিল্লা সদর দক্ষিণ" }
  ]
}
```

---

### Reports

**POST** `/reports` — Report a Listing (Public)
```
Body: { "listingId": "guid", "reason": "Fake", "comment": "This property doesn't exist" }
```

---

## 4. API Endpoints – Phase 2-4

### Phase 2A — Core + Materials Marketplace

**Verification & Company:**

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/verifications` | Required | Submit NID (upload front/back) [P3] |
| GET | `/verifications/me` | Required | Check own verification status [P3] |
| POST | `/companies` | Company | Create company profile |
| GET | `/companies/{id}` | Public | Company profile + listings |
| PUT | `/companies/{id}` | Owner | Update company |
| POST | `/listings/{id}/video` | Owner | Confirm uploaded video [P2B construction, P3 public] |
| GET | `/listings/price-suggestion?districtId=10&upazilaId=85` | Public | Price range suggestion |

**Materials Marketplace:**

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/materials/categories` | Public | List material categories |
| GET | `/materials` | Public | Search/browse materials |
| POST | `/materials` | Dealer | Create material price listing |
| PUT | `/materials/{id}` | Owner | Update material price |
| GET | `/materials/{id}/price-history` | Public | Price history |
| GET | `/materials/compare` | Public | Compare prices across dealers |
| POST | `/materials/calculator` | Public | Calculate total construction cost |

**Wishlist:**

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/wishlist` | Required | Get wishlisted properties & materials |
| POST | `/wishlist/properties` | Required | Add property to wishlist |
| POST | `/wishlist/materials` | Required | Add material to wishlist |
| DELETE | `/wishlist/properties/{id}` | Required | Remove property |
| DELETE | `/wishlist/materials/{id}` | Required | Remove material |

**Malaysia Locations:**

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/locations/malaysia/states` | Public | Malaysian states |
| GET | `/locations/malaysia/states/{id}/districts` | Public | Malaysian districts |

### Phase 2B — Construction Portal

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/construction/companies` | Constructor | Register company |
| GET | `/construction/companies/{id}` | Public | Company profile |
| POST | `/construction/projects` | Constructor | Create project |
| GET | `/construction/projects/{id}` | Auth | Project detail |
| GET | `/construction/clients/me/projects` | ConstrClient | Client's projects |
| GET | `/construction/clients/me/payments` | ConstrClient | Client's payments |
| POST | `/construction/projects/{id}/progress` | Constructor | Update progress |
| GET | `/construction/projects/{id}/progress` | Auth | Progress dashboard |
| POST | `/construction/projects/{id}/notices` | Constructor | Post notice |
| GET | `/construction/projects/{id}/notices` | Auth | Notice board |
| POST | `/construction/calculator` | Public | Flat price calculator |

### Phase 3

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/auth/phone-login` | Firebase Phone Token | Phone OTP login [P3] |
| POST | `/auth/link-phone` | Required | Link phone number to account [P3] |
| GET/POST/DELETE | `/search-alerts` | Required | Manage search alerts |
| POST | `/ai/predict-price` | Public | ML price prediction |
| POST | `/ai/voice-search` | Public | Speech → search params |

### Phase 4

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/payments/featured-listing` | Required | Pay for featured |
| POST | `/subscriptions` | Required | Subscribe to plan |
| GET | `/blogs` | Public | Blog articles |
| POST | `/ai/chat` | Public | AI chatbot |
| GET | `/recommendations` | Required | Personalized listings |

---

## 5. Mobile Screens Plan – All Phases

### ✅ Phase 1 — MVP Screens (COMPLETE)

| # | Screen | Navigation | Auth | Key UI Elements |
|---|--------|-----------|------|-----------------|
| 1 | **Splash** | App launch → Home | None | Logo animation, auto-login check |
| 2 | **Home** | Tab 1 | Public | Search bar, location quick-select, recent listings grid, "নতুন" badge |
| 3 | **Search / Filter** | Tab 2 or from Home search | Public | Filter sheet (location/price/size), listing cards, sort, infinite scroll |
| 4 | **Listing Detail** | Tap listing card | Public | Image carousel (swipe), all property data, mini map, seller card, WhatsApp button, share sheet, report |
| 5 | **Create Listing** | Tab 3 (+ button) | Required | Step 1: Details → Step 2: "Is Owner" toggle (if No: show fields for agent's contact number, property owner name, and owner contact number) → Step 3: Location + Map Pin → Step 4: Photos (camera/gallery, up to 10) → Step 5: Preview & Publish |
| 6 | **My Listings** | Tab 4 | Required | **Two tabs: "As Seller" and "As Agent"** (`?isOwner=true/false`). Cards show `primaryImageUrl`, `thumbnailUrl`, and status badges. Swipe to edit/delete. |
| 7 | **Profile (Public)** | Tap seller name anywhere | Public | Photo, name, type, verified badge, rating stars, bio, listing grid |
| 8 | **Edit Profile** | From Profile tab | Required | Name, phone, bio, **user type (Seller/Agent dropdown)**, profile photo |
| 9 | **Login** | Redirect when auth needed | Public | Email/Password form + Google sign-in button, platform intro |
| 10 | **Settings** | Tab 5 → gear icon | Public | Language toggle, about, contact, **profile picture dropdown** (Profile, My Listings, Logout) |

### 🟡 Phase 2A — Additional Screens

| # | Screen | Auth | Key UI Elements |
|---|--------|------|-----------------|
| 12 | **NID Verification** | Required | Camera capture NID front/back, upload progress, status tracker |
| 13 | **Company Profile** | Public | Company logo, info, listing grid |
| 14 | **Video Player** | Public | In listing detail — inline video with controls |
| 15 | **Materials Marketplace** | Public | Category tabs, material cards with prices, sort/filter, compare prices |
| 16 | **Material Detail** | Public | Price comparison across dealers, provider list, add to wishlist |
| 17 | **Dealer Dashboard** | Dealer | Manage material prices, add/edit items |
| 18 | **Dealer Registration** | Required | Register as material dealer form |
| 19 | **Property & Materials Wishlist** | Required | Wishlisted properties and materials, price change notifications |
| 20 | **Construction Cost Calculator** | Public | Select materials, enter quantities, total cost calculation |

### 🟡 Phase 2B — Additional Screens

| # | Screen | Auth | Key UI Elements |
|---|--------|------|-----------------|
| 18 | **Construction Portal** | Constructor | Company dashboard — projects list, client overview |
| 19 | **Project Management** | Constructor | Manage project details, clients, installments |
| 20 | **Flat Price Calculator** | Constructor | Input costs, calculate per sqft price, profit margin |
| 21 | **Client Portal** | ConstrClient | Project progress, payment status, notices |
| 22 | **Client Payments** | ConstrClient | Installment schedule, payment proof upload |

### 🟠 Phase 3 — Additional Screens

| # | Screen | Auth | Key UI Elements |
|---|--------|------|-----------------|| 23 | **NID Verification** | Required | Camera capture NID front/back, upload progress, status tracker |
| 24 | **Phone Login** | Public | Phone input (+880), OTP input (6 digits), countdown timer, resend button |
| 25 | **Video Player** | Public | In listing detail — inline video with controls || 26 | **Notifications** | Required | Push notification list |
| 27 | **Map View** | Public | Full-screen map with markers |
| 28 | **Search Alerts** | Required | Saved searches, toggle active/inactive |

### 🔴 Phase 4 — Additional Screens

| # | Screen | Auth | Key UI Elements |
|---|--------|------|-----------------|
| 29 | **Feature Listing** | Owner | Select duration, pay via bKash/Nagad |
| 30 | **Subscription Plans** | Required | Plan comparison cards |
| 31 | **Blog** | Public | Article cards, reader |
| 32 | **AI Chat** | Public | Chat interface |

---

## 6. Navigation Structure

### Bottom Navigation Bar (5 tabs)

```
┌────────┬────────┬────────┬────────┬────────┐
│  🏠    │  🔍    │   ➕   │  📋    │  👤    │
│ Home   │ Search │  Add   │  My    │Profile │
│        │        │Listing │Listings│        │
└────────┴────────┴────────┴────────┴────────┘
```

| Tab | Screen | Notes |
|-----|--------|-------|
| 🏠 Home | Home / landing | Default tab |
| 🔍 Search | Search + filters | With sort and infinite scroll |
| ➕ Add | Create listing | Requires auth (redirect to login if not) |
| 📋 My Listings | Seller dashboard | Requires auth |
| 👤 Profile | Own profile / Login | Shows login if not authenticated |

### Navigation Flow

```
Home → Listing Detail → Seller Profile → Seller's Listings → Another Listing Detail
Home → Search → Filter → Results → Listing Detail → WhatsApp Contact
Profile → Edit Profile
My Listings → Edit Listing → Update Photos → Save
Login → Google Auth / Email Auth → Return to Previous Screen
```

> **Note:** The bottom navigation above applies to the main app. Construction Portal and Client Portal may have their own separate navigation when accessed via `construction.mypropertymart.com`.

---

## 7. Flutter Project Hints

### Recommended Packages

| Package | Purpose |
|---------|---------|
| `firebase_auth` | Authentication (Email/Password + Google; Phone OTP in P3) |
| `google_sign_in` | Google SSO |
| `dio` | HTTP client (better than http for interceptors) |
| `flutter_map` + `latlong2` | OpenStreetMap (free, no API key) |
| `cached_network_image` | Image caching |
| `image_picker` + `image_cropper` | Camera/gallery + crop |
| `flutter_image_compress` | Compress before upload |
| `shared_preferences` | Local storage (favorites P1, settings) |
| `firebase_analytics` | Google Analytics 4 |
| `firebase_messaging` | Push notifications (P3) |
| `flutter_localizations` | Bangla i18n |
| `url_launcher` | Open WhatsApp, phone calls |
| `share_plus` | Share listing via WhatsApp/other apps |
| `shimmer` | Loading skeleton animations |
| `fl_chart` | Interactive charts for progress dashboard, finance charts [P2B] |
| `intl` | Malay/Bengali number/currency formatting |
| `flutter_html` | Render notice board content |

### Key Implementation Notes

- **Image compression:** Resize to max 1920px width, JPEG 80% quality before upload
- **Offline favorites (P1):** Use `shared_preferences` to store favorite listing IDs locally
- **Pull-to-refresh:** On Home and Search screens
- **Infinite scroll:** Load next page when user scrolls near bottom
- **Deep linking:** Support `https://mypropertymart.com/listings/{id}` opening directly in app
- **RTL support:** Not needed for Bangla (Bangla is LTR like English)
- **Multi-language support:** Use Flutter localization for Bangla, English, Bahasa Malay
- **Language config:** Language shown depends on `IsMalaysia` config from API
- **Construction portal:** Screens use charts/graphs for progress visualization (`fl_chart`)

---

## 8. Offline & Performance Strategy

| Strategy | Implementation |
|----------|---------------|
| **Image caching** | `cached_network_image` — cache thumbnails and full images |
| **API response caching** | Cache location data locally (divisions/districts/upazilas change very rarely) |
| **Local favorites** | Store in `shared_preferences` until P3 server sync |
| **Recently viewed** | Store last 20 listing IDs locally |
| **Error handling** | Show cached data with "offline" banner when no internet |
| **Retry logic** | Auto-retry failed API calls (exponential backoff via Dio interceptor) |
| **Lazy loading** | Load listing images on scroll, not all at once |
| **Thumbnail first** | Show `thumbnailUrl` in lists, `imageUrl` in detail view |

---

## 9. Google Analytics 4 (Mobile)

### Setup

1. Add `firebase_analytics` package to Flutter project
2. GA4 is automatically linked when you create a Firebase project with Analytics enabled
3. No separate Measurement ID needed — Firebase handles it

### Implementation

```dart
// Initialize in main.dart
final analytics = FirebaseAnalytics.instance;

// Track screen views
await analytics.logScreenView(screenName: 'home', screenClass: 'HomeScreen');

// Track custom events
await analytics.logEvent(name: 'listing_view', parameters: {
  'listing_id': listingId,
  'property_type': 'Land',
  'district': 'Cumilla',
});

await analytics.logEvent(name: 'whatsapp_click', parameters: {
  'listing_id': listingId,
  'seller_id': sellerId,
});

await analytics.logEvent(name: 'search', parameters: {
  'district': 'Cumilla',
  'min_price': 500000,
  'max_price': 2000000,
});
```

### Key Events to Track

| Event | Parameters | When |
|-------|-----------|------|
| `screen_view` | `screen_name` | Every screen navigation |
| `listing_view` | `listing_id`, `property_type`, `district` | Open listing detail |
| `listing_create` | `property_type`, `listing_type` | Publish listing |
| `search` | `district`, `price_range`, `size_range` | Execute search |
| `whatsapp_click` | `listing_id`, `seller_id` | Click WhatsApp contact |
| `share` | `listing_id`, `method` | Share listing |
| `login` | `method` (google/email) | User logs in |
| `filter_use` | `filter_type` | Apply search filter |
| `image_upload` | `count` | Upload listing photos |
| `material_view` | `material_id`, `category` | View material detail |
| `material_compare` | `category`, `count` | Compare materials |
| `wishlist_add` | `type` (property/material), `id` | Add to wishlist |
| `calculator_use` | `type` (cost/flat_price) | Use calculator |
| `construction_progress_view` | `project_id` | View project progress |

---

*This document is the mobile developer's primary reference. For full API implementation details see `04_API_AND_DATABASE.md`. For web UI guide see `05_WEB_UI_GUIDE.md`.*
