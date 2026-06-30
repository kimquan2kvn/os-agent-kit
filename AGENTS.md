# os-agent-kit — Codex Project Context

Repo này là bộ agent config được extract từ dự án **B2B Login to Access**, dùng để bootstrap AI agent cho project mới trong team BSS.

---

## Đọc theo thứ tự này trước khi làm bất cứ việc gì

1. `.claude/CLAUDE.md` — kiến trúc dự án, module map, hard rules
2. `docs/README.md` — index toàn bộ business & technical docs
3. `docs/technical/be/` — technical docs cho backend features
4. `TEAM_STATUS.md` — ai đang làm gì, branch nào

---

## Agents

Agent files cho Codex nằm trong `.codex/agents/`. Đọc file tương ứng và dùng nội dung làm prompt khi `spawn_agent`:

| Agent | File | Khi nào dùng |
|-------|------|-------------|
| `coder` | `.codex/agents/coder.md` | Viết code, follow coding conventions |
| `planner` | `.codex/agents/planner.md` | Lập kế hoạch implementation trước khi code |
| `reviewer` | `.codex/agents/reviewer.md` | Review code hoặc MR |

Bật multi-agent: thêm `[features] multi_agent = true` vào `~/.codex/config.toml`

---

## Cấu trúc repo

```
os-agent-kit/
├── .claude/           ← Claude Code: CLAUDE.md, agents/, skills/
├── .gemini/           ← Gemini CLI: agents/
├── .codex/            ← Codex: agents/
├── docs/              ← Business & technical documentation
├── templates/         ← Templates cho feature workflow
├── hooks/             ← Git hooks
├── AGENTS.md          ← File này (entry point cho Codex)
├── GEMINI.md          ← Entry point cho Gemini CLI
├── CLAUDE.md          ← Entry point cho Claude Code
└── TEAM_STATUS.md
```

---

## Multi-dev protocol

- KHÔNG sử dụng agent dev khi feature-context.md chưa được Dev Lead approve
- LUÔN đọc feature-context.md của feature hiện tại trước khi bắt đầu code
- KHÔNG sửa file ngoài danh sách "Files sẽ bị ảnh hưởng" trong feature-context.md
- Cập nhật TEAM_STATUS.md mỗi khi bắt đầu hoặc kết thúc session làm việc

---

## Hard Rules (từ .claude/CLAUDE.md)

1. KHÔNG thêm business logic vào `login-api` (v1) — chỉ `login-api-v2`
2. KHÔNG sửa `upload.service.ts` mà không hiểu flow upload
3. KHÔNG trực tiếp gọi Shopify REST trong Controller
4. Khi sửa Hide Section/Block → đọc `docs/technical/be/hide-section.md` trước
5. Mọi entity mới đều cần migration
6. Auth guard là global — route public dùng `@Public()` hoặc thêm vào `skipShopGuard`
