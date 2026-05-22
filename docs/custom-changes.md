# Custom Changes

Ghi chép các thay đổi tùy chỉnh so với upstream `github.com/ArcReel/ArcReel/main`.
Khi pull upstream mới, review file này để biết cần re-apply gì.

---

## 2026-05-22 — Initial sync v0.15.0

### 1. Default locale: `zh` → `vi`
**File:** `lib/i18n/__init__.py:31`
```python
DEFAULT_LOCALE = "vi"  # upstream: "zh"
```
**Lý do:** Browser `Accept-Language` thường là `en-US`; nếu không match `vi/en/zh` thì fallback cũ là `zh`. Đổi default sang `vi` để trải nghiệm mặc định phù hợp hơn.

### 2. Timezone + Windows UTF-8
**File:** `.env` local, không commit
```bash
TZ=Asia/Saigon
PYTHONUTF8=1
```
**Lý do:** Múi giờ Việt Nam + giảm lỗi encoding trên Windows.

### 3. Auth credentials
**File:** `.env` local, không commit

Không ghi credential thật vào repo. Nếu cần chạy local, tự cấu hình các biến auth trong `.env` riêng trên máy.

### 4. Dev startup script
**File:** `run.sh` local, gitignored
**Lý do:** Khởi động nhanh backend + frontend với SSL workaround cho môi trường local/proxy.

### 5. Gitignore local runtime files
**File:** `.gitignore`
```gitignore
run.sh
.gitnexus/*
```
**Lý do:** Không commit script local và dữ liệu GitNexus runtime.

### 6. Claude Code permissions
**File:** `.claude/settings.json`
- Thêm một số command allowlist phục vụ dev local: `cat`, `cp`, `sed`, `which`, `uv run python`, `uv pip`, etc.

### 7. Tài liệu hướng dẫn + state machine
**File:** `docs/user-guide.md`, `docs/state-machine-workflow.md`
**Lý do:** Tài liệu tiếng Việt để dễ trace, dễ dùng, dễ hiểu workflow.

### 8. Queue/worker hardening
**File:**
- `lib/generation_worker.py`
- `lib/db/repositories/task_repo.py`
- `lib/generation_queue_client.py`
- `tests/test_task_repo.py`
- `tests/test_generation_queue_client.py`
- `tests/test_generation_worker_module.py`
- `tests/test_task_cancel_router.py`

**Thay đổi:**
- Fix head-of-line blocking: nếu provider pool đang đầy, worker đưa task đó về cuối queue và tiếp tục tìm task của provider khác còn slot.
- Guard `mark_succeeded`: chỉ task còn `running` mới được chuyển sang `succeeded`, tránh ghi đè task đã terminal/cancelled.
- Running cancel semantics: task đang chạy không bị hứa là đã hủy; API trả `skipped_running` và message rõ ràng.
- Agent queue client: `enqueue_task_only()` không check worker online trước khi enqueue; worker offline chỉ được báo trong `wait_for_task()` / `enqueue_and_wait()` sau grace period.
- Orphan dependency: task queued trỏ tới dependency đã mất sẽ bị mark `failed` thay vì kẹt queued vĩnh viễn.

**Lý do:** Giữ queue đúng nghĩa async/durable, tăng throughput khi nhiều provider, và giảm race/corruption trạng thái task.

---

## Template cho thay đổi mới

```markdown
## YYYY-MM-DD — Mô tả ngắn

### Tiêu đề thay đổi
**File:** `path/to/file.py:123`
```python
# code thay đổi
```
**Lý do:** Giải thích tại sao.
```
