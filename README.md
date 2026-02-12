# Vercel React Best Practices — Hiểu từ Gốc đến Ngọn

> **Dành cho PM / người cần nắm skill này mà không cần mô tả đúng "từ khóa"**

---

## Phần 1: Vấn Đề Trước Khi Hiểu Skill

### 1.1 Project React/Next.js thường gặp gì?

Bạn có một web React hoặc Next.js (vd **panoee-studio-public** — xem tour 360). Theo thời gian có thể xảy ra:

```
• Trang lần đầu mở rất chậm, màn trắng lâu
• Build hoặc "npm run dev" chậm
• Form / nút đôi khi gửi 2 lần
• Trên Safari ẩn danh bị lỗi hoặc crash
• Danh sách scene/comment đôi khi sai thứ tự hoặc lỗi lạ
• Trên màn hình đôi khi thấy số "0" không mong muốn
```

**Bạn không nhất thiết biết gọi tên:** "waterfall", "bundle", "re-render", "immutability"... Bạn chỉ thấy **triệu chứng**.

### 1.2 Skill là gì trong ngữ cảnh này?

**Skill** = một bộ hướng dẫn (file markdown + nhiều file rule) để **AI Agent** (Cursor, Claude...) biết:
- Khi nào nên áp dụng (task liên quan React, Next.js, performance, bundle...)
- Có những **rule** nào (57 rule trong 8 nhóm)
- **Sai** thế nào, **Đúng** thế nào, **vì sao**

```
┌─────────────────────────────────────────────────────────┐
│              VERCEL REACT BEST PRACTICES SKILL           │
│                                                          │
│  SKILL.md      → Mô tả skill, khi nào dùng, 8 nhóm      │
│  AGENTS.md     → Full 57 rule cho Agent đọc             │
│  rules/*.md    → Từng rule: Sai / Đúng / Impact        │
│                                                          │
│  Agent đọc → Chọn rule phù hợp task → Sửa code theo    │
└─────────────────────────────────────────────────────────┘
```

**Vấn đề nhiều người gặp:**  
"Tôi đâu có nói 'tối ưu bundle' hay 'waterfall' — tôi chỉ nói 'trang chậm'. Vậy skill có áp dụng không?"

→ **Có.** Phần 3 bên dưới giải thích **3 cách** vẫn áp dụng được khi bạn **không mô tả đúng từ khóa**.

---

## Phần 2: Tám Nhóm — Nói Nôm Na Cho Dễ Nhớ

Skill chia 57 rule thành **8 nhóm** theo mức độ ảnh hưởng. Bạn chỉ cần nhớ **tên nhóm + một câu** là đủ để biết khi nào Agent sẽ nghĩ tới nhóm đó.

| Nhóm | Nói nôm na | Triệu chứng thường gặp |
|------|------------|------------------------|
| **1. Waterfalls** | Chờ A xong mới B xong mới C → chậm | Trang trắng lâu, load từng thứ một |
| **2. Bundle** | Gói JS quá nặng → tải lâu, build lâu | Build chậm, lần đầu mở trang chậm |
| **3. Server** | Code chạy trên server; cần auth khi sửa/xóa data | (Khi có tính năng cần đăng nhập) |
| **4. Client** | Code trên browser: fetch, localStorage, scroll | Gọi API trùng, Safari ẩn danh crash |
| **5. Re-render** | React vẽ lại; setState/effect sai → bug | Form gửi 2 lần, list cập nhật sai |
| **6. Rendering** | Cách hiển thị ẩn/hiện, điều kiện | Số 0 hiện trên màn hình, list dài giật |
| **7. JS** | Cách viết JS: không sửa mảng tại chỗ, v.v. | Danh sách sai thứ tự, lỗi khó bắt |
| **8. Advanced** | Tối ưu ít gặp (init một lần, ref...) | Làm sau khi đã ổn 7 nhóm trên |

**Một câu tóm tắt:**

> Bạn thấy **triệu chứng** (chậm, lỗi, giật, 2 lần...) → Agent map vào **nhóm** (Waterfalls, Bundle, Client...) → Agent chọn **rule** trong nhóm đó → Đọc file rule → Sửa code theo "Đúng".

---

## Phần 3: Khi Bạn Không Mô Tả Đúng "Từ Khóa" — Vẫn Áp Dụng Thế Nào?

Trong thực tế bạn hay nói: **"trang chậm"**, **"build lâu"**, **"đôi khi lỗi"**, **"giật"** — **không** nói "waterfalls", "bundle", "toSorted". Khi đó Agent có thể không tự load skill hoặc không chọn đúng rule.

**Vẫn áp dụng được bằng 3 cách sau.**

### 3.1 Cách 1 — Bạn chủ động gọi skill (không cần mô tả đúng từ khóa)

```
Bạn: "Audit project theo Vercel React best practices"
     hoặc @.agents/skills/vercel-react-best-practices

Agent: Load skill → Rà theo 8 nhóm / checklist → Báo chỗ vi phạm + đề xuất sửa
```

**Kết quả:** Agent tự rà và báo. Bạn **không cần** nói "bundle", "waterfall".

### 3.2 Cách 2 — Mô tả theo triệu chứng, Agent map sang nhóm/rule

Bạn chỉ cần nói **triệu chứng** (hoặc mong muốn). Agent sẽ **map** sang nhóm và rule tương ứng:

```
┌─────────────────────────────────────────────────────────┐
│  BẠN NÓI (triệu chứng)     →  AGENT MAP SANG            │
├─────────────────────────────────────────────────────────┤
│  "Trang lần đầu mở rất chậm"  → Waterfalls + Bundle     │
│  "Build / dev chậm"           → Bundle (barrel imports)│
│  "Form gửi 2 lần"             → Re-render (effect vs   │
│                                   event handler)        │
│  "Safari ẩn danh lỗi"         → Client (localStorage    │
│                                   try/catch)            │
│  "Danh sách sai thứ tự"       → JS (toSorted, không    │
│                                   mutate)               │
│  "Thấy số 0 trên màn hình"    → Rendering (conditional │
│                                   render)               │
│  "Rà toàn bộ / audit"         → Quick wins + Top fixes  │
└─────────────────────────────────────────────────────────┘
```

Bạn có thể thêm một câu: **"kiểm tra theo Vercel React best practices"** hoặc **@ skill** — Agent vừa nghe triệu chứng vừa dùng skill để chọn rule.

### 3.3 Cách 3 — Làm theo checklist (không phụ thuộc mô tả task)

Bạn không mô tả vấn đề gì cả. Chỉ nói:

```
"Làm lần lượt theo checklist trong doc step-by-step"
hoặc "Làm case 1, case 2, case 3..."
```

Agent (hoặc bạn) làm theo từng case trong file [react-best-practices-step-by-step-examples.md](./react-best-practices-step-by-step-examples.md): Case 1 (optimizePackageImports) → Case 2 (toSorted) → Case 3 (localStorage) → ...

**Tóm lại:**

> Bạn **không bắt buộc** mô tả đúng từ khóa.  
> Có thể: **(1)** gọi skill / audit theo skill, **(2)** mô tả triệu chứng + nhắc "theo Vercel skill", hoặc **(3)** làm theo checklist. Agent vẫn load skill và áp dụng đúng hướng.

---

## Phần 4: Sai vs Đúng — Vài Ví Dụ Nhanh

Để "dễ hiểu hơn", dưới đây là vài ví dụ **Sai** vs **Đúng** theo từng nhóm (đúng với cách trình bày trong newfile: vấn đề → so sánh).

### 4.1 Waterfalls — Chờ tuần tự

**Sai (trang chậm):**
```
Lấy user xong → mới lấy config → mới lấy data
→ 3 lần chờ mạng cộng lại
```

**Đúng:**
```
Lấy user, config, data cùng lúc (Promise.all hoặc start sớm)
→ Chỉ chờ 1 lần (bằng lâu nhất trong 3 cái)
```

### 4.2 Bundle — Import cả "kho"

**Sai:**
```js
import { Tooltip, Popover, Button } from 'antd'
// Kéo theo rất nhiều module → build chậm, trang tải lâu
```

**Đúng (hoặc dùng config):**
```js
// Cách 1: next.config.js thêm optimizePackageImports: ['antd', ...]
// Cách 2: import từng component
import Button from 'antd/Button'
import Tooltip from 'antd/Tooltip'
```

### 4.3 Client — localStorage không phòng thủ

**Sai:**
```js
localStorage.setItem('config', JSON.stringify(obj))
const x = localStorage.getItem('config')
// Safari ẩn danh / hết quota → throw → crash
```

**Đúng:**
```js
try {
  localStorage.setItem('config:v1', JSON.stringify(onlyNeededFields))
} catch (e) { /* fallback */ }
try {
  const x = localStorage.getItem('config:v1')
} catch { return null }
```

### 4.4 Re-render — Gửi form bằng state + effect

**Sai:**
```
User bấm Submit → set state "submitted" = true
→ useEffect thấy "submitted" mới gọi API
→ Dễ gọi 2 lần hoặc không đúng lúc
```

**Đúng:**
```
User bấm Submit → trong hàm onClick/onSubmit gọi API luôn
→ Không dùng state + effect cho hành động "gửi"
```

### 4.5 JS — Sửa mảng tại chỗ

**Sai:**
```js
const list = scenes.sort((a, b) => ...)
// .sort() sửa luôn mảng scenes → dễ gây bug với React state/props
```

**Đúng:**
```js
const list = scenes.toSorted((a, b) => ...)
// hoặc [...scenes].sort(...) — tạo mảng mới, không sửa mảng gốc
```

---

## Phần 5: Trong Project Hiện Tại (panoee) — Nhận Lại Gì, Thấy Gì, Kiểm Tra Sao?

### 5.1 Bạn sẽ nhận lại được gì

| Hạng mục | Mô tả ngắn |
|----------|------------|
| **Báo cáo audit** | Danh sách file/đoạn code vi phạm theo từng nhóm (có sẵn trong `react-best-practices-assessment.md`). |
| **Code đã sửa** | Config (next.config), refactor (toSorted, localStorage, effect→event, v.v.). |
| **Checklist** | Quick wins / Top fixes đánh dấu Done. |
| **Convention** | Ghi lại trong README/CONTRIBUTING để sau này code đúng chuẩn. |

### 5.2 Bạn sẽ thấy được những gì

| Triệu chứng trước | Sau khi áp dụng đúng |
|-------------------|----------------------|
| Trang trắng / loading lâu | Trang hoặc block nội dung hiện sớm hơn; request chạy song song. |
| Build lâu, file JS lớn | Build nhanh hơn; First Load JS nhỏ hơn. |
| Form gửi 2 lần | Submit 1 lần; cập nhật đúng. |
| Safari ẩn danh crash | Không crash; có try/catch, fallback. |
| Danh sách sai thứ tự / lỗi | Dùng toSorted/immutable; hành vi ổn định. |
| Số 0 hiện trên màn hình | Dùng điều kiện rõ (count > 0 ? ... : null). |

### 5.3 Kiểm tra nhanh

- **Bundle:** Build trước/sau → so sánh "First Load JS", kích thước chunk.
- **JS (toSorted):** Search `.sort(` trong code → không còn mutate state/props.
- **localStorage:** Mở trang ở chế độ ẩn danh → không crash; code có try/catch + version key.
- **Waterfalls:** Network tab → request độc lập chạy song song, không nối đuôi.
- **Re-render:** Submit 1 lần → chỉ 1 request trong Network.

---

## Phần 6: Luồng Suy Luận Của Skill (Tổng Hợp)

```
┌─────────────────────────────────────────────────────────┐
│  BƯỚC 1  Bạn hoặc Agent có:                              │
│          • Task ("tối ưu bundle") HOẶC                   │
│          • Triệu chứng ("trang chậm") HOẶC               │
│          • Yêu cầu ("audit theo Vercel skill")           │
├─────────────────────────────────────────────────────────┤
│  BƯỚC 2  Agent đối chiếu với SKILL.md / AGENTS.md       │
│          → Chọn nhóm (Bundle, Waterfalls, JS...)         │
│          → Chọn rule (vd bundle-barrel-imports)         │
├─────────────────────────────────────────────────────────┤
│  BƯỚC 3  Agent đọc rules/<tên-rule>.md                   │
│          → Nắm Sai / Đúng / Impact                       │
├─────────────────────────────────────────────────────────┤
│  BƯỚC 4  Agent tìm trong project file/đoạn code          │
│          khớp pattern "Sai"                              │
├─────────────────────────────────────────────────────────┤
│  BƯỚC 5  Agent đề xuất hoặc thực hiện thay đổi          │
│          theo "Đúng"                                     │
├─────────────────────────────────────────────────────────┤
│  BƯỚC 6  Bạn / Agent kiểm tra (build, chạy app, rà code)│
└─────────────────────────────────────────────────────────┘
```

**Một câu tóm tắt:**

> Bạn không cần nói đúng từ khóa. Nói **triệu chứng** hoặc **gọi skill** hoặc **làm theo checklist** → Agent vẫn map đúng nhóm/rule và áp dụng trong project.

---

## Tài liệu liên quan

- **Giải thích đơn giản từng thuật ngữ + case:** [react-best-practices-giai-thich-don-gian.md](./react-best-practices-giai-thich-don-gian.md)
- **Ví dụ step-by-step trong chính project:** [react-best-practices-step-by-step-examples.md](./react-best-practices-step-by-step-examples.md)
- **Assessment + quick wins/top fixes:** [react-best-practices-assessment.md](./react-best-practices-assessment.md)
- **Cấu trúc skill + 57 rule:** [vercel-react-best-practices-skill-expanded.md](./vercel-react-best-practices-skill-expanded.md)
