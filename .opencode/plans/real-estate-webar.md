# Real Estate WebAR Platform — Kế hoạch chi tiết (SRS + Steps + Outputs)

> Mục tiêu: Build một nền tảng WebAR mã nguồn mở cho ngành bất động sản, tương tự Zappar/8th Wall nhưng tự host, cho phép nhân viên sale quét catalog/brochure để hiện model 3D tòa nhà, thông tin dự án ngay trên trình duyệt điện thoại.

---

## 1. SOFTWARE REQUIREMENTS SPECIFICATION (SRS)

### 1.1 Tổng quan hệ thống

| Thuộc tính | Giá trị |
|---|---|
| Tên dự án | RealAR – Real Estate WebAR Platform |
| Loại | Web Application (PWA-ready) |
| Người dùng | Nhân viên sale BDS, khách hàng |
| Môi trường | Trình duyệt mobile (Chrome/Safari), desktop |
| Ngôn ngữ | Tiếng Việt (UI), TypeScript (code) |

### 1.2 Actors (người dùng hệ thống)

| Actor | Mô tả |
|---|---|
| **Admin** | Quản lý dự án BDS, upload model 3D, upload catalog image, compile .mind file, tạo QR |
| **Sale Staff** | Dùng điện thoại, mở URL/quét QR, hướng camera vào catalog để xem AR |
| **Khách hàng** | Cùng xem với sale staff trên điện thoại sale |

### 1.3 Functional Requirements

#### FR-01: AR Viewer (ứng dụng WebAR)
| ID | Mô tả | Độ ưu tiên |
|---|---|---|
| FR-01-01 | Xin quyền camera và hiển thị camera feed fullscreen | MUST |
| FR-01-02 | Load file .mind (image targets) từ server theo project ID | MUST |
| FR-01-03 | Nhận diện ảnh từ catalog (image tracking) bằng MindAR | MUST |
| FR-01-04 | Load và render model 3D (.glb) từ server lên anchor khi nhận diện | MUST |
| FR-01-05 | Hiển thị overlay: tên dự án, giá, diện tích phía trên model | MUST |
| FR-01-06 | Khi tap/click vào vùng AR → slide-up panel thông tin chi tiết | MUST |
| FR-01-07 | QR code entry: quét QR → redirect đến AR viewer đúng project | MUST |
| FR-01-08 | Loading screen animation trong lúc MindAR khởi động (3-5s) | MUST |
| FR-01-09 | Hỗ trợ nhiều image targets trong 1 catalog (multi-target) | SHOULD |
| FR-01-10 | Nút "Liên hệ ngay" trong detail panel → mở phone/form | SHOULD |

#### FR-02: Admin Dashboard
| ID | Mô tả | Độ ưu tiên |
|---|---|---|
| FR-02-01 | Đăng nhập bằng email/password, JWT token | MUST |
| FR-02-02 | Dashboard tổng quan: số dự án, lượt xem AR (analytics) | MUST |
| FR-02-03 | Tạo/sửa/xóa dự án BDS (tên, giá, diện tích, mô tả, ảnh bìa) | MUST |
| FR-02-04 | Upload ảnh catalog/brochure cho từng dự án | MUST |
| FR-02-05 | Compile ảnh catalog → file .mind (image target) trực tiếp trong admin | MUST |
| FR-02-06 | Upload model 3D (.glb/.gltf) cho từng dự án | MUST |
| FR-02-07 | Tạo và download QR code cho từng dự án | MUST |
| FR-02-08 | Preview link AR viewer cho từng dự án | MUST |
| FR-02-09 | Xem analytics: lượt quét QR, lượt view AR theo dự án | SHOULD |

#### FR-03: Backend API
| ID | Mô tả | Độ ưu tiên |
|---|---|---|
| FR-03-01 | Auth API: POST /api/auth/login, POST /api/auth/logout | MUST |
| FR-03-02 | Projects CRUD: GET/POST/PUT/DELETE /api/projects | MUST |
| FR-03-03 | File upload: POST /api/upload (image, .glb, .mind) | MUST |
| FR-03-04 | Compile API: POST /api/compile → nhận ảnh, trả về .mind file | MUST |
| FR-03-05 | Static file serve: /files/:filename | MUST |
| FR-03-06 | Analytics API: POST /api/analytics/track, GET /api/analytics/:projectId | SHOULD |

### 1.4 Non-Functional Requirements

| ID | Mô tả |
|---|---|
| NFR-01 | AR Viewer load trong < 5 giây trên mạng 4G |
| NFR-02 | MindAR tracking phải chạy ≥ 30fps trên iPhone 11 / Android mid-range |
| NFR-03 | HTTPS bắt buộc (camera API requirement) |
| NFR-04 | Mobile-first UI: tất cả tương tác tối ưu cho màn hình < 430px |
| NFR-05 | Hỗ trợ Chrome Android 90+, Safari iOS 14.3+ |
| NFR-06 | Admin dashboard responsive cho desktop (1280px+) |
| NFR-07 | File .glb tối đa 50MB, ảnh catalog tối đa 5MB |

---

## 2. KIẾN TRÚC HỆ THỐNG

### 2.1 Cấu trúc thư mục (monorepo)

```
real-estate-ar/
├── apps/
│   ├── ar-viewer/                    # WebAR App
│   │   ├── src/
│   │   │   ├── components/
│   │   │   │   ├── ARScene.tsx       # MindAR + Three.js core
│   │   │   │   ├── ModelViewer.tsx   # GLB model loader/renderer
│   │   │   │   ├── InfoOverlay.tsx   # Overlay tên/giá/diện tích
│   │   │   │   ├── DetailPanel.tsx   # Slide-up detail panel
│   │   │   │   ├── LoadingScreen.tsx # Loading animation
│   │   │   │   └── QRScanner.tsx     # QR code scan entry
│   │   │   ├── hooks/
│   │   │   │   ├── useMindAR.ts      # MindAR initialization hook
│   │   │   │   └── useProject.ts     # Fetch project data from API
│   │   │   ├── pages/
│   │   │   │   ├── ARPage.tsx        # Main AR experience page
│   │   │   │   └── ScanQRPage.tsx    # QR scan landing page
│   │   │   ├── types/
│   │   │   │   └── project.ts        # Project TypeScript types
│   │   │   ├── App.tsx
│   │   │   └── main.tsx
│   │   ├── public/
│   │   │   └── models/               # Demo .glb model files
│   │   ├── index.html
│   │   ├── package.json
│   │   └── vite.config.ts
│   │
│   └── admin/                        # Admin Dashboard App
│       ├── src/
│       │   ├── components/
│       │   │   ├── layout/
│       │   │   │   ├── Sidebar.tsx
│       │   │   │   ├── Header.tsx
│       │   │   │   └── Layout.tsx
│       │   │   ├── projects/
│       │   │   │   ├── ProjectList.tsx
│       │   │   │   ├── ProjectForm.tsx
│       │   │   │   ├── ProjectCard.tsx
│       │   │   │   └── QRCodeDisplay.tsx
│       │   │   ├── upload/
│       │   │   │   ├── ImageUploader.tsx
│       │   │   │   ├── ModelUploader.tsx
│       │   │   │   └── MindCompiler.tsx  # Compile .mind in browser
│       │   │   └── analytics/
│       │   │       └── StatsCard.tsx
│       │   ├── pages/
│       │   │   ├── LoginPage.tsx
│       │   │   ├── DashboardPage.tsx
│       │   │   ├── ProjectsPage.tsx
│       │   │   ├── ProjectDetailPage.tsx
│       │   │   └── AnalyticsPage.tsx
│       │   ├── services/
│       │   │   └── api.ts            # Axios API client
│       │   ├── store/
│       │   │   └── authStore.ts      # Zustand auth state
│       │   ├── App.tsx
│       │   └── main.tsx
│       ├── package.json
│       └── vite.config.ts
│
├── server/                           # Express Backend
│   ├── src/
│   │   ├── routes/
│   │   │   ├── auth.ts
│   │   │   ├── projects.ts
│   │   │   ├── upload.ts
│   │   │   ├── compile.ts
│   │   │   └── analytics.ts
│   │   ├── middleware/
│   │   │   ├── auth.ts               # JWT middleware
│   │   │   └── upload.ts             # Multer config
│   │   ├── db/
│   │   │   ├── schema.ts             # SQLite schema
│   │   │   └── index.ts              # Database connection
│   │   ├── services/
│   │   │   └── mindCompiler.ts       # Server-side .mind compiler
│   │   └── index.ts                  # Express app entry
│   ├── uploads/                      # Uploaded files storage
│   │   ├── images/
│   │   ├── models/
│   │   └── targets/                  # .mind files
│   ├── package.json
│   └── tsconfig.json
│
├── package.json                      # Root workspace config
└── README.md
```

### 2.2 Database Schema (SQLite)

```sql
-- Users table
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  name TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Projects table (dự án BDS)
CREATE TABLE projects (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  slug TEXT NOT NULL UNIQUE,         -- URL-friendly ID, dùng trong QR
  name TEXT NOT NULL,                -- Tên dự án
  developer TEXT,                    -- Chủ đầu tư
  price_from INTEGER,                -- Giá từ (VND)
  price_to INTEGER,                  -- Giá đến (VND)
  area_from REAL,                    -- Diện tích từ (m²)
  area_to REAL,                      -- Diện tích đến (m²)
  description TEXT,                  -- Mô tả chi tiết
  address TEXT,                      -- Địa chỉ
  features TEXT,                     -- JSON array: ["Hồ bơi", "Gym", ...]
  cover_image TEXT,                  -- Đường dẫn ảnh bìa
  mind_file TEXT,                    -- Đường dẫn .mind file
  model_file TEXT,                   -- Đường dẫn .glb file
  model_scale REAL DEFAULT 0.1,      -- Scale của model 3D
  contact_phone TEXT,
  contact_email TEXT,
  status TEXT DEFAULT 'active',      -- active | inactive | draft
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Project images (gallery)
CREATE TABLE project_images (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
  image_path TEXT NOT NULL,
  caption TEXT,
  sort_order INTEGER DEFAULT 0
);

-- Analytics events
CREATE TABLE analytics (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  project_id INTEGER REFERENCES projects(id),
  event_type TEXT NOT NULL,   -- 'qr_scan' | 'ar_view' | 'detail_open' | 'contact_tap'
  user_agent TEXT,
  ip_address TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 3. PHÂN TÍCH KỸ THUẬT QUAN TRỌNG

### 3.1 MindAR Integration — Cách hoạt động

```
Luồng AR:
1. Admin upload ảnh catalog (JPG/PNG)
2. MindAR compiler (browser-based WebWorker) xử lý ảnh:
   - Detect feature points (góc, texture pattern)
   - Encode thành .mind binary format
3. .mind file lưu server, link với project
4. AR Viewer:
   a. Load camera stream (getUserMedia)
   b. Khởi tạo MindARThree({ imageTargetSrc: '.mind URL' })
   c. MindAR liên tục analyze camera frame
   d. Khi detect match → fire 'targetFound' event
   e. Three.js renderer hiển thị model 3D tại vị trí anchor
```

**Key API calls:**
```typescript
// Initialization
const mindarThree = new MindARThree({
  container: containerRef.current,
  imageTargetSrc: `${API_URL}/files/targets/${project.mind_file}`,
  maxTrack: 1,
  filterMinCF: 0.001,  // giảm jitter
  filterBeta: 0.001,
});

// Event listeners
const anchor = mindarThree.addAnchor(0); // target index 0
anchor.group.add(modelMesh);

anchor.onTargetFound = () => setTracking(true);
anchor.onTargetLost = () => setTracking(false);

await mindarThree.start();
```

### 3.2 Image Target Compilation — Trong admin

MindAR cung cấp `Compiler` class có thể chạy trong browser:
```typescript
import { Compiler } from 'mind-ar/dist/mindar-image.prod.js';

const compiler = new Compiler();
await compiler.compileImageTargets([imageElement], (progress) => {
  setCompileProgress(progress * 100); // 0-100%
});
const buffer = await compiler.exportData();
// buffer → Blob → upload lên server
```

### 3.3 3D Model Display

```typescript
// Three.js GLTF loader
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader';

const loader = new GLTFLoader();
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/'); // compressed models
loader.setDRACOLoader(dracoLoader);

loader.load(modelUrl, (gltf) => {
  const model = gltf.scene;
  model.scale.set(scale, scale, scale);
  anchor.group.add(model);
  // Animate model rotation
  renderer.setAnimationLoop(() => {
    model.rotation.y += 0.01;
    renderer.render(scene, camera);
  });
});
```

### 3.4 QR Code

- Mỗi project có slug unique (e.g., `du-an-sun-grand-city`)
- AR Viewer URL: `https://ar.example.com/view/{slug}`
- QR code generate bằng `qrcode` npm package trên client
- QR hiển thị trong admin, có thể download PNG

---

## 4. IMPLEMENTATION STEPS CHI TIẾT

---

### STEP 1: Setup Monorepo

**Mục tiêu:** Khởi tạo toàn bộ project structure, cài dependencies

**Các task:**
1. Tạo root `package.json` với npm workspaces config
2. Tạo `apps/ar-viewer/` bằng `npm create vite@latest` (React + TypeScript)
3. Tạo `apps/admin/` bằng `npm create vite@latest` (React + TypeScript)
4. Tạo `server/` với `package.json` (Express + TypeScript)
5. Cài dependencies cho từng app:
   - ar-viewer: `mind-ar`, `three`, `@react-three/fiber`, `@react-three/drei`, `react-router-dom`, `axios`
   - admin: `react-router-dom`, `axios`, `zustand`, `@tanstack/react-query`, `qrcode`, `react-dropzone`, `recharts`, `shadcn/ui`, `tailwindcss`
   - server: `express`, `better-sqlite3`, `bcryptjs`, `jsonwebtoken`, `multer`, `cors`, `dotenv`
6. Setup TypeScript tsconfig cho từng package
7. Setup Tailwind CSS + shadcn/ui cho admin
8. Tạo `.env.example` files

**Output sau step này:**
- `npm run dev` chạy được ar-viewer tại `http://localhost:5173`
- `npm run dev` chạy được admin tại `http://localhost:5174`
- `npm run dev` chạy được server tại `http://localhost:3000`
- Không có lỗi TypeScript compile

---

### STEP 2: Backend API + Database

**Mục tiêu:** Build Express server với đầy đủ CRUD API, auth, file upload

**Các task:**
1. **Database setup** (`server/src/db/`):
   - Tạo `schema.ts` với SQL DDL cho 4 tables (users, projects, project_images, analytics)
   - Tạo `index.ts` kết nối SQLite, chạy migrations, seed admin user mặc định
   - Admin mặc định: `admin@realear.vn` / `Admin@123`

2. **Auth routes** (`server/src/routes/auth.ts`):
   - `POST /api/auth/login` → validate credentials → return JWT (expires 24h)
   - `POST /api/auth/logout` → client-side token clear
   - `GET /api/auth/me` → return current user info

3. **Auth middleware** (`server/src/middleware/auth.ts`):
   - Verify Bearer JWT token
   - Attach `req.user` nếu valid

4. **Projects routes** (`server/src/routes/projects.ts`):
   - `GET /api/projects` → list all (có pagination, filter by status)
   - `GET /api/projects/:slug` → get 1 project (PUBLIC, không cần auth)
   - `POST /api/projects` → tạo mới (auth required)
   - `PUT /api/projects/:id` → cập nhật (auth required)
   - `DELETE /api/projects/:id` → xóa (auth required)

5. **Upload routes** (`server/src/routes/upload.ts`):
   - `POST /api/upload/image` → upload ảnh (multer, lưu vào `uploads/images/`)
   - `POST /api/upload/model` → upload .glb file (lưu vào `uploads/models/`)
   - `POST /api/upload/target` → upload .mind file đã compile (lưu vào `uploads/targets/`)
   - `GET /files/*` → serve static files từ `uploads/` folder

6. **Analytics routes** (`server/src/routes/analytics.ts`):
   - `POST /api/analytics/track` → log event (project_id, event_type)
   - `GET /api/analytics/:projectId` → stats cho 1 project (auth)

7. **Error handling & CORS config**

**Output sau step này:**
- `GET http://localhost:3000/api/projects` → trả về `[]` JSON
- `POST /api/auth/login` với credentials đúng → trả về JWT token
- `POST /api/upload/image` với form-data → trả về file URL
- File `server/uploads/` được tạo tự động
- Tất cả routes có proper error handling (404, 401, 500)
- Test được bằng curl hoặc Postman

---

### STEP 3: AR Viewer Core

**Mục tiêu:** Build WebAR experience hoàn chỉnh — camera, tracking, 3D model

**Các task:**

1. **Custom hook `useMindAR`** (`ar-viewer/src/hooks/useMindAR.ts`):
   ```
   Input: containerRef, mindFileUrl, modelUrl, scale
   Output: { isStarted, isTracking, error, start, stop }
   ```
   - Khởi tạo MindARThree
   - Load GLTFLoader + DRACOLoader
   - Setup anchor events (targetFound/targetLost)
   - Cleanup khi unmount (stop AR, dispose Three.js objects)
   - Handle lỗi: camera permission denied, file not found

2. **Component `ARScene`** (`ar-viewer/src/components/ARScene.tsx`):
   - Container fullscreen (100vw, 100vh)
   - Gọi `useMindAR` hook
   - Quản lý state: loading → started → tracking → lost
   - Pass tracking state xuống children components

3. **Component `ModelViewer`** (`ar-viewer/src/components/ModelViewer.tsx`):
   - Load .glb model với progress indicator
   - Auto-rotate animation khi đang tracking
   - Optimize: frustum culling, LOD nếu model phức tạp

4. **Component `InfoOverlay`** (`ar-viewer/src/components/InfoOverlay.tsx`):
   - Chỉ hiện khi `isTracking = true`
   - Hiện floating card với: Tên dự án, giá từ X tỷ, diện tích Xm²
   - Animation: fade-in khi appear
   - Position: top 20% của screen (không che camera)

5. **Component `LoadingScreen`** (`ar-viewer/src/components/LoadingScreen.tsx`):
   - Full-screen overlay với logo + animation
   - Progress steps: "Đang khởi động camera..." → "Đang tải mô hình..." → "Sẵn sàng quét!"
   - Hướng dẫn scan: ảnh minh hoạ catalog + text "Hướng camera vào catalog"

6. **Component `DetailPanel`** (`ar-viewer/src/components/DetailPanel.tsx`):
   - Slide-up sheet từ bottom (CSS transition)
   - Trigger: tap/click vào canvas khi đang tracking
   - Content: ảnh gallery (swiper), mô tả dự án, features list, nút "Gọi ngay", nút "Email"
   - Dismiss: swipe down hoặc tap backdrop

7. **Page `ARPage`** (`ar-viewer/src/pages/ARPage.tsx`):
   - Route: `/view/:slug`
   - Fetch project data từ `GET /api/projects/:slug`
   - Render: LoadingScreen → ARScene → InfoOverlay + DetailPanel
   - Track analytics: 'ar_view' event on load, 'detail_open' on panel open

8. **Page `ScanQRPage`** (`ar-viewer/src/pages/ScanQRPage.tsx`):
   - Route: `/`
   - Landing page đẹp với logo, tagline
   - Nút "Quét QR" → mở camera scan QR (dùng `html5-qrcode` hoặc jsQR)
   - Redirect đến `/view/:slug` sau khi scan thành công
   - Alternative: input field nhập slug thủ công

**Output sau step này:**
- Mở `http://localhost:5173/view/demo-project` → thấy camera feed
- Đưa ảnh test image vào camera → model 3D xuất hiện lên ảnh
- Info overlay hiện với tên/giá/diện tích
- Tap → detail panel slide up
- Loading screen animation đẹp
- Chạy tốt trên Chrome desktop (webcam)

---

### STEP 4: Admin Dashboard

**Mục tiêu:** Build admin panel để quản lý dự án, upload files, tạo QR

**Các task:**

1. **Auth & Layout:**
   - `LoginPage.tsx`: Form email/password, gọi `/api/auth/login`, lưu JWT vào localStorage
   - `authStore.ts` (Zustand): lưu user state, token, isAuthenticated
   - `Layout.tsx`: Sidebar navigation + header, protected route wrapper
   - Sidebar items: Dashboard, Dự án, Thêm dự án, Analytics

2. **Dashboard Page** (`DashboardPage.tsx`):
   - Stats cards: Tổng dự án, Dự án active, Lượt quét AR hôm nay, Lượt xem tuần
   - Quick access: 3 dự án gần nhất
   - Charts (Recharts): Line chart lượt xem 7 ngày, Pie chart top dự án

3. **Project List Page** (`ProjectsPage.tsx`):
   - Bảng danh sách dự án: tên, giá, trạng thái, ngày tạo, actions
   - Filter: all / active / draft / inactive
   - Search bar
   - Nút "Thêm dự án mới"
   - Action buttons: Xem AR | Sửa | Xóa | Copy QR link

4. **Project Form Page** (`ProjectDetailPage.tsx`):
   - Form 2 cột cho màn desktop, 1 cột cho mobile
   - **Tab 1 - Thông tin cơ bản:**
     - Tên dự án (required)
     - Chủ đầu tư
     - Giá từ - đến (number input, format tiền VND)
     - Diện tích từ - đến (m²)
     - Mô tả (rich text hoặc textarea)
     - Địa chỉ
     - Tiện ích (tag input: Hồ bơi, Gym, Trường học...)
     - Thông tin liên hệ (phone, email)
     - Status dropdown
   - **Tab 2 - Media:**
     - Upload ảnh bìa (react-dropzone, preview)
     - Upload gallery (multiple images, drag reorder)
     - Upload model 3D (.glb) với file size indicator
   - **Tab 3 - AR Setup:**
     - `MindCompiler.tsx` component:
       - Drag & drop ảnh catalog để compile
       - Hiển thị progress bar khi đang compile (0-100%)
       - Preview feature points visualization sau khi compile xong
       - Nút "Lưu target" → upload .mind file lên server
     - Model scale slider (0.01 - 1.0)
     - Preview link AR viewer
   - **Tab 4 - QR Code:**
     - Hiển thị QR code lớn (300x300px)
     - Nút "Download PNG"
     - Nút "Copy link"
     - Hướng dẫn in QR lên catalog

5. **MindCompiler Component** (`upload/MindCompiler.tsx`) - Component quan trọng nhất:
   ```
   State machine:
   idle → uploading_image → compiling (0→100%) → success | error
   ```
   - Import MindAR Compiler class từ `mind-ar/dist/mindar-image.prod.js`
   - Hiển thị canvas preview feature points
   - Worker-based compilation không block UI
   - Feedback rõ ràng: "Ảnh tốt / Ảnh cần nhiều texture hơn để tracking tốt"

**Output sau step này:**
- Đăng nhập được bằng admin@realear.vn / Admin@123
- Tạo dự án mới, điền thông tin, save thành công
- Upload ảnh catalog → thấy progress bar compile → .mind file được lưu
- Upload model .glb → file được serve tại `/files/models/xxx.glb`
- Trang QR hiện QR code, download được PNG
- Link "Xem AR" mở đúng ar-viewer với đúng project

---

### STEP 5: Integration & Polish

**Mục tiêu:** Kết nối đầy đủ frontend-backend, tối ưu UX mobile

**Các task:**

1. **Kết nối AR Viewer với API thực:**
   - Fetch project data từ server thay vì hardcode
   - Error handling: project không tồn tại → 404 page đẹp
   - Analytics tracking tự động

2. **Mobile UI optimization:**
   - Test trên iPhone/Android thực (hoặc Chrome DevTools device mode)
   - Touch interactions cho Detail Panel (touch-action: pan-y)
   - Safe area insets (notch, home indicator)
   - Font size tối thiểu 16px để không zoom khi focus input
   - Landscape mode handling

3. **AR Performance tuning:**
   - filterMinCF và filterBeta settings để giảm jitter
   - Model complexity: đảm bảo .glb < 5MB cho load nhanh
   - Lazy load: chỉ load model khi tracking lần đầu

4. **Loading & Error states:**
   - Skeleton loaders trong admin
   - Toast notifications (shadcn/ui Toaster): upload thành công, lỗi API
   - Confirm dialog trước khi xóa dự án

5. **HTTPS setup for local dev:**
   - `vite.config.ts` với `server.https` dùng mkcert certificate
   - Để test camera trên điện thoại kết nối cùng WiFi

6. **Deploy configuration:**
   - `vercel.json` cho ar-viewer và admin (static + API proxy)
   - `render.yaml` hoặc `Dockerfile` cho server
   - `.env.production` template

**Output sau step này:**
- Toàn bộ luồng hoạt động end-to-end:
  1. Admin tạo dự án → upload .mind + .glb → lấy QR
  2. Sale staff quét QR bằng điện thoại
  3. AR viewer mở, loading screen → hướng camera vào catalog
  4. Model 3D xuất hiện → thông tin overlay → tap → detail
- Không có console errors
- UI đẹp, smooth animations
- Deploy được lên Vercel + Render

---

## 5. DEPENDENCIES ĐẦY ĐỦ

### apps/ar-viewer/package.json
```json
{
  "dependencies": {
    "mind-ar": "^1.2.5",
    "three": "^0.160.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "react-router-dom": "^6.22.0",
    "axios": "^1.6.0",
    "html5-qrcode": "^2.3.8"
  },
  "devDependencies": {
    "@types/three": "^0.160.0",
    "vite": "^5.0.0",
    "@vitejs/plugin-react": "^4.2.0",
    "typescript": "^5.3.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.0"
  }
}
```

### apps/admin/package.json
```json
{
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "react-router-dom": "^6.22.0",
    "axios": "^1.6.0",
    "zustand": "^4.5.0",
    "@tanstack/react-query": "^5.17.0",
    "mind-ar": "^1.2.5",
    "qrcode": "^1.5.3",
    "react-dropzone": "^14.2.3",
    "recharts": "^2.10.0",
    "lucide-react": "^0.312.0",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-tabs": "^1.0.4",
    "@radix-ui/react-toast": "^1.1.5",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0"
  },
  "devDependencies": {
    "vite": "^5.0.0",
    "@vitejs/plugin-react": "^4.2.0",
    "typescript": "^5.3.0",
    "tailwindcss": "^3.4.0"
  }
}
```

### server/package.json
```json
{
  "dependencies": {
    "express": "^4.18.2",
    "better-sqlite3": "^9.4.3",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.2",
    "multer": "^1.4.5",
    "cors": "^2.8.5",
    "dotenv": "^16.4.1",
    "uuid": "^9.0.0",
    "slugify": "^1.6.6"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/better-sqlite3": "^7.6.8",
    "@types/bcryptjs": "^2.4.6",
    "@types/jsonwebtoken": "^9.0.5",
    "@types/multer": "^1.4.11",
    "@types/cors": "^2.8.17",
    "tsx": "^4.7.0",
    "typescript": "^5.3.0"
  }
}
```

---

## 6. API CONTRACT ĐẦY ĐỦ

### POST /api/auth/login
```
Request:  { email: string, password: string }
Response: { token: string, user: { id, email, name } }
Error:    401 { error: "Invalid credentials" }
```

### GET /api/projects
```
Query: ?page=1&limit=10&status=active
Response: {
  data: Project[],
  total: number,
  page: number
}
```

### GET /api/projects/:slug (PUBLIC)
```
Response: {
  id, slug, name, developer,
  priceFrom, priceTo, areaFrom, areaTo,
  description, address, features: string[],
  coverImage: string (URL),
  mindFile: string (URL),
  modelFile: string (URL),
  modelScale: number,
  images: { url, caption }[],
  contactPhone, contactEmail
}
```

### POST /api/projects (AUTH)
```
Request body: FormData hoặc JSON với tất cả Project fields
Response: { id, slug, ...created Project }
```

### POST /api/upload/image (AUTH)
```
Request: multipart/form-data, field "file"
Response: { url: "/files/images/uuid.jpg", filename: "uuid.jpg" }
```

### POST /api/upload/target (AUTH)
```
Request: multipart/form-data, field "file" (.mind binary)
Response: { url: "/files/targets/uuid.mind", filename: "uuid.mind" }
```

### POST /api/analytics/track (PUBLIC)
```
Request: { projectId: number, eventType: "qr_scan"|"ar_view"|"detail_open"|"contact_tap" }
Response: { ok: true }
```

---

## 7. UI/UX SPECIFICATIONS

### AR Viewer — Color Scheme
```css
--primary: #1a1a2e       /* Dark navy */
--accent: #e94560        /* Red accent */
--overlay-bg: rgba(0, 0, 0, 0.7)
--overlay-text: #ffffff
--card-bg: rgba(255, 255, 255, 0.95)
```

### AR Viewer — Component Sizes (Mobile)
- InfoOverlay card: max-width 90vw, padding 16px, border-radius 12px
- DetailPanel height: 70vh max, border-radius 20px top corners
- Loading screen: fullscreen, centered content
- Font sizes: title 20px, price 24px bold, area 16px

### Admin Dashboard — Sidebar
```
Logo: RealAR (với AR icon)
Navigation:
  - Dashboard (icon: LayoutDashboard)
  - Dự án (icon: Building2)
  - Thêm mới (icon: PlusCircle)
  - Analytics (icon: BarChart3)
  - Cài đặt (icon: Settings)
```

---

## 8. CHECKLIST TRƯỚC KHI BẮT ĐẦU CODE

- [x] Tech stack đã xác nhận: React + Three.js + MindAR + Express + SQLite
- [x] Deploy target: Vercel (frontend) + Railway/Render (backend)
- [x] Auth approach: JWT (đơn giản, đủ dùng)
- [x] File storage: Local filesystem (có thể migrate sang S3 sau)
- [x] Demo model: dùng free .glb từ Sketchfab, admin có thể upload model thật
- [x] HTTPS local dev: mkcert hoặc Vite https plugin
- [x] MindAR compiler: browser-side trong Admin tab 3 (không cần server compile)

---

## 9. RỦI RO VÀ CÁCH XRLY (Risk Mitigation)

| Rủi ro | Xác suất | Giải pháp |
|---|---|---|
| Image tracking kém chất lượng (ảnh catalog ít texture) | CAO | Hiện warning trong compiler khi feature points < threshold; gợi ý dùng ảnh nhiều chi tiết hơn |
| Camera permission bị deny trên iOS Safari | TRUNG BÌNH | Clear error UI với hướng dẫn từng bước bật permission trong Settings |
| .glb model quá lớn → load chậm | TRUNG BÌNH | Validate file size (max 50MB upload), recommend Draco compression, hiện loading % |
| MindAR không support một số Android cũ | THẤP | Fallback: hiện 3D model viewer thông thường (không AR) nếu WebGL/Camera không support |
| HTTPS required cho camera | CAO (local dev) | Document rõ ràng; dùng mkcert cho dev, Vercel tự động HTTPS cho prod |

---

*Plan version: 1.0 — ngày 27/02/2026*
