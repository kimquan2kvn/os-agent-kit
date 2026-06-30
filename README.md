# os-agent-kit

Bộ agent config và workflow templates được extract từ dự án **B2B Login to Access**, dùng để bootstrap AI agent cho project mới trong team BSS.

Hỗ trợ đa nền tảng: **Claude Code** · **Gemini CLI** · **Codex**

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

| AI Tool | File cần đọc đầu tiên |
|---------|----------------------|
| Claude Code | [`CLAUDE.md`](CLAUDE.md) → [`.claude/CLAUDE.md`](.claude/CLAUDE.md) |
| Gemini CLI | [`GEMINI.md`](GEMINI.md) (tự động include `.claude/CLAUDE.md`) |
| Codex | [`AGENTS.md`](AGENTS.md) → [`references/codex-tools.md`](references/codex-tools.md) |

Sau đó đọc [`docs/README.md`](docs/README.md) — index toàn bộ business & technical docs.

### 4. Copy config vào project mới

Khi bắt đầu một project mới, copy các file cần thiết theo tool đang dùng:

**Claude Code:**
```bash
cp -r os-agent-kit/.claude/ <your-project>/
```

**Gemini CLI:**
```bash
cp -r os-agent-kit/.claude/ <your-project>/
cp os-agent-kit/GEMINI.md <your-project>/
cp -r os-agent-kit/references/ <your-project>/
```

**Codex:**
```bash
cp -r os-agent-kit/.claude/ <your-project>/
cp os-agent-kit/AGENTS.md <your-project>/
cp -r os-agent-kit/references/ <your-project>/
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
├── references/        ← Tool mapping cho từng AI platform
│   ├── gemini-tools.md   ← Claude Code → Gemini CLI tool names
│   └── codex-tools.md    ← Claude Code → Codex tool names
├── templates/         ← Templates cho feature workflow
├── hooks/             ← Git hooks (cài bằng core.hooksPath)
├── CLAUDE.md          ← Entry point cho Claude Code
├── GEMINI.md          ← Entry point cho Gemini CLI
├── AGENTS.md          ← Entry point cho Codex
└── TEAM_STATUS.md     ← Trạng thái làm việc của team
```

---

## Links quan trọng

| File | Mô tả |
|------|-------|
| [`CLAUDE.md`](CLAUDE.md) | Entry point cho Claude Code |
| [`GEMINI.md`](GEMINI.md) | Entry point cho Gemini CLI |
| [`AGENTS.md`](AGENTS.md) | Entry point cho Codex |
| [`.claude/CLAUDE.md`](.claude/CLAUDE.md) | Project context cho AI (source of truth) |
| [`references/gemini-tools.md`](references/gemini-tools.md) | Tool mapping cho Gemini CLI |
| [`references/codex-tools.md`](references/codex-tools.md) | Tool mapping cho Codex |
| [`TEAM_STATUS.md`](TEAM_STATUS.md) | Ai đang làm gì |
| [`hooks/README.md`](hooks/README.md) | Cách cài git hooks |
