# KrewLens — Implementation Plan

Field workforce management app for pesticide spraying business.
Admin tracks worker locations in real-time and manages hotel job assignments.

---

## Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| Android | Kotlin + Jetpack Compose | Modern, less boilerplate, first-class Google support |
| Architecture (Android) | MVVM + Repository pattern | Industry standard, testable, clean separation |
| Maps | Google Maps SDK for Android | Real-time worker pins on map |
| GPS | FusedLocationProviderClient | Battery-efficient location from Google Play Services |
| HTTP Client | Retrofit2 + OkHttp3 | Standard Android networking |
| DI | Hilt | Official Android DI, integrates with ViewModel |
| Async | Kotlin Coroutines + Flow | Reactive streams for location and UI state |
| Local storage | DataStore (token) + Room (job cache) | Offline support |
| Real-time | WebSocket (STOMP via OkHttp) | Live location streaming to admin |
| Push notifications | Firebase Cloud Messaging (FCM) | Free, reliable, deep Android OS integration |
| Backend | Java Spring Boot | Robust, typed, production-ready |
| Auth | Spring Security + JWT | Stateless, role-based (ADMIN / WORKER) |
| ORM | Spring Data JPA + Hibernate | Standard JPA with MySQL |
| Database | MySQL | Relational, structured job/user data |
| Real-time (server) | Spring WebSocket + STOMP | Broadcasts worker locations to subscribed admins |
| Push (server) | Firebase Admin SDK | Backend sends FCM notifications |

---

## Project Structure (Monorepo)

```
KrewLens/
├── android/                        # Android app (Kotlin + Jetpack Compose)
│   └── app/
│       └── src/main/
│           ├── java/com/krewlens/
│           │   ├── data/           # Repositories, API services, DB DAOs
│           │   ├── domain/         # Models, use cases
│           │   ├── ui/             # Screens, ViewModels, navigation
│           │   │   ├── auth/       # Login screen
│           │   │   ├── admin/      # Map, workers list, job management
│           │   │   └── worker/     # My jobs, job detail
│           │   ├── service/        # Background location service
│           │   └── di/             # Hilt modules
│           └── res/
├── backend/                        # Spring Boot API
│   └── src/main/java/com/krewlens/
│       ├── config/                 # Security, WebSocket, Firebase config
│       ├── controller/             # REST controllers
│       ├── service/                # Business logic
│       ├── repository/             # JPA repositories
│       ├── model/                  # JPA entities
│       ├── dto/                    # Request/Response DTOs
│       ├── security/               # JWT filter, UserDetailsService
│       └── websocket/              # STOMP message handlers
└── PLAN.md
```

---

## Database Schema (MySQL)

### `users`
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK AUTO_INCREMENT | |
| name | VARCHAR(100) | |
| email | VARCHAR(150) UNIQUE | login credential |
| password_hash | VARCHAR(255) | BCrypt |
| role | ENUM('ADMIN','WORKER') | role-based access |
| fcm_token | VARCHAR(255) | updated from app on login |
| is_active | BOOLEAN DEFAULT TRUE | soft disable |
| last_latitude | DECIMAL(10,7) | last known location |
| last_longitude | DECIMAL(10,7) | last known location |
| last_location_at | DATETIME | when last GPS ping received |
| created_at | DATETIME | |
| updated_at | DATETIME | |

### `jobs`
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK AUTO_INCREMENT | |
| title | VARCHAR(200) | e.g. "Spray Room 301-310" |
| description | TEXT | instructions |
| hotel_name | VARCHAR(200) | |
| address | VARCHAR(500) | |
| latitude | DECIMAL(10,7) | hotel GPS coords |
| longitude | DECIMAL(10,7) | hotel GPS coords |
| status | ENUM('PENDING','ASSIGNED','IN_PROGRESS','COMPLETED','CANCELLED') | |
| assigned_to | BIGINT FK -> users.id (nullable) | null = unassigned |
| created_by | BIGINT FK -> users.id | admin who created |
| scheduled_at | DATETIME | when to do the job |
| completed_at | DATETIME (nullable) | |
| created_at | DATETIME | |
| updated_at | DATETIME | |

### `location_history` *(optional, for audit trail)*
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK AUTO_INCREMENT | |
| user_id | BIGINT FK -> users.id | |
| latitude | DECIMAL(10,7) | |
| longitude | DECIMAL(10,7) | |
| recorded_at | DATETIME | |

---

## REST API Design

**Base URL:** `https://api.krewlens.com/api`
**Auth header:** `Authorization: Bearer <jwt_token>`

### Auth
| Method | Endpoint | Access | Description |
|---|---|---|---|
| POST | `/auth/login` | Public | Email + password → JWT token + role |
| POST | `/auth/refresh` | Auth | Refresh JWT token |
| PUT | `/auth/fcm-token` | Auth | Worker/Admin updates FCM token |

### Users (Admin only)
| Method | Endpoint | Access | Description |
|---|---|---|---|
| GET | `/users` | ADMIN | List all workers |
| POST | `/users` | ADMIN | Create worker account |
| PUT | `/users/{id}` | ADMIN | Update worker details |
| DELETE | `/users/{id}` | ADMIN | Deactivate worker |
| GET | `/users/locations` | ADMIN | Get all workers' last known locations |

### Jobs
| Method | Endpoint | Access | Description |
|---|---|---|---|
| GET | `/jobs` | ADMIN = all jobs; WORKER = only assigned | List jobs |
| POST | `/jobs` | ADMIN | Create new job |
| GET | `/jobs/{id}` | Auth | Get job details |
| PUT | `/jobs/{id}` | ADMIN | Update job |
| DELETE | `/jobs/{id}` | ADMIN | Delete job |
| PATCH | `/jobs/{id}/assign` | ADMIN | Assign job to worker (body: `{workerId}`) |
| PATCH | `/jobs/{id}/unassign` | ADMIN | Remove worker from job |
| PATCH | `/jobs/{id}/status` | WORKER | Worker updates status (IN_PROGRESS / COMPLETED) |

### Location
| Method | Endpoint | Access | Description |
|---|---|---|---|
| POST | `/location` | WORKER | Worker sends GPS ping `{latitude, longitude}` |

---

## WebSocket (STOMP) Design

**Endpoint:** `wss://api.krewlens.com/ws`
**Auth:** Pass JWT as query param on connect: `/ws?token=<jwt>`

| Direction | Destination | Description |
|---|---|---|
| Worker → Server | `/app/location` | Worker publishes GPS coords |
| Server → Admin | `/topic/locations` | Server broadcasts all worker positions |
| Server → Admin | `/topic/jobs` | Server broadcasts job create/update/delete events |

Admin app subscribes to `/topic/locations` on login. Receives:
```json
{
  "workerId": 5,
  "workerName": "Rajan",
  "latitude": 28.6139,
  "longitude": 77.2090,
  "timestamp": "2026-03-07T10:30:00Z"
}
```

---

## Push Notification Flows (FCM)

| Trigger | Recipient | Title | Body |
|---|---|---|---|
| Job assigned to worker | Worker | "New Job Assigned" | "You have a new job: {hotel_name}" |
| Job updated | Worker | "Job Updated" | "{title} has been updated" |
| Job cancelled | Worker | "Job Cancelled" | "{title} has been cancelled" |
| Worker starts job | Admin | "Job Started" | "{workerName} started {title}" |
| Worker completes job | Admin | "Job Completed" | "{workerName} completed {title}" |

Backend sends FCM via Firebase Admin SDK whenever these events occur in the service layer.

---

## Android App — Screens & Navigation

```
NavGraph
├── AuthGraph
│   └── LoginScreen
└── MainGraph (role-based entry)
    ├── ADMIN role →
    │   ├── AdminMapScreen       (bottom nav: Map)
    │   ├── WorkersListScreen    (bottom nav: Workers)
    │   ├── JobsListScreen       (bottom nav: Jobs)
    │   ├── CreateEditJobScreen
    │   ├── JobDetailScreen      (assign worker, change status)
    │   └── WorkerDetailScreen
    └── WORKER role →
        ├── MyJobsScreen         (bottom nav: My Jobs)
        ├── JobDetailScreen      (view details, update status)
        └── ProfileScreen        (bottom nav: Profile)
```

### Background Location Service
- `LocationForegroundService` (Android Foreground Service)
- Runs while worker is logged in
- Sends GPS ping to `POST /api/location` every **30 seconds**
- Also sends via WebSocket to `/app/location` for real-time admin map updates
- Shows persistent notification: "KrewLens is tracking your location"

---

## Spring Boot Backend — Package Structure

```
com.krewlens
├── KrewLensApplication.java
├── config/
│   ├── SecurityConfig.java          # JWT filter chain, role rules
│   ├── WebSocketConfig.java         # STOMP endpoint, message broker
│   └── FirebaseConfig.java          # Initialize Firebase Admin SDK
├── security/
│   ├── JwtUtil.java
│   ├── JwtAuthFilter.java
│   └── CustomUserDetailsService.java
├── model/
│   ├── User.java                    # @Entity
│   ├── Job.java                     # @Entity
│   └── LocationHistory.java         # @Entity
├── dto/
│   ├── LoginRequest/Response
│   ├── JobRequest/Response
│   ├── UserRequest/Response
│   └── LocationUpdate
├── repository/
│   ├── UserRepository.java
│   ├── JobRepository.java
│   └── LocationHistoryRepository.java
├── service/
│   ├── AuthService.java
│   ├── UserService.java
│   ├── JobService.java
│   ├── LocationService.java
│   └── NotificationService.java     # FCM push logic
├── controller/
│   ├── AuthController.java
│   ├── UserController.java
│   ├── JobController.java
│   └── LocationController.java
└── websocket/
    └── LocationMessageHandler.java  # Handles /app/location, broadcasts to /topic/locations
```

---

## Optimizations & Decisions

| # | Decision | Reasoning |
|---|---|---|
| 1 | `location_history` table is **included** (not optional) | Needed for audit trail if dispute arises about whether a worker visited a site |
| 2 | WebSocket auth via **HandshakeInterceptor** (not query param) | JWT in query param gets logged in server access logs — security risk |
| 3 | **Global exception handler** in Spring Boot from day 1 | Uniform error responses; avoids inconsistent 500s during development |
| 4 | **Seed SQL script** for default admin user | Can't test login without it; BCrypt hash generated once and committed |
| 5 | **`docker-compose.yml`** for local MySQL | No manual MySQL install; entire dev environment spins up with one command |
| 6 | **Standard API response wrapper** `ApiResponse<T>` | Consistent `{success, data, message}` shape across all endpoints |
| 7 | Location interval: **30s normally, 10s when IN_PROGRESS** | Balance between battery life and accuracy when worker is actively on site |
| 8 | **Logout stops location service** and disconnects WebSocket | Prevents ghost tracking after session ends |
| 9 | **401 interceptor in OkHttp** on Android | Handles token expiry globally; redirects to login without per-screen handling |
| 10 | Add `phone_number` column to `users` table | Useful for admin to call a worker directly from the app |

---

## Third-Party Services to Set Up

1. **Firebase Project**
   - Enable Firebase Cloud Messaging (FCM)
   - Download `google-services.json` → place in `android/app/`
   - Download Firebase Admin SDK service account JSON → place in `backend/src/main/resources/`

2. **Google Maps**
   - Enable Maps SDK for Android in Google Cloud Console
   - Create and restrict Android API key → add to `AndroidManifest.xml`

3. **MySQL Database**
   - Local: `docker-compose.yml` with MySQL 8 image
   - Production: Railway / AWS RDS / PlanetScale

4. **Backend Hosting (production)**
   - Railway / Render / AWS EC2 for Spring Boot JAR

---

## Implementation Phases — Detailed Task Breakdown

---

### Phase 1 — Project Setup & Infrastructure

**Backend**
- [ ] 1.1 Generate Spring Boot project at `start.spring.io` — dependencies: Web, Security, Data JPA, WebSocket, MySQL Driver, Validation, Lombok
- [ ] 1.2 Open in IntelliJ, verify project compiles and runs (blank app)
- [ ] 1.3 Create `application.properties` with MySQL datasource config (local Docker URL)
- [ ] 1.4 Create `docker-compose.yml` with MySQL 8 service (port 3306, named volume)
- [ ] 1.5 Write `schema.sql` DDL — tables: `users`, `jobs`, `location_history`
- [ ] 1.6 Write `seed.sql` — insert default admin user with BCrypt hashed password
- [ ] 1.7 Configure Hibernate to validate schema on startup (`ddl-auto=validate`)
- [ ] 1.8 Create `ApiResponse.java` generic wrapper `{success, data, message}`
- [ ] 1.9 Create `GlobalExceptionHandler.java` — handle 400, 401, 403, 404, 500

**Android**
- [ ] 1.10 Create Android project in Android Studio — Kotlin, Jetpack Compose, min SDK 26
- [ ] 1.11 Add all Gradle dependencies — Retrofit2, OkHttp3, Hilt, Maps Compose, FCM, Room, DataStore, Navigation Compose, Coroutines
- [ ] 1.12 Create `KrewLensApp.kt` — `@HiltAndroidApp` application class
- [ ] 1.13 Add `@AndroidEntryPoint` to `MainActivity.kt`
- [ ] 1.14 Create placeholder `google-services.json` stub (will replace after Firebase setup step)

**Third-party**
- [ ] 1.15 Create Firebase project → add Android app → download real `google-services.json`
- [ ] 1.16 Enable Firebase Cloud Messaging in Firebase console
- [ ] 1.17 Download Firebase Admin SDK service account JSON for backend
- [ ] 1.18 Enable Google Maps SDK for Android in Google Cloud Console → create API key

**Checkpoint:** Docker MySQL running, Spring Boot starts and connects to DB, Android project builds clean.

---

### Phase 2 — Authentication

**Backend**
- [ ] 2.1 Create `User.java` `@Entity` — all columns from schema
- [ ] 2.2 Create `UserRepository.java` — add `findByEmail()` method
- [ ] 2.3 Create `JwtUtil.java` — `generateToken()`, `validateToken()`, `extractEmail()`, `extractRole()`
- [ ] 2.4 Create `CustomUserDetailsService.java` — loads user by email, maps role to `GrantedAuthority`
- [ ] 2.5 Create `JwtAuthFilter.java` — reads `Authorization` header, validates token, sets `SecurityContext`
- [ ] 2.6 Create `SecurityConfig.java` — permit `/api/auth/**`, secure all other routes, inject filter
- [ ] 2.7 Create `LoginRequest.java` / `LoginResponse.java` DTOs
- [ ] 2.8 Create `AuthService.java` — find user by email, BCrypt match, generate JWT, return token + role
- [ ] 2.9 Create `AuthController.java` — `POST /api/auth/login`
- [ ] 2.10 Add `PUT /api/auth/fcm-token` endpoint — authenticated, updates `fcm_token` on current user
- [ ] 2.11 Test login with Postman — valid credentials → JWT, invalid → 401

**Android**
- [ ] 2.12 Create `AuthApiService.kt` Retrofit interface — `login()` call only
- [ ] 2.13 Create `NetworkModule.kt` Hilt module — provide Retrofit + OkHttp with base URL
- [ ] 2.14 Create `TokenDataStore.kt` — read/write JWT token and role using DataStore
- [ ] 2.15 Create `DataStoreModule.kt` Hilt module — provide DataStore instance
- [ ] 2.16 Create `AuthRepository.kt` — call login API, save token + role to DataStore
- [ ] 2.17 Create `AuthViewModel.kt` — `login(email, password)`, expose `UiState` (loading/success/error)
- [ ] 2.18 Create `LoginScreen.kt` — email field, password field, login button, error message
- [ ] 2.19 Create `NavGraph.kt` — `AuthGraph` (Login) and stub `AdminGraph` / `WorkerGraph`
- [ ] 2.20 On app launch: check DataStore for saved token → if present skip login, navigate by role
- [ ] 2.21 Add `AuthInterceptor` to OkHttp — attaches `Authorization: Bearer` header to every request
- [ ] 2.22 Add `401UnauthorizedInterceptor` — on 401 response, clear token, navigate to login

**Checkpoint:** Login works end-to-end. Admin token routes to admin stub screen. Worker token routes to worker stub screen.

---

### Phase 3 — Worker Management (Admin)

**Backend**
- [ ] 3.1 Create `UserRequest.java` DTO — name, email, password, phone_number
- [ ] 3.2 Create `UserResponse.java` DTO — id, name, email, phone_number, role, is_active, last_location_at
- [ ] 3.3 Create `UserService.java` — `listWorkers()`, `createWorker()`, `updateWorker()`, `deactivateWorker()`
- [ ] 3.4 Create `UserController.java` — `GET /api/users`, `POST /api/users`, `PUT /api/users/{id}`, `DELETE /api/users/{id}`
- [ ] 3.5 Secure all user endpoints with `@PreAuthorize("hasRole('ADMIN')")`
- [ ] 3.6 Test: create worker via Postman, verify password is BCrypt hashed in DB

**Android**
- [ ] 3.7 Add user API calls to `ApiService.kt` — list workers, create, update, deactivate
- [ ] 3.8 Create `UserRepository.kt`
- [ ] 3.9 Create `WorkersListViewModel.kt` — fetch and hold workers list
- [ ] 3.10 Create `WorkersListScreen.kt` — scrollable list, each row shows name + phone + active status
- [ ] 3.11 Create `CreateEditWorkerScreen.kt` — form with name, email, password, phone fields
- [ ] 3.12 Add admin bottom nav bar — tabs: Map, Workers, Jobs
- [ ] 3.13 Wire Workers screen into admin nav graph

**Checkpoint:** Admin can add a worker from the app. Worker can log in with those credentials.

---

### Phase 4 — Job Management

**Backend**
- [ ] 4.1 Create `Job.java` `@Entity` — all columns from schema
- [ ] 4.2 Create `JobRepository.java` — add `findByAssignedTo()`, `findAllByOrderByCreatedAtDesc()`
- [ ] 4.3 Create `JobRequest.java` / `JobResponse.java` DTOs
- [ ] 4.4 Create `JobService.java` — `createJob()`, `updateJob()`, `deleteJob()`, `assignJob()`, `unassignJob()`, `updateJobStatus()`
- [ ] 4.5 Role filter in `getJobs()` — admin gets all, worker gets only their own
- [ ] 4.6 Create `JobController.java` — all 8 job endpoints
- [ ] 4.7 On `assignJob()`: validate worker exists and is active; change status to `ASSIGNED`
- [ ] 4.8 On `updateJobStatus()` to `IN_PROGRESS` or `COMPLETED`: validate caller is the assigned worker
- [ ] 4.9 Test all endpoints with Postman

**Android (Admin)**
- [ ] 4.10 Add job API calls to `ApiService.kt`
- [ ] 4.11 Create `JobRepository.kt`
- [ ] 4.12 Create `JobsListViewModel.kt` + `JobsListScreen.kt` — list with status chips, swipe to delete
- [ ] 4.13 Create `CreateEditJobScreen.kt` — fields: title, hotel name, address, lat/lng (map picker), description, scheduled time
- [ ] 4.14 Create admin `JobDetailScreen.kt` — show all details, dropdown to assign worker, cancel button

**Android (Worker)**
- [ ] 4.15 Create `MyJobsViewModel.kt` + `MyJobsScreen.kt` — list of assigned jobs sorted by schedule
- [ ] 4.16 Create worker `JobDetailScreen.kt` — job info, "Start Job" and "Complete Job" buttons
- [ ] 4.17 Status button logic: show "Start Job" if `ASSIGNED`, "Complete Job" if `IN_PROGRESS`, disabled if `COMPLETED`

**Checkpoint:** Admin creates a job, assigns it to a worker. Worker sees it in "My Jobs" and can update status.

---

### Phase 5 — Real-Time Location Tracking

**Backend**
- [ ] 5.1 Create `WebSocketConfig.java` — STOMP endpoint at `/ws`, simple message broker for `/topic`
- [ ] 5.2 Create `WebSocketHandshakeInterceptor.java` — extract JWT from `token` header on handshake, authenticate
- [ ] 5.3 Create `LocationUpdateMessage.java` DTO — workerId, workerName, latitude, longitude, timestamp
- [ ] 5.4 Create `LocationMessageHandler.java` — `@MessageMapping("/location")`, update DB, broadcast to `/topic/locations`
- [ ] 5.5 Create `LocationService.java` — update `last_latitude`, `last_longitude`, `last_location_at` on user; save row to `location_history`
- [ ] 5.6 Create `LocationController.java` — `POST /api/location` (REST fallback path)
- [ ] 5.7 Add `GET /api/users/locations` — returns all workers with last known lat/lng for initial map load
- [ ] 5.8 Test WebSocket with a STOMP browser client (e.g., `websocat` or `stomp.js` test page)

**Android (Worker)**
- [ ] 5.9 Request permissions in `LoginScreen.kt` — `ACCESS_FINE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`
- [ ] 5.10 Create `LocationForegroundService.kt` — extends `Service`, `@AndroidEntryPoint`
- [ ] 5.11 In service: set up `FusedLocationProviderClient`, request updates every 30s (`LocationRequest`)
- [ ] 5.12 On location callback: POST to `POST /api/location` via Retrofit
- [ ] 5.13 On location callback: send via WebSocket to `/app/location`
- [ ] 5.14 Show required persistent foreground notification while service runs
- [ ] 5.15 Start service on successful login (worker role only), stop service on logout
- [ ] 5.16 Increase location interval to 10s when job status is `IN_PROGRESS`

**Android (Admin)**
- [ ] 5.17 Add `maps-compose` Google Maps dependency
- [ ] 5.18 Create `AdminMapScreen.kt` — full-screen `GoogleMap` composable
- [ ] 5.19 On screen open: call `GET /api/users/locations` → place initial `Marker` for each worker
- [ ] 5.20 Set up WebSocket client in `AdminMapViewModel.kt` — connect on screen enter, disconnect on exit
- [ ] 5.21 Subscribe to `/topic/locations` — on each message, update or add marker on map
- [ ] 5.22 Add `MarkerInfoWindow` on tap — shows worker name, last seen timestamp

**Checkpoint:** Admin opens map, sees worker pins. Worker drives around (or use emulator GPS mock), pins move in real-time.

---

### Phase 6 — Push Notifications

**Backend**
- [ ] 6.1 Add `firebase-admin` dependency to `pom.xml`
- [ ] 6.2 Create `FirebaseConfig.java` — initialize `FirebaseApp` from service account JSON in resources
- [ ] 6.3 Create `NotificationService.java` — `sendNotification(fcmToken, title, body, Map<String,String> data)`
- [ ] 6.4 In `JobService.assignJob()` — call `NotificationService` to notify assigned worker
- [ ] 6.5 In `JobService.updateJob()` — notify worker if their job was changed
- [ ] 6.6 In `JobService.deleteJob()` / cancel — notify worker their job was cancelled
- [ ] 6.7 In `JobService.updateJobStatus(IN_PROGRESS)` — notify all admins job has started
- [ ] 6.8 In `JobService.updateJobStatus(COMPLETED)` — notify all admins job is done
- [ ] 6.9 Test: trigger job assignment from Postman → receive push notification on device

**Android**
- [ ] 6.10 Add `google-services.json` (real Firebase file) to `android/app/`
- [ ] 6.11 Add `com.google.firebase:firebase-messaging` to `build.gradle`
- [ ] 6.12 Create `KrewMessagingService.kt` — extends `FirebaseMessagingService`
- [ ] 6.13 Override `onNewToken()` — POST new token to `PUT /api/auth/fcm-token`
- [ ] 6.14 Override `onMessageReceived()` — build and show `NotificationCompat` notification
- [ ] 6.15 Add `data` payload to notifications (`jobId`, `type`) — used for deep linking
- [ ] 6.16 Handle notification tap — `PendingIntent` opens app and navigates to the specific job detail screen
- [ ] 6.17 In `AuthViewModel` after login success: trigger FCM token refresh upload

**Checkpoint:** Admin assigns job → worker's phone buzzes with notification → tap opens that job's detail screen.

---

### Phase 7 — Polish & Hardening

**UX**
- [ ] 7.1 Add loading skeleton/shimmer states on all list screens
- [ ] 7.2 Add empty state UI — no jobs, no workers, no location yet
- [ ] 7.3 Add `Snackbar` error messages for all API failures
- [ ] 7.4 Add pull-to-refresh on jobs list and workers list
- [ ] 7.5 Add confirmation `AlertDialog` before deleting a job or deactivating a worker

**Offline & Reliability**
- [ ] 7.6 Set up Room database with `JobEntity` and `JobDao`
- [ ] 7.7 Cache jobs list in Room — load from Room first, refresh from API in background
- [ ] 7.8 Worker can view their jobs even with no internet connection
- [ ] 7.9 Queue failed location POST requests and retry when network returns

**Security & Build**
- [ ] 7.10 Add ProGuard rules for Retrofit, Hilt, Room in `proguard-rules.pro`
- [ ] 7.11 Move API base URL, Maps key to `BuildConfig` fields (not hardcoded in source)
- [ ] 7.12 Add `INTERNET`, `FOREGROUND_SERVICE`, location permission declarations to `AndroidManifest.xml` (verify all present)
- [ ] 7.13 Handle `SecurityException` if location permission is revoked while service is running

**Testing**
- [ ] 7.14 End-to-end smoke test: admin login → create job → assign worker → worker notified → worker starts job → admin sees status change → worker completes → admin notified
- [ ] 7.15 Test on Android 12+ (background location permission requires extra step on newer OS)
- [ ] 7.16 Test location tracking with Android Emulator GPS mock routes
- [ ] 7.17 Generate signed release APK, verify ProGuard doesn't break anything

---

## Key Decisions Summary

- **Single APK, two roles** — admin and worker share one app; login role determines navigation graph
- **Location update interval** — 30s idle, 10s when a job is `IN_PROGRESS`
- **Last-known location on `users` table** — fast admin map initial load; `location_history` for full audit trail
- **WebSocket (STOMP) for real-time** — admin map updates without polling; worker pushes via WebSocket + REST fallback
- **JWT stateless auth** — token in Android DataStore; 401 interceptor handles expiry globally
- **Docker Compose for dev** — one command starts MySQL locally; no manual DB install needed
