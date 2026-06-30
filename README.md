# os-agent-kit

Bộ agent config và workflow templates được extract từ dự án **B2B Login to Access**, dùng để bootstrap AI agent cho project mới trong team BSS.

Hỗ trợ đa nền tảng: **Claude Code** · **Gemini CLI** · **Codex**

---

## Mục đích

Member clone repo này để lấy toàn bộ cấu hình AI agent, docs tham chiếu (`docs/`), templates workflow (`templates/`), và git hooks (`hooks/`) — không cần setup lại từ đầu.

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

| AI Tool | Entry point | Agent files |
|---------|-------------|-------------|
| Claude Code | [`CLAUDE.md`](CLAUDE.md) | `.claude/agents/` |
| Gemini CLI | [`GEMINI.md`](GEMINI.md) | `.gemini/agents/` |
| Codex | [`AGENTS.md`](AGENTS.md) | `.codex/agents/` |

Sau đó đọc [`docs/README.md`](docs/README.md) — index toàn bộ business & technical docs.

### 4. Copy config vào project mới

Khi bắt đầu một project mới, copy thư mục tương ứng với tool đang dùng:

**Claude Code:**
```bash
cp -r os-agent-kit/.claude/ <your-project>/
cp os-agent-kit/CLAUDE.md <your-project>/
```

**Gemini CLI:**
```bash
cp -r os-agent-kit/.gemini/ <your-project>/
cp os-agent-kit/GEMINI.md <your-project>/
```

**Codex:**
```bash
cp -r os-agent-kit/.codex/ <your-project>/
cp os-agent-kit/AGENTS.md <your-project>/
```

Sau đó cập nhật `.claude/CLAUDE.md` (hoặc entry point tương ứng) theo đúng kiến trúc của project mới.

---

## Cấu trúc thư mục

```
os-agent-kit/
├── .claude/           ← Claude Code: CLAUDE.md, agents/ (frontmatter), skills/
│   ├── CLAUDE.md
│   ├── agents/
│   │   ├── coder.md
│   │   ├── planner.md
│   │   └── reviewer.md
│   └── skills/
│       └── coding-convention.md
├── .gemini/           ← Gemini CLI: agents/ (native format, save_memory)
│   └── agents/
│       ├── coder.md
│       ├── planner.md
│       └── reviewer.md
├── .codex/            ← Codex: agents/ (native format, update_plan/spawn_agent)
│   └── agents/
│       ├── coder.md
│       ├── planner.md
│       └── reviewer.md
├── docs/              ← Business & technical documentation
│   ├── business/      ← User-facing features và business logic
│   ├── technical/     ← Architecture và technical specs
│   └── plans/         ← Feature plans đã viết sẵn
├── templates/         ← Templates cho feature workflow
├── hooks/             ← Git hooks (cài bằng core.hooksPath)
├── CLAUDE.md          ← Entry point cho Claude Code
├── GEMINI.md          ← Entry point cho Gemini CLI (@include .claude/CLAUDE.md)
├── AGENTS.md          ← Entry point cho Codex
└── TEAM_STATUS.md     ← Trạng thái làm việc của team
```

---

## So sánh format agent files theo tool

| | Claude Code | Gemini CLI | Codex |
|-|-------------|------------|-------|
| **Directory** | `.claude/agents/` | `.gemini/agents/` | `.codex/agents/` |
| **Frontmatter** | Có (`name`, `model`, `color`, `memory`) | Không | Không |
| **Memory** | Tự động qua `memory:` field | `save_memory(key, value)` | File-based thủ công |
| **Task tracking** | `TodoWrite` | `write_todos` | `update_plan` |
| **Subagents** | `Agent` tool | `@generalist` | `spawn_agent` / `wait_agent` |
| **File read** | `Read` | `read_file` | Native file tools |

---

## Links quan trọng

| File | Mô tả |
|------|-------|
| [`CLAUDE.md`](CLAUDE.md) | Entry point cho Claude Code |
| [`GEMINI.md`](GEMINI.md) | Entry point cho Gemini CLI |
| [`AGENTS.md`](AGENTS.md) | Entry point cho Codex |
| [`.claude/CLAUDE.md`](.claude/CLAUDE.md) | Project context — source of truth |
| [`TEAM_STATUS.md`](TEAM_STATUS.md) | Ai đang làm gì |
| [`hooks/README.md`](hooks/README.md) | Cách cài git hooks |
