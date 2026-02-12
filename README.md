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

### 1.3 Tại sao cần skill cho AI Agent?

AI Agent (Cursor, Claude...) khi làm task **không tự biết** project nên tuân theo bộ rule nào. Nó cần được **"cho biết"**:

```
❌ KHÔNG CÓ SKILL:
Bạn: "Trang chậm, sửa giúp tôi"
Agent: "Tôi sẽ tối ưu... (đoán chung chung, có thể sửa chỗ ít impact)"

✅ CÓ SKILL (bạn @ skill hoặc nói "theo Vercel React best practices"):
Agent: Load skill → Biết 8 nhóm, 57 rule, Sai/Đúng từng rule
       → Chọn Waterfalls + Bundle → Tìm _app, next.config, getServerSideProps
       → Đề xuất cụ thể (Promise.all, optimizePackageImports, v.v.)
```

**Vấn đề cốt lõi:** Không có skill = Agent **tự suy luận** (tốn tokens, có thể sai ưu tiên). Có skill = Agent **đọc rule** → áp dụng đúng pattern, đúng impact.

### 1.4 "Nhưng đã có file assessment rồi mà?" — Câu hỏi quan trọng

Nhiều người hỏi: **"Đã có react-best-practices-assessment.md liệt kê quick wins và top fixes rồi. Tại sao còn cần skill?"**

**Assessment (file markdown)** = **Danh sách** đã rà cho project này: chỗ nào vi phạm, nên sửa gì. Agent (hoặc người) **đọc** file đó để biết.

**Skill (SKILL.md + AGENTS.md + rules/)** = **Bộ chuẩn** cho mọi React/Next.js: **tại sao** sai, **đúng** phải làm gì, **impact** ra sao. Agent dùng skill để:
- **Khi chưa có assessment:** Tự rà project theo 57 rule và sinh ra báo cáo tương tự.
- **Khi đã có assessment:** Hiểu sâu từng mục (vd "toSorted" không chỉ là "đổi chữ" mà vì immutability, stale closure).
- **Khi viết code mới:** Không vi phạm rule ngay từ đầu.

```
┌─────────────────────────────────────────────────────────┐
│  ASSESSMENT (file .md)     =  "Kết quả rà cho 1 project" │
│  → Đọc để biết: panoee chỗ nào cần sửa                  │
├─────────────────────────────────────────────────────────┤
│  SKILL (rules + AGENTS)    =  "Chuẩn cho mọi project"   │
│  → Agent dùng để: rà, sửa, viết code mới đúng chuẩn     │
└─────────────────────────────────────────────────────────┘
```

**Một câu:** Assessment = danh sách vi phạm của **project này**. Skill = bộ luật **Sai/Đúng** để Agent rà mọi project và sửa đúng cách.

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

### 2.1 Vấn đề và hậu quả từng nhóm (tóm tắt)

| Nhóm | Vấn đề điển hình | Hậu quả nếu không sửa |
|------|-------------------|------------------------|
| **Waterfalls** | Request A xong mới gọi B; client đợi multilanguages xong mới render. | Trang trắng lâu, TTFB cao, user bỏ trang. |
| **Bundle** | Import barrel (`antd`, `@ant-design/icons`) kéo cả kho. | Build lâu, First Load JS lớn, cold start chậm. |
| **Server** | API/Server Action không check auth khi có mutation. | Ai cũng gọi được xóa/sửa (bảo mật). |
| **Client** | Nhiều component gọi cùng API; localStorage không try/catch. | Request trùng, crash Safari ẩn danh. |
| **Re-render** | setState dựa trên state cũ (stale); logic submit trong useEffect. | Double submit, list sai thứ tự, bug khó tái hiện. |
| **Rendering** | `count && <Badge>` khi count = 0 → hiện "0". | UI sai (số 0 hiện chữ), list dài giật. |
| **JS** | `arr.sort()` mutate; nhiều lần .find() trên cùng mảng. | Bug state, re-render thừa, chậm. |
| **Advanced** | Init chạy mỗi lần mount; effect subscribe không cleanup. | Memory leak, chạy 2 lần (Strict Mode). |

### 2.2 Luồng từ "triệu chứng" đến nhóm (sơ đồ)

```
Bạn nói: "trang chậm" / "build lâu" / "form gửi 2 lần" / "Safari crash"
                    │
                    ▼
        ┌───────────────────────┐
        │ Skill / Agent map     │
        │ triệu chứng → nhóm   │
        └───────────┬───────────┘
                    │
    ┌───────┬───────┼───────┬───────┐
    ▼       ▼       ▼       ▼       ▼
 Waterfall Bundle Client Re-render ...
    │       │       │       │
    ▼       ▼       ▼       ▼
 Rule A   Rule B   Rule C   Rule D
    │       │       │       │
    ▼       ▼       ▼       ▼
 Code     Code     Code     Code
 (_app,   (next.   (SWR,    (setState
  GSSP)    config)  localStorage) handler)
```

Chi tiết bảng "triệu chứng → nhóm" nằm ở **Phần 3** và file **react-best-practices-step-by-step-examples.md** (bảng "Triệu chứng → Nhóm / Rule").

### 2.3 Đào sâu hai nhóm điển hình: Waterfalls và Bundle

**Waterfalls — Vấn đề và tại sao:**

Trang web cần nhiều thứ: user, config, danh sách scene, đa ngôn ngữ... Nếu code viết kiểu "lấy A xong mới lấy B, B xong mới lấy C", tổng thời gian = A + B + C. User thấy màn trắng hoặc loading rất lâu. **Tại sao hay xảy ra:** Dễ viết theo kiểu "cần user để gọi API tiếp" hoặc "cần multilanguages xong mới render layout" → vô tình tạo chuỗi chờ. **Hậu quả:** TTFB (Time to First Byte) cao, FCP (First Contentful Paint) trễ, tỷ lệ thoát trang tăng.

**Bundle — Vấn đề và tại sao:**

Thư viện UI như `antd` có hàng trăm component. Nếu bạn viết `import { Button } from 'antd'`, bundler có thể kéo theo cả "barrel file" và nhiều module liên quan. **Tại sao:** Barrel export (file index re-export tất cả) khiến cây dependency lớn. **Hậu quả:** Build chậm, file JS gửi xuống browser nặng → lần đầu mở trang chậm (cold start). **Giải pháp:** Next 13.5+ có `optimizePackageImports`: bạn khai báo package, Next tự chuyển sang chỉ import đúng thứ cần → không đụng từng dòng code.

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

### 3.4 So sánh: Có skill vs Không có skill (khi bạn nói "trang chậm")

| Khía cạnh | Không có skill | Có skill (gọi @ hoặc "theo Vercel React best practices") |
|-----------|----------------|----------------------------------------------------------|
| **Agent làm gì** | Đoán chung: "có thể do ảnh, do API..." → đề xuất chung chung. | Map "trang chậm" → Waterfalls + Bundle → đọc rule → tìm _app, next.config, getServerSideProps → đề xuất cụ thể (Promise.all, optimizePackageImports). |
| **Kết quả** | Có thể sửa chỗ ít impact trước; bỏ sót waterfall hoặc barrel. | Ưu tiên đúng: Waterfalls (TTFB) và Bundle (First Load) trước. |
| **Bạn cần nói** | Có thể phải tự nói "waterfall", "bundle" mới trúng. | Chỉ cần "trang chậm" + "theo skill" hoặc @ skill. |

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

### 4.6 Rendering — Điều kiện số (count = 0)

**Sai:**
```jsx
{ count && <Badge>{count}</Badge> }
// Khi count = 0 → JavaScript coi 0 là falsy → vẫn render "0" (vì 0 && <Badge> trả về 0)
```

**Đúng:**
```jsx
{ count > 0 ? <Badge>{count}</Badge> : null }
// Hoặc: { !!count && <Badge>{count}</Badge> } — rõ ràng boolean
```

**Trong project:** Rà các chỗ `something.length && <...>` hoặc `count && <...>` trong panoee; nếu là số có thể bằng 0 thì đổi sang điều kiện rõ ràng.

### 4.7 Server — Auth khi có mutation (chỉ khi cần)

**Lưu ý:** Web public xem tour **không cần đăng nhập**. Phần này chỉ áp dụng **khi** có Server Action hoặc API route **sửa/xóa dữ liệu** (admin, collaboration, form nhạy cảm).

**Sai (khi đã có Server Action xóa/sửa):**
```js
// Trong Server Action hoặc API route: nhận request xóa comment
export async function deleteComment(id) {
  await db.comments.delete(id)  // Không kiểm tra ai gọi → ai cũng xóa được
}
```

**Đúng:**
```js
export async function deleteComment(id) {
  const session = await getServerSession()
  if (!session?.user) throw new Error('Unauthorized')
  // Kiểm tra quyền: user có được phép xóa comment này không
  await db.comments.delete(id)
}
```

**Trong project (panoee):** Hiện dùng Pages Router + Redux/saga, rewrite ra API ngoài; chưa có Next API route / Server Action trong repo. Khi **thêm** API route hoặc "use server" làm việc nhạy cảm → bắt buộc check session + quyền trong từng action.

### 4.8 Advanced — Init chạy một lần (tóm tắt)

**Sai:** Logic khởi tạo toàn cục đặt trong `useEffect` không guard → Strict Mode / remount chạy 2 lần.

**Đúng:** Dùng biến guard (vd `didInit`) hoặc init ngoài component; hoặc chấp nhận chạy 2 lần nếu logic idempotent.

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

### 5.4 Ví dụ cụ thể từng bước (trong chính project)

Để thấy rõ **trigger → rule → code hiện tại → bước sửa → kiểm tra**, dùng file [react-best-practices-step-by-step-examples.md](./react-best-practices-step-by-step-examples.md). Dưới đây là **tóm tắt đường dẫn và bước** cho 3 case đầu:

| Case | File / vị trí | Bước ngắn gọn |
|------|----------------|---------------|
| **1 — optimizePackageImports** | `panoee-studio-public/src/next.config.js` (object `nextConfig`) | Thêm `experimental: { optimizePackageImports: ['antd', '@ant-design/icons', '@ant-design/compatible', '@ant-design/cssinjs'] }` → build lại, so sánh First Load JS. |
| **2 — toSorted** | `containers/Home/utils.ts` (dòng ~174: `scenes.sort`); `containers/Home/hook/useCommentScene.tsx` (`.slice().sort`) | Đổi `scenes.sort(...)` → `scenes.toSorted(...)` (hoặc `[...scenes].sort(...)`); `.slice().sort` → `.toSorted`. Search `.sort(` không còn mutate state/props. |
| **3 — localStorage** | `src/utils/localStorage.ts` | Thêm version key (vd `config:v1`), bọc get/set trong try/catch, chỉ lưu field cần → test Safari ẩn danh. |

Case 4 (Waterfalls — `_app`, getServerSideProps) và Case 5 (conditional render) cũng có trong file step-by-step với bảng từng bước và đoạn code mẫu.

### 5.5 Scenario: Một ngày làm việc với skill (minh họa)

**Buổi sáng — Bạn không nói đúng từ khóa:**

- Bạn: "Trang tour lần đầu mở chậm quá."
- Bạn thêm: "Kiểm tra theo Vercel React best practices giúp tôi."
- Agent: Load skill → map "chậm" → Waterfalls + Bundle → đọc rule → tìm `_app.tsx`, `next.config.js`, getServerSideProps → đề xuất: Promise.all / start sớm cho request, thêm optimizePackageImports.

**Buổi chiều — Làm theo checklist:**

- Bạn: "Làm case 1 và 2 trong doc step-by-step."
- Agent: Mở step-by-step-examples → Case 1 (next.config), Case 2 (utils.ts, useCommentScene) → thực hiện từng bước, báo đã sửa và gợi ý kiểm tra (build, search `.sort(`).

**Cuối ngày — Kiểm tra:**

- Build trước/sau → so sánh First Load JS.
- Mở trang ẩn danh (Safari/Chrome) → không crash.
- Network tab → request chạy song song, không nối đuôi.

Như vậy bạn **không cần** nói "waterfall", "barrel", "toSorted" — chỉ cần triệu chứng + "theo skill" hoặc checklist.

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

## Phần 7: Cấu Trúc Skill Chi Tiết (SKILL.md, AGENTS.md, rules/)

Skill Vercel React Best Practices **không** dùng thư mục `references/` hay `scripts/`. Chỉ có:

| Thành phần | Vai trò | Agent dùng thế nào |
|------------|--------|---------------------|
| **SKILL.md** | Mô tả skill: tên, khi nào áp dụng (React/Next.js, performance, bundle...), Quick Reference 8 nhóm. | Đọc đầu tiên → quyết định có load skill không; biết 8 nhóm để map task → nhóm. |
| **AGENTS.md** | Toàn bộ 57 rule: tên rule, nhóm, mức ưu tiên, tóm tắt Sai/Đúng. | Đọc khi cần danh sách đầy đủ; chọn rule cụ thể theo task. |
| **rules/*.md** | Từng file một rule: mô tả, Sai (anti-pattern), Đúng (pattern), Impact, đôi khi ví dụ code. | Đọc **sau khi** chọn rule → lấy chính xác "Sai" và "Đúng" để tìm trong code và sửa. |

**Luồng thực tế:**

```
Task / triệu chứng
       → SKILL.md (có áp dụng không? nhóm nào?)
       → AGENTS.md hoặc Quick Reference (rule nào?)
       → rules/<tên-rule>.md (Sai / Đúng cụ thể)
       → Tìm trong project → Sửa theo "Đúng"
```

**So sánh với "chỉ đọc assessment":** Assessment cho bạn **danh sách chỗ sửa** của một project. Skill cho Agent **định nghĩa** mỗi chỗ đó **sai thế nào** và **đúng phải làm gì** — dùng được cho mọi project và cho code mới.

---

## Phần 8: Quick Wins vs Top Fixes — Giải Thích Dài Hơn

### 8.1 Quick wins (làm nhanh, ít thay đổi)

| Mục | Giải thích ngắn | Trong panoee |
|-----|-----------------|--------------|
| **optimizePackageImports** | Một dòng config trong `next.config.js`; Next 13.5+ tự đổi barrel import thành direct. | Thêm `experimental.optimizePackageImports: ['antd', '@ant-design/icons', ...]` → build và First Load JS giảm rõ. |
| **toSorted** | Đổi `.sort()` thành `.toSorted()` (hoặc `[...arr].sort()`) để không mutate. | `containers/Home/utils.ts`, `useCommentScene.tsx` — vài chỗ sort; đổi xong search `.sort(` không còn trên state/props. |
| **localStorage** | Version key + try/catch + chỉ lưu field cần. | `src/utils/localStorage.ts` — bọc get/set, test Safari ẩn danh. |
| **Conditional render** | Số: `count > 0 ? <Badge> : null` thay vì `count && <Badge>`. | Rà component có count/length → sửa điều kiện. |

**Đặc điểm:** Ít đụng logic nghiệp vụ, ít rủi ro; hiệu quả rõ (bundle, ổn định, đúng UI).

### 8.2 Top fixes (cần phân tích hơn)

| Mục | Giải thích ngắn | Trong panoee |
|-----|-----------------|--------------|
| **Waterfalls** | Request chạy song song (Promise.all hoặc start sớm); client không đợi từng thứ tuần tự. | `_app.tsx`: multilanguages / init; getServerSideProps: nhiều fetch → gom song song. |
| **Effect vs event** | Hành động "gửi / submit" gọi API trong **event handler**, không dùng state + useEffect. | Rà form / nút submit → chuyển logic gọi API vào onClick/onSubmit. |
| **SWR (hoặc tương đương)** | Gom request trùng key; cache; tránh nhiều component gọi cùng API. | useGetTenant ở NotFound, collaboration, preview → có thể dùng SWR cùng key. |
| **Server auth (khi có mutation)** | Khi thêm Server Action / API route sửa-xóa: check session + quyền trong action. | Hiện chưa có; khi thêm tính năng cần đăng nhập để sửa/xóa → áp dụng. |

**Đặc điểm:** Ảnh hưởng lớn (TTFB, trải nghiệm, bảo mật) nhưng cần hiểu luồng hiện tại (Redux, saga, _app) rồi mới refactor an toàn.

### 8.3 Thứ tự ưu tiên gợi ý

1. **Quick wins** (optimizePackageImports → toSorted → localStorage → conditional render).
2. **Waterfalls** (client _app + server getServerSideProps).
3. **Re-render** (effect vs event, setState functional).
4. **Client** (SWR cho useGetTenant nếu cần).
5. **Server auth** chỉ khi đã có Server Action/API mutation nhạy cảm.

### 8.4 Bảng impact (CRITICAL / HIGH / MEDIUM) — nhắc nhanh

| Nhóm / mục | Impact | Lý do |
|------------|--------|--------|
| Waterfalls (client + server) | **CRITICAL** | Trực tiếp TTFB, thời gian thấy nội dung. |
| optimizePackageImports (Bundle) | **CRITICAL** | Trực tiếp build time và First Load JS. |
| Server auth (khi có mutation) | **CRITICAL** | Bảo mật: ai cũng gọi được nếu không check. |
| toSorted, localStorage, conditional render | **HIGH** | Ổn định, đúng UI, không crash. |
| Effect vs event, SWR | **HIGH** | UX (không double submit), ít request thừa. |
| Rendering (list dài, content-visibility) | **MEDIUM** | Mượt hơn khi list rất dài. |
| Advanced (init, ref) | **LOW** | Tối ưu thêm, ít khi là nút thắt. |

### 8.5 Ví dụ lệnh kiểm tra (sau khi sửa)

- **Build và so sánh:** `cd panoee-studio-public && npm run build` — xem log "First Load JS", thời gian build.
- **Tìm chỗ còn mutate sort:** Trong repo search `\.sort\(` — không còn gọi trực tiếp trên state/props (phải là toSorted hoặc `[...].sort`).
- **Test localStorage:** Mở app trong cửa sổ ẩn danh (Safari/Chrome) → vào trang dùng localStorage → không crash.
- **Network:** DevTools → Network → reload trang → xem request có chạy song song (nhiều request cùng start) thay vì nối đuôi.

---

## Phần 9: Cheat Sheet — Một Dòng Nhắc Nhanh

| Bạn muốn | Nói / làm gì |
|----------|---------------|
| Rà toàn bộ project | "Audit theo Vercel React best practices" hoặc @ skill. |
| Trang chậm | "Trang chậm, kiểm tra theo Vercel React best practices" → Agent nghĩ Waterfalls + Bundle. |
| Build chậm | "Build chậm, tối ưu theo skill" → Bundle (optimizePackageImports). |
| Form gửi 2 lần | "Form gửi 2 lần" → Re-render (effect vs event handler). |
| Safari crash | "Safari ẩn danh lỗi" → Client (localStorage try/catch). |
| Làm theo từng bước | "Làm case 1, 2, 3 trong doc step-by-step" hoặc mở [react-best-practices-step-by-step-examples.md](./react-best-practices-step-by-step-examples.md). |
| Kiểm tra sau khi sửa | Build trước/sau; search `.sort(`, `localStorage`; Network tab (song song?); test ẩn danh. |

---

## Phần 10: Tóm Tắt Cuối

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  VERCEL REACT BEST PRACTICES — TÓM TẮT CUỐI                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. Skill = bộ rule (SKILL + AGENTS + rules/) để Agent rà & sửa React/Next  │
│  2. 8 nhóm: Waterfalls, Bundle, Server, Client, Re-render, Rendering, JS,   │
│     Advanced — map từ triệu chứng (chậm, lỗi, 2 lần...) → nhóm → rule       │
│  3. Bạn không cần nói đúng từ khóa: gọi skill / nói triệu chứng / checklist │
│  4. Quick wins: optimizePackageImports, toSorted, localStorage, conditional  │
│  5. Top fixes: Waterfalls, effect→event, SWR, server auth (khi có mutation) │
│  6. Assessment = kết quả rà 1 project; Skill = chuẩn Sai/Đúng mọi project  │
│  7. Kiểm tra: build, search .sort(, localStorage, Network, Safari ẩn danh   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Điểm cần lưu ý (giới hạn / khi nào cẩn thận)

- **Skill không thay code thay bạn:** Agent đề xuất hoặc sửa theo rule; bạn vẫn nên review (đặc biệt Waterfalls, Redux/saga) vì refactor có thể đụng luồng nghiệp vụ.
- **Assessment đã rà cho panoee:** Nếu codebase thay đổi nhiều (thêm trang, đổi thư viện), nên chạy lại audit hoặc làm lại checklist.
- **Server auth:** Chỉ áp dụng khi **đã có** Server Action hoặc API route làm việc nhạy cảm; web public xem tour không cần login.

---

