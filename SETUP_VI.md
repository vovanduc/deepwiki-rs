# 🇻🇳 Hướng Dẫn Cài Đặt và Sử Dụng Litho (deepwiki-rs)

> Hướng dẫn đầy đủ bằng tiếng Việt để bắt đầu với Litho - công cụ tạo tài liệu kiến trúc tự động bằng AI

## 📋 Mục Lục

- [Giới Thiệu](#-giới-thiệu)
- [Yêu Cầu Hệ Thống](#-yêu-cầu-hệ-thống)
- [Cài Đặt](#-cài-đặt)
- [Cấu Hình LLM Models](#-cấu-hình-llm-models)
- [Cách Sử Dụng](#-cách-sử-dụng)
- [Quy Trình Hoạt Động](#-quy-trình-hoạt-động)
- [Tối Ưu Chi Phí](#-tối-ưu-chi-phí)
- [Xử Lý Lỗi Thường Gặp](#-xử-lý-lỗi-thường-gặp)
- [Câu Hỏi Thường Gặp](#-câu-hỏi-thường-gặp)

---

## 🎯 Giới Thiệu

**Litho** (tên package: `deepwiki-rs`) là công cụ tự động tạo tài liệu kiến trúc phần mềm bằng AI, được viết bằng Rust để đảm bảo hiệu năng cao.

### Litho làm được gì?

- ✅ **Tự động phân tích** codebase của bạn (hỗ trợ nhiều ngôn ngữ: Rust, Python, Java, JavaScript, TypeScript, Go, Kotlin, v.v.)
- ✅ **Tạo tài liệu kiến trúc C4** (Context, Container, Component, Code) chuyên nghiệp
- ✅ **Sinh diagram Mermaid** tự động với kiểm tra syntax
- ✅ **Hiểu được luồng dữ liệu** và dependencies giữa các modules
- ✅ **Tạo deep-dive documentation** cho các modules quan trọng

### Tại sao nên dùng Litho?

- 🚀 **Tiết kiệm thời gian**: Không cần viết tài liệu kiến trúc thủ công
- 📊 **Chuyên nghiệp**: Tài liệu theo chuẩn C4 model
- 🔄 **Luôn cập nhật**: Chạy lại là có tài liệu mới nhất
- 🌍 **Đa ngôn ngữ**: Hỗ trợ tiếng Anh, Trung, Nhật, Hàn, Đức, Pháp, Nga
- 💰 **Linh hoạt**: Chọn LLM provider phù hợp với ngân sách

---

## 💻 Yêu Cầu Hệ Thống

### Bắt buộc

- **Rust** 1.70 trở lên ([Cài đặt Rust](https://rustup.rs/))
- **Cargo** (đi kèm với Rust)
- **API key** của một trong các LLM providers:
  - OpenAI (GPT-4o, GPT-4o-mini)
  - Anthropic (Claude)
  - DeepSeek (phổ biến tại VN, giá rẻ)
  - Google Gemini
  - Hoặc chạy local với Ollama (miễn phí)

### Khuyến nghị

- **RAM**: Tối thiểu 4GB, khuyến nghị 8GB+
- **Disk**: ~100MB cho binary + cache space
- **Internet**: Cần kết nối để gọi LLM API (trừ khi dùng Ollama)

---

## 📦 Cài Đặt

### Phương Án 1: Cài từ crates.io (Khuyến Nghị)

Cách đơn giản nhất, không cần clone source code:

```bash
cargo install deepwiki-rs
```

Sau khi cài xong, bạn có thể dùng lệnh `deepwiki-rs` ở bất kỳ đâu.

### Phương Án 2: Build từ Source Code

Nếu bạn muốn customize hoặc contribute:

```bash
# 1. Clone repository
git clone https://github.com/sopaco/deepwiki-rs.git
cd deepwiki-rs

# 2. Build project (release mode để tối ưu hiệu năng)
cargo build --release

# 3. Binary nằm ở đây
./target/release/deepwiki-rs --version

# 4. (Tùy chọn) Cài đặt global
cargo install --path .
```

### Kiểm Tra Cài Đặt

```bash
deepwiki-rs --version
# Output: deepwiki-rs 1.2.1
```

---

## 🤖 Cấu Hình LLM Models

Đây là phần **QUAN TRỌNG NHẤT** - bạn cần cấu hình model để Litho hoạt động.

### Hiểu về 2 Loại Model

Litho sử dụng **2 loại model** cho các task khác nhau:

| Loại Model | Tên Parameter | Mục Đích | Khi Nào Dùng |
|------------|---------------|----------|--------------|
| **Efficient** | `--model-efficient` | Model nhanh, rẻ | Quét code, phân tích cơ bản, extract structure |
| **Powerful** | `--model-powerful` | Model mạnh, đắt hơn | Phân tích phức tạp, tạo tài liệu chất lượng cao |

**Ví dụ thực tế:**
- **Efficient**: GPT-4o-mini, Claude Haiku, Gemini Flash (~$0.075-0.25/1M tokens)
- **Powerful**: GPT-4o, Claude Sonnet, Gemini Pro (~$1.25-3.00/1M tokens)

### Các Provider Phổ Biến

#### 1. 🌟 DeepSeek (Khuyến nghị cho người Việt Nam)

**Ưu điểm**: Rất rẻ (~$0.14/1M tokens), API nhanh, chất lượng tốt cho code
**Nhược điểm**: Ít document hơn OpenAI

```bash
# Bước 1: Đăng ký tài khoản tại https://platform.deepseek.com/
# Bước 2: Lấy API key
# Bước 3: Set environment variable
export LITHO_LLM_API_KEY="sk-xxxxxxxxx"

# Bước 4: Chạy Litho
deepwiki-rs -p ./my-project \
  --llm-provider deepseek \
  --llm-api-base-url https://api.deepseek.com/v1 \
  --model-efficient deepseek-chat \
  --model-powerful deepseek-chat
```

**Chi phí ước tính**: Dự án 5000 files ~ $1-2 USD

#### 2. 💎 Google Gemini (Rẻ nhất)

**Ưu điểm**: Rẻ nhất (~$0.075/1M tokens cho Flash), context window lớn
**Nhược điểm**: Đôi khi kém ổn định hơn

```bash
# Bước 1: Lấy API key tại https://aistudio.google.com/app/apikey
# Bước 2: Set API key
export LITHO_LLM_API_KEY="AIzaSy..."

# Bước 3: Chạy Litho
deepwiki-rs -p ./my-project \
  --llm-provider gemini \
  --llm-api-base-url https://generativelanguage.googleapis.com/v1beta \
  --model-efficient gemini-1.5-flash \
  --model-powerful gemini-1.5-pro
```

**Chi phí ước tính**: Dự án 5000 files ~ $0.50-1 USD

#### 3. 🔥 OpenAI (Chất lượng cao nhất)

**Ưu điểm**: Chất lượng tốt nhất, stable, document đầy đủ
**Nhược điểm**: Đắt nhất

```bash
# Bước 1: Lấy API key tại https://platform.openai.com/api-keys
# Bước 2: Set API key
export LITHO_LLM_API_KEY="sk-proj-..."

# Bước 3: Chạy Litho
deepwiki-rs -p ./my-project \
  --llm-api-base-url https://api.openai.com/v1 \
  --model-efficient gpt-4o-mini \
  --model-powerful gpt-4o
```

**Chi phí ước tính**: Dự án 5000 files ~ $3-5 USD

#### 4. 🧠 Anthropic Claude (Tốt cho code analysis)

**Ưu điểm**: Rất giỏi phân tích code, reasoning tốt
**Nhược điểm**: Khá đắt, API có thể slow

```bash
# Bước 1: Lấy API key tại https://console.anthropic.com/
# Bước 2: Set API key
export LITHO_LLM_API_KEY="sk-ant-..."

# Bước 3: Chạy Litho
deepwiki-rs -p ./my-project \
  --llm-provider anthropic \
  --llm-api-base-url https://api.anthropic.com \
  --model-efficient claude-3-haiku-20240307 \
  --model-powerful claude-3-5-sonnet-20241022
```

**Chi phí ước tính**: Dự án 5000 files ~ $2-4 USD

#### 5. 🏠 Ollama (Chạy Local - MIỄN PHÍ)

**Ưu điểm**: Hoàn toàn miễn phí, offline, bảo mật
**Nhược điểm**: Cần GPU mạnh, chậm hơn, chất lượng thấp hơn

```bash
# Bước 1: Cài Ollama - https://ollama.ai/download
# Bước 2: Pull model
ollama pull llama3.1

# Bước 3: Chạy Litho (không cần API key)
deepwiki-rs -p ./my-project \
  --llm-provider ollama \
  --llm-api-base-url http://localhost:11434 \
  --model-efficient llama3.1 \
  --model-powerful llama3.1
```

**Chi phí**: $0 (nhưng tốn điện + GPU)

### Cấu Hình bằng File litho.toml

Thay vì gõ dài dòng trên command line, bạn có thể tạo file `litho.toml`:

```bash
# Copy file example
cp litho.toml.example litho.toml

# Edit file
nano litho.toml
```

Ví dụ nội dung `litho.toml`:

```toml
[llm]
provider = "deepseek"
api_base_url = "https://api.deepseek.com/v1"
api_key = "${LITHO_LLM_API_KEY}"
model_efficient = "deepseek-chat"
model_powerful = "deepseek-chat"
max_tokens = 8192
temperature = 0.1
max_parallels = 3

[cache]
enabled = true
```

Sau đó chỉ cần chạy:

```bash
export LITHO_LLM_API_KEY="your-key"
deepwiki-rs -p ./my-project
```

---

## 🚀 Cách Sử Dụng

### Lệnh Cơ Bản

```bash
# Phân tích project hiện tại
deepwiki-rs

# Phân tích project cụ thể
deepwiki-rs -p ./path/to/project

# Chỉ định thư mục output
deepwiki-rs -p ./src -o ./my-docs
```

### Lệnh Nâng Cao

```bash
# Chỉ định model và language
deepwiki-rs -p ./src \
  --model-efficient gpt-4o-mini \
  --model-powerful gpt-4o \
  --target-language en \
  --llm-api-key sk-xxx

# Skip các giai đoạn (khi đã có cache)
deepwiki-rs -p ./src \
  --skip-preprocessing \
  --skip-research

# Chạy lại từ đầu (xóa cache)
deepwiki-rs -p ./src --force-regenerate

# Disable ReAct mode (không auto-scan files)
deepwiki-rs -p ./src --disable-preset-tools

# Tăng parallel processing (nếu có API limit cao)
deepwiki-rs -p ./src --max-parallels 5
```

### Output Structure

Sau khi chạy xong, bạn sẽ có thư mục `litho.docs/`:

```
litho.docs/
├── 1、项目概述.md             # Tổng quan dự án
├── 2、架构概览.md             # Tổng quan kiến trúc
├── 3、工作流程.md             # Quy trình làm việc
├── 4、深入探索/               # Phân tích chi tiết
│   ├── 配置与基础设施域.md
│   ├── 智能分析代理域.md
│   ├── 文档生成域.md
│   └── ...
└── 5、边界调用.md             # API boundaries
```

---

## ⚙️ Quy Trình Hoạt Động

Litho hoạt động qua **4 giai đoạn** tuần tự:

### Giai Đoạn 1: Preprocessing (Tiền Xử Lý)

**Mục đích**: Quét và phân tích codebase

**Các bước**:
1. Quét tất cả files trong project
2. Phân tích cấu trúc (classes, functions, imports)
3. Trích xuất comments và documentation
4. Xác định dependencies giữa các modules
5. Tạo **CodeInsight** cho mỗi file

**Model dùng**: `model_efficient` (vì phải xử lý nhiều files)

**Output**: Lưu vào Memory - project structure, code insights, relationships

### Giai Đoạn 2: Research & Analysis (Nghiên Cứu)

**Mục đích**: AI agents phân tích sâu codebase

**Các AI Agents chạy song song**:

| Agent | Nhiệm Vụ | Model |
|-------|----------|-------|
| **SystemContextResearcher** | Phân tích context hệ thống | Efficient |
| **DomainModulesDetector** | Xác định domain modules | Efficient |
| **ArchitectureResearcher** | Phân tích kiến trúc | Powerful |
| **WorkflowResearcher** | Trích xuất business workflows | Powerful |
| **BoundaryAnalyzer** | Phân tích API boundaries | Efficient |
| **KeyModulesInsight** | Deep dive vào modules quan trọng | Powerful |

**Model dùng**: Mix giữa `efficient` và `powerful` tùy độ phức tạp

**Output**: Research reports lưu vào Memory

### Giai Đoạn 3: Documentation Generation (Tạo Tài Liệu)

**Mục đích**: Tạo tài liệu markdown có cấu trúc

**Các Editor Agents**:

| Agent | Tạo Gì | Model |
|-------|--------|-------|
| **OverviewEditor** | Project overview | Powerful |
| **ArchitectureEditor** | C4 architecture diagrams | Powerful |
| **WorkflowEditor** | Workflow documentation | Powerful |
| **BoundaryEditor** | API documentation | Powerful |
| **KeyModulesInsightEditor** | Deep dive articles | Powerful |

**Model dùng**: Chủ yếu `model_powerful` để đảm bảo chất lượng

**Output**: Các file .md trong document tree

### Giai Đoạn 4: Verification & Enhancement (Kiểm Tra)

**Mục đích**: Validate và fix syntax errors

**Các bước**:
1. Validate Mermaid diagram syntax
2. Check document completeness
3. Auto-repair broken diagrams
4. Generate summary reports

**Output**: Ghi files ra disk (`./litho.docs/`)

---

## 💰 Tối Ưu Chi Phí

### Chiến Lược Tiết Kiệm

#### 1. Dùng Cache

```bash
# Cache được bật mặc định, lưu tại .litho/cache/
# Khi chạy lần 2, sẽ reuse cached results

# Nếu muốn tắt cache (không khuyến nghị)
deepwiki-rs -p ./src --no-cache
```

**Tiết kiệm**: ~70-90% chi phí khi chạy lại

#### 2. Chọn Model Rẻ

```bash
# Dùng cùng model rẻ cho cả 2 loại
deepwiki-rs -p ./src \
  --model-efficient gemini-1.5-flash \
  --model-powerful gemini-1.5-flash
```

**Tiết kiệm**: ~80% so với dùng GPT-4o

#### 3. Skip Các Giai Đoạn Không Cần

```bash
# Nếu đã có preprocessing cache
deepwiki-rs -p ./src --skip-preprocessing

# Nếu chỉ muốn generate docs lại
deepwiki-rs -p ./src --skip-preprocessing --skip-research
```

**Tiết kiệm**: ~50-70% chi phí

#### 4. Giảm Max Parallels

```bash
# Chạy tuần tự thay vì song song (chậm hơn nhưng ít lỗi)
deepwiki-rs -p ./src --max-parallels 1
```

**Tiết kiệm**: Tránh bị rate limit → ít retry → ít phí

#### 5. Lọc Files Không Cần Thiết

Chỉnh file `litho.toml`:

```toml
[ignore_patterns]
directories = [
    "node_modules",
    "target",
    "test",           # Bỏ test files
    "examples",       # Bỏ examples
]
extensions = [
    ".test.js",
    ".spec.ts",
]
```

**Tiết kiệm**: ~30-50% nếu project có nhiều test files

### Bảng So Sánh Chi Phí

Dự án mẫu: **5000 files, ~2M tokens input**

| Provider | Model Setup | Chi Phí Ước Tính | Thời Gian |
|----------|-------------|------------------|-----------|
| **Gemini Flash** | flash/flash | ~$0.15 | ~5 phút |
| **DeepSeek** | chat/chat | ~$0.28 | ~7 phút |
| **OpenAI Mini** | 4o-mini/4o-mini | ~$0.30 | ~6 phút |
| **Mixed (Budget)** | 4o-mini/4o | ~$1.50 | ~8 phút |
| **OpenAI Premium** | 4o/4o | ~$5.00 | ~10 phút |
| **Claude** | haiku/sonnet | ~$3.50 | ~12 phút |
| **Ollama** | llama3.1/llama3.1 | $0 (FREE) | ~30 phút |

---

## 🔧 Xử Lý Lỗi Thường Gặp

### Lỗi 1: "API key not found"

```bash
Error: LLM API key not configured
```

**Nguyên nhân**: Chưa set API key

**Giải pháp**:
```bash
# Cách 1: Environment variable
export LITHO_LLM_API_KEY="your-key-here"

# Cách 2: Command line
deepwiki-rs --llm-api-key "your-key-here" -p ./src

# Cách 3: File litho.toml
# Chỉnh api_key = "your-key" trong file
```

### Lỗi 2: "Rate limit exceeded"

```bash
Error: Rate limit reached for requests
```

**Nguyên nhân**: Gọi API quá nhanh

**Giải pháp**:
```bash
# Giảm parallel processing
deepwiki-rs -p ./src --max-parallels 1

# Hoặc tăng retry delay trong litho.toml
[llm]
retry_delay_ms = 5000  # Tăng từ 2000 lên 5000
```

### Lỗi 3: "Connection timeout"

```bash
Error: Request timeout after 120 seconds
```

**Nguyên nhân**: API slow hoặc model quá lớn

**Giải pháp**:
```bash
# Tăng timeout
deepwiki-rs -p ./src

# Trong litho.toml:
[llm]
timeout_seconds = 300  # Tăng từ 120 lên 300
```

### Lỗi 4: "Out of memory"

```bash
Error: Cannot allocate memory
```

**Nguyên nhân**: Project quá lớn

**Giải pháp**:
```bash
# Giảm max_tokens
deepwiki-rs -p ./src --max-tokens 4000

# Chia nhỏ project
deepwiki-rs -p ./src/module1
deepwiki-rs -p ./src/module2
```

### Lỗi 5: "Invalid Mermaid syntax"

```bash
Warning: Mermaid diagram validation failed
```

**Nguyên nhân**: AI tạo diagram sai syntax

**Giải pháp**: Litho tự động sửa ở giai đoạn 4. Nếu vẫn lỗi:

```bash
# Dùng Mermaid Fixer (companion tool)
# https://github.com/sopaco/mermaid-fixer
```

### Lỗi 6: Cache bị corrupt

```bash
Error: Failed to deserialize cache
```

**Giải pháp**:
```bash
# Xóa cache và chạy lại
rm -rf .litho/cache/
deepwiki-rs -p ./src --force-regenerate
```

---

## ❓ Câu Hỏi Thường Gặp

### 1. Litho có hỗ trợ ngôn ngữ lập trình nào?

Hỗ trợ đầy đủ:
- **Backend**: Rust, Python, Java, JavaScript, TypeScript, Go, Kotlin, C#
- **Frontend**: React, Vue, Svelte

Hỗ trợ cơ bản (extract structure): Hầu hết ngôn ngữ phổ biến khác

### 2. Litho có gửi code của tôi lên đâu không?

- **Có**, nếu bạn dùng cloud LLM (OpenAI, Claude, etc.) - code sẽ được gửi qua API
- **Không**, nếu bạn dùng Ollama (chạy local)

**Khuyến nghị**: Đọc Privacy Policy của provider bạn chọn.

### 3. Tôi có thể dùng Litho offline không?

**Có**, với Ollama:
```bash
ollama pull llama3.1
deepwiki-rs --llm-provider ollama -p ./src
```

### 4. File cache lưu ở đâu? Có thể xóa không?

Cache lưu tại `.litho/cache/` trong project folder.

```bash
# Xem kích thước cache
du -sh .litho/

# Xóa cache
rm -rf .litho/

# Hoặc dùng flag
deepwiki-rs --force-regenerate
```

### 5. Litho có thể tích hợp vào CI/CD không?

**Có**, ví dụ với GitHub Actions:

```yaml
name: Generate Docs
on: [push]
jobs:
  docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions-rs/toolchain@v1
      - run: cargo install deepwiki-rs
      - run: deepwiki-rs -p ./src
        env:
          LITHO_LLM_API_KEY: ${{ secrets.LITHO_API_KEY }}
      - uses: actions/upload-artifact@v2
        with:
          name: docs
          path: litho.docs/
```

### 6. Tài liệu sinh ra có chính xác không?

**Phụ thuộc vào**:
- Chất lượng model (GPT-4o > GPT-4o-mini > Llama)
- Chất lượng code (code có comments tốt → docs tốt)
- Cấu trúc project (organized project → better analysis)

**Khuyến nghị**: Review tài liệu trước khi sử dụng.

### 7. Mất bao lâu để chạy?

**Phụ thuộc vào**:
- Kích thước project
- Model chọn
- API speed
- Có cache hay không

**Ước tính**:
- Dự án nhỏ (< 100 files): 1-2 phút
- Dự án trung bình (< 1000 files): 5-10 phút
- Dự án lớn (< 10000 files): 20-60 phút

### 8. Tôi có thể customize template không?

Hiện tại: **Không** (templates được hardcode trong agents)

**Workaround**: Fork project và chỉnh sửa prompts trong các agent files.

### 9. Litho có hỗ trợ monorepo không?

**Có**, nhưng nên chạy từng sub-project:

```bash
deepwiki-rs -p ./packages/frontend -o ./docs/frontend
deepwiki-rs -p ./packages/backend -o ./docs/backend
```

### 10. Chi phí thực tế là bao nhiêu?

**Project của tôi (Litho chính nó)**:
- Files: ~75 Rust files
- Tokens: ~500K input
- Provider: DeepSeek
- Chi phí: **~$0.07 USD**
- Thời gian: ~3 phút

---

## 📚 Tài Liệu Tham Khảo

- **GitHub**: https://github.com/sopaco/deepwiki-rs
- **English README**: [README.md](./README.md)
- **Chinese README**: [README_zh.md](./README_zh.md)
- **Config Example**: [litho.toml.example](./litho.toml.example)
- **Claude Code Guide**: [CLAUDE.md](./CLAUDE.md)

### Ecosystem Tools

- **Litho Book**: Markdown reader để xem docs - https://github.com/sopaco/litho-book
- **Mermaid Fixer**: Tool fix Mermaid syntax - https://github.com/sopaco/mermaid-fixer

---

## 🆘 Hỗ Trợ

### Gặp vấn đề?

1. **Đọc phần troubleshooting** phía trên
2. **Check GitHub Issues**: https://github.com/sopaco/deepwiki-rs/issues
3. **Tạo issue mới** với thông tin:
   - Litho version: `deepwiki-rs --version`
   - OS: `uname -a`
   - Command bạn chạy
   - Error message đầy đủ
   - Config file (ẩn API key)

### Muốn contribute?

```bash
# Fork repo
git clone https://github.com/your-username/deepwiki-rs
cd deepwiki-rs

# Tạo branch
git checkout -b feature/my-feature

# Code, test, commit
cargo test
git commit -m "Add feature X"

# Push và tạo PR
git push origin feature/my-feature
```

---

## 📝 License

MIT License - Xem file [LICENSE](./LICENSE)

---

**Chúc bạn tạo tài liệu thành công với Litho! 🎉**

*Nếu thấy hữu ích, hãy cho repo một ⭐ trên GitHub!*
