# 🎬 Remote Video Platform – Microservices Architecture

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docker](https://img.shields.io/badge/Docker-Supported-2496ED?logo=docker&logoColor=white)](#-quick-start)
[![Architecture](https://img.shields.io/badge/Architecture-Microservices-success)](#-architecture-overview)

A complete **video streaming platform** built with a **decoupled microservices architecture**. Features secure JWT authentication with refresh tokens, HLS adaptive streaming, and Docker orchestration.

> **Note:** This is a **demonstration project** designed for local development and portfolio purposes. It showcases microservices architecture, JWT authentication, and HLS video streaming in a self-contained Docker environment.

---

## 🚀 Quick Start

Run the entire platform with a single command:

```bash
# 1. Clone this repository
git clone https://github.com/anp3l/remote-video-platform.git
cd remote-video-platform

# 2. Configure environment
cp .env.example .env
# Edit .env with your RSA keys and secrets (see instructions inside)

# 3. Start all services
docker-compose up --build
```

**Access Points:**

- 🎨 **Frontend**: http://localhost:4200
- 🔐 **Auth API**: http://localhost:4000/api-docs
- 🎬 **Video API**: http://localhost:3070/api-docs
- 💾 **MongoDB**: localhost:27020

---

## 🏗️ Architecture Overview

This platform implements a **true microservices pattern** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                        │
│   ┌─────────────────────────────────────────────────────┐   │
│   │   Angular Frontend (Port 4200)                      │   │
│   │   - Material Design UI                              │   │
│   │   - HLS Video Player (Video.js)                     │   │
│   │   - Dual Token Management (Access + Refresh)        │   │
│   │   - Automatic Token Refresh                         │   │
│   │   - Preemptive Expiry Detection                     │   │
│   └───────────┬──────────────────────┬──────────────────┘   │
└───────────────┼──────────────────────┼──────────────────────┘
                │                      │
                │ JWT Requests         │ Video/Auth API Calls
                │ (Auto-refresh)       │
                ▼                      ▼
┌───────────────────────────┐   ┌─────────────────────────────┐
│   AUTH SERVER (IdP)       │   │   VIDEO SERVER (Resource)   │
│   Port: 4000              │   │   Port: 3070                │
├───────────────────────────┤   ├─────────────────────────────┤
│ - User Registration       │   │ - Video Upload              │
│ - Login / Logout          │   │ - FFmpeg Transcoding        │
│ - RS256 JWT Issuance      │   │ - HLS Adaptive Streaming    │
│ - Refresh Token Rotation  │   │ - HMAC Signed URLs          │
│                           │   │ - Per-User Isolation        │
│ ✅ Issues Tokens (15m/7d) │   │ - Thumbnail Generation      │
│                           │   │                             │
│                           │   │ ✅ Verifies Tokens (RSA)    │
└───────────┬───────────────┘   └──────────┬──────────────────┘
            │                              │
            │ authdb                       │ videodb
            ▼                              ▼
        ┌─────────────────────────────────────┐
        │    SHARED MONGODB INSTANCE          │
        │    Port: 27020                      │
        │    - User accounts & sessions       │
        │    - Video metadata                 │
        └─────────────────────────────────────┘
```

---

## 🎯 Key Design Principles

### 1. **Stateless Authentication with Refresh Tokens**
- Auth Server issues **short-lived access tokens** (15 minutes) using RS256
- **Long-lived refresh tokens** (7 days) stored in database with rotation
- Video Server verifies access tokens using public key (no database lookups)
- Client automatically refreshes tokens before expiry (preemptive refresh)
- Enables horizontal scaling and zero session storage

### 2. **Service Isolation**
- Each service has its own MongoDB database (`authdb` / `videodb`)
- Services communicate via REST APIs, not direct database access
- Frontend connects to both services independently
- Shared infrastructure (MongoDB) with logical separation

### 3. **Multi-Layer Security**
- **Layer 1:** JWT authentication for API access (RS256 signed by Auth Server)
- **Layer 2:** HMAC-signed URLs for video streaming (prevents hotlinking)
- **Layer 3:** Per-user data isolation enforced by `userId` claims in JWT
- **Layer 4:** Refresh token rotation (prevents token reuse attacks)

---

## 📦 Services

### 1. [Auth Server](https://github.com/anp3l/auth-server) (Identity Provider)

**Technology:** Node.js, Express, TypeScript, MongoDB, Bcrypt, JWT  
**Responsibilities:**

- User signup and login with bcrypt password hashing
- RS256 JWT token issuance (access + refresh)
- Refresh token rotation and revocation
- Secure logout with token cleanup

**Endpoints Used by Frontend:**
```
Authentication Flow:
- POST /auth/signup              - Register new user
- POST /auth/login               - Login with credentials
- POST /auth/refresh-token       - Refresh access token (automatic)
- POST /auth/revoke-token        - Logout from current device
```

**Additional Backend Capabilities** (not yet used by frontend UI):
```
Session Management:
- POST /auth/revoke-all-tokens   - Logout from all devices
- GET  /auth/refresh-tokens      - List active sessions

Password Management:
- POST /auth/forgot-password     - Request password reset
- POST /auth/reset-password      - Reset with token
- POST /auth/change-password     - Change password

Admin Panel:
- GET    /admin/users            - List all users
- PUT    /admin/users/:id/role   - Change user role
- DELETE /admin/users/:id        - Delete user
- GET    /admin/stats            - Platform statistics
- GET    /admin/audit-logs       - View audit logs
```

**Why Separate?**  
Decoupling identity management from business logic allows:
- Reusing the same Auth Server for multiple resource servers
- Independent scaling based on authentication load
- Security-focused updates without touching video processing code
- Ready for future frontend features (admin panel, profile management)

---

### 2. [Video Server](https://github.com/anp3l/remote-video-server) (Resource Server)

**Technology:** Node.js, Express, TypeScript, MongoDB, FFmpeg  
**Responsibilities:**

- Video upload and metadata management
- Adaptive Bitrate Transcoding (1080p/720p/480p/360p HLS)
- Secure streaming via HMAC-signed URLs
- Thumbnail generation (static + animated WebP)
- Background processing with status polling

**Key Features:**
- **Stateless Auth:** Verifies JWT signatures without calling Auth Server
- **Background Processing:** Asynchronous transcoding with status endpoints
- **Storage Efficiency:** Persistent Docker volumes for video files
- **Quality Adaptive Streaming:** HLS master playlist with automatic quality switching

---

### 3. [Frontend Client](https://github.com/anp3l/remote-video-client) (Angular SPA)

**Technology:** Angular 20, Material Design, Tailwind CSS, Video.js, jwt-decode  
**Responsibilities:**

- User authentication (login/signup/logout)
- Video library with upload, edit, delete, playback
- HLS streaming with adaptive quality selection
- Automatic token refresh handling

**Implemented Features:**
- **Auth Pages**: Login and signup forms with validation
- **Video Library**: Responsive grid/list view with search, filter, sort
- **Video Upload**: Dialog with file picker and progress tracking
- **Video Player**: HLS player in dialog with quality selector (1080p/720p/480p/360p)
- **Video Management**: Edit metadata and delete with confirmation

**Architecture Highlights:**
- **Dual API Integration:** Orchestrates requests to Auth (4000) and Video (3070) servers
- **Smart Token Management:** 
  - Stores access token (15min) + refresh token (7d) in localStorage
  - Preemptive refresh 30 seconds before expiry
  - Automatic retry on 401 errors
- **Dual Interceptor System:**
  - `authInterceptor`: Checks token expiry, adds Authorization header
  - `authRefreshInterceptor`: Handles 401 with automatic token refresh
- **Dockerized Nginx:** Production-ready static file serving

**Not Yet Implemented** (backend endpoints available):
- Profile management
- Active sessions view
- Admin panel
- Password reset flow

---

## 🔐 Security & Authentication Flow

### Initial Login Flow
```
┌─────────┐                  ┌──────────────┐                ┌──────────────┐
│ Client  │                  │ Auth Server  │                │ Video Server │
└────┬────┘                  └──────┬───────┘                └───────┬──────┘
     │                              │                                │
     │ 1. POST /auth/login          │                                │
     │    {email, password}         │                                │
     │ ────────────────────────────>│                                │
     │                              │                                │
     │                   2. Validate credentials                     │
     │                      Hash comparison (bcrypt)                 │
     │                      Generate access token (RS256)            │
     │                      Generate refresh token (UUID)            │
     │                      Store refresh token in DB                │
     │                              │                                │
     │ 3. Return both tokens        │                                │
     │    {accessToken, refreshToken, user}                          │
     │ <────────────────────────────│                                │
     │                              │                                │
     │ 4. Store in localStorage     │                                │
     │    - access_token (15min)    │                                │
     │    - refresh_token (7d)      │                                │
```

### Token Refresh Flow (Automatic)
```
     │                              │                                │
     │ 5. Check token expiry        │                                │
     │    (30s before expiration)   │                                │
     │                              │                                │
     │ 6. POST /auth/refresh-token  │                                │
     │    {refreshToken}            │                                │
     │ ────────────────────────────>│                                │
     │                              │                                │
     │                   7. Validate refresh token                   │
     │                      Check not revoked/expired                │
     │                      Issue new access token                   │
     │                      Rotate refresh token (new UUID)          │
     │                      Invalidate old refresh token             │
     │                              │                                │
     │ 8. Return new tokens         │                                │
     │   {accessToken, refreshToken}|                                │
     │ <────────────────────────────│                                │
     │                              │                                │
     │ 9. Update localStorage       │                                │
     │    (seamless, no user action)│                                │
```

### Authenticated Video Upload
```
     │                              │                                │
     │ 10. POST /videos             │                                │
     │     Authorization: Bearer <access_token>                      │
     │     Content-Type: multipart/form-data                         │
     │     Body: {title, description, file}                          │
     │ ─────────────────────────────────────────────────────────────>│
     │                              │                                │
     │                              │  11. Verify JWT signature      │
     │                              │       (using public key)       │
     │                              │  12. Extract userId from JWT   │
     │                              │  13. Validate file size/type   │
     │                              │  14. Save to /uploads/{userId}/│
     │                              │  15. Create DB entry           │
     │                              │  16. Queue FFmpeg job          │
     │                              │                                │
     │ 17. {videoId, status: "processing"}                           │
     │ <─────────────────────────────────────────────────────────────│
     │                              │                                │
     │ 18. Poll GET /videos/:id/status                               │
     │     (every 3 seconds)        │                                │
     │ ─────────────────────────────────────────────────────────────>│
     │                              │                                │
     │                              │ Background: FFmpeg transcoding │
     │                              │   - Generate HLS playlists     │
     │                              │   - Multiple qualities         │
     │                              │   - Extract thumbnails         │
     │                              │                                │
     │ 19. {status: "completed", streamUrl}                          │
     │ <─────────────────────────────────────────────────────────────│
     │                              │                                │
     │ 20. Stream video via HMAC URL│                                │
     │     (HLS adaptive playback)  │                                │
```

**Note:** All other video operations (GET, PUT, DELETE) follow the same JWT verification pattern:
- Extract token → Verify signature → Extract userId → Enforce per-user isolation

### Logout Flow
```
     │                              │                                │
     │ 21. POST /auth/revoke-token  │                                │
     │     {refreshToken}           │                                │
     │     Authorization: Bearer... │                                │
     │ ────────────────────────────>│                                │
     │                              │                                │
     │                  22. Mark token as revoked                    │
     │                      Clear localStorage                       │
     │                              │                                │
     │ 23. Success response         │                                │
     │ <────────────────────────────│                                │
     │                              │                                │
     │ 24. Redirect to login        │                                │
```

**Security Features Explained:**

- ✅ **Short-lived access tokens (15 minutes)** - Limits exposure window if token is compromised
- ✅ **Refresh token rotation** - One-time use tokens prevent replay attacks
- ✅ **Preemptive renewal** - Client refreshes 30s before expiry (zero visible 401 errors)
- ✅ **Automatic retry** - Failed requests are retried after token refresh

**Why RS256 (RSA) instead of HS256 (HMAC)?**
- **Public Key Distribution:** Video Server doesn't need the private key (zero-trust model)
- **Multi-Service Support:** Same Auth Server can serve multiple resource servers
- **Industry Standard:** Uses the same cryptographic approach as OAuth 2.0 / OpenID Connect

---

## 🎥 Video Processing Pipeline

The platform uses **FFmpeg** for professional-grade transcoding:

| Quality | Resolution | Bitrate  | Audio    | Segment Size |
|---------|-----------|----------|----------|--------------|
| 1080p   | 1920x1080 | 5000kbps | 192kbps  | 4s           |
| 720p    | 1280x720  | 2800kbps | 192kbps  | 4s           |
| 480p    | 854x480   | 1400kbps | 128kbps  | 4s           |
| 360p    | 640x360   | 800kbps  | 96kbps   | 4s           |

**Output:** HLS master playlist with automatic quality switching based on network conditions.

**Processing Flow:**
1. User uploads video via frontend
2. Video Server stores original file
3. FFmpeg transcodes to multiple qualities (background job)
4. HLS segments (.m3u8 + .ts files) generated
5. Thumbnails extracted (static + animated WebP)
6. Client polls status endpoint until complete
7. Video appears in library

---

## 🛠️ Tech Stack

| Layer              | Technology                                           |
|--------------------|-----------------------------------------------------|
| **Frontend**       | Angular 20, Material Design, Tailwind, Video.js     |
| **Backend**        | Node.js 20+, Express, TypeScript                    |
| **Authentication** | JWT (RS256), Refresh Tokens, Bcrypt, jwt-decode    |
| **Database**       | MongoDB 7.0 (authdb, videodb)                       |
| **Media**          | FFmpeg (H.264/AAC), HLS Protocol                    |
| **DevOps**         | Docker, Docker Compose, Nginx                       |
| **Documentation**  | Swagger/OpenAPI                                     |

---

## 📋 Prerequisites

- **Docker Desktop** or Docker Engine + Docker Compose
- **OpenSSL** (for RSA key generation)

---

## ⚙️ Configuration

### Step 1: Generate RSA Keys

```bash
# Generate private key (2048-bit)
openssl genrsa -out private.pem 2048

# Extract public key
openssl rsa -in private.pem -pubout -out public.pem

# Convert to Base64 (Linux/Mac)
cat private.pem | base64 | tr -d '\n'
cat public.pem | base64 | tr -d '\n'

# Windows PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("./private.pem"))
[Convert]::ToBase64String([IO.File]::ReadAllBytes("./public.pem"))
```

### Step 2: Generate Streaming Secret

```bash
openssl rand -base64 32
```

### Step 3: Configure Environment

```bash
cp .env.example .env
# Edit .env with the Base64 strings from above
```

### Required Environment Variables

```env
# RSA Keys for JWT signing/verification
PRIVATE_KEY_BASE64=your_base64_private_key
PUBLIC_KEY_BASE64=your_base64_public_key

# JWT Token lifetimes
ACCESS_TOKEN_EXPIRY=15m
REFRESH_TOKEN_EXPIRY=7d

# Video streaming security
STREAM_SECRET=your_base64_streaming_secret

# Email addresses (mock mode, logged to console)
EMAIL_FROM=noreply@demo.com
SUPPORT_EMAIL=support@demo.com
```

**Complete example in [`.env.example`](.env.example)**

---

## 🚦 Development & Deployment

### Start All Services

```bash
docker-compose up --build
```

**First startup takes ~2 minutes:**
1. MongoDB initializes
2. Auth Server starts
3. Video Server starts  
4. Frontend builds (Angular)

### Start Individual Services

```bash
docker-compose up auth-service       # Auth only
docker-compose up video-service      # Video only
docker-compose up client             # Frontend only
```

### Stop & Clean

```bash
docker-compose down                  # Stop services
docker-compose down -v               # Stop + delete volumes (⚠️ deletes videos)
```

### View Logs

```bash
docker-compose logs -f auth-service   # Auth logs
docker-compose logs -f video-service  # Video logs
docker-compose logs -f client         # Frontend logs
```

### Rebuild Single Service

```bash
docker-compose up --build auth-service
```

---

## 🧪 Testing the Platform

### 1. Access Frontend
Navigate to [http://localhost:4200](http://localhost:4200/)

### 2. Create Account
Sign up with email/password (e.g., `demo@example.com` / `Demo123!`)

**Check the auth-server logs** to see the mock welcome email:
```bash
docker-compose logs -f auth-service
# Look for: [MOCK EMAIL] Welcome to Remote Video Platform
```

### 3. Upload Video
- Click the upload button (top right)
- Select an MP4/MOV file
- Add title and description (category optional)
- Submit and watch the progress bar

### 4. Monitor Processing
- Video card shows "Processing" status
- Frontend polls `/videos/:id/status`
- Watch FFmpeg transcoding in video-server logs:

```bash
docker-compose logs -f video-service
# You'll see FFmpeg output for 1080p, 720p, 480p, 360p
```

### 5. Video Management
Once processing completes:
- **Play**: Click the video card → HLS player opens with quality selector
- **Edit**: Click edit icon → Update title/description/category
- **Delete**: Click delete icon → Confirmation dialog
- **Search**: Use search bar to filter by title/description
- **Filter**: Filter by category dropdown
- **Sort**: Sort by date/title/duration
- **View**: Toggle between grid and list view

### 6. Test Token Refresh
Want to see automatic token refresh?

```bash
# Option 1: Wait 14 minutes (access token expires at 15min)
# - Navigate around the app
# - Check browser console: "🔄 Token about to expire, refreshing preemptively..."
# - No 401 errors!

# Option 2: Quick test with short expiry
# Edit .env:
ACCESS_TOKEN_EXPIRY=2m
# Restart: docker-compose restart auth-service
# Token will refresh after 90 seconds
```

### 7. Test Logout
- Click logout button (top right)
- Redirected to login page
- Refresh token revoked on server
- Check auth-server logs: `POST /auth/revoke-token 200`

---

### 🎯 Demo Script (5 Minutes)

Quick walkthrough for recruiters:

1. **Start**: `docker-compose up` → show all 4 services starting
2. **Signup**: Create account → show mock email in auth-server logs
3. **Upload**: Upload video → show processing status, FFmpeg logs
4. **Play**: Click video → show HLS player with quality selector (try switching qualities)
5. **Edit**: Update video metadata → instant update
6. **Search**: Filter videos → real-time search
7. **Logout**: Show token revocation in logs

**Key talking points:**
- ✅ Microservices architecture (3 independent services)
- ✅ JWT + refresh token with automatic renewal
- ✅ Real-time video processing (FFmpeg HLS adaptive streaming)
- ✅ Production-ready patterns (stateless auth, token rotation)
- ✅ Responsive UI (Material Design + Tailwind)

---

## 🗂️ Data Persistence

All data is persisted in Docker volumes:

- **`mongo-data`:** User accounts, refresh tokens, and video metadata
- **`video-uploads`:** Original videos + HLS segments + thumbnails

**Backup Strategy:**

```bash
# Export MongoDB
docker exec video-platform-db mongodump --out /backup

# Copy uploaded videos
docker cp video-server:/app/uploads ./backup/uploads
```

**Restore:**

```bash
# Restore MongoDB
docker exec video-platform-db mongorestore /backup

# Restore videos
docker cp ./backup/uploads video-server:/app/uploads
```

---

## ✅ Completed Features

### Authentication
- ✅ **User Signup/Login**: Email and password authentication
- ✅ **Refresh Token System**: Automatic token renewal (15min access / 7d refresh)
- ✅ **Token Rotation**: Single-use refresh tokens prevent replay attacks
- ✅ **Preemptive Refresh**: Client refreshes 30s before expiry (zero 401 errors)
- ✅ **Secure Logout**: Revokes refresh token on server

### Video Management
- ✅ **HLS Adaptive Streaming**: Multi-quality transcoding (1080p/720p/480p/360p)
- ✅ **Background Processing**: Asynchronous FFmpeg transcoding
- ✅ **Video Upload**: File picker with progress tracking
- ✅ **Video Editing**: Update title, description, category
- ✅ **Video Deletion**: Safe deletion with confirmation
- ✅ **Thumbnail Generation**: Static + animated WebP previews
- ✅ **Search & Filter**: Real-time search, category filtering, sorting
- ✅ **Responsive UI**: Grid/list view toggle, mobile-friendly

### Security
- ✅ **Stateless Authentication**: RS256 JWT verification
- ✅ **HMAC-Signed URLs**: Secure video streaming
- ✅ **Per-User Isolation**: Users can only access their own videos

---

## 🔮 Future Improvements

### Frontend Features (Backend Ready, UI Missing)
These features have **working backend endpoints** but need frontend implementation:

- [ ] **Profile Page**: View and edit user information
- [ ] **Active Sessions Management**: List and revoke tokens (`GET /auth/refresh-tokens`)
- [ ] **Admin Panel**: User management dashboard (`GET /admin/users`, `PUT /admin/users/:id/role`)
- [ ] **Password Reset**: Forgot password flow (`POST /auth/forgot-password`, `/auth/reset-password`)
- [ ] **Change Password**: In-app password change (`POST /auth/change-password`)
- [ ] **User Role Indicator**: Show admin/customer badge in UI

### Video Features
- [ ] **Video Playlists**: Organize videos into collections
- [ ] **Drag-and-Drop Upload**: Drop zone instead of file picker
- [ ] **Video Sharing**: Public links with expiry

### System Architecture
- [ ] **JWKS Auto-Discovery**: `GET /.well-known/jwks.json` endpoint
- [ ] **Key Rotation**: Automated RSA key pair rotation
- [ ] **Rate Limiting**: API abuse prevention

### Analytics
- [ ] **View Count**: Track video views
- [ ] **Watch Time**: Track how long users watch
- [ ] **Storage Dashboard**: Per-user storage quotas

---


## 📄 License

This project is licensed under the MIT License - see individual repositories for details.

---

## 🔗 Repository Links

- **Platform Orchestration (This Repo)**: [https://github.com/anp3l/remote-video-platform](https://github.com/anp3l/remote-video-platform)
- **Auth Server**: [https://github.com/anp3l/auth-server](https://github.com/anp3l/auth-server)
- **Video Server**: [https://github.com/anp3l/remote-video-server](https://github.com/anp3l/remote-video-server)
- **Frontend Client**: [https://github.com/anp3l/remote-video-client](https://github.com/anp3l/remote-video-client)

---

**Built by [Andrea Peluso](https://www.linkedin.com/in/adr-peluso/)**