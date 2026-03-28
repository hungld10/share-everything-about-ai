# Cấu trúc repo tham khảo

Khi tạo tài liệu mới, đặt file theo cấu trúc sau:

```
share-everything-about-ai/
├── README.md                              ← Cập nhật mục lục tổng
├── docs/
│   ├── README.md                          ← Cập nhật danh sách tài liệu
│   ├── claude-code-skills-huong-dan-truc-quan.html   ← Ví dụ: HTML standalone
│   ├── skill-system/
│   │   ├── skill-system.md                ← Ví dụ: tài liệu MD
│   │   ├── skill-system-visual.html       ← Ví dụ: tài liệu HTML visual
│   │   └── images/                        ← Ảnh minh hoạ
│   └── <topic-moi>/                       ← Thư mục cho chủ đề mới
│       ├── <topic-name>.md
│       ├── <topic-name>-visual.html
│       └── images/
└── .claude/
    └── skills/
        └── create-visual-doc/             ← Skill này
```

## Convention đặt tên

- Thư mục: `kebab-case`, tiếng Anh (ví dụ: `agent-architecture`, `rag-system`)
- File MD: `<topic-name>.md`
- File HTML: `<topic-name>-visual.html`
- Ảnh: `docs/<topic>/images/fig-01-description.png`

## Format entry trong README

Trong `README.md` (root) và `docs/README.md`, mỗi tài liệu ghi trên 1 dòng:

```markdown
- [Tên tài liệu](path/to/file.md) · [🎨 Visual](path/to/file-visual.html)
```
