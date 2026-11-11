# Thiết kế: bộ harness dùng chung cho Roo/Zoo Code và Codex trên Yocto / TelAF / C++

- **Ngày:** 2026-11-11
- **Trạng thái:** bản nháp chờ duyệt
- **Tài liệu liên quan:** [Khảo sát cộng đồng](../../research/2026-11-11-community-survey.md)

Trong tài liệu này, câu nào ghi **(quyết định của người viết)** là lựa chọn của người thiết kế, chủ dự án chưa xác nhận riêng. Những phần còn lại là điều chủ dự án đã nêu hoặc đã đồng ý.

## 1. Ý định

Một kỹ sư làm việc trên dự án embedded Linux lớn (nhiều layer Yocto, nhiều app TelAF/Legato, C++17) dùng AI coding agent qua một endpoint tuỳ chỉnh. Hiện tại agent gây khó chịu ở ba điểm:

1. Dừng lại xin phép chạy lệnh quá nhiều lần.
2. Không theo format và commit rule của dự án.
3. Mất mạch sau khi context bị nén, và mỗi agent mới phải tìm hiểu lại dự án từ đầu (cấu trúc layer, recipe nằm đâu, cách build từng component, source tổ chức ra sao).

Bộ harness phải cho phép agent tự xử lý trọn một tác vụ điển hình với ít can thiệp, kể cả khi không có web search hoặc không có mạng, và không làm hỏng build hay cache.

### Bối cảnh cố định

- **Nền tảng:** TelAF bản mới nhất trên SA525M. Theo manifest công khai, nền Yocto là kirkstone; phiên bản chính xác do bước khảo sát trên máy dự án xác định.
- **Build full khoảng 8 giờ.** Build lại một recipe hoặc một app TelAF (bằng `mkapp` và các lệnh `mk` khác) thì nhanh.
- **Chuẩn code:** MISRA C++:2023 trên C++17, kiểm tra bằng Coverity. Chưa có coding standard nội bộ. Đã có CI.
- **Debug:** trên board thật, nhưng agent không được chạm vào board. Người lấy log về, agent điều tra từ log vào source.
- **Endpoint:** phục vụ Responses API, native function calling, có API embedding. Nhiều model: loại mạnh (Sonnet, Sol), loại rẻ (Haiku, Luna, Qwen 27B).
- **Hai công cụ thay thế nhau:** Roo Code (hoặc fork kế tục Zoo Code) và Codex. Mỗi phiên làm việc chỉ chọn một, cả hai phải làm được cùng bộ việc.
- **Máy dự án:** clone được từ GitHub, cài được Python, ripgrep, ctags. Không có đường đưa gì ngược về repo này.

## 2. Mục tiêu, ngoài phạm vi, điều phải tránh

### Mục tiêu

1. Agent tự chạy các lệnh an toàn mà không hỏi, và không thể chạy các lệnh nguy hiểm.
2. Agent chỉ dừng chờ người ở hai chỗ: khi xong tác vụ, và khi có nguy cơ vi phạm kiến trúc đã được ghi lại.
3. Kết quả tuân theo format và commit rule của dự án, được kiểm tra bằng máy.
4. Trạng thái công việc và hiểu biết về dự án nằm trên đĩa, sống sót qua compact, và dùng được khi đổi công cụ.
5. Model mạnh điều phối, model rẻ implement; skill viết cho model yếu nhất đọc được.
6. Mọi chức năng cốt lõi chạy được khi hoàn toàn offline. Mạng chỉ dùng để tải KB.
7. Một nguồn sự thật cho cả hai công cụ.
8. Harness tự cải thiện được tại chỗ, có người duyệt.
9. Sau một lần cài đặt, mọi việc thiết lập còn lại trên máy dự án đều có kịch bản soạn sẵn, chỉ cần kích hoạt.

### Ngoài phạm vi

- Deploy, flash, gdb trực tiếp hoặc bất kỳ thao tác nào trên board.
- Agent tự build image, SDK hoặc build full.
- Web search.
- Đồng bộ ngược từ máy dự án về repo này.
- Thay thế Coverity trong việc phán quyết MISRA.

### Điều phải tránh

- Agent xoá hoặc làm bẩn `tmp`, `sstate-cache`, `downloads`.
- Agent tự nới quyền cho chính nó mà không ai duyệt.
- Agent bịa tên API, biến Yocto hoặc lệnh khi không tra được.
- Agent báo "đã kiểm tra" ở mức cao hơn mức thực sự đã chạy.
- Nguyên văn luật MISRA, tài liệu hạn chế hoặc chi tiết nội bộ dự án xuất hiện trong repo public này.
- Một harness cần người trông chừng nhiều hơn chính agent.

## 3. Kiến trúc tổng thể

Hướng đã chọn: **lõi chung + bộ chuyển đổi sinh tự động**. Một thư mục nguồn chứa luật, skill, vai trò, chính sách lệnh và tool. Một bộ sinh tạo ra cấu hình riêng của từng công cụ. Không ai sửa tay file đã sinh.

```
        Repo harness (repo này, sau bàn giao là repo git cục bộ)
        ┌──────────────────────────────────────────────────────┐
        │ core/      luật, vai trò, chính sách lệnh, skill     │
        │ tools/     một lệnh dòng lệnh duy nhất (Python)      │
        │ adapters/  bộ sinh cho Roo/Zoo và cho Codex          │
        │ kb/        danh sách nguồn, script tải, đánh chỉ mục │
        │ evals/     dự án mẫu và bộ tác vụ đánh giá           │
        └───────────────┬──────────────────────────────────────┘
                        │ cài đặt + sinh cấu hình
        ┌───────────────▼───────────────┐   ┌──────────────────────────┐
        │ Workspace dự án                │   │ Thư mục trí nhớ dự án    │
        │ AGENTS.md, .agents/skills/     │◄──┤ (git cục bộ riêng)       │
        │ .roomodes, .roo/, .codex/      │   │ bản đồ, thẻ component,   │
        │ (loại khỏi git của dự án)      │   │ kiến trúc, ADR, bài học, │
        └───────┬───────────────┬───────┘   │ tác vụ, KB               │
                │               │           └──────────────────────────┘
          Roo / Zoo Code      Codex
          (tuần tự)           (song song được)
```

### Ba vùng

| Vùng | Nội dung | Vị trí |
|---|---|---|
| Harness | luật chung, skill, tool, vai trò, chính sách lệnh, bộ sinh, bộ đánh giá | repo này; trên máy dự án là một bản clone, từ đó phát triển độc lập |
| Trí nhớ dự án | mọi thứ đặc thù dự án | thư mục cục bộ trên máy dự án, có git riêng để xem lịch sử và hoàn tác **(quyết định của người viết)** |
| File cài vào workspace | cấu hình mà hai công cụ đọc | sinh trong workspace dự án, loại khỏi git của dự án |

Không có gì từ vùng trí nhớ hay workspace được quay về repo public.

### Bàn giao một lần

Repo này được dựng tới một bản hoàn chỉnh, máy dự án clone về **một lần**, và từ đó harness phát triển tiếp tại chỗ. Không có luồng cập nhật từ repo này sang máy dự án sau bàn giao, và không có luồng ngược lại. Hệ quả:

- Các mốc ở mục 14 là thứ tự dựng trong repo này, không phải các đợt phát hành.
- Bản bàn giao phải tự đủ: mọi thứ kế thừa được chép vào `vendor/`, không tải thêm lúc cài.
- Mọi giả định về dự án phải chung chung và tự dò; chỗ nào không dò được thì kịch bản hỏi người.
- Tài liệu bảo trì phải đủ để một người hoặc agent khác phát triển tiếp mà không có ngữ cảnh của bản thiết kế này.

### Cấu trúc repo harness

```
core/
  AGENTS.md.tmpl            luật chung, ngắn, trỏ tới skill và trí nhớ
  roles/                    sáu vai trò: mục đích, hạng model, file được sửa, skill
  policy/commands.yaml      nguồn gốc duy nhất của chính sách lệnh
  skills/
    process/                skill quy trình kế thừa, đã rút gọn
    yocto/  telaf/  cpp/  investigate/
    playbooks/              kịch bản thiết lập và bảo trì, kích hoạt bằng một lệnh
  templates/                mẫu plan, brief, thẻ component, ADR, bài học
tools/                      lệnh `harness` và thư viện của nó
adapters/roo/  adapters/codex/
kb/                         sources.yaml, script tải, script đánh chỉ mục
vendor/                     bản chép nguyên gốc từ cộng đồng + ghi chú nguồn, phiên bản, giấy phép
evals/                      dự án mẫu, tác vụ, kết quả mong đợi
docs/                       tài liệu bảo trì, thiết kế, khảo sát
install.py
```

### Cấu trúc thư mục trí nhớ (trên máy dự án)

```
project.yaml        dữ kiện dò được + cấu hình theo máy (đường dẫn, mức mạng, model)
map/                layer, recipe, component, cây source rút gọn, tags
cards/              mỗi component một thẻ
architecture.md     ràng buộc kiến trúc, viết dạng kiểm tra được
adr/                quyết định kiến trúc (mẫu MADR)
conventions.md      format, commit rule
dont.md             "đừng làm X vì Y"
lessons.md          bài học rút ra
friction.jsonl      nhật ký chỗ vấp, do tool tự ghi
tasks/<id>/         plan, các brief, tiến độ, bằng chứng, giả thuyết
kb/                 snapshot tài liệu, tài liệu nội bộ, chỉ mục
proposals/          đề xuất thay đổi harness chờ duyệt
```

## 4. Vai trò và quy trình

### Sáu vai trò

| Vai trò | Hạng model | Được làm | Không được làm |
|---|---|---|---|
| Điều phối | mạnh | làm rõ yêu cầu, lập plan, viết brief, nghiệm thu, đối chiếu kiến trúc | sửa source |
| Implement Yocto | rẻ | sửa `.bb`, `.bbappend`, `.conf`, `.inc`, patch | sửa source C++, định nghĩa TelAF |
| Implement TelAF | rẻ | sửa `.adef`, `.cdef`, `.sdef`, `.api` và code component | sửa recipe |
| Implement C++ | rẻ | sửa source và test C++ | sửa recipe, định nghĩa TelAF |
| Điều tra | mạnh | đọc log, báo cáo Coverity, source; ghi giả thuyết và bằng chứng | sửa bất cứ gì ngoài thư mục tác vụ |
| Review | mạnh hoặc trung bình | đối chiếu diff với brief, chuẩn code, commit rule, kiến trúc | sửa source |

Sáu vai trò và việc tách implementer theo mảng là **(quyết định của người viết)**. Mỗi vai trò có danh tính cố định và đọc cùng một trí nhớ; phiên chạy thì dùng xong bỏ.

### Vòng đời một tác vụ

1. **Làm rõ và lập plan** (điều phối): đọc bản đồ, thẻ liên quan, `architecture.md`. Viết `plan.md`.
2. **Đối chiếu kiến trúc:** nếu plan đụng một ràng buộc trong `architecture.md` hoặc một ADR, dừng lại, nêu ràng buộc bị đụng và các phương án. Đây là một trong hai điểm dừng.
3. **Cắt brief:** mỗi brief khép kín và không chồng file với brief khác.
4. **Implement:** implementer đọc brief và thẻ được chỉ định, sửa, chạy kiểm tra tới bậc brief yêu cầu, ghi tiến độ.
5. **Review:** đối chiếu diff với brief, quy ước và kiến trúc.
6. **Ghi lại:** cập nhật thẻ component và bài học. Bắt buộc trước khi báo xong.
7. **Trình kết quả:** diff, bậc kiểm chứng đã đạt, commit message soạn sẵn. Đây là điểm dừng thứ hai. Agent không tự commit **(quyết định của người viết)**.

Việc nhỏ (một file, không đụng kiến trúc) đi thẳng tới implementer, bỏ bước 1 đến 3.

### Nội dung một brief

Mục tiêu; file được sửa; thẻ và tài liệu phải đọc; các bước nếu đã biết; lệnh kiểm tra và bậc phải đạt; tiêu chí xong; điều kiện phải dừng và báo lại. Implementer có một luật cứng: cần sửa file ngoài brief thì dừng và báo điều phối, không tự mở rộng.

### Sống sót qua compact và đổi công cụ

Mọi trạng thái nằm trong `tasks/<id>/`. Sau compact, hoặc khi mở phiên mới ở công cụ kia, việc đầu tiên agent làm là đọc `progress.md` của tác vụ đang mở. Trong cùng một tác vụ, điều phối tiếp tục giao cho đúng sub-agent đang làm nếu công cụ cho phép (Codex), không thì giao brief kế tiếp cho phiên mới (Roo/Zoo).

### Chia việc song song

- Brief ghi rõ danh sách file sở hữu. Hai brief không được có file chung.
- Codex chạy các brief độc lập song song. Roo/Zoo chạy lần lượt. Quy trình phải đúng ở chế độ tuần tự; song song chỉ để nhanh hơn.
- Lệnh build luôn đi qua một khoá: mỗi lúc chỉ một lệnh build chạy, kể cả khi nhiều sub-agent cùng làm.

## 5. Chính sách lệnh và hàng rào

Một file `policy/commands.yaml` là nguồn gốc. Bộ sinh tạo ra danh sách cho phép / cấm của Roo/Zoo và file rule của Codex.

| Mức | Ví dụ | Hành vi |
|---|---|---|
| Tự chạy | tìm kiếm, đọc git, lệnh `harness`, `bitbake -e`, `bitbake -p`, `bitbake-layers show-*`, format, `clang-tidy` | không hỏi |
| Tự chạy, xếp hàng | build một recipe, build một app bằng `mk`, `devtool` | không hỏi, qua khoá build, có giới hạn thời gian |
| Phải hỏi | build image hoặc SDK, `git commit`, `git push`, `cleansstate` | dừng chờ người |
| Cấm | xoá `tmp`, `sstate-cache`, `downloads`; `cleanall`; build `world`; `adb`, `fastboot`, `ssh` tới target; `git push --force`, `git reset --hard`, `git clean` | từ chối |

**Lệnh không nằm trong mức nào** thì rơi vào mức "phải hỏi", và lần hỏi đó được ghi vào nhật ký chỗ vấp để vòng tự cải thiện đề xuất xếp nó vào một mức **(quyết định của người viết)**. Danh sách khởi đầu vì thế phải đủ rộng cho công việc thường ngày, nếu không mục tiêu "chỉ dừng ở hai chỗ" không đạt.

**Mức độ bảo đảm:** danh sách cho phép / cấm khớp theo tiền tố nên là hàng rào chống sai sót, không phải ranh giới bảo mật; một lệnh bọc trong `bash -c` hoặc trong script có thể lọt. Thiết kế chấp nhận điều đó và bù bằng nhiều lớp: cấm các dạng bọc lệnh phổ biến, lệnh `harness` tự kiểm tra, sandbox và hook phía Codex, và một kịch bản tuỳ chọn đặt quyền chỉ đọc ở mức hệ điều hành cho `sstate-cache` và `downloads` **(quyết định của người viết)**.

Vì hầu hết việc đi qua lệnh `harness`, chỉ cần cho phép một tiền tố là hết phần lớn các lần xin phép. Lệnh `harness` tự kiểm tra đối số của nó theo cùng chính sách, nên nó không trở thành cửa hậu.

**Kiểm tra bắt buộc không dựa vào hook của công cụ**, vì Roo/Zoo không có hook:

- Git hook (pre-commit, commit-msg) kiểm tra format và commit rule. Dự án hiện chưa có hook nào; bộ cài vẫn phải nối thêm chứ không ghi đè nếu sau này có.
- `harness check` chạy bậc 0 cho các file vừa sửa; skill yêu cầu agent gọi nó sau mỗi lần sửa.
- Phía Codex, hook `PostToolUse` gọi cùng lệnh đó như một lớp bổ sung, và `PreToolUse` chặn lệnh cấm.

Cảnh báo thay đổi lớn: `harness check` báo khi diff vượt ngưỡng số file hoặc số dòng, hoặc đụng file ngoài brief.

## 6. Thang kiểm chứng

| Bậc | Kiểm tra | Ai chạy |
|---|---|---|
| 0 | format, `clang-tidy`, parse recipe, kiểm tra commit message | agent, sau mỗi lần sửa |
| 1 | build app hoặc component vừa đụng bằng `mk`; unit test trên host nếu có | agent |
| 2 | build recipe liên quan | agent, qua khoá build |
| 3 | build system, image, SDK | người hoặc CI |
| 4 | Coverity, chạy trên board | người hoặc CI; kết quả quay lại làm đầu vào |

Brief ghi bậc phải đạt. Báo cáo ghi bậc thực sự đã đạt kèm đầu ra của lệnh. Lệnh `mk` cụ thể cho từng loại file được xác định từ source thật trong bước khảo sát, không điền sẵn bằng phỏng đoán.

## 7. Tool

Một lệnh dòng lệnh duy nhất, viết bằng Python chỉ dùng thư viện chuẩn, dựa vào `ripgrep` và `ctags`. Tên lệnh `harness` là **(quyết định của người viết)**. MCP không nằm trong bản bàn giao; một lớp MCP mỏng bọc lên chính lệnh này có thể thêm sau.

| Nhóm | Việc | Ghi chú |
|---|---|---|
| `survey` | dò phiên bản Yocto, layer, recipe, component TelAF, script môi trường; sinh `project.yaml` và `map/` | báo rõ thứ dò được và thứ không dò được; không đoán |
| `map` | tra layer, recipe và các `.bbappend` chồng lên, component và API, symbol | đọc từ chỉ mục đã sinh |
| `bb` | bọc `bitbake -e` (lấy một biến), `bitbake-layers`, `oe-pkgdata-util`, parse, build một recipe | đầu ra rút gọn; build qua khoá |
| `mk` | build một app hoặc component TelAF | lệnh thật lấy từ `project.yaml` |
| `log` | rút gọn log lớn, khớp mẫu lỗi đã biết, trích vùng quanh lỗi đầu tiên | cho log build, log framework, kernel oops, báo cáo Coverity |
| `kb` | tìm trong KB, tải snapshot, thêm tài liệu | chỉ `kb fetch` dùng mạng |
| `task` | tạo, xem, tiếp tục tác vụ; nhận brief | ghi vào `tasks/` |
| `check` | bậc 0 cho file vừa sửa, kiểm tra phạm vi so với brief, cảnh báo thay đổi lớn | |
| `friction` / `retro` | ghi chỗ vấp; tổng hợp thành đề xuất | xem mục 10 |
| `gen` / `doctor` | sinh lại cấu hình; kiểm tra cài đặt và liệt kê bước tay còn thiếu | |
| `eval` | chạy bộ tác vụ đánh giá | |

Mọi đầu ra của tool đều ngắn theo mặc định và có tuỳ chọn xem thêm, vì context của model rẻ ngắn.

## 8. Skill

Tất cả theo chuẩn `SKILL.md`, đặt ở `.agents/skills/` để cả hai công cụ đọc.

**Ngôn ngữ:** mọi thứ agent đọc (`AGENTS.md`, skill, vai trò, mẫu, đầu ra của tool) viết bằng tiếng Anh. Tài liệu dành cho người (thiết kế, hướng dẫn bảo trì) viết bằng tiếng Việt. File trong thư mục trí nhớ do agent ghi cũng bằng tiếng Anh **(quyết định của người viết)**.

**Quy tắc viết cho model yếu:** mỗi skill ngắn; các bước đánh số; lệnh ghi nguyên văn; có điều kiện dừng rõ ràng; không dựa vào suy luận ngầm; chi tiết dài tách sang file tham khảo chỉ đọc khi cần. Mọi skill được thử bằng model yếu nhất qua bộ đánh giá trước khi nhận.

| Nhóm | Skill | Nguồn |
|---|---|---|
| Quy trình | brainstorm, lập plan, debug có hệ thống, kiểm chứng trước khi báo xong, review | kế thừa từ superpowers, rút gọn |
| Yocto | viết và sửa recipe, bbappend; quản lý layer; `devtool` và patch; kiểm tra dependency, license; cấu hình image và SDK; đọc lỗi build | lấy `yocto-agent-skills` làm nền, đối chiếu với kirkstone |
| TelAF / Legato | tạo component và app; thêm API (`.api`, `.cdef`, `.adef`, `.sdef`); binding; build bằng `mk`; đọc log framework | tự viết, dựa trên source TelAF công khai |
| C++ | chọn design pattern; refactor; review theo MISRA C++:2023 (bằng mã luật) và CERT; sửa theo báo cáo Coverity | tự viết, mượn cấu trúc từ cộng đồng |
| Điều tra | từ log tới source; sổ giả thuyết; mẫu lỗi (linker, thiếu dependency, race, rò bộ nhớ, IPC) | tự viết |
| Kịch bản | xem mục 11 | tự viết |

**Luật tra cứu, áp dụng cho mọi skill:** tra KB trước, rồi đọc source và header, rồi `man` và `--help`, cuối cùng mới hỏi người. Không được dùng một API, biến hay lệnh mà chưa thấy nó trong một trong các nguồn đó.

**Luật điều tra:** tái hiện hoặc xác định điều kiện, thu thập log, khoanh vùng, nêu giả thuyết, kiểm chứng, đề xuất sửa, kiểm tra hồi quy. Mỗi kết luận kèm bằng chứng (dòng log, file và dòng source). Giả thuyết đã loại được ghi vào `hypotheses.md` kèm lý do.

## 9. Knowledge base

| Phần | Nội dung | Cách vào KB |
|---|---|---|
| Công khai | Yocto kirkstone, BitBake manual, SEI CERT C/C++, CWE, C++ Core Guidelines, man-pages | `kb fetch` tải khi có mạng, đóng thành snapshot có version |
| Hạn chế | tài liệu Qualcomm, tài liệu TelAF dựng từ source, tài liệu nội bộ | `kb add` tại máy dự án, không bao giờ rời máy |
| MISRA | mã luật, diễn giải tự viết, ví dụ sửa, ánh xạ sang checker Coverity | viết dần tại máy dự án; không có nguyên văn luật |
| Kinh nghiệm | mẫu lỗi đã gặp, ví dụ recipe, pattern catalog | harness kèm bản khởi đầu; lớn dần qua vòng tự cải thiện |

**Tra cứu:** mặc định là tìm toàn văn bằng SQLite FTS5 (có sẵn trong Python). Vector search qua API embedding của endpoint là tuỳ chọn bật thêm sau, phía sau cùng lệnh `kb search`, để không phải sửa skill.

**Hành vi theo mức mạng** (cấu hình `network: auto | off` trong `project.yaml`):

| Thành phần | Có mạng | Có mạng, không web search | Offline |
|---|---|---|---|
| Luật, skill, vai trò, chính sách lệnh | như nhau | như nhau | như nhau |
| Tool `survey`, `map`, `bb`, `mk`, `log`, `task`, `check` | như nhau | như nhau | như nhau |
| `kb search` | cục bộ | cục bộ | cục bộ |
| `kb fetch` | tải và làm mới snapshot | tải và làm mới snapshot | báo không có mạng, dùng snapshot hiện có |
| Vector search (tuỳ chọn) | dùng endpoint | dùng endpoint | lùi về tìm toàn văn |
| Model | qua endpoint | qua endpoint | cần endpoint nội bộ; ngoài phạm vi harness |

## 10. Tự cải thiện

1. **Ghi chỗ vấp tự động:** lệnh bị chặn, kiểm tra thất bại, lần phải hỏi người, lần tra KB không ra, lần implementer phải dừng vì brief thiếu. Ghi vào `friction.jsonl`.
2. **Rà soát định kỳ** (kích hoạt bằng một lệnh): điều phối đọc nhật ký, gom nhóm, đề xuất thay đổi.
3. **Hai loại thay đổi:**
   - Trí nhớ (thẻ, bài học, mẫu lỗi, KB kinh nghiệm): áp dụng ngay, lưu trong git của thư mục trí nhớ.
   - Luật, skill, vai trò, chính sách lệnh, tool: chỉ là đề xuất trên một nhánh của repo harness cục bộ, chờ người duyệt.
4. **Cổng chất lượng:** đề xuất loại thứ hai phải chạy lại bộ tác vụ đánh giá và không làm kết quả tệ đi trước khi được nhận.

Agent không bao giờ tự sửa chính sách lệnh có hiệu lực.

## 11. Kịch bản soạn sẵn

Sau lần cài đặt, mỗi việc thiết lập còn lại là một skill kích hoạt bằng một lệnh. Mỗi kịch bản tự làm phần dò được, chỉ hỏi phần thiếu, và ghi tiến độ để chạy tiếp được.

| Kịch bản | Việc |
|---|---|
| Khởi tạo dự án | chạy `survey`, sinh bản đồ, hỏi những gì không dò được |
| Nhập quy ước | đọc commit rule và cấu hình format, sinh `conventions.md` và git hook |
| Ghi nhận kiến trúc | phỏng vấn ngắn, sinh `architecture.md` và các ADR đầu tiên |
| Dựng thẻ component | sinh thẻ từ bản đồ và source, từng component một |
| Ánh xạ Coverity | nhập báo cáo mẫu, lập bảng checker sang mã luật MISRA |
| Nhập tài liệu | thêm tài liệu Qualcomm và nội bộ vào KB, đánh chỉ mục |
| Thêm mẫu lỗi | từ một log thật, tạo mẫu nhận diện mới |
| Bật vector search | kiểm tra API embedding, đánh chỉ mục, so sánh với tìm toàn văn |
| Chuyển bộ đánh giá | tạo tác vụ đánh giá trên dự án thật từ mẫu |
| Rà soát | vòng tự cải thiện ở mục 10 |
| Nghiệm thu | chạy `doctor`, kiểm thử chính sách lệnh, bộ đánh giá với từng hạng model; ghi kết quả |

## 12. Phần chung và phần riêng

| Thành phần | Chung | Roo / Zoo Code | Codex |
|---|---|---|---|
| Luật | `AGENTS.md` | tự nạp | tự nạp, giới hạn 32 KiB |
| Skill | `.agents/skills/` | tự nạp | tự nạp |
| Vai trò | `core/roles/` | `.roomodes`, giới hạn file bằng `fileRegex` | `.codex/agents/*.toml`, giới hạn bằng chỉ dẫn + `harness check` |
| Model theo vai trò | hạng model ghi trong vai trò | gán tay trong extension; `doctor` liệt kê | trường `model` trong file agent |
| Chính sách lệnh | `commands.yaml` | setting cho phép / cấm | `.codex/rules/*.rules` |
| Kiểm tra sau sửa | `harness check` + git hook | chỉ dẫn trong mode | thêm hook |
| Song song | brief không chồng file, khoá build | tuần tự | sub-agent song song |
| Cấu hình theo máy | mẫu trong repo | tiền tố setting khác nhau giữa Roo và Zoo | provider đặt ở cấu hình người dùng |

`doctor` kiểm tra và in ra đúng các bước tay còn thiếu trên máy đó.

## 13. Bộ tác vụ đánh giá

Bốn tác vụ mẫu, chạy trên dự án mẫu trong repo này và chuyển sang dự án thật bằng kịch bản:

1. Thêm một recipe mới.
2. Sửa một lỗi build.
3. Điều tra một crash từ log (không có board).
4. Thêm một API Legato/TelAF vào một app.

Mỗi tác vụ có đầu vào cố định, tiêu chí đạt kiểm tra bằng máy, và chạy được khi không có mạng. Chấm theo: có đạt không, số lần phải hỏi người, có lệnh nào bị chặn không, bậc kiểm chứng đạt được. Chạy với từng hạng model để biết model yếu nhất còn làm được gì.

Ngoài ra có kiểm thử cho chính harness: lệnh cấm bị từ chối ở cả hai định dạng sinh ra, bộ sinh cho đầu ra ổn định, `survey` chạy đúng trên dự án mẫu.

### Kiểm chứng được gì ở đâu

| Nơi | Có gì | Kiểm chứng được |
|---|---|---|
| Máy dựng harness | Codex với một model rẻ (Luna), Haiku, không có SDK TelAF, không có endpoint của dự án | skill và vai trò với hai model rẻ; bộ sinh; chính sách lệnh phía Codex; tool với `bitbake` và `mk` giả lập |
| Máy dự án | endpoint với mọi model, Roo/Zoo Code, SDK thật, source thật | model yếu nhất (Qwen); bộ chuyển đổi Roo/Zoo; `survey` và `mk` trên source thật |

Vì vậy bản bàn giao kèm một **kịch bản nghiệm thu** chạy trên máy dự án: `doctor`, kiểm thử chính sách lệnh trên công cụ thật, rồi bốn tác vụ đánh giá với từng hạng model. Phần nào chưa qua nghiệm thu tại chỗ thì được ghi rõ là chưa kiểm chứng. Cấu hình sinh ra cho Roo/Zoo trước lúc đó chỉ được kiểm tra về định dạng.

Dự án mẫu gồm một cây layer nhỏ, một app TelAF mẫu và các log đóng hộp. Kiểm thử tool dùng `bitbake` giả lập trả đầu ra ghi sẵn; chạy với Yocto thật là tuỳ chọn **(quyết định của người viết)**.

## 14. Lộ trình tới bản bàn giao

| Mốc | Nội dung | Giá trị | Độ khó |
|---|---|---|---|
| M1 | Cài đặt, bộ sinh, `AGENTS.md`, chính sách lệnh, git hook, `doctor` | cao: hết cảnh bấm duyệt, đúng quy ước | trung bình |
| M2 | `survey`, bản đồ, `map`, tool `bb` và `mk`, khoá build | cao: hết cảnh tìm hiểu lại | trung bình |
| M3 | Vai trò, brief, thư mục tác vụ, skill quy trình | cao: sống sót qua compact, model rẻ làm được việc | cao |
| M4 | Skill Yocto, TelAF, C++; tool `log`; skill điều tra | cao | cao |
| M5 | KB: tải, snapshot, tìm toàn văn | trung bình | thấp |
| M6 | Bộ đánh giá, vòng tự cải thiện, kịch bản, tài liệu bảo trì | cao về lâu dài | trung bình |

Bộ đánh giá tối thiểu (một tác vụ) được dựng từ M1 và lớn dần, để mỗi mốc đều đo được.

## 15. Để lại cho người dựng

- Định dạng chính xác của `commands.yaml`, vai trò, brief và `project.yaml`.
- Cách `survey` nhận diện từng loại dữ kiện.
- Danh sách mẫu lỗi khởi đầu.
- Cách loại file sinh ra khỏi git khi workspace là nhiều repo quản lý bằng `repo`.
- Chọn skill nào của cộng đồng để chép vào và rút gọn tới mức nào.

## 16. Chưa biết, sẽ rõ trên máy dự án

- Extension là Roo Code hay Zoo Code, và phiên bản Codex.
- Công cụ được mở từ thư mục nào, và thư mục đó có phải gốc của một git repo không. Bộ cài phải xử lý được cả hai trường hợp: một repo, và nhiều repo dưới một gốc chung **(quyết định của người viết)**.
- Baseline chính xác và Yocto có bị nhà cung cấp sửa không.
- Cách lấy log trên target và vị trí core dump.
- Script build và đường dẫn SDK.
- Toolchain, quyết định phiên bản `clang-tidy` dùng được.

Harness không được giả định giá trị nào cho các mục này; `survey` và kịch bản khởi tạo sẽ điền chúng.
