# Coder Agent

ALWAYS use the context7 MCP Server to read relevant documentation. Do this every time you are working with a language, framework, library etc. Never assume that you know the answer as these things change frequently. Your training date is in the past so your knowledge is likely out of date, even if it is a technology you are familiar with.

## Code Navigation — Bắt buộc dùng Codegraph

Trước khi đọc hay sửa bất kỳ file nào, **bắt buộc** dùng Codegraph MCP để định vị symbol, file và luồng gọi:

- Tìm symbol theo tên → `codegraph_search`
- Hiểu một feature/khu vực code → `codegraph_context` (gọi trước tiên)
- Truy vết luồng từ X đến Y → `codegraph_trace`
- Xem ai gọi hàm này → `codegraph_callers`
- Xem hàm này gọi gì → `codegraph_callees`
- Đánh giá impact khi sửa → `codegraph_impact`
- Xem source/signature của symbol → `codegraph_node`
- Khám phá nhiều symbol liên quan → `codegraph_explore`
- Liệt kê file trong thư mục → `codegraph_files`

**Không được** dùng shell search hay file read để tìm kiếm khi Codegraph đã có thể trả lời.

## Coding Conventions — Tuân thủ .claude/skills/coding-convention.md

Đọc file `.claude/skills/coding-convention.md` trước khi bắt đầu viết code. Áp dụng toàn bộ quy tắc cho mọi đoạn code được viết hoặc sửa:

**Core Principles:**
- KISS: đặt tên rõ ràng, ưu tiên `const` hơn `let`, tránh `var`
- Fail Fast: validate input sớm, return/throw ngay, tránh nested sâu
- SRP: mỗi function/component/file chỉ làm một việc
- DIP: module cấp cao phụ thuộc abstraction, không phụ thuộc implementation cụ thể
- Không over-engineering

**Frontend (React/TypeScript):**
- Component → PascalCase, hook → `useCamelCase`, handler → `handleCamelCase`, boolean → `is/has/should`
- Business logic vào custom hooks hoặc services, không để trong component
- Chỉ dùng `useMemo`/`useCallback` khi thực sự cần
- Tuân thủ Shopify Polaris, tránh custom styling trừ khi Polaris không hỗ trợ
- Không `import * as`, dùng lazy-load cho component nặng

**Backend (NestJS/Node.js):**
- Folder → kebab-case, variable → camelCase, class → PascalCase
- Controller-service pattern: controller xử lý routing + validation, service xử lý business logic
- Không đặt DB query trong controller
- Tránh N+1 query — dùng eager loading / join thay vì loop
- Shopify API: dùng GraphQL thay REST, dùng `@bss-sbc/shopify-api-fetcher` cho throttling

**Comments:**
- Chỉ comment khi giải thích business rule, hạn chế kỹ thuật, hoặc workaround
- Không comment những gì tên hàm/biến đã nói rõ

## Mandatory Coding Principles

1. Structure — dùng layout nhất quán, group theo feature, entry point rõ ràng
2. Architecture — flat và explicit hơn abstraction, minimize coupling
3. Functions — control flow linear, tránh nested sâu, pass state explicitly
4. Naming — tên mô tả được mục đích, comment chỉ cho invariants và constraints
5. Logging — structured logs tại key boundaries, errors explicit và informative
6. Regenerability — mỗi file/module có thể rewrite độc lập mà không break hệ thống
7. Modifications — khi extend/refactor, follow existing patterns

## Task Tracking

Dùng `update_plan` để track tiến độ các task đang làm:

```
update_plan(steps=[
  {"id": "1", "title": "Đọc docs liên quan", "done": true},
  {"id": "2", "title": "Khám phá codebase qua codegraph", "done": false},
  {"id": "3", "title": "Implement feature", "done": false},
])
```

## Subagents

Nếu cần spawn subagent cho task độc lập (yêu cầu bật `[features] multi_agent = true` trong `~/.codex/config.toml`):

```
agent_id = spawn_agent(prompt="<task cụ thể>")
result = wait_agent(agent_id)
close_agent(agent_id)
```

## Environment Detection

Trước khi làm việc với git worktrees hoặc branches:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → đang trong linked worktree
- `BRANCH` trống → detached HEAD, không thể push/PR
