# Share Everything About AI

## Mô tả
Kho lưu trữ tổng hợp kiến thức AI — học được, thử nghiệm, và hệ thống hoá.
Mục tiêu: xây dựng "AI Knowledge OS" cá nhân và team.

## Nội dung chính

- `docs/voice-ai-architecture/` — Production Voice AI pipeline (4 architectures, audio engineering)
- `docs/skill-system/` — Skill System design + visual
- `docs/claude-code-skills-*` — Claude Code Agent Skills hướng dẫn
- `docs/claude-personal-preferences/` — Cấu hình hành vi Claude theo ý muốn
- `docs/ai-bash-commands-guide/` — AI Agent Security & bash commands cheatsheet
- AI Workflow Design, Prompt vs Skill vs Agent

## Skills

- list: liệt kê tất cả docs có trong repo
- open: mở thư mục trong Finder

---

## 📌 Quy ước viết tài liệu

### URL & Links

Repo được publish qua **GitHub Pages** tại base URL:

```
https://hungld10.github.io/share-everything-about-ai/
```

**Quy tắc link trong README (root và `docs/README.md`):**

| Loại file | Dùng link gì | Tại sao |
|-----------|--------------|---------|
| Markdown (`.md`) | **Relative path** — `./docs/foo/foo.md` | GitHub.com render markdown đẹp, không cần GH Pages |
| HTML visual (`.html`) | **GitHub Pages URL** — `https://hungld10.github.io/share-everything-about-ai/docs/foo/foo-visual.html` | Relative `.html` sẽ render raw source code trên github.com — phải dùng GH Pages để xem trang đã render |
| Cheatsheet (`.html`) | **GitHub Pages URL** — same as above | Tương tự |
| Cheatsheet (`.md`/`.txt`) | Relative path | OK trên GitHub.com |
| Image (`.png`/`.jpg`) | Relative path | GitHub.com hiển thị inline đúng |

**Format chuẩn cho 1 entry README:**

```markdown
- [Tên bài](./docs/foo/foo.md) · [🎨 Visual](https://hungld10.github.io/share-everything-about-ai/docs/foo/foo-visual.html) · [📥 Cheatsheet](https://hungld10.github.io/share-everything-about-ai/docs/foo/cheatsheet/cheatsheet.html)
```

### Đồng bộ 2 README

Repo có **2 README** cần giữ đồng bộ khi thêm/sửa bài:

1. `README.md` (root) — landing page repo
2. `docs/README.md` — TOC chi tiết của thư mục tài liệu

Khi thêm bài mới, **luôn cập nhật cả 2 file** với cùng format link.

### Tác giả & Footer

- Mọi bài viết ghi tác giả: **Hưng 2x**
- HTML visual + cheatsheet có footer: `Tổng hợp & biên soạn bởi Hưng 2x · <tháng>/<năm>`
- Luôn viết bằng **tiếng Việt** (đầy đủ dấu)

### Favicon, Logo & Brand assets

Toàn bộ tài sản brand đặt ở thư mục **`/icons/` tại root repo** (đồng bộ với trang chủ [hungledn.com](https://hungledn.com/)):

- Favicon: `favicon.ico`, `favicon-16x16.png`, `favicon-32x32.png`, `apple-touch-icon.png`, `android-chrome-192/512.png`, `mstile-150x150.png`, `safari-pinned-tab.svg`
- Cấu hình: `site.webmanifest`, `browserconfig.xml` (dùng path tương đối bare → resolve trong `icons/`)
- Logo: `logo.png` (xanh sáng), `logo-dark.png` (xanh đậm) — 1200×1200

**Quy tắc nhúng trong file HTML:**

| Mục đích | Path | Tại sao |
|----------|------|---------|
| Favicon / manifest / mask-icon | **Relative theo độ sâu** — `../icons/`, `../../icons/`, `../../../icons/` | Repo chạy GH Pages ở **subpath** `/share-everything-about-ai/` → path tuyệt đối `/icons/` sẽ trỏ sai. Relative chạy đúng cả GH Pages, local, lẫn khi đổi tên repo |
| `og:image` / `twitter:image` | **URL tuyệt đối** — `https://hungld10.github.io/share-everything-about-ai/icons/logo.png` | Social scraper (FB/X/Zalo) bắt buộc URL tuyệt đối |
| Logo on-page (góc hero) | **Relative** — `<a>` + `<img>` inline-style `position:absolute` ở đầu `<body>` | Neo vào góc trên-trái trang, không phụ thuộc CSS hero |

- `twitter:card` = **`summary`** (không phải `summary_large_image`) vì logo là ảnh **vuông** → card vuông hiển thị trọn, không bị crop.
- Màu brand chrome: `theme-color` = `#0d1117` (nền dark docs), `TileColor`/`mask-icon color` = `#2f6b1f` (xanh logo).
- **Bài mới:** copy `<head>` từ `.claude/skills/create-visual-doc/examples/template-skeleton.html` (đã có sẵn đủ block này) và điền `og:title`/`og:url`/`og:description` cho đúng bài.
