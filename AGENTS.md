# TÀI LIỆU KIẾN TRÚC MULTI-AGENT CỦA BỘ CÔNG CỤ ANY2VIDEO

Tài liệu này mô tả chi tiết kiến trúc Multi-Agent, thiết kế hệ thống, các luồng tương tác (Orchestration Flow) và vai trò của từng AI Agent trong dự án `any2video`.

---

## 1. System Architecture Overview

Dự án `any2video` không sử dụng các framework Multi-Agent truyền thống (như LangChain hay AutoGen) duy trì trạng thái qua RAM. Thay vào đó, nó hoạt động dưới dạng **Coding Agent Skill Toolbox** (đặc biệt thiết kế cho **Claude Code** hoặc **Antigravity**).

Triết lý thiết kế dựa trên **Sequential File-based Orchestration với Quality Gates**:
- **Trạng thái (State)** được lưu trữ, chia sẻ và cập nhật hoàn toàn thông qua hệ thống tệp tin (`workspace/runs/<slug>/`).
- **Phân chia Agent theo Tier**: Sử dụng các model có năng lực cao (Claude Opus/Sonnet) cho công việc lập kế hoạch, logic và phán đoán (judgment). Giao việc lặp đi lặp lại hoặc dispatch công cụ cho các model chi phí thấp (Claude Haiku).
- Hệ thống thiết lập **các vòng lặp tự chữa lành (Self-healing loops)**: Các công cụ kiểm duyệt (Critic/Gate) sẽ đánh giá output của Agent và trả về mã lỗi JSON chi tiết, buộc Agent phải tự động sửa (regenerate) đúng phần lỗi trước khi được phép tiến hành bước tiếp theo.

### Workflow Diagram (Mermaid.js)

```mermaid
graph TD
    %% Define System Actors
    User([User / Trigger])
    Opus[The Orchestrator / Planner Agent<br/>(Claude Opus)]
    Sonnet_Critic[Narrative Critic Agent<br/>(Claude Sonnet)]
    Haiku_Exec[Media Executor Agent<br/>(Claude Haiku)]
    Scene_Gate{Scene Gate / Evaluator<br/>(Automated DOM + Agent)}
    FileSystem[(Workspace Filesystem<br/>State / Memory)]

    %% Connections
    User --"Input URL/Text"--> Opus

    subgraph Phase 1 & 2: Extract & Plan
        Opus --"Clone repo / Fetch HTML"--> FileSystem
        Opus --"Draft analysis.md & plan.md"--> FileSystem
    end

    subgraph Gate 2: Critic Evaluation
        FileSystem --"Read plan.md & analysis.md"--> Sonnet_Critic
        Sonnet_Critic --"JSON Array of Issues"--> Opus
        Opus --"Fix failing scenes"--> FileSystem
    end

    subgraph Phase 3 & 4: TTS & Visuals
        Opus --"Approve Script (Checkpoint 1)"--> Haiku_Exec
        Haiku_Exec --"Dispatch TTS & Update duration_sec"--> FileSystem
        Opus --"Draft HTML (template_render)"--> FileSystem
    end

    subgraph Gates 3 & 4: Render Quality
        FileSystem --"HTML/PNGs"--> Scene_Gate
        Scene_Gate --"Pass/Fail Issues (Text Clip/Overlap)"--> Opus
        Opus --"Regenerate faulty HTML"--> FileSystem
    end

    subgraph Phase 5 & 6: Compose & Delivery
        Scene_Gate --"Pass = True"--> Haiku_Exec
        Haiku_Exec --"FFmpeg Compose & Send Telegram"--> FileSystem
    end
```

---

## 2. Discovered Agent Profiles

Dựa trên mã nguồn và hướng dẫn trong `skill/SKILL.md`, các Agent và Sub-Agent sau đây đã được định nghĩa và triển khai.

### 2.1. The Orchestrator / Planner Agent
- **Agent Name & Role:** Deep Extractor, Narrative Planner & Visual Designer.
- **System Prompt / Persona:** Đọc trực tiếp mã nguồn (KHÔNG chỉ đọc README), tìm ra cơ chế hoạt động, luồng đi thực sự (architecture & flow) và điểm yếu thực tế (honest caveats) của repo/công cụ. Lên kế hoạch kịch bản theo đúng 9 nguyên tắc kể chuyện (pain-first, contrast, 2nd-person address) và thiết kế hình ảnh bằng cách điền tham số vào bộ Template JSON (không tự viết HTML trừ trường hợp bất đắc dĩ).
- **Inputs & Outputs:**
  - **Inputs:** Raw text, GitHub Repos, URL Web, Hình ảnh (thông qua Claude Vision).
  - **Outputs:** Lập và lưu cấu trúc `analysis.md` và `plan.md` (YAML).
- **Tools & Capabilities:** Lệnh Bash (`git clone`), `lib.cli init` (để phân tích định tuyến), OCR, WebFetch.
- **Underlying LLM:** Claude Opus (yêu cầu năng lực phán đoán, tóm tắt và thiết kế kịch bản).

### 2.2. The Narrative Critic Agent
- **Agent Name & Role:** Gate 2 Critic (Người Đánh Giá Kịch Bản).
- **System Prompt / Persona:** Kiểm tra file `plan.md` dựa vào `analysis.md`. Đảm bảo kịch bản không chứa văn phong liệt kê, không dùng CTA chung chung (vd: "Ghé repo xem thử"), có chứa từ nối (connectors), có sử dụng ngôi thứ hai ("bạn"), và mọi chi tiết kỹ thuật/con số đều khớp với mã nguồn được chứng minh trong `analysis.md`.
- **Inputs & Outputs:**
  - **Inputs:** `plan.md`, `analysis.md`.
  - **Outputs:** Trả về JSON chứa danh sách `issues` nếu có lỗi.
- **Tools & Capabilities:** Thực thi script `python -m lib.critic.plan_critic`.
- **Underlying LLM:** Claude Sonnet (tối ưu chi phí, chuyên đánh giá text logic).

### 2.3. The Scene Gate (Visual Evaluator)
- **Agent Name & Role:** Automated Gatekeeper kết hợp Agent khắc phục lỗi UI.
- **System Prompt / Persona:** Đóng vai trò là vòng bảo vệ (Pre-render Gate). Cấm việc xuất xưởng bất kỳ scene nào bị tràn lề (outside safe zones), bị cắt chữ (đặc biệt dấu tiếng Việt như Ậ, Ỗ), các block text chèn lên nhau, hộp UI trống rỗng hoặc hình ảnh bị lỗi tải.
- **Inputs & Outputs:**
  - **Inputs:** File `scenes/<id>.html`.
  - **Outputs:** Thông báo dạng Text/JSON (`pass:false` hoặc `pass:true`).
- **Tools & Capabilities:** Thực thi script `python -m lib.critic.scene_gate all <plan.md>`. Agent sẽ đọc lỗi và tự điều chỉnh padding, line-height hoặc độ dài văn bản trong file template/YAML.
- **Underlying LLM:** Automated Python DOM Inspector (Script) + Claude Opus (để fix lỗi sinh ra).

### 2.4. The Media Executor Agent
- **Agent Name & Role:** Tool Dispatcher (TTS, Render, FFMpeg Compose).
- **System Prompt / Persona:** Tiếp nhận kế hoạch đã được duyệt. Phân tích các bước để kích hoạt TTS (đo thời lượng thật), kích hoạt Playwright Render và cấu hình lệnh ghép FFmpeg (hard-cut, không crossfade tiếng) rồi gửi kết quả qua Telegram.
- **Inputs & Outputs:**
  - **Inputs:** Lệnh thực thi bash.
  - **Outputs:** File `.mp3`, `.png`, `final.mp4`.
- **Tools & Capabilities:** `lib.tts.narrate`, `lib.render.playwright_render`, `lib.compose.ffmpeg_compose`, `lib.notify.telegram`.
- **Underlying LLM:** Claude Haiku (do tính chất lặp đi lặp lại và thực thi câu lệnh có cấu trúc chuẩn).

---

## 3. Communication & Orchestration Flow

Kiến trúc chạy theo dạng **Sequential Flow with Mandatory Validation Gates**.

1. **State Management & Context Preservation**:
   - Agent không truyền memory qua mảng tin nhắn (Message History) như LangGraph, mà dùng **Filesystem Context**.
   - Tất cả kết quả của một run nằm độc lập ở `workspace/runs/<slug>/`. Context truyền sang các sub-agent bao gồm đường dẫn đọc file (`analysis.md`, `plan.md`). Khi Agent mất kết nối hoặc đổi phiên, nó chỉ cần chạy lệnh `python -m any2video.lib.cli status <slug>` để khôi phục toàn bộ tiến độ.

2. **Self-Healing Error Handling Pattern**:
   - Khi Agent gọi các Validation Script (ví dụ: `lib.critic.plan_critic`), nếu có lỗi (Exit 1), script trả về dữ liệu chuẩn JSON (chứa `scene_id`, `kind`, `hint`).
   - System prompt ép Agent KHÔNG được lập kế hoạch lại toàn bộ video. Nó phải dùng JSON này để tìm và sửa ĐÚNG vào scene đang bị lỗi trong file `plan.md`, lưu lại và chạy lại Critic cho đến khi Exit 0 (Exit code chuẩn - AI tự biết lỗi gì mà sửa).

3. **Human-in-the-loop (Checkpoints)**:
   - Hệ thống có 2 chốt chặn do người duyệt qua Telegram:
     - **Checkpoint 1 (Script before TTS)**: Duyệt văn bản và phiên âm (phonetics) để tránh lỗi phát âm.
     - **Checkpoint 2 (Stills before Render)**: Xem frame ảnh (PNG) tĩnh trước khi kích hoạt Playwright tạo video tốn thời gian.

---

## 4. Developer Guide: Adding a New Agent

Để thêm một phân hệ chuyên môn mới (Agent hoặc Critic) vào luồng của `any2video` mà không phá vỡ tính đồng bộ, Developer cần làm theo 4 bước sau:

- **Bước 1: Tạo Schema Validation & Template**
  Định nghĩa contract (Inputs/Outputs dạng markdown hoặc JSON schema) và thêm file mô tả vào thư mục `skill/templates/` (ví dụ: `new-agent-schema.md`). Đảm bảo quy định rõ các ràng buộc (constraints).

- **Bước 2: Viết Python Tool/Critic (Executable JSON API)**
  Tạo script Python độc lập trong `lib/critic/` (ví dụ: `new_gate.py`). Script này bắt buộc:
  - Nhận tham số qua CLI argument (vd đường dẫn file).
  - Trả kết quả chuẩn dạng JSON ra `stdout`.
  - Trả về Exit Code `0` nếu Pass, và Exit Code `1` nếu Fail (kèm thông báo chi tiết mảng `issues` để AI Agent tự chữa).

- **Bước 3: Tích hợp vào Orchestrator Prompt**
  Cập nhật mảng `next_steps` trong `lib/cli.py` (hàm `init`) để hướng dẫn Claude Agent (The Orchestrator) cách gọi tool vừa tạo và xử lý khi bị báo lỗi.
  Ví dụ: *"Run `python -m lib.critic.new_gate <plan.md>`. If exit 1, read the JSON issues, fix the specific `<id>` in `plan.md`, and re-run."*

- **Bước 4: Cập nhật SKILL.md (System Prompt lõi)**
  Bổ sung một mục giải thích ngắn ở `skill/SKILL.md` dưới phần Workflow để các model Claude trong những lượt hội thoại sau hiểu ngữ cảnh và biết khi nào nên gọi Agent này.
