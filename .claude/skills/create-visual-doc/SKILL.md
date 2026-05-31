---
name: create-visual-doc
description: "Tạo bộ tài liệu hoàn chỉnh gồm bản Markdown chi tiết và bản HTML trực quan từ một chủ đề cho trước. Kích hoạt khi user yêu cầu tạo tài liệu, viết guide, tạo hướng dẫn, hoặc document một chủ đề vào repo. Ví dụ: 'tạo tài liệu về RAG', 'viết guide về agent architecture', 'document cách dùng MCP servers'."
---

# Skill: Tạo Bộ Tài Liệu Trực Quan

Skill này tạo bộ tài liệu hoàn chỉnh (Markdown + HTML trực quan) cho repo `share-everything-about-ai`. Luôn viết bằng **tiếng Việt**. Tác giả: **Hưng 2x**.

## Quy trình tổng thể

Skill hỗ trợ **2 workflow** phụ thuộc vào tình huống:

1. **Workflow 1: Tạo tài liệu mới từ không** → Theo 4 bước bên dưới
2. **Workflow 2: Cải thiện tài liệu hiện tại** → Bỏ qua Bước 1, bắt đầu từ Bước 2

## Quy trình 4 bước

### Bước 1 — Nghiên cứu & Tổng hợp tài liệu Markdown

**Mục tiêu:** Tạo file `.md` chi tiết, đầy đủ, chính xác. *(Bỏ qua nếu tài liệu đã tồn tại)*

1. **Xác định scope rõ ràng TRƯỚC KHI viết:**
   - Bài này giải thích **cái gì**? (learning objective cụ thể)
   - Bài này thuộc **tầng nào** trong series? (foundations, engineering, architecture...)
   - **Những topic nào KHÔNG thuộc** bài này (dù liên quan), và sẽ được cover ở bài nào?
   - Độ sâu phù hợp: bài foundations chỉ cần hiểu "what + why", không cần deep-dive "how every bit works"
   - Ghi ra danh sách **in-scope** và **out-of-scope** trước khi viết

   Ví dụ scope cho bài WebSocket (Tầng 1 — Nền tảng):
   - ✅ In-scope: HTTP vs WebSocket, handshake, frame overhead, 2 patterns, lifecycle, WSS
   - ❌ Out-of-scope: Frame structure detail (bit-level) → Tầng 2, Failure modes (5 scenarios) → Tầng 4, Latency budget → Bài 06, gRPC/MQTT comparison → Bài 07, Redis catastrophe → Bài 12

2. **Phân tích chủ đề:** Xác định phạm vi, các khía cạnh cần bao quát, đối tượng người đọc.
3. **Tìm kiếm bổ sung:** Dùng web search để tìm:
   - Tài liệu chính thức (official docs, blog posts)
   - Bài viết chuyên sâu từ cộng đồng (Medium, Dev.to, GitHub)
   - Số liệu, thống kê, case study thực tế
   - Hình ảnh, sơ đồ, demo nếu có
3. **Viết tài liệu Markdown** theo cấu trúc:
   ```markdown
   # Tiêu đề chủ đề

   ## Mục lục
   - [1. Phần 1](#...)
   - ...

   ## 1. Giới thiệu / Tổng quan
   ## 2. Các khái niệm cốt lõi
   ## 3. Kiến trúc / Mô hình / Framework
   ## 4. Hướng dẫn thực hành
   ## 5. Best practices & Gotchas
   ## 6. So sánh với các giải pháp khác (nếu phù hợp)
   ## 7. Tài nguyên tham khảo

   ## Nguồn tham khảo
   [1] Tên nguồn — URL
   [2] ...
   ```
5. **Chú thích nguồn:** Mỗi claim quan trọng cần ghi chú nguồn tham khảo `【[số](url)】`.

---

### Bước 2 — Đánh giá & Chỉnh sửa Nội dung

**Mục tiêu:** Cải thiện chất lượng, độ sâu, và sự hấp dẫn của nội dung Markdown. *(Áp dụng cho TẤT CẢ tài liệu, kể cả những bài mới viết)*

**Thời gian:** 45–60 phút

1. **Kiểm tra scope TRƯỚC TIÊN (scope audit):**
   - Đọc lại danh sách in-scope/out-of-scope đã xác định ở Bước 1
   - Mỗi section hiện tại có thuộc scope không? Nếu không → đánh dấu để cắt
   - Có section nào đang duplicate nội dung của bài khác không? → preview ngắn là đủ, không expand
   - **Target length:** Bài Tầng 1 (foundations): 200–400 dòng. Bài Tầng 2–3: 400–600 dòng. Bài Tầng 4: 600–800 dòng. Nếu vượt quá → dấu hiệu scope creep.

2. **Đánh giá nội dung** theo 7 tiêu chí:
   - **Độ sâu:** Có giải thích đủ các khái niệm cốt lõi? Có ví dụ cụ thể, code snippet thực tế?
   - **Sự hấp dẫn:** Có hook từ đầu? Có câu hỏi rhetorical, so sánh bất ngờ, hay anecdote?
   - **Tính thực tiễn:** Có best practices? Có gotchas production thực tế?
   - **Liên tục series:** Có liên kết đến bài trước/sau? Có preview ngắn topic của bài tiếp theo?
   - **Cấu trúc:** Có mục lục rõ ràng? Có transition mượt giữa các section?
   - **Tonality:** Có phù hợp với đối tượng và tầng bài viết?
   - **Chính xác:** Số liệu, thuật ngữ, code có đúng không? Có deprecated info?

3. **Xác định khoảng cách cải thiện (trong scope):**
   - Mỗi section có thiếu ví dụ cụ thể không?
   - Có gotchas/pitfalls quan trọng cần thêm vào không?
   - Các bài sau cần prerequisite gì từ bài này? Đã đủ chưa?

4. **Cải thiện nội dung:**
   - Cải thiện quality của nội dung **đã có trong scope** — không thêm topic mới ngoài scope
   - Thêm ví dụ, numbers thực tế, production insights
   - Tối ưu wording để rõ, sắc nét, hấp dẫn
   - Đảm bảo đầy đủ dấu tiếng Việt, không lỗi typo
   - **KHÔNG nhúng HTML/CSS vào file .md** — dùng ASCII diagrams, tables, code blocks thay thế

5. **Output:** Updated `.md` file với nội dung cải thiện, sạch Markdown thuần

---

### Bước 3 — Thêm Minh Hoạ Trực Quan

**Mục tiêu:** Tạo/cập nhật các hình ảnh minh hoạ, diagram, và trang HTML để giúp người mới dễ hiểu nội dung.

**Thời gian:** 2–4 giờ

#### A. Thêm hình ảnh cho file Markdown

**a. Lên danh sách hình cần tạo:**
- Mỗi section chính (h2) cần ít nhất 1 hình minh hoạ
- Xác định loại hình phù hợp:
  - **Sơ đồ kiến trúc / flow:** Cho phần kiến trúc, mô hình, framework
  - **Infographic tổng quan:** Cho phần giới thiệu, tổng quan
  - **So sánh trực quan:** Cho phần so sánh giải pháp
  - **Minh hoạ khái niệm:** Cho phần khái niệm trừu tượng
  - **Quy trình từng bước:** Cho phần hướng dẫn thực hành
  - **Bảng tổng hợp / checklist:** Cho phần best practices

**b. Tạo hình ảnh bằng AI:**
- Sử dụng tool tạo ảnh AI (image generation) để tạo từng ảnh
- Prompt nên mô tả rõ: nội dung, phong cách (clean, professional, diagram-style), màu (tím #534AB7, xanh lá #1D9E75, xanh dương #378ADD)
- Ưu tiên: flat illustration, clean diagram, modern infographic
- Tránh: quá nhiều text (khó đọc), quá phức tạp

**c. Lưu trữ & nhúng:**
- Lưu vào `docs/<topic-folder>/images/` với tên: `fig-01-mo-ta.png`, `fig-02-mo-ta.png`
- Nhúng vào `.md`:
  ```markdown
  ![Mô tả ngắn](./images/fig-01-mo-ta.png)
  *Figure 1 — Tiêu đề hình ảnh*
  *Chú thích mô tả nội dung hình ảnh thể hiện.*
  ```
- Mỗi hình PHẢI có: alt text, figure title, chú thích

**d. Nếu không có tool tạo ảnh AI:**
- Để placeholder: `<!-- TODO: Tạo ảnh minh hoạ cho section này -->`
- Mô tả chi tiết ảnh cần tạo
- Thông báo cho user những ảnh nào chưa được tạo

#### B. Tạo/cập nhật tài liệu HTML trực quan

**Mục tiêu:** Chuyển Markdown thành trang HTML đẹp, dễ đọc, hấp dẫn người mới.

1. **Sử dụng design system** của repo (xem `references/design-system.css`).
2. **Bắt buộc có các thành phần trực quan:**
   - **Hero section:** Tiêu đề + subtitle + 3–4 thống kê nổi bật
   - **SVG diagrams:** Sơ đồ kiến trúc, flow, so sánh — inline SVG
   - **Card grids:** Phân loại, danh mục dùng grid cards + icon + ví dụ
   - **Stepper/Timeline:** Quy trình nhiều bước
   - **Good vs Bad boxes:** So sánh cách làm đúng/sai
   - **Highlight boxes:** Trích dẫn, insights quan trọng, tips
   - **Code blocks:** Ví dụ code với syntax highlighting (`code-key`, `code-str`, `code-comment`, `code-punct`)
   - **Comparison tables:** Bảng so sánh nếu cần
   - **Resource cards:** Link tài nguyên tham khảo
3. **Nguyên tắc thiết kế:**
   - Hướng đến người mới — ngôn ngữ dễ hiểu, ví dụ cụ thể
   - Tạo hứng thú — mỗi section ≥1 thành phần trực quan
   - Responsive — hoạt động tốt trên mobile
   - Tiếng Việt đầy đủ dấu
4. **Footer:** Luôn ghi `Tổng hợp & biên soạn bởi Hưng 2x` kèm nguồn tham khảo và tháng/năm cập nhật.

**Output:** Updated `.md` file (với hình ảnh) + created/updated `.html` file + image assets (nếu cần)

---

### Bước 4 — Kiểm Tra & Sửa Lỗi

**Mục tiêu:** Đảm bảo chất lượng cả 2 tài liệu trước khi deploy.

**Thời gian:** 30–45 phút

1. **Kiểm tra Markdown file:**
   - Chính xác: Thông tin, số liệu, thuật ngữ có đúng?
   - Ngữ nghĩa: Câu rõ ràng, mạch lạc, không mơ hồ?
   - Tiếng Việt: Đầy đủ dấu, không lẫn không dấu?
   - Nguồn: Tất cả claims quan trọng có chú thích nguồn?
   - Hình ảnh MD: Mỗi section có ảnh? Ảnh có alt text, figure title, chú thích? File ảnh tồn tại?

2. **Kiểm tra HTML file:**
   - W3C validation: Không có syntax error, lỗi encoding
   - SVG text: Các ký tự tiếng Việt có đầy đủ dấu?
   - Responsive: Hoạt động tốt trên mobile, tablet, desktop
   - Links: Tất cả link tham khảo có hoạt động?
   - Render: HTML hiển thị đúng, không lỗi layout?
   - Tiếng Việt: Đầy đủ dấu, không mất ký tự

3. **Sửa lỗi:**
   - Fix tất cả lỗi tìm được (encoding, syntax, typo, dấu)
   - Test lại trên 3 device size (mobile, tablet, desktop)
   - Commit with semantic message (e.g., `feat: enhance 02-websocket content`, `fix: escape ampersands in HTML`)

4. **Cập nhật README (BẮT BUỘC cả 2 file):**

   Repo có **2 README** cần đồng bộ — KHÔNG được chỉ update 1 file:
   - `README.md` (root) — landing page repo
   - `docs/README.md` — TOC thư mục tài liệu

   **URL pattern bắt buộc — đây là lỗi rất hay gặp:**

   | Loại file | Dùng link gì |
   |-----------|--------------|
   | Markdown (`.md`) | **Relative path** — `./docs/foo/foo.md` |
   | HTML visual / Cheatsheet HTML | **GitHub Pages URL** — `https://hungld10.github.io/share-everything-about-ai/docs/foo/foo-visual.html` |
   | Cheatsheet `.md`/`.txt` | Relative path |

   ⚠️ **Tại sao:** Relative `.html` paths trên github.com sẽ render **raw HTML source code**, không phải trang đã render. Phải dùng GitHub Pages URL để link mở đúng trang visual.

   **Format chuẩn 1 entry:**

   ```markdown
   ### 🔖 Tên section
   - [Tên bài](./docs/foo/foo.md) · [🎨 Visual](https://hungld10.github.io/share-everything-about-ai/docs/foo/foo-visual.html) · [📥 Cheatsheet](https://hungld10.github.io/share-everything-about-ai/docs/foo/cheatsheet/cheatsheet.html)
   ```

   Trong `docs/README.md`, prefix `./docs/` được bỏ (vì đang ở trong `docs/` rồi):

   ```markdown
   - [Tên bài](./foo/foo.md) · [🎨 Visual](https://hungld10.github.io/share-everything-about-ai/docs/foo/foo-visual.html) · ...
   ```

   GitHub Pages URL luôn full path từ `https://hungld10.github.io/share-everything-about-ai/docs/...` bất kể đang ở file README nào.

5. **Đặt tên file** (nếu là file mới):
   - Markdown: `docs/<topic-folder>/<topic-name>.md`
   - HTML: `docs/<topic-folder>/<topic-name>-visual.html`
   - Images: `docs/<topic-folder>/images/`
   - Tên dùng kebab-case, tiếng Anh

6. **Deploy:**
   - Commit tất cả file cùng với semantic message
   - Push lên GitHub
   - Verify trên web

**Output:** Clean, deployed files (MD + HTML + image assets)

## Gotchas

### Bước 1 & 2 — Scope Control ⚠️ (Lỗi phổ biến nhất)

- **Scope creep trong series:** Khi viết bài thuộc series, rất dễ "tiện tay" thêm nội dung của bài khác vào. Ví dụ thực tế: bài WebSocket (Tầng 1) đã bị nhồi thêm Failure Modes (Tầng 4), Latency Budget (Bài 06), Redis Catastrophe (Bài 12), Protocol Comparison (Bài 07). Kết quả: 1,600 dòng thay vì 300 dòng, reader bị overwhelmed.
- **Triệu chứng scope creep:** Bài vượt quá target length; Có section "sâu hơn cần thiết" cho tầng đó; Có content preview bài sau nhưng bị expand thành full section; Có nhiều hơn 1 visualization cho cùng 1 concept.
- **Kiểm tra nhanh:** Hỏi "Nếu tôi cắt section này, reader có hiểu được topic chính của bài không?" — nếu Yes → có thể cắt.
- **Deep dive vs. surface mention:** Nội dung của bài sau chỉ được **nhắc ngắn** (1–2 câu) kèm "Chi tiết ở Bài XX". Không expand thành subsection đầy đủ.

### Bước 2 — Không nhúng HTML vào file .md ⚠️

- **KHÔNG nhúng `<div>`, `<style>`, `<script>` vào file .md.** HTML embedded trong Markdown chỉ render đúng trên web. Trên GitHub, Obsidian, VS Code preview → reader thấy raw HTML code, không đọc được nội dung.
- **Thay thế HTML components bằng:**
  - ASCII diagrams cho flow/architecture: ``` ─ ═ → ↓ ║ ▼ ```
  - Markdown tables cho comparison
  - Code blocks với ví dụ thực tế
  - Blockquotes (`>`) cho highlight/warning
- **HTML components chỉ được phép trong file `.html`** — không bao giờ trong file `.md`

### Bước 2 — Redundancy (Trình bày cùng concept nhiều lần)

- **Mỗi concept chỉ được trình bày MỘT lần** trong file .md, ở vị trí phù hợp nhất. Nếu cần nhắc lại ở section khác → dùng link hoặc "như đã đề cập ở trên."
- **Dấu hiệu redundancy:** Cùng một diagram/table xuất hiện 2–3 lần theo dạng khác; text giải thích và visual giải thích cùng concept nhưng trùng lặp hoàn toàn.
- **Rule of thumb:** 1 concept → 1 visual đơn giản nhất (ASCII hoặc table). Visual phức tạp hơn → dành cho file `.html`.

### Bước 3 (Visual Elements)

- **Tiếng Việt không dấu trong HTML:** Khi viết file HTML lớn, rất dễ bị mất dấu tiếng Việt. Luôn kiểm tra lại toàn bộ text trong HTML có đầy đủ dấu.
- **SVG text encoding:** Text tiếng Việt có dấu trong SVG element `<text>` cần đảm bảo file được lưu UTF-8. Nếu ký tự đặc biệt bị lỗi, dùng HTML entities.
- **File quá lớn:** Nếu HTML vượt 1500 dòng, cân nhắc chia thành các section có thể collapse hoặc tách file.
- **Hình ảnh trong MD vs HTML:** File .md dùng ảnh PNG/JPG từ thư mục `images/` (tạo bằng AI). File HTML dùng inline SVG diagrams (vẽ trực tiếp). Hai loại KHÁC nhau — MD cần ảnh AI đẹp để minh hoạ, HTML cần SVG để giải thích chi tiết.
- **Thiếu ảnh:** Nếu tool tạo ảnh AI không khả dụng, PHẢI để placeholder rõ ràng và thông báo cho user.

### Bước 4 (QA & Deployment)

- **Copy-paste nguồn:** Không copy nguyên văn từ nguồn. Tổng hợp, biên dịch và viết lại bằng ngôn ngữ của mình.
- **Đồng bộ 2 file:** Sau khi sửa 1 file (MD/HTML), luôn kiểm tra file còn lại có cần update theo không.
- **README format:** Gộp MD và Visual vào 1 dòng bằng ký tự `·` ngăn cách, không để 2 dòng riêng.
- **Git commit messages:** Dùng semantic messages: `feat:`, `fix:`, `docs:`, `enhance:`, `refactor:`
  - `feat: add WebSocket fundamental article`
  - `enhance: improve latency discussion with production examples`
  - `fix: escape ampersands in HTML for W3C validation`
  - `refactor: remove scope creep from article 02, trim to 276 lines`

### Bước 4 — Link URL pattern ⚠️ (Lỗi đã xảy ra nhiều lần)

- **Quên 1 trong 2 README:** Skill ban đầu chỉ ghi "cập nhật `docs/README.md`, root README nếu cần" — kết quả thực tế là `README.md` ở root bị bỏ sót. **Luôn update CẢ 2 file** trong cùng commit.
- **Relative path cho `.html` = LINK SAI:** Click vào `./docs/foo/foo-visual.html` trên github.com sẽ render raw source code, người dùng thấy code HTML không phải trang visual. **Bắt buộc dùng full GitHub Pages URL** `https://hungld10.github.io/share-everything-about-ai/docs/foo/foo-visual.html` cho mọi link HTML.
- **Mixed pattern:** `.md` dùng relative, `.html` dùng absolute GH Pages URL — đây là pattern chuẩn của repo này, không phải bug. Lý do: github.com render `.md` đẹp nên relative tốt; còn `.html` thì github.com show raw, github.io mới render.
- **Quick check sau khi thêm entry README:**
  - Mở README trên github.com → click thử Visual link → có hiện trang đã render không?
  - Click thử Cheatsheet link → có thấy nút "🖨️ In / Lưu PDF" không?
  - Nếu thấy raw HTML code → URL đang sai pattern.
