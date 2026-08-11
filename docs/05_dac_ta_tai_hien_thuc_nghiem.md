# Đặc tả yêu cầu tái hiện thực nghiệm

Tài liệu tổng hợp toàn bộ tham số, dữ liệu, hạ tầng và ước lượng chi phí cần thiết để tái hiện P1
(MASA) và P2 (Skill-MAS), kèm đánh giá tính khả thi theo từng mức ngân sách.

---

## 1. Bảng tham số đầy đủ

### 1.1 P1 — MASA

| Nhóm | Tham số | Giá trị | Nguồn |
|---|---|---|---|
| **Giai đoạn 1 — hill climbing** | số vòng tối đa `I` | 10 | Phụ lục E.1 |
| | patience `p` | 3 | Phụ lục E.1 |
| | kích thước lịch sử `K` (skill set điểm cao đưa cho teacher) | 5 | Phụ lục E.1 |
| | quy tắc chấp nhận | phần thưởng điều chỉnh trung bình **vượt nghiêm ngặt** giá trị tốt nhất | Phụ lục E.1 |
| **Giai đoạn 2 — UCB tree search** | hằng số khám phá `C` | 1.4 | Phụ lục E.2 |
| | số vòng mỗi loại tác vụ `J` | 10 | Phụ lục E.2 |
| | số episode đánh giá mỗi node `N` | 100 | Phụ lục E.2 |
| **Hàm phần thưởng** | `R = SR − λ·NHR` | `λ ∈ [0,1]` (giá trị cụ thể không nêu) | §3.1 |
| **Truy hồi** | bộ mã hoá | Qwen3-Embedding-0.6B | §3.1 |
| | `k_G`, `k_T` | **không nêu** | — |
| **Teacher LLM** | | DeepSeek-V4-Pro | §4.1 |
| **Rewriter — kiến trúc** | backbone | Qwen3-4B | Phụ lục F |
| | chế độ huấn luyện | full-parameter SFT | Phụ lục F |
| | độ chính xác | BF16 | Phụ lục F |
| **Rewriter — tối ưu** | optimizer | AdamW | Phụ lục F |
| | learning rate | 1e-5 | Phụ lục F |
| | schedule | cosine, warmup ratio 0.1 | Phụ lục F |
| | gradient checkpointing | bật | Phụ lục F |
| | effective batch size | 4 (per-device 1 × grad accum 4) | Phụ lục F |
| | số epoch | 5 | Phụ lục F |
| | max sequence length | 4096 | Phụ lục F |
| | chọn checkpoint | theo hội tụ training loss | Phụ lục F |
| **Rewriter — dữ liệu** | rewriter kết hợp | 769 mẫu (ALFWorld Pick/Look/Pick2 + WebShop + Search) | Phụ lục F |
| | rewriter đặc thù môi trường | 499 mẫu (WebShop + Search) | Phụ lục F |
| | augmentation — noise ratio | 0.3 | Phụ lục F |
| | augmentation — partial keep ratio | 0.6 | Phụ lục F |
| **Backbone đánh giá** | | Qwen3-4B/8B/14B/32B, **non-thinking mode** | §4.1 |
| | bổ trợ | Gemma3-4B/12B/27B | Phụ lục C.3 |

### 1.2 P2 — Skill-MAS

| Nhóm | Tham số | Giá trị | Nguồn |
|---|---|---|---|
| **Vòng lặp tiến hoá** | số vòng `R` | 10 | Bảng 5 |
| | số rollout mỗi tác vụ `K` | 5 | Bảng 5 |
| | chọn skill cuối | validation score cao nhất trong `R` vòng | §3 |
| **Lời gọi LLM** | temperature | 1.0 | Bảng 5 |
| | max_tokens | 32.768 | Bảng 5 |
| **Chọn tác vụ** | điểm ưu tiên | `p_i = ½(ũ_i + d̃_i)` sau min–max normalization | §3.3.1 |
| | ngưỡng chọn | elbow — `j* = argmax_j \|δ_j − δ_{j+1}\| + 1` | §3.3.1 |
| **Phản tỉnh** | phân hoạch quỹ đạo | median split thành `H_i` / `L_i` | §3.3.2 |
| | định dạng đầu ra | strict JSON, không markdown fence | Hình 19–20 |
| **Tối ưu skill** | giới hạn cắt tỉa | ≤ 1 element mỗi stage section mỗi vòng | Hình 21 |
| | giới hạn bổ sung | ≤ 1 conceptual upgrade mỗi module mỗi vòng | Hình 21 |
| | abstraction firewall | cấm ví dụ đặc thù miền / chi tiết cú pháp / heuristic cứng | Hình 21 |
| **Meta-agent** | reasoning effort | "low" cho Gemini-3.1-Flash và GPT-5.4-Nano; mặc định cho Qwen3.5-Plus và DeepSeek-V4-Flash | Phụ lục D.2 |
| | ràng buộc | **cùng một LLM cho mọi cấu phần trong một MAS** | §4.1 |
| **Đánh giá** | LLM-as-a-judge | Gemini-3.1-Flash | Phụ lục D.2 |
| | chuẩn hoá chỉ số | [0, 100%] | §4 |
| **Baseline** | AFlow | tối đa 10 vòng tìm kiếm, đánh giá validation 3 lần mỗi vòng | Phụ lục D.2 |
| | các baseline khác | theo nguyên thiết lập gốc | Phụ lục D.2 |

---

## 2. Dữ liệu và môi trường

### 2.1 P1

| Môi trường | Nguồn | Ghi chú tái hiện |
|---|---|---|
| ALFWorld | Shridhar và cộng sự, 2020 — công khai | 6 loại tác vụ; tiến hoá dùng training split, đánh giá dùng test split |
| WebShop | Yao và cộng sự, 2022a — công khai | Có sẵn success flag tự động |
| Search-QA | NQ, TriviaQA, PopQA, HotpotQA, 2Wiki, MuSiQue, Bamboogle — công khai | Tiến hoá **chỉ trên NQ và HotpotQA**; năm dataset còn lại là out-of-domain |
| Skill library khởi tạo | SkillRL (Xia và cộng sự, 2026) | **Phụ thuộc bên ngoài bắt buộc** — nếu không truy cập được, cần tự xây library thay thế, và kết quả không còn so sánh trực tiếp với công bố |

Toàn bộ backbone Qwen3 và Gemma3 là open-weight, cho phép tự host.

### 2.2 P2

| Benchmark | Validation | Test | Nguồn | Ghi chú |
|---|---|---|---|---|
| DeepResearchBench | 16 | 84 | Du và cộng sự, 2025 | 100 tác vụ, 22 lĩnh vực; chấm bằng LLM theo 4 chiều |
| Humanity's Last Exam-Math | 32 | 168 | Phan và cộng sự, 2025 | Lấy mẫu 200 câu từ subset MATH |
| BrowseComp-Plus | 32 | 168 | Chen và cộng sự, 2025b | Lấy mẫu 200 câu; corpus cố định |
| VitaBench | 16 | 84 | He và cộng sự, 2025b | 66 tool trong môi trường mô phỏng; chấm theo rubric |

Meta-agent gồm hai mô hình proprietary (Gemini-3.1-Flash, GPT-5.4-Nano) — **bắt buộc dùng API trả
phí**, không thay thế được bằng mô hình tự host nếu muốn tái hiện đúng bảng kết quả.

---

## 3. Ước lượng chi phí

### 3.1 P1 — chi phí tính toán

Công bố **không cung cấp** bảng chi phí. Ước lượng dưới đây suy ra từ tham số đã nêu.

**Số episode cho một backbone, môi trường ALFWorld (6 loại tác vụ):**

| Thành phần | Công thức | Ước lượng |
|---|---|---|
| Giai đoạn 2 — tree search | 6 loại × `J=10` vòng × `N=100` episode | **6.000 episode** |
| Giai đoạn 1 — hill climbing | `I=10` vòng × (rollout trên toàn task suite) | phụ thuộc kích thước eval set, bậc 10³ |
| **Tổng, một backbone, một môi trường** | | **bậc 10⁴ episode** |
| **Tổng, 4 backbone × 3 môi trường** | | **bậc 10⁵ episode** |

Mỗi episode gồm hàng chục bước, mỗi bước là một lời gọi LLM. Cộng thêm chi phí teacher DeepSeek-V4-Pro
cho mỗi lần phân tích thất bại và viết lại.

**Huấn luyện rewriter** là phần rẻ nhất: Qwen3-4B full-parameter SFT, 769 mẫu, 5 epoch, max seq 4096,
effective batch 4. Với gradient checkpointing, cấu hình này khả thi trên một GPU 80 GB (A100/H100);
trên 40 GB cần LoRA thay cho full-parameter, và kết quả sẽ lệch khỏi công bố.

### 3.2 P2 — chi phí API

Công bố cung cấp số liệu trực tiếp.

**Chi phí tiến hoá trên validation set** (Bảng 6, trung bình qua bốn benchmark, USD):

| Gemini-3.1-Flash | GPT-5.4-Nano | Qwen3.5-Plus | DeepSeek-V4-Flash |
|---|---|---|---|
| 9,35 | 31,36 | 59,06 | 24,54 |

**Chi phí suy luận trên test set** (Bảng 1, cột Avg.Cost, USD): `Skill-MAS-optimized` dao động
2,12–5,22 tuỳ meta-agent.

**Ước lượng tổng để tái hiện toàn bộ Bảng 1:**

| Hạng mục | Ước lượng |
|---|---|
| Tiến hoá, 4 meta-agent × 4 benchmark | ≈ 4 × (9,35 + 31,36 + 59,06 + 24,54) ≈ **500 USD** |
| Suy luận `Skill-MAS-optimized` + `init`, 4 meta-agent × 4 benchmark | ≈ **100–200 USD** |
| Năm baseline, 4 meta-agent × 4 benchmark | phụ thuộc mạnh vào baseline; EvoAgent 8,20 USD/benchmark trên Gemini là mức cao nhất ghi nhận. Ước **300–600 USD** |
| **Tổng tái hiện đầy đủ** | **bậc 1.000 USD** |

Ghi chú: các con số trên là ước lượng suy ra từ số liệu công bố, không phải báo cáo của tác giả; biến
động giá API và số lần chạy lại do lỗi kỹ thuật có thể làm lệch đáng kể.

---

## 4. Yêu cầu hạ tầng

### 4.1 P1

| Hạng mục | Yêu cầu tối thiểu | Ghi chú |
|---|---|---|
| GPU suy luận | 1 × 80 GB cho Qwen3-32B ở BF16 | 4B/8B/14B chạy được trên 24–48 GB |
| GPU huấn luyện rewriter | 1 × 80 GB (full-parameter SFT Qwen3-4B, seq 4096) | LoRA cho phép hạ xuống 24 GB nhưng lệch cấu hình gốc |
| API bên ngoài | DeepSeek-V4-Pro (teacher) | Bắt buộc; số lời gọi tỉ lệ với số vòng tìm kiếm |
| Framework | vLLM hoặc tương đương cho batched inference | Bắt buộc trên thực tế: bậc 10⁵ episode không khả thi với inference tuần tự |
| Song song hoá | Các cây của Giai đoạn 2 độc lập hoàn toàn | Đây là điểm song song hoá hiệu quả nhất |

### 4.2 P2

| Hạng mục | Yêu cầu |
|---|---|
| GPU | **Không bắt buộc** nếu dùng meta-agent qua API |
| API bên ngoài | Gemini, OpenAI, Qwen, DeepSeek — bốn nhà cung cấp |
| Sandbox thực thi | Bắt buộc: hệ sinh và **chạy** mã Python cho MAS workflow |
| Môi trường tool | VitaBench yêu cầu môi trường mô phỏng 66 tool |
| Corpus truy hồi | BrowseComp-Plus yêu cầu corpus cố định kèm BM25 index |

Lưu ý kỹ thuật: `MAS_BUILD_CONTRACT` (Hình 17–18) chỉ rõ hệ dùng **BM25 (truy hồi từ khoá)** cho các
agent `multi_turn_search`, kèm hệ quả thiết kế: nếu agent thượng nguồn chỉ sinh mô tả trừu tượng
(*"amphibian"*, *"early 1990s"*), planner BM25 không tạo được truy vấn chính xác. Ràng buộc này phải
được tái hiện đúng, nếu không kết quả trên BrowseComp-Plus sẽ không so sánh được.

---

## 5. Lộ trình tái hiện theo mức ngân sách

### 5.1 Mức tối thiểu — xác minh phát hiện cốt lõi của P1

**Mục tiêu.** Tái hiện §2 của P1 (thí nghiệm sơ bộ về granularity), không tái hiện phương pháp.

| Hạng mục | Yêu cầu |
|---|---|
| Backbone | Qwen3-4B và Qwen3-8B (đủ để tái hiện hiện tượng nghịch lý ở 8B) |
| Môi trường | ALFWorld, validation split |
| Điều kiện | 4 mức: No Skill / Concise / Moderate / Detailed |
| Phần cần tự làm | Viết `Concise` và `Detailed` từ library gốc, giữ nguyên nội dung nguyên lý |
| Hạ tầng | 1 GPU 48 GB, vLLM |
| Giá trị thu được | Xác nhận hoặc bác bỏ kết luận trung tâm của P1 ở chi phí thấp nhất |

Đây là điểm khởi đầu hợp lý vì §2 của P1 là phần có tỉ lệ giá trị bằng chứng trên chi phí cao nhất
trong cả hai công bố.

### 5.2 Mức trung bình — tái hiện Skill-MAS trên một benchmark

**Mục tiêu.** Tái hiện vòng lặp tiến hoá của P2 trên một benchmark, một meta-agent.

| Hạng mục | Lựa chọn khuyến nghị |
|---|---|
| Meta-agent | Gemini-3.1-Flash — chi phí tiến hoá thấp nhất (9,35 USD) |
| Benchmark | BrowseComp-Plus hoặc HLE-Math — không cần môi trường 66 tool như VitaBench |
| Chi phí ước tính | 10–30 USD tiến hoá + 3–5 USD suy luận |
| Rủi ro chính | Judge mặc định là Gemini, trùng meta-agent; nên dùng judge độc lập và ghi nhận sai lệch |

### 5.3 Mức đầy đủ — tái hiện cả hai công bố

Bậc 1.000 USD chi phí API cho P2, cộng bậc 10⁵ episode LLM inference cho P1. Không khuyến nghị trừ
khi có mục tiêu so sánh trực tiếp với một phương pháp mới.

---

## 6. Danh mục rủi ro triển khai

| Rủi ro | Mức | Biện pháp giảm nhẹ |
|---|---|---|
| **Phụ thuộc skill library gốc của SkillRL (P1)** | Cao | Nếu không truy cập được, tự xây library và ghi rõ rằng kết quả không so sánh trực tiếp với công bố |
| **Tham số `k_G`, `k_T` không được nêu (P1)** | Trung bình | Quét vài giá trị (ví dụ 1, 3, 5) và báo cáo độ nhạy; đây đồng thời là một ablation mà công bố gốc còn thiếu |
| **Giá trị `λ` trong hàm phần thưởng không được nêu (P1)** | Trung bình | Thử `λ ∈ {0, 0.25, 0.5}`; `λ = 0` là biến thể chỉ dùng SR, hữu ích làm mốc đối chiếu |
| **Xung đột vai trò judge (P2)** | Cao | Dùng judge khác meta-agent; nếu tài nguyên cho phép, chấm chéo bằng hai judge và báo cáo mức bất đồng |
| **Validation set nhỏ, 16 tác vụ (P2)** | Cao | Chạy tối thiểu 3 seed và báo cáo độ lệch chuẩn; đây là điểm yếu đã xác định của công bố gốc |
| **Sinh mã MAS không chạy được (P2)** | Cao | Tái hiện đầy đủ `MAS_BUILD_CONTRACT`; ghi nhận tỉ lệ sinh mã hợp lệ như một chỉ số riêng — baseline MAS² thất bại hoàn toàn ở khâu này |
| **Skill phình sau nhiều vòng** | Trung bình | Áp đủ ba ràng buộc của P2; ghi lại số token của tài liệu skill sau mỗi vòng như chỉ số giám sát |
| **Thiếu reward tự động trong miền đích** | Cao | Nếu miền đích không có success flag, cần thiết kế evaluator trước khi triển khai bất kỳ phần nào của hai pipeline |
| **Biến động API và deprecation mô hình** | Trung bình | Ghi lại chính xác định danh mô hình và ngày chạy; kết quả không tái lặp được sau khi nhà cung cấp cập nhật endpoint |

---

## 7. Thứ tự triển khai khuyến nghị

Nếu mục tiêu là xây dựng năng lực nghiên cứu trên hướng này thay vì tái hiện thuần tuý, thứ tự sau
tối ưu hoá tỉ lệ giá trị thu được trên chi phí:

1. **Tái hiện §2 của P1** (mức 5.1) — xác nhận hiện tượng nền, chi phí thấp nhất, và cho một kết quả
   có thể báo cáo độc lập.
2. **Cài đặt multi-trajectory rollout và hai thống kê `u`, `d`** — cơ chế rẻ nhất trong cả hai công bố,
   dùng lại được ở mọi thí nghiệm sau.
3. **Cài đặt elbow detection** — vài dòng mã, dùng chung với bước 2.
4. **Cài đặt một vòng lặp reflect–rewrite tối giản** với ba ràng buộc kiểm soát tăng trưởng của P2,
   chạy trên môi trường đã có ở bước 1.
5. Chỉ khi bốn bước trên đã ổn định mới cân nhắc UCB tree search (P1, Giai đoạn 2) hoặc chưng cất
   rewriter — hai cấu phần đắt nhất và phụ thuộc nhiều nhất vào hạ tầng.
