# os-agent-kit

Bộ agent config và workflow templates được extract từ dự án **B2B Login to Access**, dùng để bootstrap Claude Code cho project mới trong team BSS.

---

## Mục đích

Member clone repo này để lấy toàn bộ cấu hình AI agent (`.claude/`), docs tham chiếu (`docs/`), templates workflow (`templates/`), và git hooks (`hooks/`) — không cần setup lại từ đầu.

---

## Hướng dẫn cho member mới

### 1. Clone repo

```bash
git clone <repo-url> os-agent-kit
cd os-agent-kit
```

### 2. Kích hoạt git hooks

```bash
git config core.hooksPath hooks
```

### 3. Đọc tài liệu theo thứ tự

1. [`CLAUDE.md`](CLAUDE.md) — entry point, cách dùng repo này
2. [`.claude/CLAUDE.md`](.claude/CLAUDE.md) — kiến trúc dự án và hard rules
3. [`docs/README.md`](docs/README.md) — index toàn bộ docs

### 4. Copy `.claude/` vào project mới

Khi bắt đầu một project mới, copy toàn bộ `.claude/` vào root của project đó:

```bash
cp -r os-agent-kit/.claude/ <your-project>/
```

Sau đó cập nhật `.claude/CLAUDE.md` theo đúng kiến trúc của project mới.

---

## Cấu trúc thư mục

```
os-agent-kit/
├── .claude/           ← Agent config: CLAUDE.md, agents, skills
├── docs/              ← Business & technical documentation
│   ├── business/      ← User-facing features và business logic
│   ├── technical/     ← Architecture và technical specs
│   └── plans/         ← Feature plans đã viết sẵn
├── templates/         ← Templates cho feature workflow
├── hooks/             ← Git hooks (cài bằng core.hooksPath)
├── CLAUDE.md          ← Entry point cho Claude Code
└── TEAM_STATUS.md     ← Trạng thái làm việc của team
```

---

## Links quan trọng

- [`CLAUDE.md`](CLAUDE.md) — hướng dẫn sử dụng repo
- [`TEAM_STATUS.md`](TEAM_STATUS.md) — ai đang làm gì
- [`.claude/CLAUDE.md`](.claude/CLAUDE.md) — project context cho AI
- [`hooks/README.md`](hooks/README.md) — cách cài git hooks
