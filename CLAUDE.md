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
