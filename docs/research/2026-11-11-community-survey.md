# Khảo sát: kế thừa gì từ cộng đồng cho bộ harness Roo Code + Codex trên Yocto / TelAF / C++

- **Ngày khảo sát:** 2026-11-11
- **Mục đích:** xác định phần nào của bộ harness có thể kế thừa từ cộng đồng, phần nào phải tự viết, trước khi chốt thiết kế.
- **Phương pháp:** bốn agent khảo sát web chạy song song, mỗi agent một mảng (Roo Code, Codex, tài sản cộng đồng, mảng chuyên môn Yocto / TelAF / C++). Kết quả được tổng hợp lại ở đây.
- **Giới hạn:** người tổng hợp chưa kiểm chứng độc lập các kết luận dưới đây. Mục nào agent khảo sát không xác nhận được đều ghi rõ "chưa xác nhận". Số sao và ngày cập nhật là tại thời điểm khảo sát.

## 1. Bốn phát hiện làm thay đổi kế hoạch

1. **Roo Code đã ngừng phát triển.** Extension đóng cửa ngày 2026-05-15, repo bị archive ở v3.54.0 và bị gỡ khỏi Marketplace. Bản kế tục đang hoạt động là fork cộng đồng **Zoo Code** (v3.96.0 ngày 2026-11-07, phát hành hằng tuần). Zoo Code giữ nguyên `.roo/`, `.roomodes`, `.rooignore` nhưng đổi tiền tố setting từ `roo-cline.*` sang `zoo-code.*`.
2. **Codex chỉ dùng Responses API.** `wire_api = "chat"` là lỗi cứng trong source. Endpoint "tương thích OpenAI" chỉ phục vụ `/v1/chat/completions` sẽ không chạy với Codex bản mới; nó phải phục vụ `POST /v1/responses`.
3. **Roo/Zoo không có hook, không chạy subtask song song, không có web search.** Subtask của Orchestrator luôn mở context mới và chạy tuần tự. Song song và hook chỉ có ở phía Codex.
4. **Không có tài sản agent nào cho Legato / TelAF.** Phần này phải tự viết hoàn toàn. Với Yocto thì đã có vài bộ skill dùng làm nền được.

## 2. Roo Code / Zoo Code

Trạng thái mô tả dưới đây là của Roo Code bản cuối (v3.53/3.54). Zoo Code kế thừa, trừ khi ghi khác.

| Chủ đề | Thực tế | Chia sẻ qua repo được không |
|---|---|---|
| Custom mode | `.roomodes` (YAML/JSON) ở gốc repo. Trường: `slug`, `name`, `description`, `roleDefinition`, `whenToUse`, `customInstructions`, `groups`. Giới hạn file được sửa bằng `["edit", {fileRegex, description}]`. `.roo/modes/` không được hỗ trợ. | Có |
| Model theo mode | Gán API configuration profile cho từng mode trong tab Prompts. Nằm trong trạng thái extension, không phải file repo. | Không, mỗi người tự cấu hình |
| Orchestrator / `new_task` | Subtask có context hoàn toàn riêng. Chỉ `message` (và `todos`) đi xuống, chỉ bản tóm tắt `attempt_completion` đi lên. Chạy tuần tự, một con tại một thời điểm. Tiếp tục một subtask cũ: chưa xác nhận. | Có (qua mode) |
| Lệnh tự chạy | `allowedCommands` và `deniedCommands`. Khớp theo tiền tố, tiền tố dài nhất thắng, deny thắng khi hoà. Lệnh nối (`&&`, `||`, `;`, `|`) bị tách ra, một lệnh con bị cấm là cả dòng bị cấm. | Là setting VS Code; đặt trong `.vscode/settings.json` chưa được kiểm chứng |
| Skill | Hỗ trợ `SKILL.md`. Vị trí: `.roo/skills/`, `.agents/skills/`, và biến thể theo mode `skills-{mode}/`. `.roo` ưu tiên hơn `.agents`. | Có |
| `AGENTS.md` và rule | Tự nạp `AGENTS.md` ở gốc workspace. Rule ở `.roo/rules/` và `.roo/rules-{slug}/`. | Có |
| Hook | Không có. Cách thay thế: git pre-commit hook, tool MCP, chỉ dẫn trong mode, `fileRegex`, danh sách cấm lệnh. Terminal của Roo đặt biến môi trường `ROO_ACTIVE`. | — |
| Nén context | Tóm tắt bằng LLM, mặc định bật, ngưỡng chỉnh được, prompt nén tuỳ biến được. Không có tính năng trí nhớ. | Một phần |
| Index codebase | Cần Qdrant và một embedding provider (hỗ trợ endpoint tương thích OpenAI và Ollama). Zoo Code thêm Semble, tìm code cục bộ không cần Qdrant. | — |
| MCP | `.roo/mcp.json` ở mức dự án, hỗ trợ stdio, có `alwaysAllow` theo từng tool. | Có |
| Web search | Không có. Tool trình duyệt cũng đã bị gỡ (02/2026). Chỉ còn đường MCP. | — |
| Tool calling | Kiểu XML đã bị gỡ; endpoint phải hỗ trợ native function calling. | — |

Nguồn: https://github.com/RooCodeInc/Roo-Code, https://github.com/Zoo-Code-Org/Zoo-Code, tài liệu tại `roocodeinc.github.io/Roo-Code` (các trang custom-modes, boomerang-tasks, auto-approving-actions, skills, custom-instructions, codebase-indexing, mcp).

Chưa xác nhận: Zoo Code có chạy subtask song song hay không (README nói có, source cho thấy vẫn một con); tài liệu của Destructive Command Guard trong Zoo Code.

## 3. Codex

Dựa trên tài liệu chính thức và `openai/codex` (bản ổn định `rust-v0.167.1`).

| Chủ đề | Thực tế | Chia sẻ qua repo được không |
|---|---|---|
| Endpoint tuỳ chỉnh | `[model_providers.<id>]` với `base_url`, `env_key`. Chỉ `wire_api = "responses"`. Tool calling với model không phải của OpenAI không được đảm bảo, phải thử. | Không, chỉ ở `~/.codex/config.toml` |
| Profile | Mỗi profile là một file `~/.codex/<name>.config.toml`, chọn bằng `--profile`. | Không |
| `AGENTS.md` | Nạp từ gốc dự án xuống thư mục đang đứng, nối lại, file sau thắng. Giới hạn 32 KiB tổng. Thư mục con bên dưới không được đọc. | Có |
| Skill | `SKILL.md` trong `.agents/skills/` (từ thư mục đang đứng lên tới gốc repo) và `~/.agents/skills`. Danh sách skill bị giới hạn ở 2% context. | Có |
| Duyệt lệnh | File rule Starlark `prefix_rule(pattern, decision = allow|prompt|forbidden)` trong `.codex/rules/`. Luật chặt nhất thắng. Kiểm thử bằng `codex execpolicy check`. Chỉ nạp khi dự án được tin cậy. | Có |
| Hook | Có: `PreToolUse` (chặn hoặc sửa lệnh), `PostToolUse`, `PreCompact`/`PostCompact`, `SessionStart`, `Stop`, `SubagentStart`/`SubagentStop`. Mỗi hook phải được duyệt tin cậy theo hash. | Có |
| Sub-agent | Chạy song song. Định nghĩa agent bằng file TOML trong `.codex/agents/` với `model` riêng. Tiếp tục được một sub-agent đang có với context của nó. Không đặt riêng được sandbox hay provider cho từng agent. | Có |
| Phiên | `codex resume`, `codex fork`. | — |
| Nén context | Tự động, có ngưỡng và prompt tuỳ biến. Tính năng memories mặc định tắt. | Một phần |
| MCP | Server stdio trong `[mcp_servers.<name>]`. Codex không còn chạy được như một MCP server. | Có (dự án tin cậy) |
| Chạy không tương tác | `codex exec` với `--json`, `--output-schema`, `--ephemeral`. Phù hợp để chạy bộ tác vụ đánh giá. | — |
| Web search | Có sẵn nhưng gần như không dùng được với provider tuỳ chỉnh. | — |

Nguồn: tài liệu Codex (mục lục tại https://developers.openai.com/codex/llms.txt), https://github.com/openai/codex.

## 4. Phần dùng chung và phần riêng

| Thành phần | Dùng chung | Riêng Roo/Zoo | Riêng Codex |
|---|---|---|---|
| Luật dự án | `AGENTS.md` ở gốc repo | — | — |
| Skill | `.agents/skills/` | `skills-{mode}/` nếu cần | — |
| Tool cục bộ | Script Python + một MCP server stdio | `.roo/mcp.json` | `.codex/config.toml` |
| Vai trò agent | Cùng định dạng brief và cùng thư mục trí nhớ | `.roomodes` | `.codex/agents/*.toml` |
| Chặn lệnh nguy hiểm | Một danh sách gốc, sinh ra hai định dạng | `deniedCommands` | `.codex/rules/*.rules` |
| Kiểm tra sau khi sửa | Git pre-commit hook + script kiểm tra gọi được bằng tay | Chỉ dẫn trong mode (không có hook) | Hook `PostToolUse` |
| Chạy song song | — | Không có | Sub-agent song song |
| Model theo vai trò | — | Cấu hình tay trong extension | Trường `model` trong file agent |

Hệ quả: một `AGENTS.md` và một `.agents/skills/` phục vụ được cả hai công cụ mà không cần công cụ đồng bộ. Những thứ không đưa vào repo được (model theo mode của Roo, provider của Codex) cần một script cài đặt hoặc hướng dẫn một lần cho mỗi máy.

## 5. Tài sản cộng đồng

### Kế thừa

| Tài sản | Dùng để làm gì | Giấy phép |
|---|---|---|
| Chuẩn `AGENTS.md` — https://agents.md | Nguồn sự thật chung | MIT |
| Agent Skills (`SKILL.md`) — https://agentskills.io | Định dạng skill chung. Frontmatter bắt buộc: `name` (trùng tên thư mục) và `description` | Apache-2.0 |
| obra/superpowers — https://github.com/obra/superpowers | Skill quy trình: brainstorm, plan, TDD, debug có hệ thống, kiểm chứng, review. Chép các thư mục skill vào repo và rút gọn cho model yếu. Không hỗ trợ chính thức Roo/Zoo | MIT |
| MADR — https://github.com/adr/madr | Mẫu ADR | MIT / CC0 |
| universal-ctags + ripgrep | Chỉ mục symbol và tìm kiếm, hoàn toàn offline | GPL-2.0 / Unlicense |
| Higangssh/yocto-agent-skills — https://github.com/Higangssh/yocto-agent-skills | Nền cho skill Yocto (debug bitbake, review recipe/layer, rootfs, BSP). Cần đối chiếu với kirkstone | MIT |
| yoctoproject/vscode-bitbake — https://github.com/yoctoproject/vscode-bitbake | Hỗ trợ soạn thảo recipe, kèm language server | MIT |
| tree-sitter-bitbake — https://github.com/tree-sitter-grammars/tree-sitter-bitbake | Parser cục bộ cho recipe | MIT |
| Luật `prefix_rule` của Codex và danh sách lệnh của Zoo | Chặn lệnh nguy hiểm bằng cơ chế sẵn có | — |
| CodeChecker — https://github.com/Ericsson/codechecker | Gom kết quả phân tích tĩnh, chạy offline | Apache-2.0 |

### Mượn ý tưởng

| Tài sản | Ý tưởng đáng lấy |
|---|---|
| OpenSpec — https://github.com/Fission-AI/OpenSpec | Đề xuất thay đổi dạng spec delta làm bản bàn giao từ orchestrator sang implementer. Hợp với codebase có sẵn |
| Compound Engineering — https://github.com/EveryInc/compound-engineering-plugin | Bước "ghi lại bài học" sau mỗi tác vụ |
| Mẫu memory-bank (roo-code-memory-bank, RooFlow) | Vài file markdown: bối cảnh hiện tại, nhật ký quyết định, tiến độ. Các bộ gốc đã cũ, chỉ lấy mẫu |
| Aider repo map — https://github.com/Aider-AI/aider | Bản đồ repo rút gọn bằng tree-sitter và xếp hạng |
| affaan-m/ECC — https://github.com/affaan-m/ECC | Cấu trúc skill C++: coding standard, reviewer, build resolver |
| Bản vá `AGENTS.md` của OE-core | Quy ước commit khi có AI tham gia (thẻ `AI-Generated:`) |
| promptfoo, Harbor, mini-swe-agent | Khung chạy bộ tác vụ đánh giá. Harbor cần Docker |

### Thử nghiệm sau

- Serena (https://github.com/oraios/serena): điều hướng symbol qua LSP, có hỗ trợ C/C++.
- Semble (https://github.com/MinishLab/semble): tìm code chạy CPU, đã tích hợp trong Zoo Code. Chưa rõ có cần tải model một lần hay không.
- codebase-memory-mcp (https://github.com/DeusData/codebase-memory-mcp): binary đơn, không cần runtime.
- BeaCox/gdb-mcp (https://github.com/BeaCox/gdb-mcp): chỉ có ích nếu sau này agent được phép chạm vào target.
- destructive_command_guard: giấy phép có điều khoản riêng, cần pháp chế xem trước.

Chưa agent nào kiểm tra các công cụ này ở quy mô một cây source Yocto lớn.

### Bỏ qua

BMAD-METHOD và spec-kit (quá nặng cho context ngắn), Agent OS, roo-commander (đã archive), các bộ SPARC (cũ từ 2025), ConPort, adr-tools, các công cụ vector search cần vector database ngoài.

### Phải tự viết

- Toàn bộ skill Legato / TelAF (`.adef`, `.cdef`, `.sdef`, `.api`, `mkapp`).
- Tool bọc `bitbake -e`, `bitbake-layers`, `devtool`, `oe-pkgdata-util`. Không tìm thấy MCP server nào làm việc này.
- Skill phân tích log: lỗi linker, core dump, kernel oops, log framework.
- Skill chọn design pattern C++.

## 6. Mảng chuyên môn

### TelAF trên SA525M

- **Source công khai:** https://git.codelinaro.org/clo/le/platform/TelAF, nhánh mặc định `telaf.lnx.1.1`. Fork Legato ở `clo/le/legato-af`.
- **Giấy phép:** TelAF là BSD-3-Clause-Clear; Legato là MPL-2.0.
- **Nền Yocto (đọc từ manifest CLO):** SA525M (`LE.UM.7.4.1.1`) dùng poky **kirkstone**; SA515M dùng thud.
- **Tài liệu:** user guide 80-41102-1 công khai trên docs.qualcomm.com. Repo có thư mục `doc/` (rst và Doxygen) nên dựng được tài liệu offline.
- **Build app:** `source /opt/qct/sa525m/environment-setup-telaf-app`, sau đó `mkapp -t sa525m -i $TELAF_INTERFACES -i $TELAF_MNGD_INTERFACES <app>.adef`, cho ra file `.update`.
- **Build TelAF:** `source set_af_env.sh sa525m; build-sa525m-af`.
- **Log:** macro `LE_INFO` và công cụ `log level` kế thừa từ Legato. Trên target dùng `logread` hay `journalctl`: chưa xác nhận.
- **Lưu ý:** bản phát hành đầy đủ cho SA525M đi qua Qualcomm ChipCode và cần đăng nhập; đường công khai bị giới hạn.

### C++ theo MISRA và security

- **clang-tidy:** không có nhóm check `misra-` hay `autosar-`. Có `cert-`, `cppcoreguidelines-`, `bugprone-`, clang-analyzer. Dùng cho CERT và Core Guidelines, chỉ trùng gián tiếp với MISRA.
- **cppcheck:** addon MISRA nhắm vào MISRA C 2012, không phải MISRA C++. Dùng để bắt lỗi chung.
- **codeql-coding-standards** (https://github.com/github/codeql-coding-standards): bộ luật mở duy nhất có phủ MISRA C++:2023 thật sự. CodeQL CLI không phải mã nguồn mở, cần kiểm tra điều khoản cho code thương mại.
- **Kiểm tra MISRA C++:2023 có chứng nhận:** chỉ có ở công cụ thương mại (Helix QAC, Parasoft).
- **Nội dung luật MISRA:** tài liệu mua theo từng người dùng và không được chia sẻ. Không đưa nguyên văn luật vào knowledge base. Chỉ lưu mã luật, diễn giải tự viết và ánh xạ sang công cụ.
- **Hướng dẫn được phép phân phối lại:** SEI CERT C/C++ (CC-BY 4.0), CWE (kèm ghi chú của MITRE), C++ Core Guidelines.

## 7. Tài liệu đóng gói offline được

| Nguồn | Giấy phép | Dạng offline |
|---|---|---|
| Yocto docs — https://docs.yoctoproject.org/downloads.html | CC-BY-SA-2.0-UK | EPUB/PDF, HTML theo release (kirkstone ở `/4.0/`), source Sphinx trong git `yocto-docs` |
| BitBake manual | CC-BY 2.5 | Dựng từ `doc/` trong git bitbake |
| TelAF docs | Bản trên web thuộc bản quyền Qualcomm | Dựng từ `doc/` của repo BSD |
| Legato docs — https://docs.legato.io/latest/ | Chưa rõ quyền phân phối lại; đóng băng ở 21.05.0 | Chỉ nên mirror nội bộ |
| Linux man-pages | Tự do theo từng trang (chưa kiểm chứng) | Tarball từ kernel.org |
| cppreference | CC-BY-SA 3.0 / GFDL (chưa kiểm chứng) | Bản archive mới nhất là 2025-02 |

## 8. Tác động tới thiết kế

1. **Nền chung là `AGENTS.md` + `.agents/skills/`.** Cả hai công cụ đọc trực tiếp, không cần đồng bộ.
2. **Chạy song song chỉ thực hiện được ở phía Codex.** Phía Roo/Zoo, orchestrator giao việc tuần tự. Thiết kế phân rã tác vụ phải chạy đúng ở cả hai chế độ.
3. **Vì Roo/Zoo không có hook,** các kiểm tra bắt buộc (format, commit rule, `bitbake -p`) phải nằm ở git hook và ở script kiểm tra mà agent được yêu cầu chạy, không dựa vào hook của công cụ. Hook của Codex là lớp bổ sung.
4. **Danh sách lệnh được phép / bị cấm cần một nguồn gốc duy nhất,** từ đó sinh ra setting của Roo/Zoo và file rule của Codex.
5. **Cấu hình theo máy là không tránh được** (model theo mode, provider). Cần script cài đặt và tài liệu hướng dẫn.
6. **Trí nhớ dự án phải nằm trên đĩa,** vì cả hai công cụ đều không có tính năng trí nhớ đủ tin cậy và subtask của Roo/Zoo luôn mở context mới.
7. **Knowledge base ghim theo kirkstone** và theo đúng nhánh TelAF của dự án.
8. **MISRA:** agent làm việc với mã luật và công cụ kiểm tra, không với nguyên văn luật.

## 9. Câu hỏi mở, cần kiểm tra trên máy dự án

1. Extension đang cài là Roo Code (bản nào) hay Zoo Code?
2. Endpoint có phục vụ Responses API (`/v1/responses`) không, và Codex đang cài là bản nào?
3. Endpoint có hỗ trợ native function calling cho mọi model định dùng không? Có API embedding không?
4. Baseline chính xác: bản SA525M.LE, nhánh `telaf.lnx.1.1.cNN/rNN`, Yocto là kirkstone gốc hay đã bị nhà cung cấp sửa?
5. Log trên target lấy bằng gì, core dump nằm ở đâu?
6. Build được điều khiển bằng script nào, SDK cài ở đường dẫn nào?
7. Yêu cầu là MISRA C++:2008, MISRA C++:2023 hay AUTOSAR C++14? Có công cụ phân tích thương mại nào không?
8. Giấy phép MISRA của tổ chức có cho phép dùng trong knowledge base nội bộ cho AI không?
9. Chuẩn C++ và phiên bản toolchain trong SDK?
10. Tài liệu Qualcomm có được phép mirror vào knowledge base offline nội bộ không?
