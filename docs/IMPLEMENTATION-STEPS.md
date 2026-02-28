# IMPLEMENTATION STEPS — RealAR Platform

**Version:** 1.0  
**Date:** 2026-02-28

---

## STEP 1 — Setup Monorepo

### Mục tiêu
Khởi tạo toàn bộ project structure, cài dependencies, 3 dev server chạy được.

### Task 1.1 — Root workspace
```
Làm gì:   Tạo thư mục gốc + package.json npm workspaces
Input:    Không có
Commands:
  mkdir real-estate-ar
  mkdir -p apps/ar-viewer apps/admin server docs

Output expect:
  package.json:
  {
    "name": "real-estate-ar",
    "workspaces": ["apps/*", "server"],
    "scripts": {
      "dev:server": "npm run dev --workspace=server",
      "dev:viewer": "npm run dev --workspace=ar-viewer",
      "dev:admin":  "npm run dev --workspace=admin",
      "dev": "concurrently ..."
    }
  }
```

### Task 1.2 — Scaffold ar-viewer
```
Làm gì:   Tạo Vite + React + TypeScript app
Command:  cd apps && npm create vite@latest ar-viewer -- --template react-ts

Output expect:
  apps/ar-viewer/
  ├── src/App.tsx
  ├── src/main.tsx
  ├── index.html
  ├── package.json  → name: "ar-viewer"
  └── vite.config.ts
```

### Task 1.3 — Scaffold admin
```
Làm gì:   Tạo Vite + React + TypeScript app
Command:  cd apps && npm create vite@latest admin -- --template react-ts

Output expect:
  apps/admin/ (cấu trúc tương tự ar-viewer)
  package.json → name: "admin", port: 5174
```

### Task 1.4 — Setup server skeleton
```
Làm gì:   Tạo Express server với health check endpoint
Files:
  server/src/index.ts
  server/package.json
  server/tsconfig.json

server/src/index.ts:
  import express from 'express'
  const app = express()
  app.get('/health', (req, res) => res.json({ ok: true }))
  app.listen(3000, () => console.log('Server :3000'))

Output expect:
  curl http://localhost:3000/health → { "ok": true }
```

### Task 1.5 — Cài dependencies ar-viewer
```
npm install (trong apps/ar-viewer/):
  mind-ar@1.2.5
  three@0.160.0
  @types/three
  react-router-dom@6
  axios

Output expect:
  TypeScript import MindARThree không lỗi
  TypeScript import THREE không lỗi
```

### Task 1.6 — Cài dependencies admin
```
npm install (trong apps/admin/):
  react-router-dom@6
  axios
  zustand@4
  @tanstack/react-query@5
  mind-ar@1.2.5
  qrcode + @types/qrcode
  react-dropzone
  recharts
  lucide-react
  tailwindcss autoprefixer postcss
  clsx tailwind-merge
  @radix-ui/react-dialog
  @radix-ui/react-tabs
  @radix-ui/react-toast

Sau đó:
  npx tailwindcss init -p
  npx shadcn-ui@latest init

Output expect:
  class="text-red-500" → màu đỏ hiện ra
  shadcn Button component import được
```

### Task 1.7 — Cài dependencies server
```
npm install (trong server/):
  express @types/express
  better-sqlite3 @types/better-sqlite3
  bcryptjs @types/bcryptjs
  jsonwebtoken @types/jsonwebtoken
  multer @types/multer
  cors @types/cors
  dotenv
  uuid @types/uuid
  slugify
  tsx

Output expect:
  npm run dev → "Server running on port 3000"
  Không có TypeScript errors
```

### Task 1.8 — .env files
```
Files tạo ra:
  .env.example (root)
  apps/ar-viewer/.env.example
  apps/admin/.env.example
  server/.env.example

Content server/.env.example:
  DATABASE_PATH=./database.db
  JWT_SECRET=change_this_to_random_64_char_string
  PORT=3000
  UPLOADS_DIR=./uploads

Content apps/ar-viewer/.env.example:
  VITE_API_URL=http://localhost:3000

Content apps/admin/.env.example:
  VITE_API_URL=http://localhost:3000
  VITE_AR_VIEWER_URL=http://localhost:5173
```

### ✅ Done khi
```
npm run dev:server  → localhost:3000/health = {"ok":true}
npm run dev:viewer  → localhost:5173 = React app
npm run dev:admin   → localhost:5174 = React app
npx tsc --noEmit    → 0 errors (tất cả packages)
```

---

## STEP 2 — Backend API + Database

### Mục tiêu
Express server với đầy đủ CRUD, auth, file upload, analytics.

### Task 2.1 — Database schema + seed
```
File:  server/src/db/schema.ts + server/src/db/index.ts

Input:  Không có (chạy khi server start lần đầu)

Logic:
  1. Mở/tạo database.db (path từ env)
  2. CREATE TABLE IF NOT EXISTS cho 4 tables
  3. Kiểm tra users table có data chưa
  4. Nếu chưa → INSERT admin:
     email: admin@realear.vn
     password_hash: bcrypt.hash("Admin@123", 10)
     name: "Admin"

Output expect:
  File database.db được tạo
  sqlite3 database.db "SELECT * FROM users" → 1 row
  sqlite3 database.db "SELECT * FROM projects" → 0 rows
  Không có migration errors khi restart server
```

### Task 2.2 — Auth middleware
```
File:  server/src/middleware/auth.ts

Input:  HTTP Request có header Authorization: Bearer <token>

Logic:
  1. Lấy token từ req.headers.authorization?.split(' ')[1]
  2. Nếu không có token → 401 { error: "Unauthorized" }
  3. jwt.verify(token, JWT_SECRET)
  4. SELECT * FROM users WHERE id = payload.id
  5. Nếu user không tồn tại → 401
  6. req.user = user; next()

Output expect:
  Request có token hợp lệ    → req.user populated, next() gọi
  Request không có token     → 401 { error: "Unauthorized" }
  Token hết hạn              → 401 { error: "Token expired" }
  Token bị giả mạo (sign sai) → 401 { error: "Invalid token" }
```

### Task 2.3 — Auth routes
```
File:  server/src/routes/auth.ts

POST /api/auth/login:
  Input:   { email: string, password: string }
  Logic:
    1. SELECT * FROM users WHERE email = ?
    2. Nếu không có user → 401
    3. bcrypt.compare(password, user.password_hash)
    4. Nếu sai password → 401
    5. jwt.sign({ id, email }, JWT_SECRET, { expiresIn: '24h' })
  Output:  200 { token: string, user: { id, email, name } }
  Error:   401 { error: "Email hoặc mật khẩu không đúng" }

GET /api/auth/me [AUTH]:
  Output:  200 { id, email, name }

Test:
  curl -X POST http://localhost:3000/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email":"admin@realear.vn","password":"Admin@123"}'
  → { "token": "eyJ..." }
```

### Task 2.4 — Projects routes
```
File:  server/src/routes/projects.ts

GET /api/projects [AUTH]:
  Input:   ?page=1&limit=10&status=active&search=sun
  Logic:
    SELECT * FROM projects
    WHERE (:status = 'all' OR status = :status)
    AND name LIKE '%' || :search || '%'
    ORDER BY created_at DESC
    LIMIT :limit OFFSET (:page-1)*:limit
  Output:  { data: Project[], total: number, page: number }

GET /api/projects/:slug [PUBLIC]:
  Logic:
    SELECT p.*, json_group_array(json_object('url',...)) as images
    FROM projects p
    LEFT JOIN project_images pi ON p.id = pi.project_id
    WHERE p.slug = :slug
    GROUP BY p.id
  Output:  Project object (full, bao gồm images[])
  Error:   404 { error: "Dự án không tồn tại" }

POST /api/projects [AUTH]:
  Input:   { name, developer, priceFrom, priceTo, areaFrom, areaTo,
             description, address, features[], contactPhone, contactEmail }
  Logic:
    1. Validate name (required)
    2. slug = slugify(name, { lower: true, locale: 'vi' })
    3. Kiểm tra slug unique, thêm -2, -3 nếu trùng
    4. INSERT INTO projects
  Output:  201 { id, slug, ...project }
  Error:   400 { error: "Tên dự án không được để trống" }

PUT /api/projects/:id [AUTH]:
  Input:   Partial<Project> — chỉ các field muốn update
  Logic:   UPDATE projects SET ...fields, updated_at = now WHERE id = ?
  Output:  200 Updated Project
  Error:   404

DELETE /api/projects/:id [AUTH]:
  Logic:
    1. SELECT project để lấy mind_file, model_file, cover_image
    2. Xóa files trên disk (không throw nếu file không tồn tại)
    3. DELETE FROM projects WHERE id = ? (CASCADE xóa images + analytics)
  Output:  200 { ok: true }
  Error:   404
```

### Task 2.5 — Upload routes
```
File:  server/src/routes/upload.ts
       server/src/middleware/upload.ts (Multer config)

Multer config:
  storage: diskStorage
  destination: tùy route
    /upload/image  → uploads/images/
    /upload/model  → uploads/models/
    /upload/target → uploads/targets/
  filename: () => uuid() + path.extname(originalname)

POST /api/upload/image [AUTH]:
  Input:   multipart/form-data field "file" (jpg/png/webp)
  Limit:   5MB
  Validate: mimetype phải là image/*
  Output:  { url: "/files/images/uuid.jpg", filename: "uuid.jpg" }
  Error:   413 { error: "File quá lớn, tối đa 5MB" }

POST /api/upload/model [AUTH]:
  Input:   .glb / .gltf
  Limit:   50MB
  Output:  { url: "/files/models/uuid.glb", filename: "uuid.glb" }

POST /api/upload/target [AUTH]:
  Input:   .mind binary
  Limit:   10MB
  Output:  { url: "/files/targets/uuid.mind", filename: "uuid.mind" }

GET /files/* [PUBLIC]:
  express.static(UPLOADS_DIR)
  Tạo thư mục uploads/ tự động nếu chưa có

Test:
  curl http://localhost:3000/files/images/test.jpg → file binary
  curl http://localhost:3000/files/nonexistent.jpg → 404
```

### Task 2.6 — Analytics routes
```
File:  server/src/routes/analytics.ts

POST /api/analytics/track [PUBLIC]:
  Input:   { projectId: number, eventType: string }
  Logic:   INSERT INTO analytics (project_id, event_type, user_agent, ip_address)
  Output:  { ok: true }
  Note:    Không throw lỗi — analytics không được làm crash app

GET /api/analytics/:projectId [AUTH]:
  Output:
    {
      totalViews:       number,  -- COUNT WHERE event_type = 'ar_view'
      totalQRScans:     number,  -- COUNT WHERE event_type = 'qr_scan'
      totalDetailOpens: number,
      last7Days: [
        { date: "2026-02-21", views: 12 },
        { date: "2026-02-22", views: 8 },
        ...
      ]
    }
```

### ✅ Done khi
```
curl test tất cả:
  POST /api/auth/login admin@realear.vn/Admin@123 → JWT token ✅
  GET  /api/projects (+ Bearer token)             → [] ✅
  POST /api/projects (+ token + body)             → {id:1, slug:"..."} ✅
  POST /api/upload/image (+ file)                 → {url, filename} ✅
  GET  /files/images/{filename}                   → file binary ✅
  POST /api/analytics/track                       → {ok:true} ✅
  DELETE /api/projects/1 (no token)               → 401 ✅
  GET  /api/projects/nonexistent                  → 404 ✅
```

---

## STEP 3 — AR Viewer Core

### Mục tiêu
WebAR experience hoàn chỉnh: camera feed → image tracking → 3D model → UI.

### Task 3.1 — Router + TypeScript types
```
File:  apps/ar-viewer/src/types/project.ts
       apps/ar-viewer/src/App.tsx

types/project.ts:
  interface Project {
    id: number
    slug: string
    name: string
    developer?: string
    priceFrom?: number
    priceTo?: number
    areaFrom?: number
    areaTo?: number
    description?: string
    address?: string
    features: string[]
    coverImage?: string
    mindFile?: string
    modelFile?: string
    modelScale: number
    contactPhone?: string
    contactEmail?: string
    images: { url: string; caption?: string }[]
  }

App.tsx routes:
  /            → ScanQRPage
  /view/:slug  → ARPage
  *            → NotFoundPage

Output expect:
  Navigate đến /view/test → ARPage render (không blank)
  Navigate đến /          → ScanQRPage render
  Navigate đến /random    → NotFoundPage render
```

### Task 3.2 — useProject hook
```
File:  apps/ar-viewer/src/hooks/useProject.ts

Input:  slug: string

Logic:
  1. useState: project, loading, error
  2. useEffect: fetch ${VITE_API_URL}/api/projects/${slug}
  3. Xử lý:
     - 404 → error = "Dự án không tồn tại hoặc đã bị xóa"
     - network fail → error = "Không thể kết nối. Kiểm tra mạng"
     - success → project = data

Output:
  { project: Project | null, loading: boolean, error: string | null }

Test:
  slug = "existing" → project populated, loading=false, error=null
  slug = "fake"     → project=null, error="Dự án không tồn tại..."
  server down       → project=null, error="Không thể kết nối..."
```

### Task 3.3 — useMindAR hook (CORE)
```
File:  apps/ar-viewer/src/hooks/useMindAR.ts

Input:
  containerRef: RefObject<HTMLDivElement>
  mindFileUrl:  string  (full URL đến .mind file)
  modelUrl:     string  (full URL đến .glb file)
  modelScale:   number

State machine bên trong:
  type ARState = 'idle' | 'requesting_camera' | 'loading' | 'started' | 'error'

Logic chi tiết:
  1. setArState('requesting_camera')
  2. new MindARThree({
       container: containerRef.current,
       imageTargetSrc: mindFileUrl,
       maxTrack: 1,
       filterMinCF: 0.001,
       filterBeta: 0.001,
       warmupTolerance: 5,
       missTolerance: 5
     })
  3. const { renderer, scene, camera } = mindarThree
  4. Setup lights:
       scene.add(new THREE.AmbientLight(0xffffff, 0.8))
       const dirLight = new THREE.DirectionalLight(0xffffff, 1.0)
       dirLight.position.set(0, 5, 5)
       scene.add(dirLight)
  5. setArState('loading')
  6. GLTFLoader + DRACOLoader:
       loader.load(modelUrl,
         (gltf) => { ... onSuccess },
         (xhr) => setLoadProgress(xhr.loaded/xhr.total*100),
         (err) => setArState('error')
       )
  7. onSuccess:
       gltf.scene.scale.set(modelScale, modelScale, modelScale)
       const anchor = mindarThree.addAnchor(0)
       anchor.group.add(gltf.scene)
       anchor.group.visible = false
       anchor.onTargetFound = () => {
         anchor.group.visible = true
         setIsTracking(true)
       }
       anchor.onTargetLost = () => {
         anchor.group.visible = false
         setIsTracking(false)
       }
  8. await mindarThree.start()
       → catch NotAllowedError → setError('camera_denied')
  9. setArState('started')
  10. renderer.setAnimationLoop(() => {
        gltf.scene.rotation.y += 0.005
        renderer.render(scene, camera)
      })

Cleanup (return từ useEffect):
  mindarThree.stop()
  renderer.setAnimationLoop(null)
  renderer.dispose()
  gltf.scene.traverse((obj: any) => {
    obj.geometry?.dispose()
    if (obj.material) {
      Array.isArray(obj.material)
        ? obj.material.forEach((m: any) => m.dispose())
        : obj.material.dispose()
    }
  })

Output:
  {
    arState: ARState,
    isTracking: boolean,
    loadProgress: number,  // 0-100
    error: string | null
  }
```

### Task 3.4 — LoadingScreen component
```
File:  apps/ar-viewer/src/components/LoadingScreen.tsx

Input:  { arState: ARState, loadProgress: number }

UI states:
  requesting_camera:
    Spinner + "Đang khởi động camera..."

  loading:
    Progress bar (0-100%)
    Text: "Đang tải dữ liệu AR... {loadProgress}%"

  started:
    Icon catalog + mũi tên camera
    Text: "Hướng camera vào catalog"
    Pulsing scan frame animation
    → component này tự ẩn sau 2 giây

Hiển thị:
  Khi arState !== 'started'  → fullscreen overlay (z-50)
  Khi arState === 'started'  → fade out + unmount sau 2s

Output expect:
  Mở page → thấy loading screen ngay
  Sau khi AR ready → loading screen fade out
  Không flicker giữa các states
```

### Task 3.5 — InfoOverlay component
```
File:  apps/ar-viewer/src/components/InfoOverlay.tsx

Input:  { project: Project, isTracking: boolean, onTap: () => void }

UI:
  Position: fixed, bottom: 120px, centered
  Card:
    backdrop-blur-md bg-white/90 rounded-2xl shadow-xl px-4 py-3
    ┌─────────────────────────────┐
    │ [Tên dự án]                 │  font-bold text-lg
    │ Từ X tỷ — Y tỷ              │  text-blue-600 font-bold text-xl
    │ X m² — Y m²  [Chủ đầu tư]  │  text-gray-500 text-sm
    │         Xem chi tiết →      │  text-right text-blue-500
    └─────────────────────────────┘

Animation:
  isTracking=true  → opacity-100 translateY(0)    transition 300ms
  isTracking=false → opacity-0   translateY(20px)  transition 300ms
  pointer-events: none khi hidden

onClick card → onTap()

Output expect:
  Không tracking → card không hiện (opacity 0)
  Bắt đầu tracking → card fade in từ dưới lên
  Click card → onTap() được gọi
```

### Task 3.6 — DetailPanel component
```
File:  apps/ar-viewer/src/components/DetailPanel.tsx

Input:  { project: Project, isOpen: boolean, onClose: () => void }

UI:
  Backdrop: fixed inset-0 bg-black/50 (click → onClose)
  Panel: fixed bottom-0 left-0 right-0
         max-h-[70vh] bg-white rounded-t-3xl overflow-y-auto
         transition: translateY(100%) → translateY(0)

  Content scroll:
    Handle bar (gray pill, centered top)
    ── Header ──
    [cover image nếu có]
    Tên dự án (text-2xl font-bold)
    Chủ đầu tư (text-gray-500)
    ── Stats row ──
    [Giá từ X–Y tỷ] | [X–Y m²]
    ── Địa chỉ ──
    📍 address text
    ── Tiện ích ──
    Badges: [Hồ bơi] [Gym] [Trường học] ...
    ── Mô tả ──
    description text (3 dòng, expand khi tap "Xem thêm")
    ── Gallery ──
    Horizontal scroll images (nếu có)
    ── Actions ──
    [📞 Gọi ngay]    → tel:contactPhone  (primary button)
    [✉️ Email]       → mailto:contactEmail (secondary button)

Swipe to close:
  onTouchStart: startY = e.touches[0].clientY
  onTouchMove:  deltaY = currentY - startY
                Nếu deltaY > 80 → onClose()

Output expect:
  isOpen=false → panel ở dưới màn hình (translateY 100%)
  isOpen=true  → panel slide up trong 350ms
  Swipe down   → panel đóng
  Tap backdrop → panel đóng
  Gọi ngay     → dialpad mở
```

### Task 3.7 — ARPage
```
File:  apps/ar-viewer/src/pages/ARPage.tsx

Route: /view/:slug

Logic:
  1. const { slug } = useParams()
  2. const { project, loading, error } = useProject(slug)
  3. Nếu loading → <Spinner />
  4. Nếu error   → <ErrorPage message={error} />
  5. Nếu project → render AR scene
  6. containerRef = useRef<HTMLDivElement>()
  7. useMindAR(containerRef, mindFileUrl, modelFileUrl, project.modelScale)
  8. [isDetailOpen, setIsDetailOpen] = useState(false)
  9. Analytics:
     useEffect: khi arState='started' → track('ar_view', project.id)
     Khi setIsDetailOpen(true) → track('detail_open', project.id)

Render:
  <div ref={containerRef} className="w-screen h-screen overflow-hidden">
    <LoadingScreen arState={arState} loadProgress={loadProgress} />
    <InfoOverlay
      project={project}
      isTracking={isTracking}
      onTap={() => setIsDetailOpen(true)}
    />
    <DetailPanel
      project={project}
      isOpen={isDetailOpen}
      onClose={() => setIsDetailOpen(false)}
    />
    {error && <ErrorOverlay error={error} />}
  </div>

Output expect:
  /view/real-project-slug:
    → Fetch project data
    → Loading screen hiện
    → Camera bật
    → Hướng webcam vào ảnh test → model 3D hiện
    → Overlay hiện tên/giá/diện tích
    → Click → detail panel slide up
    → Gọi ngay → dialpad
```

### Task 3.8 — ScanQRPage
```
File:  apps/ar-viewer/src/pages/ScanQRPage.tsx

Route: /

UI:
  Centered layout:
  Logo RealAR
  "Trải nghiệm bất động sản trong AR"
  [📷 Quét mã QR] button → bật html5-qrcode scanner
  Scanner overlay khi active (viewfinder)
  ── hoặc ──
  Input: "Nhập mã dự án" + [→ Đi] button
  → navigate('/view/' + slug)

QR scan logic:
  import { Html5QrcodeScanner } from 'html5-qrcode'
  Khi quét được text chứa '/view/' → extract slug → navigate

Output expect:
  Click "Quét QR" → camera mở
  Quét QR demo   → redirect /view/slug
  Nhập slug tay  → redirect /view/slug
```

### ✅ Done khi
```
localhost:5173/view/test-project:
  ✅ LoadingScreen hiện với progress
  ✅ Camera feed fullscreen (webcam desktop)
  ✅ Đưa ảnh test card vào webcam → model .glb xuất hiện
  ✅ Model rotate tự động
  ✅ InfoOverlay fade-in khi tracking
  ✅ Click overlay → DetailPanel slide up
  ✅ Swipe down panel → đóng
  ✅ Button Gọi ngay → dialpad (mobile) hoặc tel: link (desktop)
  ✅ Không có console errors
  ✅ Memory cleanup khi navigate đi trang khác
```

---

## STEP 4 — Admin Dashboard

### Mục tiêu
Admin panel đầy đủ: login, CRUD dự án, upload files, compile .mind, QR.

### Task 4.1 — Auth flow
```
Files:
  apps/admin/src/store/authStore.ts
  apps/admin/src/pages/LoginPage.tsx
  apps/admin/src/components/layout/ProtectedRoute.tsx
  apps/admin/src/services/api.ts

api.ts (Axios instance):
  baseURL: VITE_API_URL
  interceptors.request: thêm Authorization: Bearer token
  interceptors.response: nếu 401 → authStore.logout() → redirect /login

authStore (Zustand):
  state: { token: string|null, user: User|null, isAuthenticated: boolean }
  login(token, user): set state + localStorage.setItem
  logout():          clear state + localStorage.removeItem + navigate('/login')
  hydrate():         load từ localStorage khi app mount

LoginPage:
  Form: email + password + submit button
  onSubmit:
    1. POST /api/auth/login
    2. authStore.login(token, user)
    3. navigate('/dashboard')
  Error state: hiện "Email hoặc mật khẩu không đúng"
  Loading state: button disabled + spinner

ProtectedRoute:
  if (!isAuthenticated) return <Navigate to="/login" />
  return <Outlet />

Output expect:
  /dashboard khi chưa login → redirect /login
  Login sai → hiện lỗi, không redirect
  Login đúng → /dashboard
  Refresh page → vẫn logged in (localStorage)
  Token expire → auto logout + redirect /login
```

### Task 4.2 — Layout + Sidebar
```
Files:
  apps/admin/src/components/layout/Layout.tsx
  apps/admin/src/components/layout/Sidebar.tsx
  apps/admin/src/components/layout/Header.tsx

Layout:
  Desktop: sidebar 240px fixed left + content area right
  Mobile:  sidebar collapse + hamburger menu

Sidebar items:
  Logo "RealAR" (top)
  ──────────────────
  LayoutDashboard  Dashboard    /dashboard
  Building2        Dự án        /projects
  PlusCircle       Thêm mới     /projects/new
  BarChart3        Analytics    /analytics
  ──────────────────
  LogOut           Đăng xuất    → authStore.logout()

Active route: highlight background + text color change

Header:
  Title (tên page hiện tại)
  Avatar circle (initials của admin name)

Output expect:
  Active route highlight đúng
  Đăng xuất → clear token → /login
  Mobile: menu có thể đóng/mở
```

### Task 4.3 — Dashboard Page
```
File:  apps/admin/src/pages/DashboardPage.tsx

API calls (parallel):
  GET /api/projects?limit=3&page=1
  GET /api/analytics/summary (hoặc tổng hợp từ nhiều calls)

UI:
  Row 1 — 4 StatsCards:
    [🏢 Tổng dự án: N]
    [✅ Đang active: N]
    [👁 AR views hôm nay: N]
    [📱 QR scans tuần này: N]

  Row 2 — Charts:
    Left: LineChart (Recharts) — lượt xem 7 ngày
    Right: BarChart — top 5 dự án

  Row 3 — Dự án mới nhất:
    3 ProjectCards với [Xem AR] [Sửa] buttons

Output expect:
  Số liệu load từ API (không hardcode)
  Charts render đúng với data
  Responsive: 1 column trên mobile
```

### Task 4.4 — Project List Page
```
File:  apps/admin/src/pages/ProjectsPage.tsx

State: search (debounce 300ms), status filter, page

API: GET /api/projects?search=&status=&page=&limit=10

UI:
  Header: "Dự án" + [+ Thêm dự án] button
  Search input (debounced)
  Filter tabs: Tất cả | Active | Draft | Inactive

  Table columns:
    Tên dự án | Giá từ | AR Setup | Trạng thái | Ngày tạo | Actions

  AR Setup column:
    ✅ Đã cấu hình (có cả mind_file + model_file)
    ⚠️ Chưa hoàn chỉnh (thiếu 1 trong 2)
    ❌ Chưa cấu hình (không có cả 2)

  Actions per row:
    [👁 Xem AR]   → window.open(AR_VIEWER_URL/view/slug)
    [✏️ Sửa]     → navigate(/projects/:id)
    [🗑 Xóa]     → confirm dialog → DELETE /api/projects/:id → refresh

  Pagination: prev/next + trang hiện tại

Output expect:
  Search "sun" → debounce 300ms → filter results
  Click Xóa → confirm dialog "Xóa dự án này?" → xóa → toast success
  Pagination hoạt động
```

### Task 4.5 — Project Form (Tab 1: Thông tin)
```
File:  apps/admin/src/pages/ProjectDetailPage.tsx

Dùng cho:
  /projects/new   → POST /api/projects
  /projects/:id   → GET để load, PUT để save

Fields + validation:
  name*           required, min 3 chars
  developer       optional text
  priceFrom       number, format "1.500.000.000 ₫", min 0
  priceTo         number, phải > priceFrom nếu có
  areaFrom        number (m²), min 0
  areaTo          number, phải > areaFrom nếu có
  description     textarea, max 2000 chars, char counter
  address         text
  features        TagInput component:
                    Enter → thêm tag
                    X → xóa tag
                    Gợi ý preset: [Hồ bơi] [Gym] [Trường học]...
  contactPhone    validate: 0[35789][0-9]{8} (VN mobile)
  contactEmail    validate email format
  status          Select: Draft | Active | Inactive

Submit button:
  Tạo mới → "Tạo dự án"
  Sửa     → "Lưu thay đổi"
  Loading state khi đang gọi API
  Redirect về /projects sau khi thành công

Output expect:
  Submit form thiếu name → validation error inline
  priceFrom > priceTo → error "Giá đến phải lớn hơn giá từ"
  Submit thành công → toast "Đã lưu" → redirect /projects
```

### Task 4.6 — Project Form (Tab 2: Media)
```
Files:
  apps/admin/src/components/upload/ImageUploader.tsx
  apps/admin/src/components/upload/ModelUploader.tsx

ImageUploader (ảnh bìa):
  UI:
    Dropzone area (react-dropzone):
      accept: {image/*: ['.jpg','.jpeg','.png','.webp']}
      maxSize: 5 * 1024 * 1024 (5MB)
    Khi có ảnh: hiện preview + [X xóa]
    Khi drop:
      1. FileReader → preview ngay (không cần upload)
      2. POST /api/upload/image
      3. Nhận URL → update form.coverImage
  Error:
    > 5MB → "Ảnh quá lớn, tối đa 5MB"
    Sai format → "Chỉ chấp nhận JPG, PNG, WebP"

ModelUploader (.glb):
  UI:
    Dropzone accept: {'.glb': [], '.gltf': []}
    maxSize: 50MB
    File info sau upload: "model.glb (24.5 MB)" + [↻ Thay thế]
    Progress bar khi đang upload
  Logic:
    Dùng axios.post với onUploadProgress callback
    → setProgress(event.loaded / event.total * 100)
  Error:
    > 50MB → "File quá lớn, tối đa 50MB"

Output expect:
  Drop ảnh → preview hiện ngay
  Drop .glb 20MB → progress bar → "model.glb (20 MB)" ✅
  File > 50MB → error message rõ ràng
  Xóa ảnh → preview mất, form.coverImage = null
```

### Task 4.7 — Project Form (Tab 3: AR Setup / MindCompiler)
```
File:  apps/admin/src/components/upload/MindCompiler.tsx

State machine:
  'idle' → 'image_loaded' → 'compiling' → 'compiled' → 'uploading' → 'success'
                                        ↘ 'error'                  ↘ 'error'

UI theo state:
  idle:
    Dropzone "Kéo thả ảnh catalog vào đây"
    Accept: image/*, maxSize: 5MB

  image_loaded:
    Preview ảnh
    [▶ Bắt đầu Compile] button

  compiling:
    Progress bar 0→100%
    "Đang phân tích ảnh... {progress}%"
    Không thể cancel (WebWorker đang chạy)

  compiled:
    Canvas hiện ảnh + chấm đỏ feature points
    Quality badge:
      < 200 points → ⚠️ vàng "Ảnh có ít điểm ({n}), tracking có thể kém"
      ≥ 200 points → ✅ xanh "Ảnh tốt ({n} điểm đặc trưng)"
      ≥ 500 points → ✅✅ "Ảnh xuất sắc ({n} điểm)"
    [💾 Lưu target] button

  uploading:
    Spinner + "Đang lưu..."

  success:
    ✅ "Đã lưu: {filename}"
    [🔄 Compile lại] button (để thử ảnh khác)

Logic compile:
  1. const { Compiler } = await import('mind-ar/dist/mindar-image.prod.js')
  2. const compiler = new Compiler()
  3. await compiler.compileImageTargets([imgElement], (p) => setProgress(p*100))
  4. const buffer = await compiler.exportData()
  5. Lấy feature points từ compiler.getDataList()
  6. Vẽ points lên canvas
  7. User click Lưu → POST /api/upload/target
  8. PUT /api/projects/:id { mindFile: filename }

Ngoài ra trong Tab 3:
  Model Scale slider:
    Range: 0.001 → 2.0 (step 0.001)
    Default: 0.1
    Live preview value: "Scale: 0.05"
    → update form.modelScale

  AR Preview link:
    "Xem AR: {VITE_AR_VIEWER_URL}/view/{slug}"
    [Copy] [Mở]

Output expect:
  Upload ảnh catalog thật → progress bar compile
  Sau compile → canvas hiện chấm đỏ phủ ảnh
  Quality badge hiện đúng
  Click Lưu → .mind được upload → success
  Scale slider → value thay đổi realtime
```

### Task 4.8 — Project Form (Tab 4: QR Code)
```
File:  apps/admin/src/components/projects/QRCodeDisplay.tsx

Input:  { slug: string }

Logic:
  arUrl = `${VITE_AR_VIEWER_URL}/view/${slug}`
  useEffect: QRCode.toCanvas(canvasRef.current, arUrl, {
    width: 300, margin: 2,
    color: { dark: '#1a1a2e', light: '#ffffff' }
  })

UI:
  Canvas 300x300 (QR code)
  URL text (monospace, copyable)
  [📋 Copy link] → navigator.clipboard.writeText → toast "Đã copy!"
  [⬇ Download PNG] →
    canvas.toBlob((blob) => {
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url; a.download = `qr-${slug}.png`; a.click()
    })
  ──────────────────────────────
  💡 Hướng dẫn:
  "In QR code này lên trang bìa catalog
   để sale staff quét khi gặp khách hàng"

Output expect:
  QR hiện ngay khi mở tab
  Điện thoại quét QR → mở đúng AR Viewer URL
  Download → file qr-{slug}.png lưu về máy
  Copy link → paste được URL đúng
```

### ✅ Done khi
```
  ✅ Login admin@realear.vn / Admin@123 thành công
  ✅ Tạo dự án "Vinhomes Central Park" → slug auto-generated
  ✅ Tab Media: upload ảnh bìa → preview, upload .glb → progress
  ✅ Tab AR Setup: upload catalog ảnh → compile → feature points → lưu .mind
  ✅ Tab QR: QR hiện, download PNG, copy link
  ✅ Quét QR bằng điện thoại → mở ar-viewer đúng
  ✅ AR viewer load đúng .mind + .glb của dự án
```

---

## STEP 5 — Integration + Polish + Deploy

### Task 5.1 — HTTPS local dev
```
Tool: mkcert

Commands:
  brew install mkcert (macOS) / choco install mkcert (Windows)
  mkcert -install
  mkcert localhost 127.0.0.1 192.168.x.x

Vite config (ar-viewer):
  import fs from 'fs'
  export default defineConfig({
    server: {
      https: {
        key: fs.readFileSync('localhost+2-key.pem'),
        cert: fs.readFileSync('localhost+2.pem'),
      },
      host: '0.0.0.0',
      port: 5173
    }
  })

Output expect:
  https://localhost:5173 → không có SSL warning
  https://192.168.x.x:5173 → trên điện thoại cùng WiFi
  Camera permission popup hiện đúng (không bị block)
```

### Task 5.2 — Mobile optimization
```
Touch interactions:
  DetailPanel swipe to close:
    onTouchStart: record startY
    onTouchMove:  if deltaY > 80px → onClose()
    CSS: touch-action: pan-y (scroll vertical, không pan horizontal)

Safe area (notch / home indicator):
  CSS:
    padding-top: env(safe-area-inset-top)
    padding-bottom: env(safe-area-inset-bottom)
  Vite config: add to index.html:
    <meta name="viewport" content="width=device-width, initial-scale=1,
      viewport-fit=cover">

Prevent iOS zoom on input focus:
  All inputs/selects: font-size: 16px minimum

Landscape mode:
  @media (orientation: landscape) {
    .info-overlay { bottom: 60px; }
    .detail-panel { max-height: 60vh; }
  }

Prevent body scroll khi DetailPanel mở:
  isOpen ? document.body.style.overflow = 'hidden'
         : document.body.style.overflow = ''
```

### Task 5.3 — Error handling
```
AR Viewer error states:

camera_denied:
  Title: "Cần quyền truy cập camera"
  iOS:    "Vào Settings → Safari → Camera → Allow"
  Android:"Tap biểu tượng 🔒 trên URL bar → Camera → Cho phép"
  Button: [🔄 Thử lại] → reload page

mind_file_not_found:
  "Dự án chưa được cấu hình AR"
  "Vui lòng liên hệ quản trị viên"

network_error:
  "Không thể kết nối. Kiểm tra kết nối mạng"
  Button: [🔄 Thử lại]

model_load_failed:
  Không hiện error to, chỉ log console
  Hiện wireframe cube thay thế model
  Tracking vẫn hoạt động bình thường

Admin toast (shadcn Toaster):
  Upload thành công → green toast 3s "Đã lưu thành công"
  Upload thất bại   → red toast persistent "Lỗi: {message}"
  Compile xong      → green "Compile xong! {n} điểm đặc trưng"
  Xóa dự án        → green 3s "Đã xóa dự án"
```

### Task 5.4 — Deploy config
```
apps/ar-viewer/vercel.json:
  {
    "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }],
    "headers": [{
      "source": "/(.*)",
      "headers": [
        { "key": "Cross-Origin-Opener-Policy", "value": "same-origin" },
        { "key": "Cross-Origin-Embedder-Policy", "value": "require-corp" }
      ]
    }]
  }

apps/admin/vercel.json:
  { "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }] }

server/Dockerfile:
  FROM node:20-alpine
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci --production
  COPY dist/ ./dist/
  EXPOSE 3000
  VOLUME ["/app/uploads", "/app/database.db"]
  CMD ["node", "dist/index.js"]

server/.env.production:
  DATABASE_PATH=/app/database.db
  UPLOADS_DIR=/app/uploads
  JWT_SECRET=<64-char random string>
  PORT=3000
  CORS_ORIGIN=https://ar.yourdomain.com,https://admin.yourdomain.com
```

### ✅ Done khi (End-to-End trên điện thoại thực)
```
1. Mở https://admin.vercel.app → Login
2. Tạo dự án "Test Project"
3. Upload ảnh catalog → compile .mind (thấy feature points canvas)
4. Upload model.glb
5. Tab QR → Download QR PNG

6. Dùng điện thoại quét QR
7. AR Viewer mở trên Safari iOS / Chrome Android
8. LoadingScreen hiện → camera bật
9. Hướng camera vào ảnh catalog (in ra hoặc hiện trên màn hình khác)
10. Model 3D tòa nhà xuất hiện trên ảnh ✅
11. InfoOverlay: "Test Project | Từ X tỷ | Y m²" ✅
12. Tap overlay → DetailPanel slide up ✅
13. Swipe down → đóng ✅
14. Gọi ngay → dialpad mở ✅
15. Analytics: admin dashboard hiện +1 ar_view ✅
```
