---
name: create-visual-doc
description: "Tạo bộ tài liệu hoàn chỉnh gồm bản Markdown chi tiết và bản HTML trực quan từ một chủ đề cho trước. Kích hoạt khi user yêu cầu tạo tài liệu, viết guide, tạo hướng dẫn, hoặc document một chủ đề vào repo. Ví dụ: 'tạo tài liệu về RAG', 'viết guide về agent architecture', 'document cách dùng MCP servers'."
---

# Skill: Tạo Bộ Tài Liệu Trực Quan

Skill này tạo bộ tài liệu hoàn chỉnh (Markdown + HTML trực quan) cho repo `share-everything-about-ai`. Luôn viết bằng **tiếng Việt**. Tác giả: **Hưng 2x**.

## Quy trình 4 bước

### Bước 1 — Nghiên cứu & Tổng hợp tài liệu Markdown

**Mục tiêu:** Tạo file `.md` chi tiết, đầy đủ, chính xác.

1. **Phân tích chủ đề:** Xác định phạm vi, các khía cạnh cần bao quát, đối tượng người đọc.
2. **Tìm kiếm bổ sung:** Dùng web search để tìm:
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
4. **Chú thích nguồn:** Mỗi claim quan trọng cần ghi chú nguồn tham khảo `【[số](url)】`.
5. **Tạo hình ảnh minh hoạ cho file Markdown** — đây là bước bắt buộc, không bỏ qua:

   **a. Lên danh sách hình ảnh cần tạo:**
   - Mỗi section chính (h2) cần ít nhất 1 hình minh hoạ
   - Xác định loại hình phù hợp cho từng section:
     - **Sơ đồ kiến trúc / flow:** Cho phần kiến trúc, mô hình, framework
     - **Infographic tổng quan:** Cho phần giới thiệu, tổng quan
     - **So sánh trực quan:** Cho phần so sánh giải pháp
     - **Minh hoạ khái niệm:** Cho phần khái niệm trừu tượng
     - **Quy trình từng bước:** Cho phần hướng dẫn thực hành
     - **Bảng tổng hợp / checklist:** Cho phần best practices

   **b. Tạo hình ảnh bằng AI:**
   - Sử dụng tool tạo ảnh AI (image generation) để tạo từng ảnh
   - Prompt tạo ảnh nên mô tả rõ: nội dung cần thể hiện, phong cách (clean, professional, diagram-style), màu sắc (phù hợp design system của repo: tím #534AB7, xanh lá #1D9E75, xanh dương #378ADD)
   - Ưu tiên phong cách: flat illustration, clean diagram, modern infographic
   - Tránh: ảnh chứa quá nhiều text (khó đọc khi thu nhỏ), ảnh quá phức tạp

   **c. Lưu trữ và nhúng:**
   - Lưu ảnh vào `docs/<topic-folder>/images/` với tên: `fig-01-mo-ta.png`, `fig-02-mo-ta.png`...
   - Nhúng vào file .md với format:
     ```markdown
     ![Mô tả ngắn](./images/fig-01-mo-ta.png)
     *Figure 1 — Tiêu đề hình ảnh*
     *Chú thích mô tả nội dung hình ảnh thể hiện.*
     ```
   - Mỗi hình PHẢI có: alt text, tiêu đề figure, chú thích mô tả

   **d. Nếu không có tool tạo ảnh AI:**
   - Ghi rõ placeholder trong file .md: `<!-- TODO: Tạo ảnh minh hoạ cho section này -->`
   - Mô tả chi tiết ảnh cần tạo trong placeholder để user có thể tự tạo sau
   - Thông báo cho user biết những ảnh nào chưa được tạo

### Bước 2 — Tạo tài liệu HTML trực quan

**Mục tiêu:** Chuyển nội dung Markdown thành trang HTML đẹp, dễ đọc, hấp dẫn người mới.

1. **Sử dụng design system** của repo (xem `references/design-system.css` trong skill này).
2. **Bắt buộc có các thành phần trực quan:**
   - **Hero section:** Tiêu đề + subtitle + 3-4 thống kê nổi bật
   - **SVG diagrams:** Sơ đồ kiến trúc, flow, so sánh — vẽ bằng inline SVG
   - **Card grids:** Phân loại, danh mục dùng grid cards có icon + ví dụ
   - **Stepper/Timeline:** Quy trình nhiều bước dùng stepper hoặc timeline
   - **Good vs Bad boxes:** So sánh cách làm đúng/sai
   - **Highlight boxes:** Trích dẫn, insights quan trọng, tips
   - **Code blocks:** Ví dụ code với syntax highlighting (dùng class `code-key`, `code-str`, `code-comment`, `code-punct`)
   - **Comparison tables:** Bảng so sánh nếu cần
   - **Resource cards:** Link tài nguyên tham khảo
3. **Nguyên tắc thiết kế:**
   - Hướng đến người mới — ngôn ngữ dễ hiểu, ví dụ cụ thể
   - Tạo hứng thú — mỗi section phải có ít nhất 1 thành phần trực quan
   - Responsive — hoạt động tốt trên mobile
   - Tiếng Việt đầy đủ dấu
4. **Footer:** Luôn ghi `Tổng hợp & biên soạn bởi Hưng 2x` kèm nguồn tham khảo và tháng/năm cập nhật.

### Bước 3 — Đánh giá & Chỉnh sửa

**Mục tiêu:** Đảm bảo chất lượng cả 2 tài liệu.

Kiểm tra theo checklist:
- [ ] **Chính xác:** Thông tin, số liệu, thuật ngữ có đúng không?
- [ ] **Đồng bộ:** Nội dung MD và HTML có khớp nhau không? Không thiếu/thừa mục nào?
- [ ] **Ngữ nghĩa:** Câu văn rõ ràng, mạch lạc, không mơ hồ?
- [ ] **Tiếng Việt:** Đầy đủ dấu, không lẫn tiếng Việt không dấu?
- [ ] **Nguồn:** Tất cả claims quan trọng đều có chú thích nguồn?
- [ ] **Hình ảnh MD:** Mỗi section chính có ảnh minh hoạ? Ảnh có alt text, figure title, chú thích? File ảnh tồn tại trong `images/`?
- [ ] **SVG trong HTML:** Các sơ đồ SVG hiển thị đúng, text đủ dấu tiếng Việt?
- [ ] **HTML:** Render đúng, responsive hoạt động?
- [ ] **Links:** Các link tham khảo hoạt động?

Sửa mọi lỗi tìm được trước khi sang bước 4.

### Bước 4 — Đặt tên & Cập nhật README

1. **Đặt tên file** theo convention:
   - Markdown: `docs/<topic-folder>/<topic-name>.md`
   - HTML: `docs/<topic-folder>/<topic-name>-visual.html`
   - Images: `docs/<topic-folder>/images/`
   - Tên folder/file dùng kebab-case, tiếng Anh

2. **Cập nhật `docs/README.md`:** Thêm entry mới vào danh sách tài liệu.

3. **Cập nhật `README.md` (root):** Thêm entry mới vào section phù hợp.
   - Format: `- [Tên tài liệu](path/to/file.md) · [🎨 Visual](path/to/file-visual.html)`

## Gotchas

- **Tiếng Việt không dấu trong HTML:** Khi viết file HTML lớn, rất dễ bị mất dấu tiếng Việt. Luôn kiểm tra lại toàn bộ text trong HTML có đầy đủ dấu.
- **SVG text encoding:** Text tiếng Việt có dấu trong SVG element `<text>` cần đảm bảo file được lưu UTF-8. Nếu ký tự đặc biệt bị lỗi, dùng HTML entities.
- **File quá lớn:** Nếu HTML vượt 1500 dòng, cân nhắc chia thành các section có thể collapse hoặc tách file.
- **Copy-paste nguồn:** Không copy nguyên văn từ nguồn. Tổng hợp, biên dịch và viết lại bằng ngôn ngữ của mình.
- **Đồng bộ 2 file:** Sau khi sửa 1 file, luôn kiểm tra file còn lại có cần update theo không.
- **README format:** Gộp MD và Visual vào 1 dòng bằng ký tự `·` ngăn cách, không để 2 dòng riêng.
- **Hình ảnh trong MD vs HTML:** File .md dùng ảnh PNG/JPG từ thư mục `images/` (tạo bằng AI). File HTML dùng inline SVG diagrams (vẽ trực tiếp). Hai loại hình ảnh này KHÁC nhau và phục vụ mục đích khác — MD cần ảnh AI đẹp để minh hoạ, HTML cần SVG tương tác để giải thích chi tiết.
- **Thiếu ảnh:** Nếu tool tạo ảnh AI không khả dụng, PHẢI để placeholder rõ ràng và thông báo cho user. Không được bỏ qua ảnh mà không nói gì.
