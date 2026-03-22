# 🏠 Property Mart BD — API Documentation

**Document Version:** 1.0  
**Created:** March 22, 2026  
**Status:** Phase 1 – Implemented  
**Source of Truth:** This document is the single reference for all API endpoints. Both `05_WEB_UI_GUIDE.md` (Angular) and `06_MOBILE_API_CONTRACT.md` (Flutter) reference this document.

---

## Table of Contents

- [1. API Basics](#1-api-basics)
- [2. Response Format](#2-response-format)
- [3. Authentication](#3-authentication)
- [4. Enums Reference](#4-enums-reference)
- [5. Phase 1 — MVP Endpoints](#5-phase-1--mvp-endpoints)
  - [5.1 Auth](#51-auth)
  - [5.2 Listings](#52-listings)
  - [5.3 File Uploads & Images](#53-file-uploads--images)
  - [5.4 Users](#54-users)
  - [5.5 Ratings](#55-ratings)
  - [5.6 Locations](#56-locations)
  - [5.7 Reports](#57-reports)
- [6. Phase 2 — Verification & Company](#6-phase-2--verification--company)
- [7. Phase 3 — Favorites, Alerts & AI](#7-phase-3--favorites-alerts--ai)
- [8. Phase 4 — Monetization & Marketplace](#8-phase-4--monetization--marketplace)
- [9. Endpoint × Screen Matrix](#9-endpoint--screen-matrix)

---

## 1. API Basics

| Item | Value |
|------|-------|
| **Base URL (dev)** | `http://localhost:5000/api/v1` |
| **Base URL (prod)** | `https://api.propertymartbd.com/api/v1` |
| **Format** | JSON — `camelCase` properties, string enums, nulls omitted |
| **Auth** | Firebase ID Token in `Authorization: Bearer <token>` header |
| **Pagination** | `?page=1&pageSize=20` (max 50 per page) |
| **Sorting** | `?sortBy=price&sortOrder=asc` |
| **Dates** | ISO 8601 UTC — `2026-03-20T10:00:00Z` |
| **IDs** | GUID (`550e8400-e29b-41d4-a716-446655440000`) for entities, `int` for locations |
| **Swagger** | `https://api.propertymartbd.com/swagger` (dev only) |

### HTTP Status Codes

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE, status change |
| 400 | Bad Request | Validation errors |
| 401 | Unauthorized | Missing or invalid Firebase token |
| 403 | Forbidden | Not the resource owner |
| 404 | Not Found | Entity does not exist |
| 429 | Rate Limited | Too many requests |
| 500 | Server Error | Unhandled exception |

---

## 2. Response Format

### Standard Response Envelope — `ApiResponse<T>`

Every endpoint returns this envelope:

```json
{
  "success": true,
  "data": { ... },
  "errors": [],
  "message": null
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

### Paginated Response — `PagedResponse<T>`

Endpoints returning collections use this extended envelope:

```json
{
  "success": true,
  "data": [ ... ],
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

---

## 3. Authentication

### Flow

```
1. User signs in via Google OAuth or Phone OTP using Firebase SDK
2. Firebase SDK returns an ID Token (auto-refreshes ~every 1 hour)
3. App calls POST /api/v1/auth/login with token in Authorization header
4. API validates token server-side → creates or returns user
5. All subsequent requests include: Authorization: Bearer <firebase-id-token>
6. On 401 → force token refresh via Firebase SDK, retry
```

### Auth Levels

| Level | Description |
|-------|-------------|
| **Public** | No token required. Anyone can call. |
| **Optional Auth** | Token accepted but not required. Behavior may vary with/without auth. |
| **Required** | Token required. Returns 401 without valid token. |
| **Owner** | Token required + must own the resource. Returns 403 if not owner. |
| **Seller/Agent** | Token required + `userType` must be `Seller` or `Agent`. Returns 403 otherwise. |
| **Admin** | Token required + `userType` must be `Admin`. |

---

## 4. Enums Reference

All enums are serialized as **strings** in JSON (e.g., `"Land"`, not `0`).

### UserType
`Seller` · `Agent` · `Buyer` · `Company` · `Admin`

### PropertyType
`Land` · `Apartment` · `Flat` · `Shop` · `EmptySpace`

### ListingType
`Sale` · `Rent`

### ListingStatus
`Active` · `Sold` · `Taken` · `Draft` · `Expired`

### LandSizeUnit
`Katha` · `Shotangsha` · `Bigha` · `Decimal` · `Acre` · `SquareFeet`

### ReportReason
`Fake` · `Fraud` · `Inappropriate` · `Duplicate` · `Other`

### ReportStatus
`Pending` · `Reviewed` · `ActionTaken` · `Dismissed`

### VerificationStatus (Phase 2)
`Pending` · `Approved` · `Rejected`

---

## 5. Phase 1 — MVP Endpoints

### 5.1 Auth

---

#### `POST /auth/login` — Login / Register via Google

| | |
|---|---|
| **Auth** | Required (Firebase Google ID Token) |
| **Web Pages** | Login |
| **Mobile Screens** | Login |

**Request:**
```
Headers: Authorization: Bearer <firebase-google-id-token>
```
```json
{
  "userType": "Seller"
}
```
> `userType` is optional — only used on first login to set user type. Accepted: `Seller`, `Buyer`, `Agent`.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@gmail.com",
    "phoneNumber": null,
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

---

#### `POST /auth/phone-login` — Login / Register via Phone OTP

| | |
|---|---|
| **Auth** | Required (Firebase Phone ID Token) |
| **Web Pages** | Phone Login |
| **Mobile Screens** | Phone Login |

**Request:**
```
Headers: Authorization: Bearer <firebase-phone-id-token>
```
```json
{
  "userType": "Seller"
}
```
> Same behavior and response as `/auth/login`. Firebase Phone Auth produces the same type of ID token; the API validates them identically.

**Response 200:** Same as `/auth/login`.

---

#### `POST /auth/link-phone` — Link Phone Number to Account

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Edit Profile |
| **Mobile Screens** | Edit Profile |

**Request:**
```json
{
  "phoneNumber": "+8801712345678"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "email": "user@gmail.com",
    "phoneNumber": "+8801712345678",
    "fullName": "Ahmed Khan",
    "fullNameBn": "আহমেদ খান",
    "profileImageUrl": "...",
    "userType": "Seller",
    "isVerified": false,
    "bio": null,
    "preferredLanguage": "bn",
    "averageRating": 4.5,
    "totalRatings": 12,
    "createdAt": "2026-03-20T10:00:00Z"
  }
}
```

---

#### `GET /auth/me` — Get Current User Profile

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Profile tab, Navbar (user info) |
| **Mobile Screens** | Profile tab, Navbar |

**Response 200:** Same as `/auth/link-phone` response (full `UserProfileResponse`).

---

### 5.2 Listings

---

#### `GET /listings` — Search / Browse Listings

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Home, Search Results |
| **Mobile Screens** | Home, Search |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `divisionId` | int? | Filter by division |
| `districtId` | int? | Filter by district |
| `upazilaId` | int? | Filter by upazila |
| `minPrice` | decimal? | Minimum price (BDT) |
| `maxPrice` | decimal? | Maximum price (BDT) |
| `minSize` | decimal? | Minimum land size |
| `maxSize` | decimal? | Maximum land size |
| `sizeUnit` | string? | Size unit for min/max filters |
| `propertyType` | string? | `Land`, `Apartment`, `Flat`, `Shop`, `EmptySpace` |
| `listingType` | string? | `Sale`, `Rent` |
| `search` | string? | Text search (title, titleBn, areaName) |
| `sortBy` | string? | `price`, `size`, `views` (default: `createdAt` desc) |
| `sortOrder` | string? | `asc`, `desc` |
| `page` | int | Page number (default: 1) |
| `pageSize` | int | Items per page (default: 20, max: 50) |

**Example:**
```
GET /listings?divisionId=2&districtId=10&minPrice=500000&maxPrice=2000000&sortBy=price&sortOrder=asc&page=1&pageSize=20
```

**Response 200:**
```json
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
      "primaryImageUrl": "https://api.propertymartbd.com/files/listing-images/2026/03/abc123-photo1.jpg",
      "thumbnailUrl": "https://api.propertymartbd.com/files/listing-images/2026/03/abc123-photo1_thumb.jpg",
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

---

#### `GET /listings/{id}` — Listing Detail

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Listing Detail |
| **Mobile Screens** | Listing Detail |

> Increments `viewCount` on each request.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "title": "৫ কাঠা জমি বিক্রয়",
    "titleBn": "৫ কাঠা জমি বিক্রয়",
    "description": "সুন্দর জায়গায় ৫ কাঠা জমি...",
    "descriptionBn": "সুন্দর জায়গায় ৫ কাঠা জমি...",
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
    "areaNameBn": "কান্দিরপাড়",
    "mouza": "কান্দিরপাড়",
    "fullAddress": "Kandirpar, Cumilla Sadar",
    "fullAddressBn": "কান্দিরপাড়, কুমিল্লা সদর",
    "latitude": 23.4607,
    "longitude": 91.1809,
    "khatianNumber": "123",
    "dagNumber": "456",
    "images": [
      {
        "id": "guid",
        "imageUrl": "https://api.propertymartbd.com/files/listing-images/2026/03/abc123-photo1.jpg",
        "thumbnailUrl": "https://api.propertymartbd.com/files/listing-images/2026/03/abc123-photo1_thumb.jpg",
        "displayOrder": 0,
        "isPrimary": true
      }
    ],
    "viewCount": 46,
    "contactCount": 5,
    "createdAt": "2026-03-15T08:00:00Z",
    "updatedAt": "2026-03-18T12:00:00Z",
    "seller": {
      "id": "guid",
      "fullName": "Ahmed Khan",
      "fullNameBn": "আহমেদ খান",
      "profileImageUrl": "https://...",
      "userType": "Seller",
      "isVerified": false,
      "bio": "Property dealer in Cumilla",
      "phoneNumber": "+8801712345678",
      "averageRating": 4.5,
      "totalRatings": 12
    }
  }
}
```

---

#### `POST /listings` — Create Listing

| | |
|---|---|
| **Auth** | Seller / Agent only |
| **Web Pages** | Create Listing |
| **Mobile Screens** | Create Listing (Add tab) |

**Request:**
```json
{
  "title": "৫ কাঠা জমি বিক্রয়",
  "titleBn": "৫ কাঠা জমি বিক্রয়",
  "description": "সুন্দর জায়গায় ৫ কাঠা জমি...",
  "descriptionBn": "সুন্দর জায়গায় ৫ কাঠা জমি...",
  "propertyType": "Land",
  "listingType": "Sale",
  "price": 5000000,
  "landSize": 5.0,
  "landSizeUnit": "Katha",
  "divisionId": 2,
  "districtId": 10,
  "upazilaId": 85,
  "areaName": "Kandirpar",
  "areaNameBn": "কান্দিরপাড়",
  "mouza": "কান্দিরপাড়",
  "fullAddress": "Kandirpar, Cumilla Sadar",
  "fullAddressBn": "কান্দিরপাড়, কুমিল্লা সদর",
  "latitude": 23.4607,
  "longitude": 91.1809,
  "khatianNumber": "123",
  "dagNumber": "456"
}
```

**Validation:**

| Field | Rule |
|-------|------|
| `title` | Required, max 200 chars |
| `description` | Required, max 5000 chars |
| `propertyType` | Required, valid enum |
| `listingType` | Required, valid enum |
| `price` | Required, > 0 |
| `divisionId` | Required |
| `districtId` | Required |
| `upazilaId` | Required |

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "new-guid",
    "title": "৫ কাঠা জমি বিক্রয়",
    "price": 5000000,
    "pricePerUnit": 1000000,
    "propertyType": "Land",
    "listingType": "Sale",
    "status": "Active",
    ...
  }
}
```

---

#### `PUT /listings/{id}` — Update Listing

| | |
|---|---|
| **Auth** | Owner only |
| **Web Pages** | Edit Listing |
| **Mobile Screens** | Edit Listing (from My Listings) |

**Request:** Same fields as create — all optional (partial update). Only send fields you want to change.

```json
{
  "price": 5500000,
  "landSize": 6.0
}
```

**Response 200:** Updated `ListingResponse`.

---

#### `DELETE /listings/{id}` — Soft Delete Listing

| | |
|---|---|
| **Auth** | Owner only |
| **Web Pages** | My Listings (delete action) |
| **Mobile Screens** | My Listings (swipe delete) |

**Response 204:** No content.

---

#### `PATCH /listings/{id}/status` — Change Listing Status

| | |
|---|---|
| **Auth** | Owner only |
| **Web Pages** | My Listings (status toggle) |
| **Mobile Screens** | My Listings |

**Request:**
```json
{
  "status": "Sold"
}
```
> Allowed values: `Active`, `Sold`, `Taken`, `Draft`

**Response 204:** No content.

---

### 5.3 File Uploads & Images

---

#### `POST /uploads` — Upload File

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Create/Edit Listing (photo upload), Edit Profile |
| **Mobile Screens** | Create Listing (Step 3: Photos), Edit Profile |

> **2-Step Flow:** First upload the file here, then attach it to a listing via `POST /listings/{id}/images`.

**Request:**
```
Content-Type: multipart/form-data
```

| Field | Type | Description |
|-------|------|-------------|
| `file` | binary | The file to upload |
| `purpose` | string (query) | `listing-images`, `nid-images`, `videos` |

**File Limits:**

| Purpose | Max Size | Allowed Types |
|---------|----------|---------------|
| `listing-images` | 5 MB | `image/jpeg`, `image/png`, `image/webp` |
| `nid-images` | 5 MB | `image/jpeg`, `image/png`, `image/webp` |
| `videos` | 50 MB | `video/mp4` |

> **Client-side:** Compress/resize images before upload — max 1920px width, 80% JPEG quality. Max 10 images per listing.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "filePath": "listing-images/2026/03/abc123-photo1.jpg",
    "url": "https://api.propertymartbd.com/files/listing-images/2026/03/abc123-photo1.jpg",
    "sizeBytes": 245000
  }
}
```

---

#### `POST /listings/{listingId}/images` — Attach Image to Listing

| | |
|---|---|
| **Auth** | Owner only |
| **Web Pages** | Create/Edit Listing |
| **Mobile Screens** | Create Listing (Step 3) |

**Request:**
```json
{
  "filePath": "listing-images/2026/03/abc123-photo1.jpg",
  "displayOrder": 0,
  "isPrimary": true
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "image-guid",
    "imageUrl": "https://api.propertymartbd.com/files/listing-images/2026/03/abc123-photo1.jpg",
    "thumbnailUrl": null,
    "displayOrder": 0,
    "isPrimary": true
  }
}
```

---

#### `DELETE /listings/{listingId}/images/{imageId}` — Delete Image

| | |
|---|---|
| **Auth** | Owner only |
| **Web Pages** | Edit Listing |
| **Mobile Screens** | Edit Listing |

**Response 204:** No content.

---

#### `PATCH /listings/{listingId}/images/reorder` — Reorder Images

| | |
|---|---|
| **Auth** | Owner only |
| **Web Pages** | Edit Listing (drag & drop) |
| **Mobile Screens** | Edit Listing |

**Request:**
```json
{
  "images": [
    { "imageId": "guid-1", "displayOrder": 0, "isPrimary": true },
    { "imageId": "guid-2", "displayOrder": 1, "isPrimary": false },
    { "imageId": "guid-3", "displayOrder": 2, "isPrimary": false }
  ]
}
```

**Response 204:** No content.

---

### 5.4 Users

---

#### `GET /users/{id}` — Public User Profile

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | User Profile |
| **Mobile Screens** | Profile (Public) |

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "email": "user@gmail.com",
    "phoneNumber": "+8801712345678",
    "fullName": "Ahmed Khan",
    "fullNameBn": "আহমেদ খান",
    "profileImageUrl": "https://...",
    "userType": "Seller",
    "isVerified": false,
    "bio": "Property dealer in Cumilla",
    "preferredLanguage": "bn",
    "averageRating": 4.5,
    "totalRatings": 12,
    "createdAt": "2026-03-20T10:00:00Z"
  }
}
```

---

#### `PUT /users/me` — Update Own Profile

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Edit Profile |
| **Mobile Screens** | Edit Profile |

**Request:** All fields optional (partial update).
```json
{
  "fullName": "Updated Name",
  "fullNameBn": "আপডেটেড নাম",
  "bio": "Property agent in Cumilla",
  "phoneNumber": "+8801712345678",
  "preferredLanguage": "bn"
}
```

**Validation:**

| Field | Rule |
|-------|------|
| `fullName` | Max 200 chars |
| `fullNameBn` | Max 200 chars |
| `bio` | Max 1000 chars |
| `phoneNumber` | Valid Bangladesh number (`+880XXXXXXXXXX`) |
| `preferredLanguage` | `bn` or `en` |

**Response 200:** Full `UserProfileResponse`.

---

#### `GET /users/{id}/listings` — User's Listings

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | User Profile (listing grid), My Listings |
| **Mobile Screens** | Profile (Public), My Listings |

**Query Parameters:**

| Parameter | Type | Default |
|-----------|------|---------|
| `page` | int | 1 |
| `pageSize` | int | 20 |

**Response 200:** Paginated `ListingResponse[]` (same as search response format).

---

#### `GET /users/{id}/ratings` — User's Ratings

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | User Profile (rating section) |
| **Mobile Screens** | Profile (Public) |

**Query Parameters:**

| Parameter | Type | Default |
|-----------|------|---------|
| `page` | int | 1 |
| `pageSize` | int | 20 |

**Response 200:**
```json
{
  "success": true,
  "data": [
    {
      "id": "guid",
      "ratedUserId": "seller-guid",
      "listingId": "listing-guid",
      "score": 4,
      "comment": "Very responsive seller",
      "raterName": "Anonymous Buyer",
      "createdAt": "2026-03-18T10:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 12,
    "totalPages": 1,
    "hasPrevious": false,
    "hasNext": false
  }
}
```

---

### 5.5 Ratings

---

#### `POST /ratings` — Create Rating

| | |
|---|---|
| **Auth** | Public (Phase 1), Required (Phase 3+) |
| **Web Pages** | Listing Detail, User Profile |
| **Mobile Screens** | Listing Detail, Profile (Public) |

> Phase 1: Anonymous ratings allowed. IP hash stored for spam prevention.

**Request:**
```json
{
  "ratedUserId": "seller-guid",
  "listingId": "listing-guid",
  "score": 4,
  "comment": "Very responsive seller",
  "raterName": "Anonymous Buyer"
}
```

**Validation:**

| Field | Rule |
|-------|------|
| `ratedUserId` | Required, must exist |
| `listingId` | Optional, must exist if provided |
| `score` | Required, 1–5 |
| `comment` | Optional, max 1000 chars |
| `raterName` | Optional, max 100 chars |

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "ratedUserId": "seller-guid",
    "listingId": "listing-guid",
    "score": 4,
    "comment": "Very responsive seller",
    "raterName": "Anonymous Buyer",
    "createdAt": "2026-03-20T10:00:00Z"
  }
}
```

---

### 5.6 Locations

---

#### `GET /locations/divisions` — All Divisions

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Home, Search, Create/Edit Listing |
| **Mobile Screens** | Home, Search, Create Listing |

> Cached server-side for 24 hours. Cache client-side — location data rarely changes.

**Response 200:**
```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "Barishal", "nameBn": "বরিশাল" },
    { "id": 2, "name": "Chattogram", "nameBn": "চট্টগ্রাম" },
    { "id": 3, "name": "Dhaka", "nameBn": "ঢাকা" },
    { "id": 4, "name": "Khulna", "nameBn": "খুলনা" },
    { "id": 5, "name": "Mymensingh", "nameBn": "ময়মনসিংহ" },
    { "id": 6, "name": "Rajshahi", "nameBn": "রাজশাহী" },
    { "id": 7, "name": "Rangpur", "nameBn": "রংপুর" },
    { "id": 8, "name": "Sylhet", "nameBn": "সিলেট" }
  ]
}
```

---

#### `GET /locations/divisions/{divId}/districts` — Districts in Division

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Search, Create/Edit Listing (cascading dropdown) |
| **Mobile Screens** | Search, Create Listing |

**Response 200:**
```json
{
  "success": true,
  "data": [
    { "id": 10, "name": "Cumilla", "nameBn": "কুমিল্লা" },
    { "id": 11, "name": "Chandpur", "nameBn": "চাঁদপুর" }
  ]
}
```

---

#### `GET /locations/districts/{distId}/upazilas` — Upazilas in District

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Search, Create/Edit Listing (cascading dropdown) |
| **Mobile Screens** | Search, Create Listing |

**Response 200:**
```json
{
  "success": true,
  "data": [
    { "id": 85, "name": "Cumilla Sadar", "nameBn": "কুমিল্লা সদর" },
    { "id": 86, "name": "Cumilla Sadar Dakshin", "nameBn": "কুমিল্লা সদর দক্ষিণ" }
  ]
}
```

---

### 5.7 Reports

---

#### `POST /reports` — Report a Listing

| | |
|---|---|
| **Auth** | Optional (logged-in user ID stored if present) |
| **Web Pages** | Listing Detail (report button) |
| **Mobile Screens** | Listing Detail (report button) |

> IP hash stored for spam prevention.

**Request:**
```json
{
  "listingId": "listing-guid",
  "reason": "Fake",
  "comment": "This property doesn't exist"
}
```

| Field | Rule |
|-------|------|
| `listingId` | Required |
| `reason` | Required — `Fake`, `Fraud`, `Inappropriate`, `Duplicate`, `Other` |
| `comment` | Optional |

**Response 201:**
```json
{
  "success": true,
  "data": {
    "message": "Report submitted"
  }
}
```

---

## 6. Phase 2 — Verification & Company

> These endpoints will be implemented in Phase 2. Schemas are already present in the database.

| Method | Endpoint | Auth | Description | Web Page | Mobile Screen |
|--------|----------|------|-------------|----------|---------------|
| POST | `/verifications` | Required | Submit NID (front/back images) | NID Verification | NID Verification |
| GET | `/verifications/me` | Required | Check own verification status | NID Verification | NID Verification |
| POST | `/companies` | Company | Create company profile | Create Company | — |
| GET | `/companies/{id}` | Public | Company profile + listings | Company Profile | Company Profile |
| PUT | `/companies/{id}` | Owner | Update company profile | Create Company | — |
| POST | `/listings/{id}/video` | Owner | Attach uploaded video to listing | Edit Listing | Edit Listing |
| GET | `/listings/price-suggestion` | Public | Price range suggestion for area | Create Listing | Create Listing |
| GET | `/admin/verifications` | Admin | List pending verifications | Admin Dashboard | — |
| PATCH | `/admin/verifications/{id}` | Admin | Approve/reject verification | Admin Review | — |

---

## 7. Phase 3 — Favorites, Alerts & AI

| Method | Endpoint | Auth | Description | Web Page | Mobile Screen |
|--------|----------|------|-------------|----------|---------------|
| GET | `/favorites` | Required | Get saved listings | Favorites | Favorites |
| POST | `/favorites` | Required | Save a listing | Listing Detail | Listing Detail |
| DELETE | `/favorites/{listingId}` | Required | Remove from favorites | Favorites | Favorites |
| POST | `/favorites/sync` | Required | Sync local favorites to server | — | (background) |
| GET | `/search-alerts` | Required | Get saved search alerts | Search Alerts | Search Alerts |
| POST | `/search-alerts` | Required | Create search alert | Search | Search |
| DELETE | `/search-alerts/{id}` | Required | Delete search alert | Search Alerts | Search Alerts |
| POST | `/ai/predict-price` | Public | ML price prediction | Create Listing | Create Listing |
| POST | `/ai/voice-search` | Public | Speech → search params | — | Search |
| GET | `/listings/nearby` | Public | Nearby listings by lat/lng | Map View | Map View |

---

## 8. Phase 4 — Monetization & Marketplace

| Method | Endpoint | Auth | Description | Web Page | Mobile Screen |
|--------|----------|------|-------------|----------|---------------|
| POST | `/payments/featured-listing` | Required | Pay for featured listing | Feature Listing | Feature Listing |
| POST | `/subscriptions` | Required | Subscribe to agent/company plan | Subscription Plans | Subscription Plans |
| GET | `/suppliers` | Public | Browse suppliers | Supplier Marketplace | Supplier Marketplace |
| POST | `/suppliers` | Supplier | Register as supplier | Supplier Marketplace | — |
| GET | `/blogs` | Public | Blog articles list | Blog Listing | Blog |
| GET | `/blogs/{slug}` | Public | Single blog article | Blog Detail | Blog |
| POST | `/ai/chat` | Public | AI chatbot | AI Chat | AI Chat |
| GET | `/recommendations` | Required | Personalized listing recommendations | Home | Home |

---

## 9. Endpoint × Screen Matrix

### Web Pages (Angular)

| Web Page | Endpoints Used |
|----------|---------------|
| **Home** | `GET /listings`, `GET /locations/divisions` |
| **Search Results** | `GET /listings`, `GET /locations/*` (all 3) |
| **Listing Detail** | `GET /listings/{id}`, `POST /ratings`, `POST /reports` |
| **Create Listing** | `POST /listings`, `POST /uploads`, `POST /listings/{id}/images`, `GET /locations/*` |
| **Edit Listing** | `PUT /listings/{id}`, `POST /uploads`, `POST /listings/{id}/images`, `DELETE /listings/{id}/images/{imageId}`, `PATCH /listings/{id}/images/reorder`, `GET /locations/*` |
| **My Listings** | `GET /users/{id}/listings`, `DELETE /listings/{id}`, `PATCH /listings/{id}/status` |
| **User Profile** | `GET /users/{id}`, `GET /users/{id}/listings`, `GET /users/{id}/ratings` |
| **Edit Profile** | `PUT /users/me`, `POST /auth/link-phone` |
| **Login** | `POST /auth/login` |
| **Phone Login** | `POST /auth/phone-login` |
| **Navbar** | `GET /auth/me` |

### Mobile Screens (Flutter)

| Mobile Screen | Endpoints Used |
|---------------|---------------|
| **Home** | `GET /listings`, `GET /locations/divisions` |
| **Search** | `GET /listings`, `GET /locations/*` (all 3) |
| **Listing Detail** | `GET /listings/{id}`, `POST /ratings`, `POST /reports` |
| **Create Listing** | `POST /listings`, `POST /uploads`, `POST /listings/{id}/images`, `GET /locations/*` |
| **My Listings** | `GET /users/{id}/listings`, `DELETE /listings/{id}`, `PATCH /listings/{id}/status` |
| **Profile (Public)** | `GET /users/{id}`, `GET /users/{id}/listings`, `GET /users/{id}/ratings` |
| **Edit Profile** | `PUT /users/me`, `POST /auth/link-phone` |
| **Login** | `POST /auth/login` |
| **Phone Login** | `POST /auth/phone-login` |
| **Settings** | `GET /auth/me` |

### Endpoints by Auth Level

| Auth Level | Endpoints |
|------------|-----------|
| **Public** | `GET /listings`, `GET /listings/{id}`, `GET /users/{id}`, `GET /users/{id}/listings`, `GET /users/{id}/ratings`, `GET /locations/*`, `POST /ratings` (P1), `POST /reports` |
| **Required** | `POST /auth/login`, `POST /auth/phone-login`, `POST /auth/link-phone`, `GET /auth/me`, `PUT /users/me`, `POST /uploads` |
| **Owner** | `POST /listings`, `PUT /listings/{id}`, `DELETE /listings/{id}`, `PATCH /listings/{id}/status`, `POST /listings/{id}/images`, `DELETE /listings/{id}/images/{imageId}`, `PATCH /listings/{id}/images/reorder` |

---

*This document is the single API reference for all clients. For web UI implementation see `05_WEB_UI_GUIDE.md`. For mobile implementation see `06_MOBILE_API_CONTRACT.md`.*
