# Queue Worker Fixes

## Goal
Sửa 5 lỗi logic trong hệ thống task queue/worker: HOL blocking, race khi mark succeeded, semantics cancel running, worker offline behavior của Agent tools, và orphan dependency.

## Scope
- `lib/generation_worker.py`
- `lib/generation_queue_client.py`
- `lib/generation_queue.py`
- `lib/db/repositories/task_repo.py`
- tests liên quan queue/task

## Non-goals
- Không thêm infra mới.
- Không cố hủy request API bên thứ ba nếu backend chưa hỗ trợ cancellation token.
- Không refactor lớn queue architecture.

## Files likely touched
- `lib/generation_worker.py`
- `lib/generation_queue_client.py`
- `lib/generation_queue.py`
- `lib/db/repositories/task_repo.py`
- `tests/test_generation_queue.py`
- `tests/test_generation_queue_client.py`
- `tests/test_task_repo.py`

## Steps
1. Trace tests hiện có và xác định seam regression.
2. Fix HOL blocking bằng provider-aware skip/requeue tránh claim lặp task đầu hàng khi pool full.
3. Guard `mark_succeeded` chỉ update task còn `running`.
4. Cập nhật cancel running semantics: API nói rõ running không bị dừng, trả `skipped_running`.
5. Agent tools enqueue trước, chỉ báo worker offline trong phase wait.
6. Orphan dependency: detect và mark failed thay vì kẹt queued.
7. Chạy targeted tests.

## Risks
- Scheduler thay đổi có thể ảnh hưởng fairness giữa provider.
- Cancel running nếu hiểu nhầm là hard cancel sẽ gây false expectation.
- Tests DB async có thể phụ thuộc fixture hiện có.

## Verification
- Targeted pytest cho queue/client/repo/router.
- Manual diff review.

## Unresolved questions
- Có cần hard-cancel API third-party không? Tạm thời không, vì cần support từng backend.
