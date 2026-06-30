# Codex Tool Mapping

Skills trong repo này dùng tên tool của Claude Code. Khi gặp các tên đó, dùng equivalent của Codex:

| Claude Code | Codex equivalent |
|-------------|-----------------|
| `Agent` tool (dispatch subagent) | `spawn_agent` |
| Multiple `Agent` calls (parallel) | Multiple `spawn_agent` calls |
| Agent trả về kết quả | `wait_agent` |
| Agent hoàn thành | `close_agent` |
| `TodoWrite` (task tracking) | `update_plan` |
| `Skill` tool (invoke a skill) | Skills load natively — follow instructions directly |
| `Read`, `Write`, `Edit` | Dùng native file tools của Codex |
| `Bash` | Dùng native shell tools của Codex |

## Bật multi-agent support

Để dùng `spawn_agent`, `wait_agent`, `close_agent`, thêm vào `~/.codex/config.toml`:

```toml
[features]
multi_agent = true
```

## Agents trong Codex

Codex không dùng `.claude/agents/` directory. Các file trong đó có phần frontmatter YAML ở đầu (giữa hai dòng `---`) — đây là metadata dành riêng cho Claude Code.

Khi dùng làm prompt cho `spawn_agent`, **chỉ lấy nội dung bên dưới dòng `---` thứ hai**:

```
# ví dụ: .claude/agents/planner.md
---                          ← BỎ QUA từ đây
name: "planner"
model: sonnet
...
---                          ← đến đây (không bao gồm)
                             ↓ Lấy từ đây trở xuống làm prompt
# Planning Agent
You create plans. You do NOT write code.
...
```

| Khi muốn dùng agent | Codex equivalent |
|---------------------|-----------------|
| `coder` agent | `spawn_agent` với prompt = body của `.claude/agents/coder.md` |
| `planner` agent | `spawn_agent` với prompt = body của `.claude/agents/planner.md` |
| `reviewer` agent | `spawn_agent` với prompt = body của `.claude/agents/reviewer.md` |

## Environment Detection

Skills liên quan đến worktrees hoặc finishing branches nên detect môi trường trước:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → đang trong một linked worktree (bỏ qua bước tạo worktree)
- `BRANCH` trống → detached HEAD (không thể branch/push/PR từ sandbox)
