# ArcReel Agent Workflow State Machine

Tài liệu này mô tả **quy ước vận hành state-machine của ArcReel Agent**: agent xác định dự án đang thiếu bước nào, rồi chạy đúng phase tiếp theo.

> Lưu ý: đây là workflow spec cho Agent, **không phải** một state-machine runtime/class duy nhất trong code.

Dùng tài liệu này để trace, debug, và hiểu flow từ lúc tạo dự án đến khi xuất video.

---

## Tổng quan phase

```text
PHASE 0   -> PHASE 1   -> PHASE 2   -> PHASE 3   -> PHASE 4   -> PHASE 5   -> PHASE 6    -> PHASE 7
Setup        Assets       Episodes     Preproc      Script       Design       Storyboard     Video
Web UI       subagent      main agent   subagent     subagent     subagent     MCP queue      MCP queue
```

| Phase | Mục tiêu | Actor chính |
|---|---|---|
| 0 | Tạo project, upload source, tạo overview | Web UI + Agent |
| 1 | Trích xuất character / scene / prop toàn cục | `analyze-assets` |
| 2 | Chia source thành từng tập | Main agent + `manage-project` scripts |
| 3 | Tiền xử lý tập theo mode | `split-*` / `normalize-*` subagent |
| 4 | Sinh JSON script | `create-episode-script` |
| 5 | Sinh sheet ảnh cho asset | `generate-assets` |
| 6 | Sinh storyboard/grid image | `generate-assets` + MCP |
| 7 | Sinh video clip | `generate-assets` + MCP |

---

## Content mode và generation mode

Hai dimension độc lập:

### `content_mode`

| Giá trị | Ý nghĩa | Script chính |
|---|---|---|
| `narration` | Thuyết minh / nói sách | `segments[]` |
| `drama` | Phim truyện / hoạt hình có cảnh và thoại | `scenes[]` |

`content_mode` được chọn khi tạo project và không được đổi sau đó qua API update.

### `generation_mode`

| Giá trị | Ý nghĩa | Visual source |
|---|---|---|
| `storyboard` | Tạo từng ảnh phân cảnh rồi sinh video | `storyboards/` |
| `grid` | Tạo ảnh lưới, cắt thành first/last frame rồi sinh video | `grids/` + `storyboards/` |
| `reference_video` | Dùng sheet ảnh character/scene/prop để sinh video trực tiếp | `reference_videos/` |

Cách resolve mode thực tế:

```text
effective_mode = episode.generation_mode ?? project.generation_mode ?? "storyboard"
```

Code tương ứng: `lib/project_manager.py::effective_mode`.

---

## State detection

Agent đọc `project.json` và kiểm tra filesystem theo thứ tự dưới đây. Gặp thiếu ở đâu thì chạy phase đó.

```text
1. project.json có đủ characters / scenes / props chưa?
   └─ Thiếu bất kỳ bucket nào -> PHASE 1

2. source/episode_{N}.txt tồn tại chưa?
   └─ Chưa -> PHASE 2

3. Step 1 draft của tập N tồn tại chưa?
   ├─ reference_video -> drafts/episode_{N}/step1_reference_units.md
   ├─ narration       -> drafts/episode_{N}/step1_segments.md
   └─ drama           -> drafts/episode_{N}/step1_normalized_script.md
   └─ Chưa -> PHASE 3

4. scripts/episode_{N}.json tồn tại chưa?
   └─ Chưa -> PHASE 4

5. Asset sheet đủ chưa?
   ├─ character.character_sheet
   ├─ scene.scene_sheet
   └─ prop.prop_sheet
   └─ Thiếu -> PHASE 5

6. storyboard/grid mode: storyboard_image đủ chưa?
   └─ Thiếu -> PHASE 6
   └─ reference_video -> skip phase này

7. video_clip đủ chưa?
   └─ Thiếu -> PHASE 7

8. Tất cả OK -> hoàn thành, gợi ý export Jianying / output video
```

---

## Chi tiết từng phase

### Phase 0 — Project setup

| Actor | Hành động |
|---|---|
| Web UI | `POST /api/v1/projects` -> `ProjectManager.create_project()` |
| Web UI | Tạo `project.json`, sync agent profile theo `content_mode` |
| Web UI | Tạo các thư mục chuẩn: `source/`, `scripts/`, `drafts/`, `characters/`, `scenes/`, `props/`, `storyboards/`, `grids/`, `videos/`, `thumbnails/`, `output/` |
| Runtime | `reference_videos/` có thể được tạo/đóng gói theo nhu cầu của reference-video flow |
| User | Upload source vào `source/` |
| Agent | Đọc `project.json`, xác nhận `title`, `content_mode`, `generation_mode`; nếu có source thì tạo overview |

`project.json` tối thiểu:

```json
{
  "schema_version": 1,
  "title": "...",
  "content_mode": "narration",
  "generation_mode": "storyboard",
  "overview": {},
  "episodes": [],
  "characters": [],
  "scenes": [],
  "props": []
}
```

### Phase 1 — Global asset extraction

| Mục | Giá trị |
|---|---|
| Trigger | `characters` / `scenes` / `props` thiếu hoặc rỗng |
| Actor | `analyze-assets` subagent |
| Input | project name, phạm vi phân tích, asset đã có |
| Output | Cập nhật `characters[]`, `scenes[]`, `props[]` trong `project.json` |
| Verify | Đọc lại `project.json`, kiểm tra bucket không còn rỗng |

### Phase 2 — Episode planning

| Mục | Giá trị |
|---|---|
| Trigger | `source/episode_{N}.txt` chưa tồn tại |
| Actor | Main agent trực tiếp, không dispatch subagent |
| Tools | `manage-project/scripts/peek_split_point.py`, `split_episode.py` |
| Output | `source/episode_{N}.txt`, thường kèm `_remaining.txt` |

Flow khuyến nghị:

1. Chọn source: ưu tiên `source/_remaining.txt`, nếu không có thì dùng file gốc.
2. Hỏi user target words/episode.
3. Peek điểm cắt tự nhiên.
4. Đề xuất anchor và chờ user confirm.
5. Chạy dry-run.
6. Nếu OK, split thật.

### Phase 3 — Preprocessing theo mode

| Trigger | Step 1 draft của tập N chưa tồn tại |
|---|---|

Routing chính xác:

```text
effective_mode == "reference_video"
  -> dispatch split-reference-video-units
  -> output drafts/episode_{N}/step1_reference_units.md

effective_mode in {"storyboard", "grid"} + content_mode == "narration"
  -> dispatch split-narration-segments
  -> output drafts/episode_{N}/step1_segments.md

effective_mode in {"storyboard", "grid"} + content_mode == "drama"
  -> dispatch normalize-drama-script
  -> output drafts/episode_{N}/step1_normalized_script.md
```

### Phase 4 — JSON script generation

| Mục | Giá trị |
|---|---|
| Trigger | `scripts/episode_{N}.json` chưa tồn tại |
| Actor | `create-episode-script` subagent |
| Tool | `mcp__arcreel__generate_episode_script({"episode": N})` |
| Output | `scripts/episode_{N}.json` + metadata trong `project.json.episodes` |

Schema theo mode:

| Mode | Schema |
|---|---|
| narration | `NarrationEpisodeScript`, có `segments[]` |
| drama | `DramaEpisodeScript`, có `scenes[]` |
| reference_video | `ReferenceVideoScript`, có `video_units[]` |

`ProjectManager._write_script_unlocked()` có lớp bảo vệ validate script: không cho ghi đè một script hợp lệ bằng bản mới hỏng cấu trúc.

### Phase 5 — Asset design

| Mục | Giá trị |
|---|---|
| Trigger | Thiếu `character_sheet`, `scene_sheet`, hoặc `prop_sheet` |
| Actor | `generate-assets` subagent, có thể chạy độc lập theo từng type |
| Tool | `mcp__arcreel__generate_assets({"type": "character|scene|prop"})` |

Dispatch rule:

```text
for type in {character, scene, prop}:
  nếu type đó còn item thiếu *_sheet -> dispatch generate-assets cho type đó
  nếu đã đủ -> bỏ qua
```

### Phase 6 — Storyboard/grid generation

| Mục | Giá trị |
|---|---|
| Trigger | storyboard/grid mode còn thiếu `storyboard_image` |
| Skip | `reference_video` bỏ qua phase này |

Routing:

```text
storyboard -> mcp__arcreel__generate_storyboards({"script": "episode_{N}.json"})
grid       -> mcp__arcreel__generate_grid({"script": "episode_{N}.json"})
```

Output chính:

- storyboard mode: `storyboards/...`
- grid mode: `grids/grid_{id}.png`, metadata `grids/grid_{id}.json`, frame cắt ra `storyboards/...`

### Phase 7 — Video generation

| Mục | Giá trị |
|---|---|
| Trigger | còn thiếu `video_clip` |
| Actor | `generate-assets` subagent |
| Tool | `mcp__arcreel__generate_video_episode({"script": "episode_{N}.json"})` |

Routing tự động trong tool:

| Script | Task route | Output |
|---|---|---|
| `segments[]` | `task_type="video"` | `videos/scene_{segment_id}.mp4` |
| `scenes[]` | `task_type="video"` | `videos/scene_{scene_id}.mp4` |
| `generation_mode == "reference_video"` hoặc `video_units[]` | `task_type="reference_video"` | `reference_videos/{unit_id}.mp4` |

---

## Subagent catalog

| Subagent | File | Vai trò |
|---|---|---|
| `analyze-assets` | `agent_runtime_profile/.claude/agents/analyze-assets.md` | Trích xuất character/scene/prop toàn cục |
| `split-narration-segments` | `agent_runtime_profile/.claude/agents/split-narration-segments.md` | Chia narration thành segment |
| `normalize-drama-script` | `agent_runtime_profile/.claude/agents/normalize-drama-script.md` | Chuẩn hóa drama thành scene |
| `split-reference-video-units` | `agent_runtime_profile/.claude/agents/split-reference-video-units.md` | Chia reference-video thành video unit |
| `create-episode-script` | `agent_runtime_profile/.claude/agents/create-episode-script.md` | Sinh JSON script từ Step 1 |
| `generate-assets` | `agent_runtime_profile/.claude/agents/generate-assets.md` | Sinh asset sheet, storyboard/grid, video |

## Skill catalog

| Skill | File | Vai trò |
|---|---|---|
| `manga-workflow` | `agent_runtime_profile/.claude/skills/manga-workflow/SKILL.narration.md` / `SKILL.drama.md` | Orchestrator instruction theo `content_mode` |
| `manage-project` | `agent_runtime_profile/.claude/skills/manage-project/` | Script chia tập |
| `generate-assets` | `agent_runtime_profile/.claude/skills/generate-assets/` | MCP wrapper sinh sheet ảnh |
| `generate-storyboard` | `agent_runtime_profile/.claude/skills/generate-storyboard/` | MCP wrapper sinh storyboard |
| `generate-grid` | `agent_runtime_profile/.claude/skills/generate-grid/` | MCP wrapper sinh grid |
| `generate-video` | `agent_runtime_profile/.claude/skills/generate-video/` | MCP wrapper sinh video |
| `generate-script` | `agent_runtime_profile/.claude/skills/generate-script/` | MCP wrapper sinh JSON script |
| `compose-video` | `agent_runtime_profile/.claude/skills/compose-video/` | FFmpeg concat/export |

---

## Confirmation protocol

Sau mỗi phase do subagent/tool hoàn thành, agent nên:

1. Tóm tắt kết quả ngắn gọn.
2. Hỏi user chọn một trong các hướng:
   - Tiếp tục phase tiếp theo.
   - Làm lại phase hiện tại với yêu cầu sửa.
   - Dừng/bỏ qua có chủ đích.
3. Chỉ chạy tiếp khi user đã xác nhận hoặc yêu cầu rõ ràng “tự chạy tiếp”.

---

## Flexible entry

Agent có thể vào workflow từ bất kỳ phase nào:

| User nói | Phase thường vào |
|---|---|
| “Phân tích nhân vật/cảnh/đạo cụ” | Phase 1 |
| “Tạo tập 2” | Phase 2 hoặc phase thiếu đầu tiên của tập 2 |
| “Tiếp tục” | Chạy state detection rồi vào phase thiếu đầu tiên |
| “Tạo phân cảnh” | Phase 6 |
| “Tạo video” | Phase 7 |

---

## Source of truth khi tài liệu lệch

Nếu tài liệu này mâu thuẫn với code/instruction runtime, ưu tiên theo thứ tự:

1. Code xử lý thật: `lib/project_manager.py`, `server/services/*`, `server/routers/*`.
2. Runtime agent docs: `agent_runtime_profile/.claude/skills/manga-workflow/` và `agent_runtime_profile/.claude/references/generation-modes.md`.
3. Tài liệu tổng quan này.
