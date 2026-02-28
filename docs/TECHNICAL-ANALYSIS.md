# TECHNICAL ANALYSIS — MindAR & WebAR

**Version:** 1.0  
**Date:** 2026-02-28

---

## 1. MindAR hoạt động như thế nào (bên dưới)

```
Camera frame (video stream 30fps)
        ↓
[WebGL / TensorFlow.js GPU backend]
        ↓
Feature Detection (ORB-like algorithm)
  • Harris Corner Detector — tìm góc, cạnh có contrast cao
  • BRIEF Descriptor — mô tả vùng 32x32px xung quanh mỗi keypoint
  • Output: ~200–1000 keypoints/frame
        ↓
Feature Matching
  • So sánh descriptors của frame hiện tại
    với descriptors đã lưu trong .mind file
  • Dùng Hamming distance (XOR bit-by-bit)
  • Brute force match hoặc FLANN index
        ↓
Pose Estimation
  • RANSAC + Homography matrix (3x3)
  • Tính ma trận biến đổi 2D → 3D
  • Xác định vị trí (x,y,z) + góc xoay (rx,ry,rz) trong camera space
        ↓
One Euro Filter (smoothing)
  • Lọc noise từ rung tay / rung camera
  • filterMinCF: cutoff frequency thấp = smooth hơn, lag hơn
  • filterBeta: tốc độ phản hồi khi di chuyển nhanh
        ↓
Anchor Transform Update
  • Cập nhật Three.js Object3D.matrix
  • Three.js render model tại đúng vị trí/góc
```

---

## 2. File .mind — Cấu trúc binary

```
.mind binary layout:
┌─────────────────────────────────┐
│  Header                          │
│  • version                       │
│  • num_targets                   │
├─────────────────────────────────┤
│  Target[0]                       │
│  ├── image_width, image_height   │
│  ├── Pyramid levels (N levels):  │
│  │   Level 0 (scale 1.0x):       │
│  │   ├── keypoints[] {x, y}      │
│  │   └── descriptors[] (binary)  │
│  │   Level 1 (scale 0.5x): ...   │
│  │   Level 2 (scale 0.25x): ...  │
│  └── matching metadata           │
├─────────────────────────────────┤
│  Target[1]: ...                  │
│  Target[N]: ...                  │
└─────────────────────────────────┘
```

**Tại sao cần pyramid levels:**
- Detect được target ở nhiều khoảng cách khác nhau
- Xa: dùng level nhỏ (0.25x) — ảnh nhỏ hơn, ít keypoints hơn nhưng đủ
- Gần: dùng level lớn (1.0x) — nhiều keypoints, tracking chính xác hơn

---

## 3. Compile .mind — Chi tiết kỹ thuật

```typescript
// Admin browser — MindCompiler.tsx
import { Compiler } from 'mind-ar/dist/mindar-image.prod.js';

// 1. Load ảnh catalog
const img = new Image();
img.src = URL.createObjectURL(file);
await img.decode();

// 2. Khởi tạo compiler
const compiler = new Compiler();

// 3. Compile (chạy trong WebWorker, không block UI)
await compiler.compileImageTargets([img], (progress: number) => {
  setProgress(Math.round(progress * 100)); // 0 → 100%
});
// Thời gian: ~5–30 giây tùy độ phức tạp ảnh + thiết bị

// 4. Export binary
const buffer = await compiler.exportData();
// buffer: ArrayBuffer (thường 500KB – 3MB)

// 5. Visualize feature points (quality check)
const dataList = compiler.getDataList();
const keyframes = dataList[0]?.matchingData?.keyframes;
const points = keyframes?.[0]?.points || []; // [{x, y}]

// Vẽ lên canvas
const canvas = document.getElementById('preview') as HTMLCanvasElement;
canvas.width = img.width;
canvas.height = img.height;
const ctx = canvas.getContext('2d')!;
ctx.drawImage(img, 0, 0);
points.forEach(p => {
  ctx.beginPath();
  ctx.arc(p.x, p.y, 3, 0, 2 * Math.PI);
  ctx.fillStyle = '#ef4444'; // red
  ctx.fill();
});

// 6. Quality feedback
const pointCount = points.length;
// < 200  → ⚠️ ảnh kém
// ≥ 200  → ✅ ảnh ổn
// ≥ 500  → ✅✅ ảnh rất tốt

// 7. Upload .mind lên server
const blob = new Blob([buffer], { type: 'application/octet-stream' });
const formData = new FormData();
formData.append('file', blob, 'target.mind');
const res = await axios.post('/api/upload/target', formData);
// → { filename: 'uuid.mind' }
```

---

## 4. AR Viewer Runtime — State Machine đầy đủ

```
                    ┌─────────┐
                    │  IDLE   │
                    └────┬────┘
                         │ component mount + project loaded
                         ▼
               ┌──────────────────┐
               │ REQUESTING_CAMERA│
               └────────┬─────────┘
              denied ◄──┤──► granted
                         │
                         ▼
               ┌──────────────────┐
               │  LOADING_ASSETS  │
               │  .mind + .glb    │◄─── progress 0→100%
               └────────┬─────────┘
              404/fail ◄─┤──► loaded
                         │
                         ▼
               ┌──────────────────┐
               │     STARTED      │ ← camera feed hiện
               │  (scanning...)   │ ← MindAR phân tích frame
               └────────┬─────────┘
                         │ targetFound
                         ▼
               ┌──────────────────┐
               │    TRACKING      │ ← model 3D hiện
               │   (model on)     │ ← InfoOverlay hiện
               └────────┬─────────┘
                         │ targetLost
                         ▼
               ┌──────────────────┐
               │      LOST        │ ← model ẩn
               │  (rescanning...) │ ← "Hướng camera vào catalog"
               └────────┬─────────┘
                         │ targetFound lại
                         └──────► TRACKING (không reload model)
```

**Model KHÔNG bị unload khi targetLost:**
```typescript
anchor.onTargetFound = () => {
  anchor.group.visible = true;  // chỉ toggle visible
  setIsTracking(true);
};
anchor.onTargetLost = () => {
  anchor.group.visible = false; // model vẫn trong memory
  setIsTracking(false);
};
// .glb load 1 lần duy nhất khi LOADING_ASSETS
// → performance tốt, không flicker khi re-detect
```

---

## 5. Camera Jitter — Phân tích và giải pháp

### Nguyên nhân jitter

```
Frame N:   pose = { x: 1.00, y: 2.00, rz: 0.10 }
Frame N+1: pose = { x: 1.02, y: 1.98, rz: 0.11 }  ← tay run nhẹ
Frame N+2: pose = { x: 0.99, y: 2.01, rz: 0.09 }

Không filter → model "nhảy" 2-3px mỗi frame → nhìn rung
```

### One Euro Filter

```
Nguyên lý:
  • Khi di chuyển CHẬM (tay run nhỏ): dùng cutoff thấp → smooth mạnh
  • Khi di chuyển NHANH (chủ động): tăng cutoff → bám sát nhanh hơn

Parameters:
  filterMinCF (minimum cutoff frequency):
    0.0001 → cực smooth, lag nhiều
    0.001  → smooth tốt, lag vừa     ← recommended
    0.01   → ít smooth, lag ít
    0.1    → gần như không filter

  filterBeta (speed coefficient):
    0      → không thích nghi theo tốc độ
    0.001  → thích nghi chậm           ← recommended thường
    1.0    → thích nghi vừa
    10     → thích nghi nhanh (tốt khi hay di chuyển camera)
```

### Tuning theo use case

```
Sale demo tại bàn (catalog đặt cố định, tay run nhẹ):
  filterMinCF: 0.001, filterBeta: 0.001

Sale cầm catalog di chuyển (trình diễn):
  filterMinCF: 0.001, filterBeta: 1.0

Môi trường rung (xe, rung động):
  filterMinCF: 0.0001, filterBeta: 0.001
```

---

## 6. Partial Detection (quét được 1 phần ảnh)

### Cơ chế hoạt động

```
Catalog 100% visible:   detect 800 keypoints match → TRACKING ✅
Catalog 60% visible:    detect 300 keypoints match → TRACKING ✅
Catalog 30% visible:    detect 80 keypoints match  → MAYBE ⚠️
Catalog bị che > 70%:   detect < 10 match          → LOST ❌
```

**MindAR detect được khi:**
- ≥ ~15 keypoints match (threshold nội bộ)
- Keypoints phân bố đủ đều (không tập trung 1 góc)
- Góc nghiêng ≤ ~60° so với mặt phẳng ảnh

### Điều kiện fail

| Điều kiện | Lý do |
|---|---|
| Che > 70% ảnh | Không đủ keypoints |
| Góc nghiêng > 60° | Perspective distortion quá lớn, homography fail |
| Ánh sáng yếu (< 50 lux) | Contrast thấp, không detect được corners |
| Glare / phản chiếu | Che phủ keypoints |
| Ảnh bị nhòe / mờ | Descriptors không match |
| Catalog bị nhăn / cong | Homography giả định phẳng, bị lệch |

---

## 7. Ảnh giống nhau — False Positive Detection

### Vấn đề trong BDS

```
Catalog 20 trang, mỗi trang có cùng layout template:
  • Header logo giống nhau
  • Color scheme giống nhau
  • Chỉ khác phần render tòa nhà

→ MindAR có thể nhầm trang 3 với trang 5 nếu render giống nhau
```

### Cơ chế MindAR xử lý nhiều targets

```
.mind file chứa N targets

Mỗi frame:
  1. Extract keypoints từ camera frame (~500 points)
  2. Match với TẤT CẢ N targets song song (GPU)
  3. Chọn target có score cao nhất:
     score = số keypoints match / tổng keypoints target
  4. Nếu score > threshold → confirm target đó

Vấn đề:
  Target A score: 0.42 (118 matches)
  Target B score: 0.44 (122 matches) ← nhầm!
  → Hiện model của target B dù đang scan target A
```

### Giải pháp đề xuất

**Giải pháp 1 (recommended): 1 dự án = 1 .mind file = 1 target**
```
Mỗi project có slug riêng
AR Viewer URL: /view/{slug}
→ Load đúng .mind file của dự án đó
→ .mind chỉ có 1 target duy nhất
→ Không có nguy cơ nhầm target

Ưu điểm: đơn giản, không bao giờ nhầm
Nhược điểm: mỗi lần mở URL mới phải load .mind mới
```

**Giải pháp 2: Tăng warmupTolerance**
```typescript
new MindARThree({
  warmupTolerance: 10,  // cần 10 frame liên tiếp match mới confirm
  missTolerance: 5,
})
// Giảm false positive nhưng vẫn có thể xảy ra
```

**Giải pháp 3: Thiết kế catalog đúng cách**
```
Mỗi trang PHẢI có vùng unique:
  ✅ Số trang lớn ở góc (font đặc biệt, size lớn)
  ✅ QR code riêng mỗi trang (texture phức tạp, unique)
  ✅ Color accent khác nhau mỗi trang
  ✅ Pattern / texture độc đáo ở border
  ❌ Tránh vùng màu đồng nhất lớn (bầu trời, tường trắng)
```

---

## 8. Các lỗi thường gặp & cách xử lý

### 8.1 Camera Permission Denied

```typescript
// Detect và xử lý
try {
  await mindarThree.start();
} catch (err: any) {
  if (err.name === 'NotAllowedError') {
    setError('camera_denied');
    // Hiện hướng dẫn:
    // iOS Safari: Settings → Safari → Camera → Allow
    // Android Chrome: tap icon lock trên URL bar → Camera → Allow
  } else if (err.name === 'NotFoundError') {
    setError('no_camera'); // tablet không có camera sau
  }
}
```

### 8.2 .mind File Load Chậm

```
.mind file 2MB trên 3G (~1Mbps):
  Load time = 2MB / 1Mbps = ~16 giây ← quá lâu

Giải pháp:
  1. Optimize khi compile: dùng ảnh catalog nhỏ hơn (< 1000px width)
     → .mind file nhỏ hơn (~500KB)
  2. Preload hint trong HTML:
     <link rel="preload" href="/files/targets/uuid.mind" as="fetch" crossorigin>
  3. Hiện loading progress rõ ràng để user không bỏ
```

### 8.3 Model .glb Quá Lớn

```
.glb 30MB trên 4G (~10Mbps):
  Load time = 30MB / 10Mbps = ~24 giây ← quá lâu

Giải pháp:
  1. Draco compression (giảm 70-90% size geometry):
     gltf-pipeline -i model.glb -o model-draco.glb --draco.compressionLevel 10
  2. Texture compression (KTX2 + Basis Universal)
  3. Admin warning khi upload > 10MB: "Model lớn có thể load chậm"
  4. Lazy load: chỉ fetch .glb sau khi .mind đã load xong
```

### 8.4 iOS Safari Đặc thù

```
Vấn đề 1: HTTPS bắt buộc
  → Local dev: mkcert localhost
  → Production: Vercel/Netlify tự có SSL

Vấn đề 2: SharedArrayBuffer cần COOP/COEP headers
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
  → Set trong vercel.json

Vấn đề 3: Auto-play video bị block
  → MindAR cần user gesture để start
  → Giải pháp: "Start AR" button trước, sau đó mới gọi mindarThree.start()

Vấn đề 4: Memory limit thấp hơn Android
  → .glb recommend < 20MB cho iOS
  → Dispose Three.js objects khi unmount
```

### 8.5 Model Scale Sai

```
Vấn đề: .glb export từ Blender/SketchUp có scale không chuẩn
  → Tòa nhà có thể nhỏ như hạt đậu hoặc to che cả màn hình

Giải pháp:
  Admin có slider scale (0.001 → 2.0)
  → PUT /api/projects/:id { model_scale: 0.05 }
  → AR Viewer: model.scale.set(scale, scale, scale)

Recommend scale mặc định: 0.1
  (model 10m thực tế → 1m trong scene → vừa tầm nhìn)
```

---

## 9. Performance Checklist cho Mobile

```
[ ] .mind file < 1MB
[ ] .glb file < 10MB (ideally < 5MB)
[ ] Draco compression bật cho .glb
[ ] Texture size ≤ 1024x1024px per texture
[ ] Polygon count ≤ 100,000 tris
[ ] filterMinCF = 0.001 (chống jitter)
[ ] maxTrack = 1 (không track nhiều target cùng lúc)
[ ] dispose() Three.js objects khi component unmount
[ ] body { overflow: hidden } khi AR active (prevent scroll)
[ ] requestAnimationFrame thay vì setInterval
[ ] COOP/COEP headers bật (SharedArrayBuffer)
```
