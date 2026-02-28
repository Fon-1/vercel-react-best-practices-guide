# ARCHITECTURE — RealAR Platform

**Version:** 1.0  
**Date:** 2026-02-28

---

## 1. Tổng quan kiến trúc

```
┌─────────────────────────────────────────────────────────────┐
│                      FRONTEND (2 apps)                       │
│                                                              │
│  ┌───────────────────────┐  ┌──────────────────────────────┐ │
│  │      ar-viewer        │  │           admin              │ │
│  │   Vite + React + TS   │  │     Vite + React + TS        │ │
│  │                       │  │                              │ │
│  │  • Camera feed        │  │  • Login page                │ │
│  │  • MindAR tracking    │  │  • CRUD dự án                │ │
│  │  • Three.js 3D        │  │  • Upload model/catalog      │ │
│  │  • Info overlay       │  │  • Compile .mind (browser)   │ │
│  │  • Detail panel       │  │  • QR code generator         │ │
│  │  • QR scan entry      │  │  • Analytics charts          │ │
│  │                       │  │                              │ │
│  │  PORT: 5173           │  │  PORT: 5174                  │ │
│  └──────────┬────────────┘  └───────────────┬──────────────┘ │
│             │ HTTP / axios                  │ HTTP / axios    │
└─────────────┼──────────────────────────────┼────────────────┘
              │                              │
              ▼                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    BACKEND (Express)                          │
│                     PORT: 3000                               │
│                                                              │
│  PUBLIC endpoints (ar-viewer):                               │
│    GET  /api/projects/:slug   → project data                 │
│    POST /api/analytics/track  → log event                    │
│    GET  /files/*              → serve .mind / .glb / images  │
│                                                              │
│  PROTECTED endpoints (admin, JWT required):                  │
│    POST   /api/auth/login                                    │
│    GET    /api/auth/me                                       │
│    GET    /api/projects                                      │
│    POST   /api/projects                                      │
│    PUT    /api/projects/:id                                  │
│    DELETE /api/projects/:id                                  │
│    POST   /api/upload/image                                  │
│    POST   /api/upload/model                                  │
│    POST   /api/upload/target                                 │
│    GET    /api/analytics/:projectId                          │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────────────────────┐  │
│  │    SQLite DB     │  │       uploads/ folder            │  │
│  │ (better-sqlite3) │  │                                  │  │
│  │                  │  │  uploads/images/   → .jpg .png   │  │
│  │  • users         │  │  uploads/models/   → .glb .gltf  │  │
│  │  • projects      │  │  uploads/targets/  → .mind       │  │
│  │  • project_imgs  │  │                                  │  │
│  │  • analytics     │  │  Serve qua GET /files/*          │  │
│  └──────────────────┘  └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Monorepo Structure

```
real-estate-ar/
├── apps/
│   ├── ar-viewer/                     # WebAR App
│   │   ├── src/
│   │   │   ├── components/
│   │   │   │   ├── ARScene.tsx        # MindAR + Three.js container
│   │   │   │   ├── ModelViewer.tsx    # GLB loader + render
│   │   │   │   ├── InfoOverlay.tsx    # Floating card tên/giá/diện tích
│   │   │   │   ├── DetailPanel.tsx    # Slide-up bottom sheet
│   │   │   │   ├── LoadingScreen.tsx  # Loading animation + steps
│   │   │   │   └── QRScanner.tsx      # Camera QR scan
│   │   │   ├── hooks/
│   │   │   │   ├── useMindAR.ts       # MindAR lifecycle hook
│   │   │   │   └── useProject.ts      # Fetch project từ API
│   │   │   ├── pages/
│   │   │   │   ├── ARPage.tsx         # /view/:slug
│   │   │   │   ├── ScanQRPage.tsx     # / (landing)
│   │   │   │   └── NotFoundPage.tsx   # 404
│   │   │   ├── types/
│   │   │   │   └── project.ts         # TypeScript interfaces
│   │   │   ├── App.tsx                # Router setup
│   │   │   └── main.tsx
│   │   ├── public/
│   │   │   └── draco/                 # DRACOLoader decoder files
│   │   ├── index.html
│   │   ├── package.json
│   │   └── vite.config.ts
│   │
│   └── admin/                         # Admin Dashboard
│       ├── src/
│       │   ├── components/
│       │   │   ├── layout/
│       │   │   │   ├── Layout.tsx
│       │   │   │   ├── Sidebar.tsx
│       │   │   │   └── Header.tsx
│       │   │   ├── projects/
│       │   │   │   ├── ProjectList.tsx
│       │   │   │   ├── ProjectForm.tsx
│       │   │   │   ├── ProjectCard.tsx
│       │   │   │   └── QRCodeDisplay.tsx
│       │   │   ├── upload/
│       │   │   │   ├── ImageUploader.tsx
│       │   │   │   ├── ModelUploader.tsx
│       │   │   │   └── MindCompiler.tsx   # ★ Core: compile .mind
│       │   │   └── analytics/
│       │   │       └── StatsCard.tsx
│       │   ├── pages/
│       │   │   ├── LoginPage.tsx
│       │   │   ├── DashboardPage.tsx
│       │   │   ├── ProjectsPage.tsx
│       │   │   ├── ProjectDetailPage.tsx
│       │   │   └── AnalyticsPage.tsx
│       │   ├── services/
│       │   │   └── api.ts             # Axios instance + interceptors
│       │   ├── store/
│       │   │   └── authStore.ts       # Zustand auth state
│       │   ├── App.tsx
│       │   └── main.tsx
│       ├── package.json
│       └── vite.config.ts
│
├── server/                            # Express Backend
│   ├── src/
│   │   ├── routes/
│   │   │   ├── auth.ts
│   │   │   ├── projects.ts
│   │   │   ├── upload.ts
│   │   │   └── analytics.ts
│   │   ├── middleware/
│   │   │   ├── auth.ts                # JWT verify
│   │   │   └── upload.ts              # Multer config
│   │   ├── db/
│   │   │   ├── schema.ts              # SQL DDL
│   │   │   └── index.ts               # SQLite connection + seed
│   │   └── index.ts                   # Express app entry
│   ├── uploads/                       # Runtime file storage
│   │   ├── images/
│   │   ├── models/
│   │   └── targets/
│   ├── package.json
│   └── tsconfig.json
│
├── docs/                              # Documentation
│   ├── SRS.md
│   ├── ARCHITECTURE.md
│   ├── TECHNICAL-ANALYSIS.md
│   └── IMPLEMENTATION-STEPS.md
├── .opencode/plans/
│   └── real-estate-webar.md          # Master plan
├── package.json                       # npm workspaces root
└── README.md
```

---

## 3. Database Schema

```sql
-- Admin accounts
CREATE TABLE users (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  email         TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  name          TEXT,
  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Dự án bất động sản
CREATE TABLE projects (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  slug          TEXT NOT NULL UNIQUE,     -- URL-friendly, dùng trong QR
  name          TEXT NOT NULL,            -- Tên dự án
  developer     TEXT,                     -- Chủ đầu tư
  price_from    INTEGER,                  -- Giá từ (VND)
  price_to      INTEGER,                  -- Giá đến (VND)
  area_from     REAL,                     -- Diện tích từ (m²)
  area_to       REAL,                     -- Diện tích đến (m²)
  description   TEXT,                     -- Mô tả chi tiết
  address       TEXT,                     -- Địa chỉ
  features      TEXT,                     -- JSON: ["Hồ bơi","Gym",...]
  cover_image   TEXT,                     -- filename trong uploads/images/
  mind_file     TEXT,                     -- filename trong uploads/targets/
  model_file    TEXT,                     -- filename trong uploads/models/
  model_scale   REAL    DEFAULT 0.1,      -- Scale model 3D
  contact_phone TEXT,
  contact_email TEXT,
  status        TEXT    DEFAULT 'draft',  -- draft | active | inactive
  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Gallery ảnh của dự án
CREATE TABLE project_images (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  project_id  INTEGER REFERENCES projects(id) ON DELETE CASCADE,
  image_path  TEXT NOT NULL,             -- filename trong uploads/images/
  caption     TEXT,
  sort_order  INTEGER DEFAULT 0
);

-- Analytics events
CREATE TABLE analytics (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  project_id   INTEGER REFERENCES projects(id) ON DELETE CASCADE,
  event_type   TEXT NOT NULL,            -- qr_scan | ar_view | detail_open | contact_tap
  user_agent   TEXT,
  ip_address   TEXT,
  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

> **Note:** DB lưu **filename** (e.g. `uuid.glb`), không lưu full URL. URL được build ở runtime: `${API_URL}/files/models/${filename}`

---

## 4. API Contract

### Auth

```
POST /api/auth/login
  Body:     { email: string, password: string }
  Success:  200 { token: string, user: { id, email, name } }
  Error:    401 { error: "Email hoặc mật khẩu không đúng" }

GET /api/auth/me  [AUTH]
  Success:  200 { id, email, name }
  Error:    401 { error: "Unauthorized" }
```

### Projects

```
GET /api/projects  [AUTH]
  Query:    ?page=1&limit=10&status=active&search=sun
  Success:  200 { data: Project[], total: number, page: number }

GET /api/projects/:slug  [PUBLIC]
  Success:  200 Project (full, bao gồm images[])
  Error:    404 { error: "Dự án không tồn tại" }

POST /api/projects  [AUTH]
  Body:     { name, developer, priceFrom, priceTo, areaFrom, areaTo,
              description, address, features[], contactPhone, contactEmail }
  Success:  201 { id, slug, ...project }
  Error:    400 { error: "Tên dự án không được để trống" }

PUT /api/projects/:id  [AUTH]
  Body:     Partial<Project>
  Success:  200 Updated Project
  Error:    404 { error: "Không tìm thấy dự án" }

DELETE /api/projects/:id  [AUTH]
  Success:  200 { ok: true }
  Error:    404 { error: "Không tìm thấy dự án" }
```

### Upload

```
POST /api/upload/image  [AUTH]
  Body:     multipart/form-data, field "file" (jpg/png/webp, max 5MB)
  Success:  200 { url: "/files/images/uuid.jpg", filename: "uuid.jpg" }
  Error:    413 { error: "File quá lớn, tối đa 5MB" }
            400 { error: "Chỉ chấp nhận file ảnh" }

POST /api/upload/model  [AUTH]
  Body:     multipart/form-data, field "file" (.glb/.gltf, max 50MB)
  Success:  200 { url: "/files/models/uuid.glb", filename: "uuid.glb" }

POST /api/upload/target  [AUTH]
  Body:     multipart/form-data, field "file" (.mind binary, max 10MB)
  Success:  200 { url: "/files/targets/uuid.mind", filename: "uuid.mind" }

GET /files/*  [PUBLIC]
  Serve static files từ uploads/ folder
```

### Analytics

```
POST /api/analytics/track  [PUBLIC]
  Body:     { projectId: number, eventType: "qr_scan"|"ar_view"|"detail_open"|"contact_tap" }
  Success:  200 { ok: true }

GET /api/analytics/:projectId  [AUTH]
  Success:  200 {
    totalViews: number,
    totalQRScans: number,
    totalDetailOpens: number,
    last7Days: [{ date: string, views: number }]
  }
```

---

## 5. Deploy Architecture

```
Production:

  ar.yourdomain.com     → Vercel (ar-viewer static build)
  admin.yourdomain.com  → Vercel (admin static build)
  api.yourdomain.com    → Render / Railway (Express server Docker)
                          Volume: /app/uploads + /app/database.db

Local dev:

  https://localhost:5173  → ar-viewer (mkcert HTTPS, camera cần HTTPS)
  http://localhost:5174   → admin
  http://localhost:3000   → server
```

### COOP Header (bắt buộc cho MindAR)

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

MindAR dùng `SharedArrayBuffer` cho WebWorker — yêu cầu 2 headers này trên production. Vercel config trong `vercel.json`.

---

## 6. Data Flow

### Luồng Admin tạo dự án AR

```
1. Admin điền form → POST /api/projects → DB insert → nhận slug
2. Admin upload ảnh catalog → POST /api/upload/image → uploads/images/
3. Admin compile .mind (browser):
   - FileReader → HTMLImageElement
   - MindAR Compiler (WebWorker) → ArrayBuffer
   - POST /api/upload/target → uploads/targets/
4. Admin upload model .glb → POST /api/upload/model → uploads/models/
5. Admin PUT /api/projects/:id { mind_file, model_file, cover_image }
6. Admin xem QR (generate client-side từ slug)
```

### Luồng Viewer xem AR

```
1. Viewer quét QR → redirect https://ar.domain.com/view/{slug}
2. ar-viewer GET /api/projects/{slug} → nhận { mindFile, modelFile, ... }
3. ar-viewer load .mind từ GET /files/targets/{mindFile}
4. ar-viewer load .glb từ GET /files/models/{modelFile}
5. MindARThree khởi động camera → scan frame liên tục
6. Khi match → anchor.onTargetFound → Three.js render model
7. POST /api/analytics/track { projectId, eventType: 'ar_view' }
```
