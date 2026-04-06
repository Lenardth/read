# Smart Booking System — Technical Documentation


**Version:** 1.0.0
**Author:** Lenard Hlabangwana
**Stack:** Laravel 12 · PHP 8.4 · SQLite (dev) / PostgreSQL (prod) · Vite 5 · Alpine.js · TailwindCSS
**Deployment:** Vercel (serverless) + Neon PostgreSQL
**Repository:** https://github.com/Lenardth/smart-trip-system

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Use Case Diagram](#3-use-case-diagram)
4. [Use Cases (Detailed)](#4-use-cases-detailed)
5. [Class Diagram](#5-class-diagram)
6. [Database Schema](#6-database-schema)
7. [API Reference](#7-api-reference)
8. [Monetization Model](#8-monetization-model)
9. [Module Breakdown](#9-module-breakdown)
10. [Deployment Architecture](#10-deployment-architecture)

---

## 1. Project Overview

Smart Booking is an AI-powered travel planning and booking platform. Users describe their mood and preferences; the system uses the Groq LLM API to generate personalised destination recommendations, then allows users to book flights and accommodations, manage itineraries, and connect with a travel community.

### Core Features

| Feature | Description |
|---|---|
| AI Trip Planning | Groq LLM generates 5 destination suggestions based on mood, budget, duration, companion, region |
| Flight Search | Real-time flight search via AeroDataBox (RapidAPI) |
| Accommodation Booking | Browse and book stays with coupon support |
| Community | Topics, groups, stories, traveller profiles |
| Wishlist | Save destinations, track across pages |
| Media Gallery | Upload photos/videos, manage personal travel memories |
| Itineraries | AI-generated PDF itineraries |
| Notifications | Real-time via Pusher + polling fallback |
| Premium Subscription | $9.99/month — removes service fees, unlocks features |
| Coupon System | Percentage and fixed-amount promo codes |
| Agency Portal | Agency users manage their own flight listings and bookings |

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      CLIENT BROWSER                      │
│  Alpine.js · Vite-bundled JS modules · CSS per-page     │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS
┌────────────────────────▼────────────────────────────────┐
│                   VERCEL SERVERLESS                      │
│  api/index.php  →  Laravel 12 Application               │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │  Routes  │  │Middleware│  │    Controllers         │  │
│  │ web.php  │  │auth/csrf │  │ 20+ controllers        │  │
│  └──────────┘  └──────────┘  └──────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Service Layer                        │   │
│  │  PricingService · AviationstackService            │   │
│  │  GeoapifyService · AiSuggestionController         │   │
│  └──────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Neon        │ │  Groq API    │ │  AeroDataBox │
│  PostgreSQL  │ │  LLM         │ │  Flights API │
│  (prod DB)   │ │  llama-3.3   │ │  (RapidAPI)  │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Technology Stack

| Layer | Technology |
|---|---|
| Backend Framework | Laravel 12 (PHP 8.4) |
| Frontend Bundler | Vite 5 + laravel-vite-plugin 1.x |
| CSS Framework | TailwindCSS 3 + custom CSS per page |
| JS Framework | Alpine.js 3 |
| Database (dev) | SQLite |
| Database (prod) | PostgreSQL via Neon |
| AI Provider | Groq (llama-3.3-70b-versatile) |
| Flight Data | AeroDataBox via RapidAPI |
| Real-time | Pusher + Laravel Echo |
| Deployment | Vercel (PHP serverless runtime) |
| PDF Generation | jsPDF (client-side) + DomPDF (server-side) |

---

## 3. Use Case Diagram

```
                    ┌─────────────────────────────────────────┐
                    │           Smart Booking System           │
                    │                                          │
  ┌──────────┐      │  ┌─────────────────────────────────┐   │
  │  Guest   │──────┼─▶│ Browse Destinations              │   │
  │  User    │      │  │ View Landing Page                │   │
  └──────────┘      │  │ Search Flights (public)          │   │
                    │  │ Browse Accommodations            │   │
                    │  │ View Community                   │   │
                    │  └─────────────────────────────────┘   │
                    │                                          │
  ┌──────────┐      │  ┌─────────────────────────────────┐   │
  │Registered│──────┼─▶│ Register / Login                 │   │
  │  User    │      │  │ Plan AI Trip                     │   │
  │(Traveler)│      │  │ Book Flights                     │   │
  └──────────┘      │  │ Book Accommodations              │   │
       │            │  │ Apply Coupon Codes               │   │
       │            │  │ Manage Wishlist                  │   │
       │            │  │ Upload Photos/Videos             │   │
       │            │  │ View/Export Itineraries          │   │
       │            │  │ Chat with Other Users            │   │
       │            │  │ Join Community Groups            │   │
       │            │  │ Post Community Topics/Stories    │   │
       │            │  │ Manage Profile                   │   │
       │            │  │ View Notifications               │   │
       │            │  │ Subscribe to Premium             │   │
       │            │  └─────────────────────────────────┘   │
       │            │                                          │
  ┌──────────┐      │  ┌─────────────────────────────────┐   │
  │  Agency  │──────┼─▶│ All Traveler Actions             │   │
  │   User   │      │  │ Manage Flight Listings           │   │
  └──────────┘      │  │ View Agency Bookings             │   │
                    │  │ Track Commission Revenue         │   │
                    │  └─────────────────────────────────┘   │
                    │                                          │
  ┌──────────┐      │  ┌─────────────────────────────────┐   │
  │  System  │──────┼─▶│ Auto-migrate DB on deploy        │   │
  │(Vercel)  │      │  │ Seed initial data                │   │
  └──────────┘      │  │ Poll notifications (5s)          │   │
                    │  │ Apply service fees on booking    │   │
                    │  └─────────────────────────────────┘   │
                    └─────────────────────────────────────────┘
```

---

## 4. Use Cases (Detailed)

### UC-01: Plan AI Trip

| Field | Detail |
|---|---|
| **Actor** | Registered User |
| **Precondition** | User is authenticated |
| **Trigger** | User navigates to /plan-trip |
| **Main Flow** | 1. User selects mood (quick pick, community pill, or custom) 2. User fills trip details (duration, companion, budget, month) 3. User sets preferences (region, accommodation, origin, experience) 4. System POSTs to /ai/suggest with payload 5. Groq LLM returns 5 destination recommendations 6. System calculates cost breakdown per destination 7. User selects a destination 8. User views/prints receipt or saves trip to dashboard |
| **Alternate Flow** | If GROQ_API_KEY missing → error shown. If throttle exceeded → retry after 1 min |
| **Postcondition** | Trip saved to DB, appears in dashboard upcoming trips |

### UC-02: Book a Flight

| Field | Detail |
|---|---|
| **Actor** | Registered User |
| **Precondition** | User authenticated, flight search results loaded |
| **Trigger** | User clicks "Book Now" on a flight card |
| **Main Flow** | 1. User searches flights (from, to, date, passengers, class) 2. System calls AeroDataBox API via AviationstackService 3. Results displayed sorted by price/duration 4. User clicks Book Now → confirmation modal 5. System creates Booking record with passenger_details JSON 6. PricingService applies 5% service fee (waived for Premium) 7. Coupon applied if provided 8. Revenue record created |
| **Postcondition** | Booking confirmed, reference number shown, booking count incremented |

### UC-03: Subscribe to Premium

| Field | Detail |
|---|---|
| **Actor** | Registered User |
| **Precondition** | User authenticated, not already premium |
| **Trigger** | User navigates to /premium and clicks Upgrade |
| **Main Flow** | 1. Modal opens with payment reference input 2. User enters payment reference (Stripe/PayPal in production) 3. System creates Subscription record (30 days) 4. User.is_premium set to true, premium_until set 5. All future bookings skip service fee |
| **Postcondition** | User is premium, no service fees on bookings |

### UC-04: Apply Coupon Code

| Field | Detail |
|---|---|
| **Actor** | Registered User |
| **Precondition** | User on booking page with items in cart |
| **Trigger** | User enters coupon code and clicks Apply |
| **Main Flow** | 1. POST /api/coupon/validate with code + subtotal 2. System checks: active, not expired, usage limit, per-user limit, min order 3. Discount calculated (percent or fixed, capped by max_discount) 4. Price summary updates live 5. On booking submit, coupon_use record created, uses_total incremented |
| **Alternate Flow** | Invalid/expired code → error message shown |

### UC-05: Manage Community

| Field | Detail |
|---|---|
| **Actor** | Any User (some actions require auth) |
| **Main Flow** | 1. Browse topics, groups, stories, travellers 2. Authenticated users can post topics, reply, create groups 3. Users can invite others to chat from community page 4. Stories shared publicly |

### UC-06: Upload Media

| Field | Detail |
|---|---|
| **Actor** | Registered User |
| **Main Flow** | 1. User opens gallery modal on dashboard 2. Drag-drop or click to select files (images/videos up to 50MB) 3. Files uploaded to /api/media/upload 4. Stored in storage/app/public/media/{user_id}/ 5. Gallery renders with edit/delete/view actions |

---

## 5. Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          MODELS                                      │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────┐         ┌──────────────────────┐
│        User          │         │    AgencyProfile      │
├──────────────────────┤         ├──────────────────────┤
│ id: bigint PK        │1───────1│ id: bigint PK         │
│ name: string         │         │ user_id: FK           │
│ email: string unique │         │ agency_name: string   │
│ password: hashed     │         │ business_registration │
│ user_type: enum      │         │ website: string       │
│   (user|agency)      │         │ rating: decimal       │
│ is_premium: bool     │         │ total_reviews: int    │
│ premium_until: ts    │         └──────────────────────┘
│ profile_picture      │
│ agency_name          │         ┌──────────────────────┐
│ bio, phone, location │         │    Subscription       │
└──────────┬───────────┘         ├──────────────────────┤
           │                     │ id: bigint PK         │
     ┌─────┼──────────────┐      │ user_id: FK           │
     │     │              │      │ plan: enum(premium)   │
     ▼     ▼              ▼      │ amount_paid: decimal  │
┌─────────┐ ┌──────────┐ ┌────┐ │ status: enum          │
│  Trip   │ │ Booking  │ │... │ │ starts_at, ends_at    │
└─────────┘ └──────────┘ └────┘ └──────────────────────┘

┌──────────────────────┐         ┌──────────────────────┐
│         Trip         │         │       Booking         │
├──────────────────────┤         ├──────────────────────┤
│ id: bigint PK        │         │ id: bigint PK         │
│ user_id: FK          │         │ user_id: FK           │
│ title: string        │         │ flight_id: FK null    │
│ destination: string  │         │ hotel_id: FK null     │
│ country: string      │         │ trip_id: FK null      │
│ mood: string         │         │ booking_reference     │
│ feeling_note         │         │ subtotal: decimal     │
│ budget: string       │         │ discount_amount       │
│ duration: string     │         │ service_fee: decimal  │
│ companion: string    │         │ total_price: decimal  │
│ region: string       │         │ coupon_code: string   │
│ accommodation        │         │ status: enum          │
│ origin: string       │         │ passenger_details:JSON│
│ month: string        │         └──────────┬────────────┘
│ estimated_cost       │                    │
│ status: enum         │         ┌──────────▼────────────┐
│ start_date, end_date │         │    RevenueRecord       │
└──────────────────────┘         ├──────────────────────┤
                                 │ booking_id: FK        │
┌──────────────────────┐         │ user_id: FK           │
│    Destination       │         │ booking_subtotal      │
├──────────────────────┤         │ discount_amount       │
│ id: bigint PK        │         │ service_fee           │
│ name: string         │         │ agency_commission     │
│ country: string      │         │ net_revenue           │
│ region: string       │         │ coupon_code           │
│ category: string     │         └──────────────────────┘
│ mood: string         │
│ description: text    │         ┌──────────────────────┐
│ image_url: string    │         │       Coupon          │
│ price_from: decimal  │         ├──────────────────────┤
│ badge: string        │         │ id: bigint PK         │
│ is_hidden_gem: bool  │         │ code: string unique   │
│ is_active: bool      │         │ type: enum(pct|fixed) │
│ lat, lng: float      │         │ value: decimal        │
└──────────────────────┘         │ min_order: decimal    │
                                 │ max_discount: decimal │
┌──────────────────────┐         │ uses_total: int       │
│  SavedDestination    │         │ uses_limit: int null  │
├──────────────────────┤         │ uses_per_user: int    │
│ id: bigint PK        │         │ is_active: bool       │
│ user_id: FK          │         │ expires_at: timestamp │
│ destination_id: FK   │         └──────────────────────┘
│ created_at           │
└──────────────────────┘         ┌──────────────────────┐
                                 │      CouponUse        │
┌──────────────────────┐         ├──────────────────────┤
│       Media          │         │ coupon_id: FK         │
├──────────────────────┤         │ user_id: FK           │
│ id: bigint PK        │         │ booking_id: FK        │
│ user_id: FK          │         │ discount_amount       │
│ file_path: string    │         └──────────────────────┘
│ file_name: string    │
│ mime_type: string    │         ┌──────────────────────┐
│ file_size: int       │         │      TripMood         │
│ type: enum(img|vid)  │         ├──────────────────────┤
│ title: string        │         │ id: bigint PK         │
│ is_favorite: bool    │         │ label: string         │
└──────────────────────┘         │ label_normalized      │
                                 │ use_count: int        │
┌──────────────────────┐         │ created_by: FK null   │
│     Itinerary        │         │ deleted_at (soft)     │
├──────────────────────┤         └──────────────────────┘
│ id: bigint PK        │
│ user_id: FK          │         ┌──────────────────────┐
│ trip_id: FK null     │         │   CommunityTopic      │
│ title: string        │         ├──────────────────────┤
│ content: JSON        │         │ id, user_id: FK       │
│ destination          │         │ title, body           │
│ duration: int        │         │ category, tags        │
│ budget_range         │         │ views, likes          │
│ travel_style         │         └──────────────────────┘
│ generated_by: AI     │
└──────────────────────┘         ┌──────────────────────┐
                                 │   CommunityReply      │
┌──────────────────────┐         ├──────────────────────┤
│      Message         │         │ id, topic_id: FK      │
├──────────────────────┤         │ user_id: FK           │
│ id: bigint PK        │         │ body: text            │
│ sender_id: FK        │         │ likes: int            │
│ receiver_id: FK      │         └──────────────────────┘
│ body: text           │
│ read_at: timestamp   │         ┌──────────────────────┐
└──────────────────────┘         │   CommunityGroup      │
                                 ├──────────────────────┤
┌──────────────────────┐         │ id, user_id: FK       │
│   Accommodation      │         │ name, description     │
├──────────────────────┤         │ category, image_url   │
│ id: bigint PK        │         │ member_count          │
│ name: string         │         └──────────────────────┘
│ city, country        │
│ style: string        │         ┌──────────────────────┐
│ budget_tier          │         │   CommunityStory      │
│ nightly_rate: float  │         ├──────────────────────┤
│ rating: int          │         │ id, user_id: FK       │
│ lat, lng: float      │         │ title, body           │
│ image_url: string    │         │ destination           │
│ is_active: bool      │         │ image_url             │
└──────────────────────┘         └──────────────────────┘
```

### Controller → Service Relationships

```
AiSuggestionController ──uses──▶ Groq API (cURL)
FlightController        ──uses──▶ AviationstackService ──▶ AeroDataBox API
AccommodationController ──uses──▶ GeoapifyService ──▶ Geoapify API
BookingController       ──uses──▶ PricingService
CouponController        ──uses──▶ PricingService
SubscriptionController  ──manages──▶ Subscription model
DashboardController     ──aggregates──▶ Trip, Booking, Media, SavedDestination
```

---

## 6. Database Schema

### 24 Tables (21 migrations)

#### users
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| name | varchar(255) | |
| email | varchar(255) | unique |
| email_verified_at | timestamp | nullable |
| password | varchar(255) | hashed |
| user_type | enum(user,agency) | default: user |
| is_premium | boolean | default: false |
| premium_until | timestamp | nullable |
| profile_picture | varchar | nullable |
| agency_name | varchar | nullable |
| bio | varchar | nullable |
| phone | varchar | nullable |
| location | varchar | nullable |
| preferences | json | nullable |
| last_login_at | timestamp | nullable |
| last_login_ip | varchar(45) | nullable |
| remember_token | varchar | nullable |
| created_at, updated_at | timestamps | |

#### trips
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → users | cascade delete |
| destination_id | FK → destinations | nullable |
| title | varchar(255) | |
| destination | varchar(255) | nullable |
| country | varchar(255) | nullable |
| mood | varchar(100) | nullable |
| feeling_note | varchar(500) | nullable |
| budget | varchar(100) | nullable (string, not decimal) |
| duration | varchar(100) | nullable |
| companion | varchar(100) | nullable |
| region | varchar(100) | nullable |
| accommodation | varchar(100) | nullable |
| origin | varchar(255) | nullable |
| month | varchar(50) | nullable |
| estimated_cost | int unsigned | nullable |
| status | enum(planned,active,completed,cancelled) | default: planned |
| start_date | date | |
| end_date | date | |
| travelers_count | int | default: 1 |
| notes | text | nullable |
| created_at, updated_at | timestamps | |

#### bookings
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → users | |
| flight_id | bigint nullable | |
| hotel_id | bigint nullable | |
| trip_id | bigint nullable | |
| booking_reference | varchar unique | auto-generated SB-XXXXXXXX |
| seats_booked | int | default: 1 |
| subtotal | decimal(10,2) | nullable |
| discount_amount | decimal(8,2) | default: 0 |
| service_fee | decimal(8,2) | default: 0 |
| total_price | decimal(10,2) | |
| coupon_code | varchar(32) | nullable |
| status | enum(pending,confirmed,completed,cancelled) | |
| passenger_details | json | nullable |
| special_requests | text | nullable |
| created_at, updated_at | timestamps | |

#### destinations
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| continent_id | FK → continents | nullable |
| name | varchar(255) | |
| country | varchar(255) | |
| region | varchar(100) | |
| category | varchar(100) | |
| mood | varchar(100) | |
| description | text | |
| image_url | varchar | nullable |
| price_from | decimal(10,2) | nullable |
| badge | varchar(100) | nullable |
| is_hidden_gem | boolean | default: false |
| is_active | boolean | default: true |
| lat, lng | decimal(10,7) | nullable |
| created_at, updated_at | timestamps | |

#### coupons
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| code | varchar(32) unique | |
| type | enum(percent,fixed) | |
| value | decimal(8,2) | |
| min_order | decimal(8,2) | default: 0 |
| max_discount | decimal(8,2) | nullable |
| uses_total | int unsigned | default: 0 |
| uses_limit | int unsigned | nullable |
| uses_per_user | int unsigned | default: 1 |
| is_active | boolean | default: true |
| expires_at | timestamp | nullable |
| description | varchar | nullable |
| created_at, updated_at | timestamps | |

#### subscriptions
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → users | cascade |
| plan | enum(premium) | |
| amount_paid | decimal(8,2) | |
| status | enum(active,cancelled,expired) | |
| starts_at | timestamp | |
| ends_at | timestamp | |
| payment_reference | varchar | nullable |
| created_at, updated_at | timestamps | |

#### revenue_records
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| booking_id | FK → bookings | cascade |
| user_id | FK → users | cascade |
| booking_subtotal | decimal(10,2) | |
| discount_amount | decimal(8,2) | default: 0 |
| service_fee | decimal(8,2) | default: 0 |
| agency_commission | decimal(8,2) | default: 0 |
| net_revenue | decimal(10,2) | |
| coupon_code | varchar | nullable |
| meta | json | nullable |
| created_at, updated_at | timestamps | |

#### Other Tables (summary)
| Table | Purpose |
|---|---|
| continents | Geographic grouping for destinations |
| saved_destinations | Wishlist (user ↔ destination pivot) |
| messages | Direct messages between users |
| notifications | Laravel notification records |
| media | User uploaded photos/videos |
| itineraries | AI-generated trip itineraries |
| accommodations | Accommodation listings |
| accommodation_searches | Search history |
| community_topics | Forum topics |
| community_replies | Replies to topics |
| community_groups | Travel groups |
| community_stories | Travel stories |
| trip_moods | Community-contributed mood labels |
| coupon_uses | Tracks which user used which coupon |
| agency_profiles | Extended profile for agency users |
| flights | Agency-listed flight records |
| hotels | Hotel records |
| cache | Laravel cache table |
| jobs | Laravel queue jobs |

---

## 7. API Reference

### Public Endpoints (no auth required)

| Method | Path | Description |
|---|---|---|
| GET | / | Landing page |
| POST | /ai/suggest | Generate AI trip suggestions |
| GET | /accommodations | Browse accommodations |
| GET | /api/accommodations | Accommodation list (JSON) |
| GET | /api/accommodation-news | Local travel news |
| GET | /discover | Discover page |
| GET | /api/discover/destinations | Destination list |
| GET | /api/discover/hidden-gems | Hidden gem destinations |
| GET | /destinations | All destinations |
| GET | /destinations/{id} | Single destination |
| GET | /community | Community page |
| GET | /api/community/topics | Community topics |
| GET | /api/community/groups | Community groups |
| GET | /api/community/stories | Community stories |
| GET | /api/trip-moods | Community mood labels |
| GET | /flights | Flight search page |
| POST | /flights/search | Search flights |

### Authenticated Endpoints

| Method | Path | Description |
|---|---|---|
| GET | /dashboard | Dashboard page |
| GET | /api/user/statistics | Stats (photos, trips, bookings, saved) |
| GET | /api/user/recent-activity | Recent activity feed |
| GET | /api/notifications | Notification list |
| POST | /api/notifications/mark-all-read | Mark all read |
| GET | /api/trips | All user trips |
| GET | /api/trips/upcoming | Planned trips |
| POST | /api/trips | Save trip |
| DELETE | /api/trips/{id} | Delete trip |
| GET | /api/media | User media |
| POST | /api/media/upload | Upload files |
| DELETE | /api/media/delete | Delete media |
| PUT | /api/media/{id} | Update media title |
| POST | /api/bookings/flight | Book a flight |
| POST | /api/bookings/accommodation | Book accommodation |
| POST | /api/coupon/validate | Validate coupon code |
| GET | /api/subscription/status | Premium status |
| POST | /api/subscription/subscribe | Subscribe to premium |
| POST | /api/subscription/cancel | Cancel subscription |
| GET | /api/wishlist/count | Wishlist count + IDs |
| POST | /wishlist | Add to wishlist |
| DELETE | /wishlist/{id} | Remove from wishlist |
| GET | /api/conversations | Chat conversations |
| GET | /api/messages/{userId} | Message thread |
| POST | /api/messages | Send message |
| POST | /api/trip-moods | Create custom mood |
| POST | /api/trip-moods/{id}/use | Increment mood usage |

---

## 8. Monetization Model

### Revenue Streams

```
┌─────────────────────────────────────────────────────────┐
│                  Revenue Model                           │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. SERVICE FEE (5%)                                     │
│     Applied to every booking (flight + accommodation)    │
│     Waived for Premium subscribers                       │
│     Example: $500 booking → $25 service fee              │
│                                                          │
│  2. PREMIUM SUBSCRIPTION ($9.99/month)                   │
│     30-day rolling subscription                          │
│     Benefits: No service fees, priority AI, PDF export   │
│     Stored in subscriptions table                        │
│                                                          │
│  3. AGENCY COMMISSION (10%)                              │
│     Platform earns 10% on agency-listed bookings         │
│     Tracked in revenue_records.agency_commission         │
│                                                          │
│  4. COUPON SYSTEM                                        │
│     Promo codes drive acquisition (WELCOME10, SAVE20)    │
│     Percentage or fixed-amount discounts                 │
│     Per-user limits, expiry dates, usage caps            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Pricing Flow

```
Booking Subtotal
      │
      ▼
Apply Coupon Discount (if valid)
      │
      ▼
After-Discount Amount
      │
      ├── Is Premium? ──YES──▶ Service Fee = $0
      │                              │
      └── Not Premium? ──▶ Service Fee = 5%
                                     │
                                     ▼
                              Total Charged
                                     │
                                     ▼
                         Revenue Record Created
                         (service_fee + agency_commission - discount)
```

### Starter Coupons

| Code | Type | Value | Limit |
|---|---|---|---|
| WELCOME10 | percent | 10% | unlimited |
| SAVE20 | percent | 20% | 100 uses, max $50 |
| FLAT25 | fixed | $25 | min order $100 |
| SUMMER15 | percent | 15% | 200 uses |
| AGENCY5 | percent | 5% | unlimited |

---

## 9. Module Breakdown

### Frontend Modules (resources/js/blade/)

| Module | File | Responsibilities |
|---|---|---|
| App Router | app.js | Dynamic imports per URL path, Alpine init |
| Base | blade/base.js | Sidebar toggle, wishlist badge, storage events |
| Global | blade/global.js | Toast, logout, notifications toggle |
| Dashboard | dashboard/index.js | Stats, trips, media gallery, notifications, real-time chat |
| Plan Trip | plan-trip/index.js | Mood selection, AI suggestions, cost breakdown, receipt PDF |
| Flights | flights/index.js | Flight search, sort, booking confirmation |
| Accommodations | accommodations/index.js | Search, map (Leaflet), news feed |
| Bookings | bookings/index.js | Booking list, filter, cancel |
| Community | community/index.js | Topics, groups, stories, travellers |
| Discover | discover/index.js | Destination grid, wishlist toggle, filters |
| Chat | chat/index.js | Real-time messaging via Pusher/Echo |
| Notifications | notifications/index.js | Notification list, mark read |
| Profile | profile/edit.js | Profile update, password change, picture upload |
| Wishlist | wishlist/index.js | Filter, remove, clear all |

### Backend Services

| Service | File | Purpose |
|---|---|---|
| PricingService | Services/PricingService.php | Service fee, coupon, revenue recording |
| AviationstackService | Services/AviationstackService.php | IATA resolution, flight search via AeroDataBox |
| GeoapifyService | Services/GeoapifyService.php | Location/geocoding data |

---

## 10. Deployment Architecture

### Vercel Serverless Setup

```
vercel.json
├── buildCommand: "npm run build"
├── functions:
│   └── api/index.php (vercel-php@0.7.2 runtime)
└── routes:
    ├── /build/* → /public/build/*
    ├── /img/*   → /public/img/*
    ├── /storage/* → /public/storage/*
    └── /*       → /api/index.php

api/index.php (bootstrap)
├── Creates /tmp directories for serverless storage
├── Overrides env: CACHE_STORE=array, SESSION_DRIVER=cookie
├── Parses DATABASE_URL → DB_* env vars
├── Smart migration: detects DB state, runs pending migrations
├── Safety check: removes stale migration records
├── Seeds on first deploy (destinations empty)
└── Handles HTTP request via Laravel kernel
```

### Environment Variables Required

| Variable | Required | Description |
|---|---|---|
| APP_KEY | ✅ | Laravel encryption key |
| APP_URL | ✅ | Full deployment URL |
| DATABASE_URL | ✅ | Neon PostgreSQL connection string |
| GROQ_API_KEY | ✅ | Groq LLM API key |
| APP_ENV | ✅ | production |
| APP_DEBUG | ✅ | false |
| PUSHER_APP_KEY | Optional | Real-time chat |
| PUSHER_APP_SECRET | Optional | Real-time chat |
| AVIATIONSTACK_KEY | Optional | Flight search |
| GEOAPIFY_KEY | Optional | Maps/geocoding |

### CI/CD Pipeline (GitHub Actions)

```
Push to MVC branch
      │
      ▼
GitHub Actions: .github/workflows/
      │
      ├── PHP tests (PHPUnit)
      │   └── 24 tests, 58 assertions
      │
      └── Vercel auto-deploy on success
```

---

*Document generated: April 2026*
*Smart Booking System — All rights reserved*
