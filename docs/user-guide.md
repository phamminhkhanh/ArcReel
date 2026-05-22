# Hướng Dẫn Sử Dụng ArcReel

ArcReel là nền tảng AI tạo video từ tiểu thuyết, hoạt động qua Web UI + AI Assistant.

---

## Bắt đầu nhanh

### 1. Khởi động

```bash
bash run.sh
# Backend:  http://localhost:1241
# Frontend: http://localhost:5173
```

### 2. Đăng nhập

Mặc định: `admin` / mật khẩu trong `.env` (`AUTH_PASSWORD`)

### 3. Cấu hình (chỉ làm 1 lần)

Vào **Cài đặt** (`/settings`), cấu hình 2 mục:

| Mục | Bắt buộc? | Giải thích |
|-----|-----------|------------|
| **ArcReel智能体** | **CÓ** | AI assistant cần Anthropic API key hoặc provider tương thích (DeepSeek, GLM, MiniMax...) |
| **AI生图/生视频** | **CÓ** | Ít nhất 1 provider cho image + 1 cho video (Gemini, Ark, Grok, OpenAI, custom) |

> **Lưu ý:** AI Assistant dùng Anthropic protocol (`/v1/messages`), không phải OpenAI protocol (`/v1/chat/completions`). Chọn preset DeepSeek/GLM/MiniMax nếu không có Anthropic key.

---

## Tạo dự án đầu tiên

### Bước 1: Tạo project trên Web UI

1. Nhấn **"Tạo dự án"**
2. Điền tên dự án
3. Chọn **Chế độ nội dung**:
   - **Thuyết minh** (narration) — 1 giọng đọc, phù hợp kể chuyện/tài liệu
   - **Phim truyện** (drama) — đối thoại nhân vật, phù hợp chuyển thể tiểu thuyết
4. Chọn **Chế độ tạo** (có thể đổi sau):
   - **图生视频** (storyboard) — từng ảnh → video
   - **宫格生视频** (grid) — ảnh ghép → cắt → video
   - **参考生视频** (reference_video) — sheet nhân vật → video trực tiếp

### Bước 2: Upload nguồn

Kéo thả file (.txt/.pdf/.docx) vào khu vực "Tệp gốc", hoặc chat với AI assistant để nó hướng dẫn.

### Bước 3: Chat với AI Assistant

Mở tab **"ArcReel Agent"** → chat tự nhiên:

| Bạn nói | Agent làm |
|---------|-----------|
| "bắt đầu làm video" | Phân tích nhân vật → chia tập → tạo kịch bản → sinh ảnh → sinh video |
| "tiếp tục" | Tự động tìm phase chưa xong và tiếp tục |
| "tạo tập 2" | Chỉ làm tập 2 (nếu đã có nhân vật) |
| "làm lại phân cảnh tập 1" | Chỉ regenerate storyboard tập 1 |
| "tạo video tập 1" | Chỉ generate video cho tập 1 |

> **Mẹo:** Đừng dùng `/manga-workflow` — chat tự nhiên là được. Agent tự biết phải làm gì.

---

## Các chế độ nội dung

### Thuyết minh (Narration)

- 1 giọng đọc xuyên suốt
- Mỗi đoạn = 1 ảnh + 1 đoạn đọc
- Phù hợp: kể chuyện, documentary, video giải thích

### Phim truyện (Drama)

- Đối thoại nhân vật + hành động
- Mỗi cảnh = 1 ảnh + hội thoại
- Phù hợp: chuyển thể tiểu thuyết, phim ngắn

---

## Các chế độ tạo (Generation Mode)

| Mode | Cách hoạt động | Ưu | Nhược |
|------|---------------|----|-------|
| **Storyboard** | Sinh ảnh phân cảnh → dùng làm start frame → sinh video | Đơn giản, kiểm soát tốt | Cần sinh cả ảnh lẫn video |
| **Grid** | Ghép nhiều ảnh vào 1 lưới → cắt → video | Tiết kiệm lượt gọi API | Phức tạp hơn |
| **Reference Video** | Dùng sheet nhân vật/cảnh làm reference → sinh video trực tiếp | Bỏ qua bước sinh ảnh | Cần reference images chất lượng |

---

## Theo dõi tiến độ

Mỗi project có các tab:

- **Tổng quan** — synopsis, genre, theme
- **Nhân vật** — danh sách + ảnh thiết kế
- **Cảnh** — danh sách + ảnh thiết kế
- **Đạo cụ** — danh sách + ảnh thiết kế
- **Tập** — danh sách tập + trạng thái từng tập

---

## Các lệnh chat hữu ích

| Lệnh | Mô tả |
|------|-------|
| "phân tích nhân vật" | Chạy Phase 1 — trích xuất nhân vật/cảnh/đạo cụ |
| "chia tập" | Chạy Phase 2 — peek + split episode |
| "tạo kịch bản tập X" | Chạy Phase 3-4 — preprocess + generate JSON script |
| "thiết kế nhân vật" | Chạy Phase 5 — generate character sheets |
| "tạo phân cảnh" | Chạy Phase 6 — generate storyboard images |
| "tạo video" | Chạy Phase 7 — generate video clips |
| "ghép video" | Dùng skill compose-video — FFmpeg ghép tập |
| "xuất剪映" | Export Jianying (CapCut) draft |

---

## Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Cách fix |
|-----|-------------|----------|
| Agent không trả lời | Thiếu Anthropic API key | Kiểm tra Settings → ArcReel智能体 |
| Agent trả lời tiếng Trung | Browser Accept-Language không có `vi` | Đã fix: `DEFAULT_LOCALE = "vi"` |
| `ascii codec` error | Python không ở UTF-8 mode | Chạy bằng `bash run.sh` (đã set `PYTHONUTF8=1`) |
| Skill `/manga-workflow` lỗi | Agent ảo giác chạy script không tồn tại | Chat tự nhiên, đừng dùng slash command |
| `project.json missing` | CWD không đúng project dir | Khởi động lại session |
| Video generation failed | Thiếu provider config | Kiểm tra Settings → AI生视频 |

---

## Kiến trúc thư mục dự án

```
projects/{tên-dự-an}/
├── project.json          # Toàn bộ metadata (nhân vật, tập, style...)
├── source/               # File nguồn (.txt/.pdf)
├── scripts/              # JSON kịch bản
├── drafts/               # File trung gian (step1)
├── characters/           # Ảnh thiết kế nhân vật
├── scenes/               # Ảnh thiết kế bối cảnh
├── props/                # Ảnh thiết kế đạo cụ
├── storyboards/          # Ảnh phân cảnh
├── grids/                # Ảnh ghép lưới
├── videos/               # Video clips đã generate
├── reference_videos/     # Video reference mode
├── thumbnails/           # Ảnh thumbnail
└── output/               # Video thành phẩm
```

## Tài liệu liên quan

- [State Machine Workflow](state-machine-workflow.md) — Chi tiết state machine & subagent
- [Custom Changes](custom-changes.md) — Các thay đổi tùy chỉnh so với upstream
