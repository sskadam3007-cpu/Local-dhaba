# Project Specification: Local Dhaba Chaat (Foodlive)

A crowdsourced, location-tagged highway and street food discovery platform. The application enables travelers to discover open roadside dhabas, street food stalls, and restaurants in real time, view food photos uploaded by recent visitors, check community ratings, and explore featured recommendations via an interactive map interface.

---

## 1. System Architecture Overview

* **Mobile Client:** Android Native (Kotlin, Jetpack Compose) / Cross-platform.
* **Map Engine:** Google Maps SDK / Mapbox SDK for Android.
* **Backend Framework:** Django REST Framework (DRF) containerized via Docker.
* **Database:** PostgreSQL with PostGIS extension for spatial queries.
* **Media Storage:** Cloud Object Storage (S3-compatible / Cloudflare R2 / OCI Object Storage).
* **Reverse Proxy & Web Server:** Nginx + Gunicorn.

---

## 2. Core Functional Requirements

### 2.1 Real-Time "Open Now" Discovery
- Query spots within an adjustable bounding box or radius ($R$ km) around the user's live GPS coordinates.
- Calculate open status using business hours combined with real-time manual overrides and community activity freshness indicators (e.g., "Active 20 mins ago").

### 2.2 Crowdsourced Live Updates & Media
- Users can log new spots or update existing stalls with one-tap GPS coordinate capture.
- Camera-first photo uploads of food items, menus, and stall hygiene.
- Fast rating submission (1 to 5 stars) and quick food category tags (Veg, Non-Veg, Tea/Snacks, Full Meals).

### 2.3 Interactive Map & Bottom Sponsored Carousel
- Custom map markers displaying food category icons and rating chips.
- Tapping a pin brings up an overview card showing latest food photos, distance, and status.
- Horizontal scrolling carousel at the bottom showcasing featured/promoted dhabas and system suggestions.
- Selecting a carousel item smoothly animates the map camera to the target dhaba marker.

---

## 3. Database Schema (PostgreSQL + PostGIS)

```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- 1. Restaurants / Dhabas / Stalls
CREATE TABLE restaurants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    location GEOGRAPHY(Point, 4326) NOT NULL,
    address_text TEXT,
    opening_time TIME NOT NULL,
    closing_time TIME NOT NULL,
    is_manually_closed BOOLEAN DEFAULT FALSE,
    average_rating NUMERIC(2, 1) DEFAULT 0.0,
    review_count INTEGER DEFAULT 0,
    is_promoted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_restaurants_location ON restaurants USING GIST(location);

-- 2. Food Items / Live Updates
CREATE TABLE live_food_updates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id UUID REFERENCES restaurants(id) ON DELETE CASCADE,
    item_name VARCHAR(150) NOT NULL,
    image_url TEXT NOT NULL,
    price NUMERIC(8, 2),
    is_available BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_food_updates_restaurant ON live_food_updates(restaurant_id);

-- 3. Reviews & Ratings
CREATE TABLE reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id UUID REFERENCES restaurants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL,
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_reviews_restaurant ON reviews(restaurant_id);
local-dhaba-chaat/
├── .gitignore
├── README.md
├── SPEC.md
├── docker-compose.yml
│
├── backend/                             # Django REST + PostGIS service
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── manage.py
│   ├── requirements.txt
│   │
│   ├── config/                          # Project configuration root
│   │   ├── __init__.py
│   │   ├── asgi.py
│   │   ├── settings.py                  # GeoDjango, DB, S3 configurations
│   │   ├── urls.py                      # Root URL routing (/api/v1/...)
│   │   └── wsgi.py
│   │
│   └── spots/                           # Core app for dhabas, updates, & reviews
│       ├── __init__.py
│       ├── admin.py                     # GeoAdmin custom layouts
│       ├── apps.py
│       ├── models.py                    # Restaurant, FoodUpdate, Review models
│       ├── serializers.py               # Spatial & JSON serializers
│       ├── views.py                     # Nearby, Promoted, & Upload ViewSets
│       ├── urls.py                      # App-level endpoints
│       ├── utils.py                     # Image compression & spatial helpers
│       ├── migrations/
│       │   └── __init__.py
│       └── tests/
│           ├── __init__.py
│           ├── test_models.py
│           └── test_spatial_api.py      # Radius & bounding box test cases
│
├── android/                             # Android Kotlin / Jetpack Compose App
│   ├── build.gradle.kts
│   ├── settings.gradle.kts
│   └── app/
│       ├── build.gradle.kts
│       └── src/
│           └── main/
│               ├── AndroidManifest.xml
│               ├── java/com/dhabachaat/
│               │   ├── data/            # Networking, DTOs, Repository pattern
│               │   │   ├── api/         # Retrofit interfaces
│               │   │   ├── model/       # Data classes (Restaurant, FoodItem)
│               │   │   └── repository/  # DhabaRepository implementation
│               │   ├── di/              # Dependency injection (Hilt modules)
│               │   ├── ui/
│               │   │   ├── map/         # MapScreen, CustomMarker, CameraController
│               │   │   ├── carousel/    # BottomSlider, DhabaSummaryCard
│               │   │   ├── upload/      # CameraCaptureBottomSheet, QuickReport
│               │   │   └── theme/       # Material3 typography, colors, theme
│               │   └── viewmodel/       # DhabaViewModel, LocationViewModel
│               └── res/
│                   ├── drawable/        # Marker icons, vector assets
│                   └── values/          # Strings, colors
│
└── deploy/                              # Production deployment configs
    ├── nginx/
    │   └── default.conf                 # Reverse proxy, SSL, media caching
    └── systemd/                         # Or cloud init / OCI deployment scripts
