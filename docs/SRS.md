# SRS — Software Requirements Specification
# RealAR – Real Estate WebAR Platform

**Version:** 1.0  
**Date:** 2026-02-28  
**Author:** RealAR Team  

---

## 1. Tổng quan hệ thống

| Thuộc tính | Giá trị |
|---|---|
| Tên dự án | RealAR – Real Estate WebAR Platform |
| Loại | Web Application (PWA-ready) |
| Mục tiêu | Nền tảng WebAR mã nguồn mở cho BDS, tương tự Zappar/8th Wall nhưng tự host |
| Môi trường | Trình duyệt mobile (Chrome Android 90+, Safari iOS 14.3+), desktop |
| Ngôn ngữ UI | Tiếng Việt |
| Ngôn ngữ code | TypeScript |

---

## 2. Actors

Hệ thống có **2 actors** thực sự tương tác với hệ thống:

| Actor | Mô tả | Quyền truy cập |
|---|---|---|
| **Admin** | Người quản lý nội dung — tạo dự án BDS, upload model 3D, upload ảnh catalog, compile .mind file, tạo QR code | Đăng nhập Dashboard → toàn quyền CRUD |
| **Viewer** | Bất kỳ ai cầm điện thoại (sale staff hoặc khách hàng) — không phân biệt | Không đăng nhập, quét QR / mở link → xem AR |

> **Lý do chỉ 2 actors:** Sale staff và khách hàng sử dụng chung 1 URL, không có permission khác nhau, không có session riêng — cùng 1 flow duy nhất.

---

## 3. Functional Requirements

### FR-01: AR Viewer (Viewer actor)

| ID | Mô tả | Độ ưu tiên |
|---|---|---|
| FR-01-01 | Xin quyền camera, hiển thị camera feed fullscreen | MUST |
| FR-01-02 | Load file .mind (image targets) từ server theo project slug | MUST |
| FR-01-03 | Nhận diện ảnh catalog bằng MindAR image tracking | MUST |
| FR-01-04 | Load và render model 3D (.glb) lên anchor khi nhận diện được | MUST |
| FR-01-05 | Hiển thị overlay: tên dự án, giá từ, diện tích khi đang tracking | MUST |
| FR-01-06 | Tap vào vùng AR → slide-up panel thông tin chi tiết dự án | MUST |
| FR-01-07 | QR code entry: quét QR → redirect đến AR viewer đúng project | MUST |
| FR-01-08 | Loading screen animation trong lúc MindAR khởi động (3–5s) | MUST |
| FR-01-09 | Hỗ trợ multi-target: nhiều trang catalog trong 1 .mind file | SHOULD |
| FR-01-10 | Nút "Gọi ngay" trong detail panel → mở dialpad phone | SHOULD |
| FR-01-11 | Hiển thị hướng dẫn khi chưa detect được ảnh | SHOULD |
| FR-01-12 | Fallback 3D viewer khi browser không hỗ trợ WebAR | COULD |

### FR-02: Admin Dashboard (Admin actor)

| ID | Mô tả | Độ ưu tiên |
|---|---|---|
| FR-02-01 | Đăng nhập bằng email/password, JWT token | MUST |
| FR-02-02 | Dashboard tổng quan: số dự án, lượt xem AR, lượt quét QR | MUST |
| FR-02-03 | Tạo / sửa / xóa dự án BDS | MUST |
| FR-02-04 | Upload ảnh catalog/brochure | MUST |
| FR-02-05 | Compile ảnh catalog → .mind file trực tiếp trong browser | MUST |
| FR-02-06 | Preview feature points sau khi compile (quality check) | MUST |
| FR-02-07 | Upload model 3D (.glb/.gltf) | MUST |
| FR-02-08 | Tạo và download QR code PNG cho từng dự án | MUST |
| FR-02-09 | Preview link AR viewer cho từng dự án | MUST |
| FR-02-10 | Điều chỉnh scale của model 3D bằng slider | MUST |
| FR-02-11 | Upload gallery ảnh cho trang chi tiết dự án | SHOULD |
| FR-02-12 | Xem analytics: lượt quét QR, lượt view AR theo dự án | SHOULD |
| FR-02-13 | Filter / search danh sách dự án | SHOULD |

### FR-03: Backend API

| ID | Mô tả | Độ ưu tiên |
|---|---|---|
| FR-03-01 | POST /api/auth/login — xác thực và cấp JWT | MUST |
| FR-03-02 | GET/POST/PUT/DELETE /api/projects — CRUD dự án | MUST |
| FR-03-03 | POST /api/upload/image — upload ảnh | MUST |
| FR-03-04 | POST /api/upload/model — upload .glb | MUST |
| FR-03-05 | POST /api/upload/target — upload .mind đã compile | MUST |
| FR-03-06 | GET /files/* — serve static files | MUST |
| FR-03-07 | POST /api/analytics/track — log AR events | SHOULD |
| FR-03-08 | GET /api/analytics/:projectId — query stats | SHOULD |

---

## 4. Non-Functional Requirements

| ID | Mô tả | Metric |
|---|---|---|
| NFR-01 | AR Viewer khởi động nhanh | < 5 giây trên mạng 4G |
| NFR-02 | MindAR tracking mượt | ≥ 30fps trên iPhone 11 / Android mid-range |
| NFR-03 | HTTPS bắt buộc | Camera API không hoạt động trên HTTP |
| NFR-04 | Mobile-first | Tất cả UI tối ưu cho màn hình ≤ 430px |
| NFR-05 | Browser support | Chrome Android 90+, Safari iOS 14.3+ |
| NFR-06 | Admin responsive | Desktop 1280px+ |
| NFR-07 | File size limits | .glb ≤ 50MB, ảnh catalog ≤ 5MB, .mind ≤ 10MB |
| NFR-08 | Open source | MIT License, tự host được |

---

## 5. User Stories

### Admin
```
AS an Admin
I WANT TO upload ảnh catalog và compile thành .mind file
SO THAT AR Viewer có thể nhận diện ảnh đó

AS an Admin
I WANT TO upload model 3D .glb cho từng dự án
SO THAT model đó hiện ra khi scan đúng trang catalog

AS an Admin
I WANT TO tạo QR code cho từng dự án
SO THAT sale staff có thể in QR lên catalog và cho khách quét

AS an Admin
I WANT TO xem số lượt xem AR theo từng dự án
SO THAT tôi biết dự án nào được quan tâm nhiều nhất
```

### Viewer (Sale Staff / Khách hàng)
```
AS a Viewer
I WANT TO quét QR code trên catalog
SO THAT tôi mở được AR experience của đúng dự án đó

AS a Viewer
I WANT TO hướng camera vào ảnh trang catalog
SO THAT model 3D tòa nhà hiện ra ngay trên catalog

AS a Viewer
I WANT TO tap vào model 3D
SO THAT tôi xem được thông tin chi tiết: giá, diện tích, tiện ích

AS a Viewer
I WANT TO nhấn nút "Gọi ngay"
SO THAT tôi liên hệ trực tiếp với chủ đầu tư
```

---

## 6. Constraints

- Phải là **open source** (MIT), không phụ thuộc dịch vụ trả phí
- AR engine: **MindAR.js** (duy nhất actively maintained, free)
- Không cần app native — chạy 100% trên trình duyệt
- Backend đơn giản, tự host được trên VPS rẻ tiền
- File storage: local filesystem (có thể migrate S3 sau)

---

## 7. Out of Scope (v1.0)

- Multi-user / role-based admin (chỉ 1 admin)
- Video overlay thay vì model 3D
- Face tracking
- Payment integration
- Mobile app (React Native)
- Multi-language (chỉ Tiếng Việt v1)
