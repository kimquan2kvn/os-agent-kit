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

**Không được** dùng `grep_search` hay `read_file` để tìm kiếm khi Codegraph đã có thể trả lời. Chỉ dùng `read_file` để xác nhận chi tiết cụ thể mà Codegraph không cover.

## Coding Conventions — Tuân thủ .claude/skills/coding-convention.md

Đọc file `.claude/skills/coding-convention.md` bằng `read_file` trước khi bắt đầu viết code. Áp dụng toàn bộ quy tắc cho mọi đoạn code được viết hoặc sửa:

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

## Gemini CLI Tools

Dùng các tool sau của Gemini CLI khi làm việc:

| Mục đích | Tool |
|----------|------|
| Đọc file | `read_file` |
| Tạo file mới | `write_file` |
| Sửa nội dung file | `replace` |
| Chạy shell command | `run_shell_command` |
| Tìm kiếm trong file | `grep_search` |
| Tìm file theo pattern | `glob` |
| Liệt kê thư mục | `list_directory` |
| Track tasks | `write_todos` |

## Memory

Dùng `save_memory` để lưu lại các phát hiện quan trọng trong quá trình code:
- Pattern mới tìm thấy trong codebase
- Quyết định kỹ thuật và lý do
- Convention đặc thù của module đang làm việc

Format: `save_memory(key="coder/<topic>", value="<nội dung ngắn gọn>")`
