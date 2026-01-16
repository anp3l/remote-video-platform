# 🎬 Remote Video Platform – Microservices Architecture

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docker](https://img.shields.io/badge/Docker-Supported-2496ED?logo=docker&logoColor=white)](#-quick-start)
[![Architecture](https://img.shields.io/badge/Architecture-Microservices-success)](#-architecture-overview)

A complete **video streaming platform** built with a **decoupled microservices architecture**. Features secure authentication, HLS adaptive streaming, and Docker orchestration.

---

## 🚀 Quick Start

Run the entire platform with a single command:

```
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
│   │   - JWT State Management                            │   │
│   └───────────┬──────────────────────┬──────────────────┘   │
└───────────────┼──────────────────────┼──────────────────────┘
                │                      │
                │ JWT Request          │ Video/Auth Requests
                ▼                      ▼
┌───────────────────────────┐  ┌─────────────────────────────┐
│   AUTH SERVER (IdP)       │  │   VIDEO SERVER (Resource)   │
│   Port: 4000              │  │   Port: 3070                │
├───────────────────────────┤  ├─────────────────────────────┤
│ - User Registration       │  │ - Video Upload              │
│ - Login / Signup          │  │ - FFmpeg Transcoding        │
│ - RS256 JWT Issuance      │  │ - HLS Adaptive Streaming    │
│ - Password Hashing        │  │ - HMAC Signed URLs          │
│                           │  │ - Per-User Isolation        │
│ ✅ Issues Tokens          │  │ ✅ Verifies Tokens (RSA)    │
└───────────┬───────────────┘  └──────────┬──────────────────┘
            │                             │
            │ MongoDB (authdb)            │ MongoDB (videodb)
            ▼                             ▼
        ┌────────────────────────────────────┐
        │    SHARED MONGODB INSTANCE         │
        │    Port: 27020                     │
        └────────────────────────────────────┘

```

---

## Key Design Principles

1.  **Stateless Authentication:**
    
    - Auth Server issues RS256-signed JWTs with a private key
        
    - Video Server verifies tokens using the public key (no database lookups)
        
    - Enables horizontal scaling and zero session storage
        
2.  **Service Isolation:**
    
    - Each service has its own MongoDB database (`authdb` / `videodb`)
        
    - Services communicate via REST APIs, not direct database access
        
    - Frontend connects to both services independently
        
3.  **Security Layers:**
    
    - **Layer 1:** JWT authentication for API access (issued by Auth Server)
        
    - **Layer 2:** HMAC-signed URLs for video streaming (prevents hotlinking)
        
    - **Layer 3:** Per-user data isolation enforced by `userId` claims
        

---

## 📦 Services

## 1. [Auth Server](https://github.com/anp3l/auth-server) (Identity Provider)

**Technology:** Node.js, Express, TypeScript, MongoDB, Bcrypt  
**Responsibilities:**

- User signup and login
    
- RS256 JWT token issuance
    
- Password security (bcrypt hashing)
    

**Why Separate?**  
Decoupling identity management from business logic allows:

- Reusing the same Auth Server for multiple resource servers
    
- Independent scaling based on authentication load
    
- Security-focused updates without touching video processing code
    

---

## 2. [Video Server](https://github.com/anp3l/remote-video-server) (Resource Server)

**Technology:** Node.js, Express, TypeScript, MongoDB, FFmpeg  
**Responsibilities:**

- Video upload and metadata management
    
- Adaptive Bitrate Transcoding (1080p/720p/480p/360p HLS)
    
- Secure streaming via HMAC-signed URLs
    
- Thumbnail generation (static + animated WebP)
    

**Key Features:**

- **Stateless Auth:** Verifies JWT signatures without calling Auth Server
    
- **Background Processing:** Asynchronous transcoding with status polling
    
- **Storage Efficiency:** Persistent Docker volumes for video files
    

---

## 3. [Frontend Client](https://github.com/anp3l/remote-video-client) (Angular SPA)

**Technology:** Angular 20, Material Design, Tailwind CSS, Video.js  
**Responsibilities:**

- User authentication flows (login/signup)
    
- Video library management (upload, edit, delete)
    
- HLS streaming playback with adaptive quality
    

**Architecture Highlights:**

- **Dual API Integration:** Orchestrates requests to Auth (port 4000) and Video (port 3070)
    
- **JWT Interceptor:** Auto-attaches tokens to Video API requests
    
- **Dockerized Nginx:** Production-ready static file serving
    

---

## 🔐 Security & Authentication Flow

```
┌─────────┐                  ┌──────────────┐                ┌──────────────┐
│ Client  │                  │ Auth Server  │                │ Video Server │
└────┬────┘                  └──────┬───────┘                └──────┬───────┘
     │                              │                               │
     │ 1. POST /auth/login          │                               │
     │ ────────────────────────────>│                               │
     │                              │                               │
     │ 2. Validate & Sign JWT       │                               │
     │    (RS256 with private key)  │                               │
     │ <────────────────────────────│                               │
     │                              │                               │
     │ 3. GET /videos (+ JWT header)│                               │
     │ ───────────────────────────────────────────────────────────> │
     │                              │                               │
     │                              │   4. Verify JWT (public key)  │
     │                              │ <──────────────────────────── │
     │                              │                               │
     │ 5. Signed streaming URL      │                               │
     │ <─────────────────────────────────────────────────────────── │
     │                              │                               │

```

**Why RS256 (RSA) instead of HS256 (HMAC)?**

- **Public Key Distribution:** Video Server doesn't need the private key (zero-trust model)
    
- **Multi-Service Support:** Same Auth Server can serve multiple resource servers
    
- **Industry Standard:** Uses the same cryptographic approach as OAuth 2.0 / OpenID Connect
    

---

## 🎥 Video Processing Pipeline

The platform uses **FFmpeg** for professional-grade transcoding:

| Quality | Resolution | Bitrate | Audio | Segment Size |
| --- | --- | --- | --- | --- |
| 1080p | 1920x1080 | 5000kbps | 192kbps | 4s  |
| 720p | 1280x720 | 2800kbps | 192kbps | 4s  |
| 480p | 854x480 | 1400kbps | 128kbps | 4s  |
| 360p | 640x360 | 800kbps | 96kbps | 4s  |

**Output:** HLS master playlist with automatic quality switching based on network conditions.

---

## 🛠️ Tech Stack

| Layer | Technology |
| --- | --- |
| **Frontend** | Angular 20, Material Design, Tailwind, Video.js |
| **Backend** | Node.js, Express, TypeScript |
| **Auth** | JWT (RS256), Bcrypt |
| **Database** | MongoDB 7.0 |
| **Media** | FFmpeg (H.264/AAC), HLS Protocol |
| **DevOps** | Docker, Docker Compose, Nginx |

---

## 📋 Prerequisites

- **Docker Desktop** or Docker Engine + Docker Compose
    
- **OpenSSL** (for RSA key generation)
    

---

## ⚙️ Configuration

## Step 1: Generate RSA Keys

```
# Generate private key
openssl genrsa -out private.pem 2048

# Extract public key
openssl rsa -in private.pem -pubout -out public.pem

# Convert to Base64 (Linux/Mac)
cat private.pem | base64 | tr -d '\n'
cat public.pem | base64 | tr -d '\n'
```

## Step 2: Generate Streaming Secret

```
openssl rand -base64 32
```

## Step 3: Update `.env`

```
cp .env.example .env
# Edit .env with the Base64 strings from above
```

---

## 🚦 Development & Deployment

## Start All Services

```
docker-compose up --build
```

## Start Individual Services

```
docker-compose up auth-service       # Auth only
docker-compose up video-service      # Video only
docker-compose up client             # Frontend only
```

## Stop & Clean

```
docker-compose down                  # Stop services
docker-compose down -v               # Stop + delete volumes (⚠️ deletes videos)
```

## View Logs

```
docker-compose logs -f auth-service   # Auth logs
docker-compose logs -f video-service  # Video logs
docker-compose logs -f client         # Frontend logs
```

---

## 🧪 Testing the Platform

1.  **Access Frontend:** [http://localhost:4200](http://localhost:4200/)
    
2.  **Create Account:** Sign up with email/password
    
3.  **Upload Video:** Select an MP4/MOV file (max 2GB)
    
4.  **Wait for Processing:** Poll status until transcoding completes
    
5.  **Stream Video:** Click the video card to open the HLS player
    

---

## 🗂️ Data Persistence

All data is persisted in Docker volumes:

- **`mongo-data`:** User accounts and video metadata
    
- **`video-uploads`:** Original videos + HLS segments
    

**Backup Strategy:**

```
# Export MongoDB
docker exec video-platform-db mongodump --out /backup

# Copy uploaded videos
docker cp video-server:/app/uploads ./backup/uploads
```

---

## 🔮 Future Improvements

This roadmap represents planned enhancements across the entire platform ecosystem:

### User Experience
- [ ] **Search & Filtering:** Full-text search across video library
- [ ] **Video Playlists:** Organize videos into custom collections
- [ ] **Drag-and-Drop Upload:** Simplified video upload interface
- [ ] **Dark Mode:** Theme toggle across frontend

### Authentication & Security
- [ ] **Password Reset:** Forgot password flow via email
- [ ] **Email Verification:** Account confirmation on signup
- [ ] **2FA Support:** Two-factor authentication

### System Architecture
- [ ] **JWKS Auto-Discovery:** Dynamic public key fetching from Auth Server
- [ ] **Admin Dashboard:** User management and storage monitoring
- [ ] **Rate Limiting:** API abuse prevention
    

---

## 📄 License

This project is licensed under the MIT License - see individual repositories for details.

---

## 🔗 Repository Links

- **Platform Orchestration (This Repo):** https://github.com/anp3l/remote-video-platform
    
- **Auth Server:** https://github.com/anp3l/auth-server
    
- **Video Server:** https://github.com/anp3l/remote-video-server
    
- **Frontend Client:** https://github.com/anp3l/remote-video-client
    

---

**Built by [Andrea Peluso](https://www.linkedin.com/in/andrea-peluso-052868386/)** | January 2026