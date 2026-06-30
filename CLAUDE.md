# os-agent-kit

Repo này là bộ agent config được extract từ dự án B2B Login to Access, dùng để bootstrap Claude Code cho project mới trong team. Member clone repo này để lấy `.claude/`, `docs/`, `templates/`, và `hooks/` — không cần setup lại từ đầu.

---

## Đọc theo thứ tự này trước khi làm bất cứ việc gì

1. `.claude/CLAUDE.md` — kiến trúc dự án, module map, hard rules
2. `docs/README.md` — index toàn bộ business & technical docs
3. `docs/technical/be/` — technical docs cho backend features
4. `TEAM_STATUS.md` — ai đang làm gì, branch nào

---

## Cách dùng repo này cho project mới

1. Clone repo về máy
2. Copy thư mục `.claude/` vào root của project mới
3. Mở `.claude/CLAUDE.md` và cập nhật phần "Overview", "Module Map", "Hard Rules" theo đúng project
4. Cập nhật paths trong `.claude/agents/reviewer.md` và `.claude/agents/planner.md` nếu cần
5. Chạy `git config core.hooksPath hooks` để kích hoạt git hooks

---

## Cấu trúc repo

```
os-agent-kit/
├── .claude/
│   ├── CLAUDE.md          ← Project context cho AI agent (cần cập nhật theo project)
│   ├── agents/            ← Subagent configs: coder, planner, reviewer
│   └── skills/            ← Coding convention và skills tái sử dụng
├── docs/
│   ├── README.md          ← Index toàn bộ docs
│   ├── business/          ← Business logic, user-facing features
│   ├── technical/         ← Technical architecture docs
│   └── plans/             ← Feature plans đã viết sẵn
├── templates/             ← Templates cho feature workflow
│   ├── feature-context.md ← "Bản hợp đồng" scope trước khi code
│   ├── test-scope.md      ← Phạm vi test cần Dev Lead approve
│   ├── test-report.md     ← Báo cáo kết quả test
│   └── pr-description.md  ← Template mô tả PR
├── hooks/
│   ├── prepare-commit-msg ← Auto prefix commit bằng tên feature
│   ├── pre-push           ← Chặn push thẳng lên main
│   └── README.md          ← Hướng dẫn cài hooks
├── CLAUDE.md              ← File này (entry point)
├── TEAM_STATUS.md         ← Cập nhật mỗi session làm việc
└── README.md              ← Hướng dẫn cho member mới
```

---

## Multi-dev protocol

- KHÔNG sử dụng agent dev khi feature-context.md chưa được Dev Lead approve
- LUÔN đọc feature-context.md của feature hiện tại trước khi bắt đầu code
- KHÔNG sửa file ngoài danh sách "Files sẽ bị ảnh hưởng" trong feature-context.md
- Cập nhật TEAM_STATUS.md mỗi khi bắt đầu hoặc kết thúc session làm việc

---

## Khi bắt đầu feature mới

1. Copy `templates/feature-context.md` vào branch mới
2. Điền đầy đủ các section (đặc biệt: Scope, Files sẽ bị ảnh hưởng, Out of scope)
3. Ping Dev Lead để review và approve
4. Chỉ bắt đầu code sau khi có approval
