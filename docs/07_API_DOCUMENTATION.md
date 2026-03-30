# 🏠 My Property Mart — API Documentation

**Document Version:** 2.0  
**Created:** March 22, 2026  
**Last Updated:** March 31, 2026  
**Status:** Phase 1 – Complete, Phase 2A – Complete, Phase 2B – Complete  
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
- [6. Phase 2A — Materials Marketplace, Wishlist & Malaysia](#6-phase-2a--materials-marketplace-wishlist--malaysia)
  - [6.1 Materials](#61-materials)
  - [6.1.1 Material Dealers](#611-material-dealers)
  - [6.2 Wishlist](#62-wishlist)
  - [6.3 Malaysia Locations](#63-malaysia-locations)
  - [6.4 Verification & Company](#64-verification--company)
- [7. Phase 2B — Construction Company Portal](#7-phase-2b--construction-company-portal)
- [8. Phase 3 — Alerts & AI](#8-phase-3--alerts--ai)
- [9. Phase 4 — Monetization](#9-phase-4--monetization)
- [10. Endpoint × Screen Matrix](#10-endpoint--screen-matrix)

---

## 1. API Basics

| Item | Value |
|------|-------|
| **Base URL (dev)** | `http://localhost:5000/api/v1` |
| **Base URL (prod)** | `https://api.mypropertymart.com/api/v1` |
| **Format** | JSON — `camelCase` properties, string enums, nulls omitted |
| **Auth** | Firebase ID Token in `Authorization: Bearer <token>` header |
| **Pagination** | `?page=1&pageSize=20` (max 50 per page) |
| **Sorting** | `?sortBy=price&sortOrder=asc` |
| **Dates** | ISO 8601 UTC — `2026-03-20T10:00:00Z` |
| **IDs** | GUID (`550e8400-e29b-41d4-a716-446655440000`) for entities, `int` for locations |
| **Swagger** | `https://api.mypropertymart.com/swagger` (dev only) |

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
1. User signs in via Email/Password or Google OAuth using Firebase SDK
2. Firebase SDK returns an ID Token (auto-refreshes ~every 1 hour)
3. App sends any request with: Authorization: Bearer <firebase-id-token>
4. API middleware validates token → auto-creates user in DB on first request (using Firebase profile: name, email, photo)
5. All subsequent requests include: Authorization: Bearer <firebase-id-token>
6. On 401 → force token refresh via Firebase SDK, retry
7. Optionally call POST /api/v1/auth/login with { "userType": "Seller" } to change user type
```

> **Auth Note (v1.2):** Phase 1-2 uses **Email/Password + Google Sign-In** only ($0 cost). Phone OTP login deferred to Phase 2C (construction clients) and Phase 3+ (public) due to Firebase SMS costs (~$0.06/SMS to Bangladesh).

### Auth Levels

| Level | Description |
|-------|-------------|
| **Public** | No token required. Anyone can call. |
| **Optional Auth** | Token accepted but not required. Behavior may vary with/without auth. |
| **Required** | Token required. Returns 401 without valid token. |
| **Owner** | Token required + must own the resource. Returns 403 if not owner. |
| **Seller/Agent** | Token required + `userType` must be `Seller` or `Agent`. Returns 403 otherwise. |
| **Dealer** | Token required + `userType` must be `Dealer`. |
| **Constructor** | Token required + `userType` must be `Constructor`. |
| **ConstrClient** | Token required + `userType` must be `ConstructionClient`. |
| **Admin** | Token required + `userType` must be `Admin`. |

---

## 4. Enums Reference

All enums are serialized as **strings** in JSON (e.g., `"Land"`, not `0`).

### UserType
`Seller` (default) · `Agent` · `Buyer` · `Company` · `Admin` · `Dealer` · `Constructor` · `ConstructionClient`

> New users are auto-created as **Seller** on first login. Users can switch to **Agent** via `PUT /users/me` or `POST /auth/login`.

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

### VerificationStatus (Phase 3)
`Pending` · `Approved` · `Rejected`

### MaterialCategory
`Cement` · `RodSteel` · `Bricks` · `Sand` · `Tiles` · `StoneChips` · `Paint` · `Plumbing` · `Electrical` · `Wood` · `Glass` · `Others`

### MaterialUnit
`Bag` · `Kg` · `Piece` · `CubicFt` · `SquareFt` · `Ton` · `Liter` · `Rod` · `Bundle` · `Other`

### ProjectStatus (Phase 2B)
`Planning` · `InProgress` · `Completed`

### InstallmentStatus (Phase 2B)
`Pending` · `Paid` · `Overdue`

### CountryCode
`BD` · `MY`

---

## 5. Phase 1 — MVP Endpoints

### 5.1 Auth

---

#### `POST /auth/login` — Login / Update User Type

| | |
|---|---|
| **Auth** | Required (Firebase ID Token — Email/Password or Google) |
| **Web Pages** | Login |
| **Mobile Screens** | Login |

**Request:**
```
Headers: Authorization: Bearer <firebase-id-token>
```
```json
{
  "userType": "Seller"
}
```
> `userType` is optional — updates the user's type if provided. Accepted: `Seller`, `Agent`. User is auto-created as **Seller** by middleware on first authenticated request.

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

#### `POST /auth/phone-login` — Login / Update User Type via Phone OTP [Phase 3]

| | |
|---|---|
| **Auth** | Required (Firebase Phone ID Token) |
| **Web Pages** | Phone Login [Phase 3] |
| **Mobile Screens** | Phone Login [Phase 3] |

> **Deferred to Phase 3 (v1.2).** Phone OTP login deferred due to Firebase SMS costs (~$0.06/SMS to Bangladesh). Same behavior and response as `/auth/login`. Firebase Phone Auth produces the same type of ID token; the API validates them identically. User is auto-created on first request. Phase 2C enables this for construction clients only.

**Request:**
```
Headers: Authorization: Bearer <firebase-phone-id-token>
```
```json
{
  "userType": "Seller"
}
```

**Response 200:** Same as `/auth/login`.

---

#### `POST /auth/link-phone` — Link Phone Number to Account [Phase 3]

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Edit Profile [Phase 3] |
| **Mobile Screens** | Edit Profile [Phase 3] |

> **Deferred to Phase 3 (v1.2).** Link phone number to existing Google/Email account. Available when phone auth is enabled in Phase 3.

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
      "primaryImageUrl": "https://api.mypropertymart.com/files/listing-images/2026/03/abc123-photo1.jpg",
      "thumbnailUrl": "https://api.mypropertymart.com/files/listing-images/2026/03/abc123-photo1_thumb.jpg",
      "viewCount": 45,
      "createdAt": "2026-03-15T08:00:00Z",
      "isOwner": true,
      "contactNumber": null,
      "ownerName": null,
      "ownerContactNumber": null,
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
        "imageUrl": "https://api.mypropertymart.com/files/listing-images/2026/03/abc123-photo1.jpg",
        "thumbnailUrl": "https://api.mypropertymart.com/files/listing-images/2026/03/abc123-photo1_thumb.jpg",
        "displayOrder": 0,
        "isPrimary": true
      }
    ],
    "viewCount": 46,
    "contactCount": 5,
    "createdAt": "2026-03-15T08:00:00Z",
    "updatedAt": "2026-03-18T12:00:00Z",
    "isOwner": true,
    "contactNumber": null,
    "ownerName": null,
    "ownerContactNumber": null,
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
| **Auth** | Required (any authenticated user; default role is Seller) |
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
  "dagNumber": "456",
  "isOwner": true,
  "contactNumber": null,
  "ownerName": null,
  "ownerContactNumber": null
}
```

**Owner / Agent Contact Fields:**

| Field | Description |
|-------|-------------|
| `isOwner` | `true` (default) = user is the property owner. `false` = user is posting as an agent |
| `contactNumber` | Agent's own contact number. **Required when `isOwner = false`**. Max 20 chars |
| `ownerName` | Property owner's name. Optional. Max 200 chars |
| `ownerContactNumber` | Property owner's phone. Optional. Max 20 chars |

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
| `contactNumber` | Required when `isOwner = false`, max 20 chars |

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
    "isOwner": true,
    "contactNumber": null,
    "ownerName": null,
    "ownerContactNumber": null,
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

**Request:** Same fields as create — all optional (partial update). Only send fields you want to change. Includes `isOwner`, `contactNumber`, `ownerName`, `ownerContactNumber`.

```json
{
  "price": 5500000,
  "landSize": 6.0,
  "isOwner": false,
  "contactNumber": "+8801712345678"
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
    "url": "https://api.mypropertymart.com/files/listing-images/2026/03/abc123-photo1.jpg",
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
    "imageUrl": "https://api.mypropertymart.com/files/listing-images/2026/03/abc123-photo1.jpg",
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
  "preferredLanguage": "bn",
  "userType": "Agent"
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
| `userType` | `Seller` or `Agent` |

**Response 200:** Full `UserProfileResponse`.

---

#### `GET /users/{id}/listings` — User's Listings

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | User Profile (listing grid), My Listings |
| **Mobile Screens** | Profile (Public), My Listings |

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | int | 1 | Page number |
| `pageSize` | int | 20 | Items per page (max 50) |
| `isOwner` | bool? | null | Filter: `true` = listings as seller/owner, `false` = listings as agent. Omit for all. Used for "My Listings" tabs |

**Response 200:** Paginated `ListingResponse[]` (same as search response format). Includes `isOwner`, `contactNumber`, `ownerName`, `ownerContactNumber` fields.

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

## 6. Phase 2A — Materials Marketplace, Wishlist & Malaysia

### 6.1 Materials

---

#### `GET /materials/categories` — List Material Categories

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Materials Marketplace |
| **Mobile Screens** | Materials Marketplace |

**Response 200:**
```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "Cement", "nameBn": "সিমেন্ট", "nameMs": "Simen", "iconUrl": null, "displayOrder": 1 },
    { "id": 2, "name": "Rod/Steel", "nameBn": "রড/স্টিল", "nameMs": "Rod/Keluli", "iconUrl": null, "displayOrder": 2 }
  ]
}
```

---

#### `GET /materials` — Search / Browse Materials

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Materials Marketplace |
| **Mobile Screens** | Materials Marketplace |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `categoryId` | int? | Filter by material category |
| `minPrice` | decimal? | Minimum price |
| `maxPrice` | decimal? | Maximum price |
| `dealerId` | guid? | Filter by dealer |
| `countryCode` | string? | `BD`, `MY` |
| `search` | string? | Text search |
| `sortBy` | string? | `price`, `rating`, `newest` |
| `sortOrder` | string? | `asc`, `desc` |
| `page` | int | Page number (default: 1) |
| `pageSize` | int | Items per page (default: 20, max: 50) |

**Response 200:** Paginated list of material listings with dealer info.

---

#### `GET /materials/{id}` — Get Material Listing Detail

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Material Detail |
| **Mobile Screens** | Material Detail |

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "categoryId": 1,
    "categoryName": "Cement",
    "itemName": "Crown Cement (50kg bag)",
    "itemNameBn": "ক্রাউন সিমেন্ট (৫০ কেজি ব্যাগ)",
    "brand": "Crown",
    "price": 480,
    "currency": "BDT",
    "unit": "Bag",
    "imageUrl": "materials/guid/photo.jpg",
    "description": "Premium quality Portland cement",
    "countryCode": "BD",
    "dealer": {
      "id": "guid",
      "shopName": "Rahim Building Materials",
      "address": "Kandirpar, Cumilla",
      "phoneNumber": "+8801712345678",
      "isVerified": false
    },
    "createdAt": "2026-03-20T10:00:00Z",
    "updatedAt": "2026-03-25T10:00:00Z"
  }
}
```

---

#### `POST /materials` — Create Material Price Listing

| | |
|---|---|
| **Auth** | Dealer |
| **Web Pages** | Dealer Dashboard |
| **Mobile Screens** | Dealer Dashboard |

**Request:**
```json
{
  "categoryId": 1,
  "itemName": "Crown Cement (50kg bag)",
  "itemNameBn": "ক্রাউন সিমেন্ট (৫০ কেজি ব্যাগ)",
  "brand": "Crown",
  "price": 480,
  "currency": "BDT",
  "unit": "Bag",
  "description": "Premium quality Portland cement"
}
```

**Response 201:** Created material listing object.

---

#### `PUT /materials/{id}` — Update Material Price

| | |
|---|---|
| **Auth** | Owner (Dealer who created it) |
| **Web Pages** | Dealer Dashboard |
| **Mobile Screens** | Dealer Dashboard |

> Price changes are tracked in `material_price_history`.

**Request:** Same fields as create — all optional (partial update). Only send fields you want to change.

**Response 200:** Updated material listing object.

---

#### `GET /materials/{id}/price-history` — Price History

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Material Detail |
| **Mobile Screens** | Material Detail |

**Response 200:**
```json
{
  "success": true,
  "data": [
    { "oldPrice": 450, "newPrice": 480, "changedAt": "2026-03-15T10:00:00Z" },
    { "oldPrice": 420, "newPrice": 450, "changedAt": "2026-02-10T08:00:00Z" }
  ]
}
```

---

#### `GET /materials/compare` — Compare Prices Across Dealers

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Materials Marketplace |
| **Mobile Screens** | Materials Marketplace |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `categoryId` | int? | Filter by category |
| `itemName` | string? | Item name to compare |
| `countryCode` | string? | `BD`, `MY` |

**Response 200:** List of same/similar items from different dealers, sorted by price.

---

#### `POST /materials/calculator` — Construction Cost Calculator

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Cost Calculator |
| **Mobile Screens** | Cost Calculator |

**Request:**
```json
{
  "items": [
    { "materialListingId": "guid", "quantity": 100, "unitPrice": 480 },
    { "materialListingId": "guid", "quantity": 500, "unitPrice": 85 },
    { "customItem": "Labor Cost", "amount": 500000 }
  ]
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "items": [
      { "name": "Crown Cement (50kg)", "quantity": 100, "unitPrice": 480, "subtotal": 48000 },
      { "name": "60 Grade Rod (12mm)", "quantity": 500, "unitPrice": 85, "subtotal": 42500 },
      { "name": "Labor Cost", "quantity": 1, "unitPrice": 500000, "subtotal": 500000 }
    ],
    "totalCost": 590500,
    "currency": "BDT"
  }
}
```

---

### 6.1.1 Material Dealers

---

#### `POST /materials/dealers/register` — Register as Material Dealer

| | |
|---|---|
| **Auth** | Required (user becomes Dealer) |
| **Web Pages** | Dealer Registration |
| **Mobile Screens** | Dealer Registration |

**Request:**
```json
{
  "shopName": "Rahim Building Materials",
  "shopNameBn": "রহিম বিল্ডিং ম্যাটেরিয়ালস",
  "phoneNumber": "+8801712345678",
  "address": "Kandirpar, Cumilla",
  "addressBn": "কান্দিরপাড়, কুমিল্লা",
  "countryCode": "BD",
  "divisionId": 1,
  "districtId": 5,
  "upazilaId": 12,
  "description": "Quality building materials at fair prices"
}
```

> **Validation:** `shopName` required (max 200), `phoneNumber` required (max 20), `address` required (max 500), `countryCode` required (`BD` or `MY`). Registration changes user's `userType` to `Dealer`.

**Response 201:** Created dealer profile object.

---

#### `GET /materials/dealers/{id}` — Get Dealer Profile

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Dealer Profile |
| **Mobile Screens** | Dealer Profile |

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "userId": "guid",
    "shopName": "Rahim Building Materials",
    "shopNameBn": "রহিম বিল্ডিং ম্যাটেরিয়ালস",
    "phoneNumber": "+8801712345678",
    "address": "Kandirpar, Cumilla",
    "addressBn": "কান্দিরপাড়, কুমিল্লা",
    "countryCode": "BD",
    "isVerified": false,
    "description": "Quality building materials at fair prices",
    "createdAt": "2026-03-20T10:00:00Z"
  }
}
```

---

#### `GET /materials/dealers/me` — Get My Dealer Profile

| | |
|---|---|
| **Auth** | Dealer |
| **Web Pages** | Dealer Dashboard |
| **Mobile Screens** | Dealer Dashboard |

**Response 200:** Same response format as `GET /materials/dealers/{id}`.

---

### 6.2 Wishlist

---

#### `GET /wishlist` — Get All Wishlisted Items

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Property Wishlist |
| **Mobile Screens** | Wishlist |

**Response 200:**
```json
{
  "success": true,
  "data": {
    "properties": [ ... ],
    "materials": [ ... ],
    "dealers": [ ... ]
  }
}
```

---

#### `POST /wishlist/properties` — Add Property to Wishlist

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Listing Detail |
| **Mobile Screens** | Listing Detail |

**Request:**
```json
{
  "listingId": "guid"
}
```

**Response 201:** Confirmation.

---

#### `DELETE /wishlist/properties/{listingId}` — Remove Property from Wishlist

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Property Wishlist |
| **Mobile Screens** | Wishlist |

**Response 204:** No content.

---

#### `POST /wishlist/materials` — Add Material to Wishlist

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Material Detail |
| **Mobile Screens** | Material Detail |

**Request:**
```json
{
  "materialListingId": "guid"
}
```

**Response 201:** Confirmation.

---

#### `DELETE /wishlist/materials/{materialListingId}` — Remove Material from Wishlist

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Property Wishlist |
| **Mobile Screens** | Wishlist |

**Response 204:** No content.

---

#### `POST /wishlist/dealers` — Add Dealer to Favorites

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Dealer Profile |
| **Mobile Screens** | Dealer Profile |

**Request:**
```json
{
  "dealerId": "guid"
}
```

**Response 201:** Confirmation.

---

#### `DELETE /wishlist/dealers/{dealerId}` — Remove Dealer from Favorites

| | |
|---|---|
| **Auth** | Required |
| **Web Pages** | Property Wishlist |
| **Mobile Screens** | Wishlist |

**Response 204:** No content.

---

### 6.3 Malaysia Locations

---

#### `GET /locations/malaysia/states` — All Malaysian States

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Search (Malaysia), Create Listing (Malaysia) |
| **Mobile Screens** | Search (Malaysia), Create Listing (Malaysia) |

**Response 200:**
```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "Johor", "nameMs": "Johor" },
    { "id": 2, "name": "Kedah", "nameMs": "Kedah" },
    { "id": 3, "name": "Kelantan", "nameMs": "Kelantan" }
  ]
}
```

---

#### `GET /locations/malaysia/states/{stateId}/districts` — Districts in State

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Search (Malaysia), Create Listing (Malaysia) |
| **Mobile Screens** | Search (Malaysia), Create Listing (Malaysia) |

**Response 200:** List of districts in the specified Malaysian state.

---

### 6.4 Verification & Company

> NID verification endpoints will be implemented in **Phase 3**. Schemas are already present in the database. NID verification moved from Phase 2 to Phase 3 (v1.2).

**Phase 3 — NID Verification (Planned)**

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/verifications` | Required | Submit NID (front/back images) |
| GET | `/verifications/me` | Required | Check own verification status |
| GET | `/admin/verifications` | Admin | List pending verifications |
| PATCH | `/admin/verifications/{id}` | Admin | Approve/reject verification |

---

#### `POST /companies` — Create Company Profile

| | |
|---|---|
| **Auth** | Required (user becomes Company) |
| **Web Pages** | Create Company |
| **Mobile Screens** | — |

**Request:**
```json
{
  "companyName": "ABC Properties Ltd.",
  "companyNameBn": "এবিসি প্রপার্টিজ লিমিটেড",
  "phoneNumber": "+8801712345678",
  "email": "info@abcproperties.com",
  "address": "123 Main Road, Dhaka",
  "logoUrl": null,
  "website": "https://abcproperties.com",
  "description": "Trusted property agency in Dhaka"
}
```

> **Validation:** `companyName` required (max 200), `phoneNumber` required (max 20), `address` required (max 500). Registration changes user's `userType` to `Company`.

**Response 201:** Created company profile.

---

#### `GET /companies/{id}` — Get Company Profile

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Company Profile |
| **Mobile Screens** | Company Profile |

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "guid",
    "userId": "guid",
    "companyName": "ABC Properties Ltd.",
    "companyNameBn": "এবিসি প্রপার্টিজ লিমিটেড",
    "phoneNumber": "+8801712345678",
    "email": "info@abcproperties.com",
    "address": "123 Main Road, Dhaka",
    "logoUrl": null,
    "website": "https://abcproperties.com",
    "description": "Trusted property agency in Dhaka",
    "isVerified": false,
    "createdAt": "2026-03-20T10:00:00Z"
  }
}
```

---

#### `PUT /companies/{id}` — Update Company Profile

| | |
|---|---|
| **Auth** | Owner (Company user who created it) |
| **Web Pages** | Create Company |
| **Mobile Screens** | — |

**Request:** Same fields as create — all optional (partial update). Only send fields you want to change.

**Response 200:** Updated company profile object.

---

#### `GET /companies/{id}/listings` — Company Property Listings

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Company Profile |
| **Mobile Screens** | Company Profile |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | int | Page number (default: 1) |
| `pageSize` | int | Items per page (default: 20, max: 50) |

**Response 200:** Paginated list of property listings belonging to the company.

---

## 7. Phase 2B — Construction Company Portal

> These endpoints serve `construction.mypropertymart.com`.

---

#### `POST /construction/companies` — Register Construction Company

| | |
|---|---|
| **Auth** | Required (user becomes Constructor) |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:**
```json
{
  "companyName": "ABC Construction Ltd.",
  "companyNameBn": "এবিসি কনস্ট্রাকশন লিমিটেড",
  "phoneNumber": "+8801712345678",
  "email": "info@abcconstruction.com",
  "address": "123 Main Road, Cumilla",
  "tradeLicense": "TRAD-2026-001",
  "description": "Premier construction company in Cumilla"
}
```

**Response 201:** Created construction company object.

---

#### `GET /construction/companies/{id}` — Company Profile

| | |
|---|---|
| **Auth** | Public |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Response 200:** Full company profile with details.

---

#### `PUT /construction/companies/{id}` — Update Company

| | |
|---|---|
| **Auth** | Owner (Constructor) |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:** Same fields as create — all optional (partial update).

**Response 200:** Updated company object.

---

#### `POST /construction/projects` — Create Building Project

| | |
|---|---|
| **Auth** | Constructor |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:**
```json
{
  "projectName": "Green Heights Residence",
  "projectNameBn": "গ্রীন হাইটস রেসিডেন্স",
  "location": "Kandirpar, Cumilla",
  "totalFloors": 10,
  "totalUnits": 40,
  "landAreaSqft": 5000,
  "description": "Premium 10-story residential building",
  "startDate": "2026-06-01",
  "expectedCompletionDate": "2028-06-01"
}
```

**Response 201:** Created project object.

---

#### `GET /construction/projects/{id}` — Project Detail

| | |
|---|---|
| **Auth** | Required (Constructor or assigned Client) |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Client Portal |

**Response 200:** Full project detail with progress and client summary.

---

#### `GET /construction/projects` — List My Company's Projects

| | |
|---|---|
| **Auth** | Constructor |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Response 200:** List of all projects belonging to the authenticated constructor's company.

---

#### `PUT /construction/projects/{id}` — Update Project

| | |
|---|---|
| **Auth** | Constructor (Owner) |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:** Same fields as create — all optional (partial update).

**Response 200:** Updated project object.

---

#### `POST /construction/projects/{id}/clients` — Add Client to Project

| | |
|---|---|
| **Auth** | Constructor |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:**
```json
{
  "clientName": "Rahim Ahmed",
  "clientPhone": "+8801812345678",
  "clientEmail": "rahim@email.com",
  "unitNumber": "4A",
  "floorNumber": 4,
  "unitSizeSqft": 1200,
  "totalPrice": 6000000,
  "currency": "BDT",
  "downPayment": 1500000
}
```

**Response 201:** Created client assignment object.

---

#### `GET /construction/projects/{id}/clients` — List Project Clients

| | |
|---|---|
| **Auth** | Constructor |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Response 200:** List of clients assigned to the project.

---

#### `POST /construction/projects/{id}/installments` — Set Installment Plan for Client

| | |
|---|---|
| **Auth** | Constructor |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:**
```json
{
  "clientId": "guid",
  "installments": [
    { "installmentNumber": 1, "amount": 300000, "dueDate": "2026-09-01" },
    { "installmentNumber": 2, "amount": 300000, "dueDate": "2026-12-01" }
  ]
}
```

**Response 201:** Created installment plan.

---

#### `POST /construction/projects/{id}/payments` — Record Payment with Proof

| | |
|---|---|
| **Auth** | Constructor or ConstrClient |
| **Web Pages** | Construction Portal, Client Portal |
| **Mobile Screens** | Construction Portal, Client Portal |

**Request:**
```json
{
  "installmentId": "guid",
  "paidDate": "2026-09-05",
  "paymentProofFilePath": "payment-proofs/guid/receipt.jpg",
  "notes": "Paid via bKash"
}
```

**Response 201:** Payment record with proof URL.

---

#### `GET /construction/clients/me/projects` — Client's Assigned Projects

| | |
|---|---|
| **Auth** | ConstrClient |
| **Web Pages** | Client Portal |
| **Mobile Screens** | Client Portal |

**Response 200:** List of projects client is assigned to with progress summary.

---

#### `GET /construction/clients/me/payments` — Client's Payment Status

| | |
|---|---|
| **Auth** | ConstrClient |
| **Web Pages** | Client Portal |
| **Mobile Screens** | Client Portal |

**Response 200:** Installment schedule with paid/pending/overdue status.

---

#### `POST /construction/projects/{id}/progress` — Update Construction Progress

| | |
|---|---|
| **Auth** | Constructor |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:**
```json
{
  "floorsCompleted": 4,
  "percentageComplete": 40.5,
  "daysRemaining": 365,
  "description": "4th floor casting completed. Starting 5th floor next week.",
  "descriptionBn": "চতুর্থ তলার ঢালাই সম্পন্ন হয়েছে।",
  "imageUrl": "progress-images/guid/photo.jpg"
}
```

**Response 201:** Created progress entry.

---

#### `GET /construction/projects/{id}/progress` — Progress Dashboard Data

| | |
|---|---|
| **Auth** | Required (Constructor or ConstrClient) |
| **Web Pages** | Construction Portal, Client Portal |
| **Mobile Screens** | Client Portal |

**Response 200:**
```json
{
  "success": true,
  "data": {
    "project": { "projectName": "Green Heights", "totalFloors": 10, "totalUnits": 40 },
    "currentProgress": {
      "floorsCompleted": 4,
      "percentageComplete": 40.5,
      "daysRemaining": 365,
      "lastUpdated": "2026-09-15T10:00:00Z"
    },
    "progressHistory": [
      { "floorsCompleted": 1, "percentageComplete": 10, "description": "Foundation", "createdAt": "2026-04-01T..." },
      { "floorsCompleted": 2, "percentageComplete": 20, "description": "2nd floor done", "createdAt": "2026-06-01T..." }
    ],
    "financeSummary": {
      "totalClients": 35,
      "totalCollected": 52500000,
      "totalPending": 157500000,
      "overdueCount": 3
    }
  }
}
```

---

#### `POST /construction/projects/{id}/notices` — Post Notice

| | |
|---|---|
| **Auth** | Constructor |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:**
```json
{
  "title": "Water Supply Update",
  "titleBn": "পানি সরবরাহ আপডেট",
  "titleMs": "Kemaskini Bekalan Air",
  "content": "Water tank installation on roof completed.",
  "contentBn": "ছাদে পানির ট্যাংক স্থাপন সম্পন্ন।",
  "contentMs": "Pemasangan tangki air di bumbung selesai.",
  "isPinned": false
}
```

**Response 201:** Created notice object.

---

#### `GET /construction/projects/{id}/notices` — Notice Board

| | |
|---|---|
| **Auth** | Required (Constructor or ConstrClient) |
| **Web Pages** | Construction Portal, Client Portal |
| **Mobile Screens** | Client Portal |

**Response 200:** Paginated list of notices, pinned ones first.

---

#### `POST /construction/calculator` — Flat Price / Profit Calculator

| | |
|---|---|
| **Auth** | Public (but typically used by Constructors) |
| **Web Pages** | Construction Portal |
| **Mobile Screens** | Construction Portal |

**Request:**
```json
{
  "landPrice": 50000000,
  "landAreaSqft": 5000,
  "constructionCostPerSqft": 2500,
  "totalBuildableSqft": 40000,
  "otherCosts": 5000000,
  "costBreakdown": [
    { "item": "Foundation", "cost": 3000000 },
    { "item": "Structure", "cost": 8000000 },
    { "item": "Finishing", "cost": 5000000 }
  ],
  "sellingPricePerSqft": 5000,
  "currency": "BDT"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "totalCost": 155000000,
    "totalSellingPrice": 200000000,
    "profitAmount": 45000000,
    "profitMarginPercentage": 29.03,
    "costPerSqft": 3875,
    "sellingPricePerSqft": 5000,
    "currency": "BDT"
  }
}
```

---

## 8. Phase 3 — Alerts, AI & Verification

> Favorites have been moved to Phase 2A Wishlist (Section 6.2). NID Verification and Phone OTP Auth moved here from Phase 2/Phase 1 respectively (v1.2).

| Method | Endpoint | Auth | Description | Web Page | Mobile Screen |
|--------|----------|------|-------------|----------|---------------|
| POST | `/auth/phone-login` | Firebase Phone Token | Phone OTP login [P3] | Phone Login | Phone Login |
| POST | `/auth/link-phone` | Required | Link phone to account [P3] | Edit Profile | Edit Profile |
| GET | `/search-alerts` | Required | Get saved search alerts | Search Alerts | Search Alerts |
| POST | `/search-alerts` | Required | Create search alert | Search | Search |
| DELETE | `/search-alerts/{id}` | Required | Delete search alert | Search Alerts | Search Alerts |
| POST | `/ai/predict-price` | Public | ML price prediction | Create Listing | Create Listing |
| POST | `/ai/voice-search` | Public | Speech → search params | — | Search |
| GET | `/listings/nearby` | Public | Nearby listings by lat/lng | Map View | Map View |

---

## 9. Phase 4 — Monetization

> Supplier marketplace has been moved to Phase 2A Materials (Section 6.1).

| Method | Endpoint | Auth | Description | Web Page | Mobile Screen |
|--------|----------|------|-------------|----------|---------------|
| POST | `/payments/featured-listing` | Required | Pay for featured listing | Feature Listing | Feature Listing |
| POST | `/subscriptions` | Required | Subscribe to agent/company plan | Subscription Plans | Subscription Plans |
| GET | `/blogs` | Public | Blog articles list | Blog Listing | Blog |
| GET | `/blogs/{slug}` | Public | Single blog article | Blog Detail | Blog |
| POST | `/ai/chat` | Public | AI chatbot | AI Chat | AI Chat |
| GET | `/recommendations` | Required | Personalized listing recommendations | Home | Home |

---

## 10. Endpoint × Screen Matrix

### Web Pages (Angular)

| Web Page | Endpoints Used |
|----------|---------------|
| **Home** | `GET /listings`, `GET /locations/divisions` |
| **Search Results** | `GET /listings`, `GET /locations/*` (all 3) |
| **Listing Detail** | `GET /listings/{id}`, `POST /ratings`, `POST /reports`, `POST /wishlist/properties` |
| **Create Listing** | `POST /listings`, `POST /uploads`, `POST /listings/{id}/images`, `GET /locations/*` |
| **Edit Listing** | `PUT /listings/{id}`, `POST /uploads`, `POST /listings/{id}/images`, `DELETE /listings/{id}/images/{imageId}`, `PATCH /listings/{id}/images/reorder`, `GET /locations/*` |
| **My Listings** | `GET /users/{id}/listings?isOwner=true\|false`, `DELETE /listings/{id}`, `PATCH /listings/{id}/status` |
| **User Profile** | `GET /users/{id}`, `GET /users/{id}/listings`, `GET /users/{id}/ratings` |
| **Edit Profile** | `PUT /users/me`, `POST /auth/link-phone` [P3] |
| **Login** | `POST /auth/login` |
| **Phone Login [P3]** | `POST /auth/phone-login` [Phase 3] |
| **Navbar** | `GET /auth/me` |
| **Materials Marketplace** | `GET /materials/categories`, `GET /materials`, `GET /materials/{id}/price-history`, `GET /materials/compare` |
| **Material Detail** | `GET /materials/{id}`, `GET /materials/{id}/price-history`, `GET /materials/compare`, `POST /wishlist/materials` |
| **Dealer Registration** | `POST /materials/dealers/register` |
| **Dealer Dashboard** | `GET /materials/dealers/me`, `POST /materials`, `PUT /materials/{id}`, `GET /materials` |
| **Property Wishlist** | `GET /wishlist`, `POST/DELETE /wishlist/properties`, `POST/DELETE /wishlist/materials`, `POST/DELETE /wishlist/dealers` |
| **Cost Calculator** | `POST /materials/calculator` |
| **Create Company** | `POST /companies`, `PUT /companies/{id}` |
| **Company Profile** | `GET /companies/{id}`, `GET /companies/{id}/listings` |
| **Admin Dashboard** | `GET /materials`, `GET /wishlist` (admin view) |
| **Construction Portal** | `POST /construction/companies`, `GET /construction/companies/{id}`, `PUT /construction/companies/{id}`, `GET /construction/projects`, `POST /construction/projects`, `GET /construction/projects/{id}`, `PUT /construction/projects/{id}`, `POST /construction/projects/{id}/clients`, `GET /construction/projects/{id}/clients`, `POST /construction/projects/{id}/installments`, `POST /construction/projects/{id}/payments`, `POST /construction/projects/{id}/progress`, `GET /construction/projects/{id}/progress`, `POST /construction/projects/{id}/notices`, `GET /construction/projects/{id}/notices` |
| **Construction Calculator** | `POST /construction/calculator` |
| **Client Portal** | `GET /construction/clients/me/projects`, `GET /construction/clients/me/payments`, `GET /construction/projects/{id}/progress`, `GET /construction/projects/{id}/notices` |

### Mobile Screens (Flutter)

| Mobile Screen | Endpoints Used |
|---------------|---------------|
| **Home** | `GET /listings`, `GET /locations/divisions` |
| **Search** | `GET /listings`, `GET /locations/*` (all 3) |
| **Listing Detail** | `GET /listings/{id}`, `POST /ratings`, `POST /reports`, `POST /wishlist/properties` |
| **Create Listing** | `POST /listings`, `POST /uploads`, `POST /listings/{id}/images`, `GET /locations/*` |
| **My Listings** | `GET /users/{id}/listings?isOwner=true\|false`, `DELETE /listings/{id}`, `PATCH /listings/{id}/status` |
| **Profile (Public)** | `GET /users/{id}`, `GET /users/{id}/listings`, `GET /users/{id}/ratings` |
| **Edit Profile** | `PUT /users/me`, `POST /auth/link-phone` [P3] |
| **Login** | `POST /auth/login` |
| **Phone Login [P3]** | `POST /auth/phone-login` [Phase 3] |
| **Settings** | `GET /auth/me` |
| **Materials Marketplace** | `GET /materials/categories`, `GET /materials`, `GET /materials/{id}/price-history`, `GET /materials/compare` |
| **Material Detail** | `GET /materials/{id}`, `GET /materials/{id}/price-history`, `GET /materials/compare`, `POST /wishlist/materials` |
| **Dealer Registration** | `POST /materials/dealers/register` |
| **Dealer Dashboard** | `GET /materials/dealers/me`, `POST /materials`, `PUT /materials/{id}`, `GET /materials` |
| **Property Wishlist** | `GET /wishlist`, `POST/DELETE /wishlist/properties`, `POST/DELETE /wishlist/materials`, `POST/DELETE /wishlist/dealers` |
| **Cost Calculator** | `POST /materials/calculator` |
| **Company Profile** | `GET /companies/{id}`, `GET /companies/{id}/listings` |
| **Construction Portal** | `POST /construction/companies`, `GET /construction/companies/{id}`, `PUT /construction/companies/{id}`, `GET /construction/projects`, `POST /construction/projects`, `GET /construction/projects/{id}`, `POST /construction/projects/{id}/clients`, `GET /construction/projects/{id}/clients`, `POST /construction/projects/{id}/installments`, `POST /construction/projects/{id}/payments`, `POST /construction/projects/{id}/progress`, `GET /construction/projects/{id}/progress`, `POST /construction/projects/{id}/notices`, `GET /construction/projects/{id}/notices` |
| **Construction Calculator** | `POST /construction/calculator` |
| **Client Portal** | `GET /construction/clients/me/projects`, `GET /construction/clients/me/payments`, `GET /construction/projects/{id}/progress`, `GET /construction/projects/{id}/notices` |

### Endpoints by Auth Level

| Auth Level | Endpoints |
|------------|-----------|
| **Public** | `GET /listings`, `GET /listings/{id}`, `GET /users/{id}`, `GET /users/{id}/listings`, `GET /users/{id}/ratings`, `GET /locations/*`, `POST /ratings` (P1), `POST /reports`, `GET /materials/categories`, `GET /materials`, `GET /materials/{id}`, `GET /materials/{id}/price-history`, `GET /materials/compare`, `POST /materials/calculator`, `GET /materials/dealers/{id}`, `GET /companies/{id}`, `GET /companies/{id}/listings`, `GET /locations/malaysia/*`, `GET /construction/companies/{id}`, `POST /construction/calculator` |
| **Required** | `POST /auth/login`, `POST /auth/phone-login` [P3], `POST /auth/link-phone` [P3], `GET /auth/me`, `PUT /users/me`, `POST /uploads`, `POST /listings`, `GET/POST/DELETE /wishlist/*`, `POST /companies`, `PUT /companies/{id}`, `POST /materials/dealers/register` |
| **Owner** | `PUT /listings/{id}`, `DELETE /listings/{id}`, `PATCH /listings/{id}/status`, `POST /listings/{id}/images`, `DELETE /listings/{id}/images/{imageId}`, `PATCH /listings/{id}/images/reorder` |
| **Dealer** | `POST /materials`, `PUT /materials/{id}`, `GET /materials/dealers/me` |
| **Constructor** | `POST /construction/companies`, `PUT /construction/companies/{id}`, `GET /construction/projects`, `POST /construction/projects`, `PUT /construction/projects/{id}`, `POST /construction/projects/{id}/clients`, `GET /construction/projects/{id}/clients`, `POST /construction/projects/{id}/installments`, `POST /construction/projects/{id}/payments`, `POST /construction/projects/{id}/progress`, `GET /construction/projects/{id}/progress`, `POST /construction/projects/{id}/notices`, `GET /construction/projects/{id}/notices` |
| **ConstrClient** | `GET /construction/clients/me/projects`, `GET /construction/clients/me/payments`, `GET /construction/projects/{id}/progress`, `GET /construction/projects/{id}/notices` |

---

*This document is the single API reference for all clients. For web UI implementation see `05_WEB_UI_GUIDE.md`. For mobile implementation see `06_MOBILE_API_CONTRACT.md`.*
