# Gemini CLI Tool Mapping

Skills trong repo này dùng tên tool của Claude Code. Khi gặp các tên đó, dùng equivalent của Gemini CLI:

| Claude Code | Gemini CLI equivalent |
|-------------|----------------------|
| `Read` (đọc file) | `read_file` |
| `Write` (tạo file) | `write_file` |
| `Edit` (sửa file) | `replace` |
| `Bash` (chạy lệnh) | `run_shell_command` |
| `Grep` (tìm trong file) | `grep_search` |
| `Glob` (tìm file theo tên) | `glob` |
| `TodoWrite` (task tracking) | `write_todos` |
| `Skill` tool (invoke a skill) | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Agent` tool (dispatch subagent) | `@agent-name` |

## Subagents trong Gemini CLI

Gemini CLI không dùng `.claude/agents/` directory. Thay vào đó dùng `@` syntax:

| Khi muốn dùng agent | Gemini CLI equivalent |
|---------------------|----------------------|
| `coder` agent | `@generalist` với prompt từ `.claude/agents/coder.md` |
| `planner` agent | `@generalist` với prompt từ `.claude/agents/planner.md` |
| `reviewer` agent | `@code-reviewer` hoặc `@generalist` với prompt từ `.claude/agents/reviewer.md` |

### Cách invoke agent trong Gemini

Các file trong `.claude/agents/` có phần frontmatter YAML ở đầu (giữa hai dòng `---`) — đây là metadata dành riêng cho Claude Code, Gemini CLI không cần phần này.

Khi dùng làm prompt cho `@generalist`, **chỉ lấy nội dung bên dưới dòng `---` thứ hai**:

```
# ví dụ: .claude/agents/coder.md
---                          ← BỎ QUA từ đây
name: "coder"
model: sonnet
...
---                          ← đến đây (không bao gồm)
                             ↓ Lấy từ đây trở xuống làm prompt
ALWAYS use #context7 MCP Server...
```

**Quy trình:**
1. Đọc `.claude/agents/<agent-name>.md`
2. Bỏ phần frontmatter (từ dòng `---` đầu đến dòng `---` thứ hai)
3. Gọi `@generalist` với phần nội dung còn lại làm prompt

### Parallel dispatch

Gemini hỗ trợ parallel subagents. Khi cần dispatch nhiều task độc lập, gọi nhiều `@generalist` cùng lúc trong một prompt.

## Gemini-specific tools

Các tool sau có trong Gemini CLI nhưng không có trong Claude Code:

| Tool | Purpose |
|------|---------|
| `list_directory` | Liệt kê files và subdirectories |
| `save_memory` | Lưu fact vào `GEMINI.md` qua các session |
| `ask_user` | Hỏi user structured input |
| `tracker_create_task` | Task management phong phú |
| `enter_plan_mode` / `exit_plan_mode` | Chuyển sang chế độ research trước khi sửa |
