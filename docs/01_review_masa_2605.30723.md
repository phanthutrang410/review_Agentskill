# P1 — MASA: Model-Aware Skill Alignment for LLM Agents

**Định danh.** arXiv 2605.30723v1 · cs.CL · nộp 29/05/2026 · 20 trang (8 trang chính, 12 trang phụ lục).
**Tác giả.** Jianxiang Yu, Jiapeng Zhu, Bochen Lin, Qier Cui, Zichen Ding, Xiang Li (liên hệ) —
East China Normal University, Thượng Hải.
**Mã nguồn.** `github.com/jianxiangyu/MASA_`
**Nhan đề đầy đủ.** *Skill is Not One-Size-Fits-All: Model-Aware Skill Alignment for LLM Agents*.

---

## 1. Bối cảnh và động cơ

### 1.1 Skill library trong nghiên cứu LLM agent

Tồn tại một hướng nghiên cứu nhằm nâng cao năng lực của LLM agent **không thông qua cập nhật trọng
số**, mà thông qua một *skill library*: tập các đơn vị **kiến thức thủ tục** (procedural knowledge)
biểu diễn bằng ngôn ngữ tự nhiên. Tại mỗi bước ra quyết định, agent truy hồi (retrieve) một số skill
liên quan và nạp chúng vào context trước khi sinh hành động.

Các hệ thống tiêu biểu được P1 dẫn: ReAct và Reflexion (phản hồi dạng văn bản làm skill in-context),
Voyager (thư viện skill tăng trưởng trong Minecraft), JARVIS-1, Ghost-in-the-Minecraft, AgentTrek,
AutoManual (quy nạp *operating manual* từ tương tác), ExpeL (chưng cất kinh nghiệm liên thử nghiệm),
SkillRL (SkillBank phân cấp, đồng tiến hoá skill với policy), SkillOS (curator dài hạn thực hiện
chèn/cập nhật/xoá skill trong SkillRepo).

### 1.2 Giả định bị chất vấn

Toàn bộ hướng nghiên cứu trên mặc định skill library là **model-agnostic**: một library được xây
dựng một lần rồi tái sử dụng trên nhiều backbone khác nhau. Ràng buộc triển khai thực tế — ngân sách
độ trễ, chi phí suy luận, khả dụng phần cứng — buộc các hệ agent phải vận hành trên các backbone có
capacity chênh lệch đáng kể thay vì luôn dùng mô hình mạnh nhất. Câu hỏi nghiên cứu do đó là: *một
cách diễn đạt skill duy nhất có phục vụ đồng đều các mô hình có capacity khác biệt đáng kể hay không?*

### 1.3 Định vị so với công trình liên quan gần nhất

P1 dẫn **SkVM (Chen và cộng sự, 2026)** như công trình đã nhận diện cùng hiện tượng, với ghi nhận
định lượng rằng **87% tác vụ có ít nhất một LLM không thu được lợi ích từ cùng một skill**. SkVM giải
quyết bằng cách *biên dịch* skill sang định dạng runtime tối ưu (code solidification, parallelization)
nhằm giảm độ trễ và chi phí token. P1 theo hướng bổ sung: **viết lại biểu đạt ngôn ngữ tự nhiên** của
skill sao cho tương thích với năng lực đọc hiểu và phong cách suy luận của từng backbone, với mục
tiêu trực tiếp là success rate.

So với các phương pháp tối ưu prompt có tính đến định danh mô hình (MAPO, PromptBridge), P1 khác ở
hai điểm: (i) đối tượng tối ưu là **skill library truy hồi động, nhiều mục** thay vì một prompt đơn
khối; (ii) quá trình tiến hoá được dẫn hướng đồng thời bởi **hồ sơ năng lực mô hình có cấu trúc** và
tín hiệu phần thưởng từ môi trường.

---

## 2. Thí nghiệm sơ bộ: chứng minh sự phụ thuộc mô hình

### 2.1 Thiết kế cô lập biến

Nguyên lý thiết kế: giữ cố định **nội dung nguyên lý** của skill library, chỉ biến thiên **độ chi
tiết biểu đạt (granularity)**. Bốn điều kiện:

| Điều kiện | Độ dài trung bình | Nguồn |
|---|---|---|
| `No Skill` | 0 token | đối chứng, library rỗng |
| `Concise` | ≈ 370 token | viết lại nén từ `Moderate` |
| `Moderate` | ≈ 1.092 token | library gốc của SkillRL (Xia và cộng sự, 2026) |
| `Detailed` | ≈ 2.737 token | viết lại mở rộng từ `Moderate` |

Ba mức skill dùng **chung pipeline truy hồi**, chung skill ID và chung độ phủ tác vụ. Mọi khác biệt
quan sát được do đó quy về một biến duy nhất.

Minh hoạ (P1, Bảng 4), cùng một general skill *"Systematic Exploration"*:

- `Concise` — *"Principle: Search all surfaces and containers once before revisiting."*
- `Moderate` — bổ sung mệnh đề ưu tiên vị trí chưa mở và một điều kiện kích hoạt ("When to apply").
- `Detailed` — sáu bước đánh số, cú pháp lệnh cụ thể (`go to countertop 1`), một trace ví dụ đầy đủ,
  và cảnh báo lỗi thường gặp (*"COMMON MISTAKE: Trying to 'heat' without first putting the object in
  the microwave → fails"*).

**Môi trường.** ALFWorld, validation split, sáu loại tác vụ.
**Backbone.** Qwen3-4B/8B/14B/32B, toàn bộ vận hành ở **non-thinking mode** nhằm loại nhiễu do cấu
hình reasoning-mode.

### 2.2 Kết quả (P1, Bảng 5 — overall success rate, %)

| Mô hình | No Skill | Concise | Moderate | Detailed | Điều kiện tối ưu |
|---|---|---|---|---|---|
| Qwen3-4B | 17.1 | 16.4 | **20.0** | 12.8 | Moderate |
| Qwen3-8B | **32.1** | 27.9 | 25.0 | 17.1 | không skill |
| Qwen3-14B | 37.9 | 36.8 | 42.1 | **47.5** | Detailed |
| Qwen3-32B | 36.4 | 40.7 | 41.4 | **42.9** | Detailed |

### 2.3 Ba kết luận

**Kết luận 1 — dạng skill tối ưu phụ thuộc mô hình, và sai lệch gây tổn hại thực.**
Không tồn tại mức granularity tối ưu đồng đều. Trường hợp cực đoan là Qwen3-8B: **cả ba mức skill
đều làm giảm hiệu năng so với điều kiện không skill** (32.1 → 27.9 / 25.0 / 17.1). Khảo sát quỹ đạo
cho thấy Qwen3-8B tự thân đã đi theo chuỗi hành động ngắn và hiệu quả; skill sai lệch áp đặt một
pattern suy luận thủ tục **ghi đè** lên chuỗi hành động tự nhiên đó, dẫn tới thăm dò dư thừa và cân
nhắc không cần thiết. Hệ quả lý thuyết: hiệu quả của một skill phụ thuộc không chỉ vào *nội dung*
mà còn vào việc *biểu đạt* của nó có tương thích với chiến lược giải quyết vấn đề mặc định của mô
hình hay không.

**Kết luận 2 — quan hệ granularity–hiệu năng không đơn điệu và không tuân theo quy mô.**
Qwen3-32B dưới `Detailed` thấp hơn Qwen3-14B 4.6 điểm, đảo ngược xu hướng scaling thông thường. Với
Qwen3-4B, `Moderate` vượt cả `Concise` lẫn `Detailed`, tức điểm tối ưu nằm ở mức trung gian không
tiếp cận được bằng quy tắc "tăng chi tiết" hoặc "nén tối đa". Hệ quả thiết kế: cần **thủ tục tìm
kiếm**, không dùng được luật cố định.

**Kết luận 3 — biến thiên theo loại tác vụ vượt biến thiên theo granularity.**
Trong cùng một cặp (mô hình, granularity), success rate chênh lệch **trên 60 điểm** giữa các loại tác
vụ: Qwen3-14B + `Concise` đạt 74.2 trên `Pick` nhưng 13.7 trên `Cool`; Qwen3-4B + `Detailed` đạt 1.6
trên `Pick` nhưng 46.7 trên `Look`. Biên độ này lớn hơn chênh lệch giữa các mức granularity tại mọi
ô. Hệ quả: tối ưu hoá toàn cục là chưa đủ; cần căn chỉnh **theo từng loại tác vụ**.

### 2.4 Kiểm chứng bổ trợ trên họ mô hình khác (Phụ lục C.3)

Lặp lại sweep trên Gemma3-4B/12B/27B (P1, Bảng 6):

| Mô hình | No Skill | Concise | Moderate | Detailed |
|---|---|---|---|---|
| Gemma3-4B | 10.7 | **12.1** | 8.6 | 0.0 |
| Gemma3-12B | 7.9 | **15.7** | 9.3 | 15.0 |
| Gemma3-27B | 21.4 | 36.4 | 35.0 | **44.3** |

Quan sát quan trọng nhất: **Gemma3-4B đạt tối ưu tại `Concise` trong khi Qwen3-4B đạt tối ưu tại
`Moderate`**. Hai mô hình cùng ngân sách tham số phản ứng với granularity theo cách khác biệt về
chất. Điều này biện minh cho việc điều kiện hoá trên **hồ sơ mô hình đầy đủ** (kiến trúc, dữ liệu
huấn luyện, quy trình alignment) thay vì dùng quy mô làm biến đại diện.

---

## 3. Phương pháp

MASA gồm hai cấu phần bổ trợ:

```
[Giai đoạn tìm kiếm]  Hierarchical Model-Conditioned Skill Evolution
                      (teacher LLM + hàng trăm đến hàng nghìn rollout)
                                    │  sinh evolution trajectories
                                    ▼
[Giai đoạn triển khai] MASA-Rewriter (Qwen3-4B, SFT)
                      một lượt forward, không cần tương tác môi trường
```

### 3.1 Hình thức hoá bài toán

Agent `F` được **đóng băng hoàn toàn**. Biến tối ưu duy nhất là skill library `S`:

```
a_t ~ F( · | τ_<t , Ŝ_t ),      Ŝ_t = TopK(S, o_t, k)
```

trong đó `τ_<t = (o_1, a_1, …, o_{t−1}, a_{t−1})` là lịch sử tương tác và `TopK` truy hồi theo cosine
similarity. Bộ mã hoá truy hồi: **Qwen3-Embedding-0.6B**.

Library có cấu trúc hai tầng (kế thừa SkillRL):

- `S^G` — **general skills**: nguyên tắc chiến lược xuyên tác vụ, dự kiến khả chuyển giữa các tác vụ
  (ví dụ *"always verify your action parsed correctly"*). Truy hồi top-`k_G`.
- `S^{T_c}` — **task-specific skills** cho từng loại tác vụ `c ∈ C`: thủ tục theo miền
  (ví dụ *"check the fridge before the counter"*). Truy hồi top-`k_T`.

### 3.2 Model card

Tín hiệu điều kiện hoá then chốt là `M_F`, một hồ sơ có cấu trúc của backbone đích, xây theo rubric
cố định gồm ba phần (Phụ lục D):

1. **Architecture metadata** — family, variant, parameter_count, kiểu kiến trúc, num_layers,
   hidden_size, num_attention_heads, num_kv_heads, context_window, vocab_size; trích trực tiếp từ
   model card công bố hoặc tệp cấu hình.
2. **Training provenance** — base hay instruct-tuned, quy trình alignment (ví dụ `"SFT + DPO + GRPO"`),
   quy mô dữ liệu huấn luyện, hỗ trợ đa ngữ; trích từ tài liệu chính thức.
3. **Capability profile** — *strengths* trích từ release note chính thức; *weaknesses* do teacher LLM
   tóm tắt từ một tập nhỏ preliminary rollout, sinh tự động, không cần chú giải thủ công.

Ví dụ trường weaknesses của Qwen3-4B: *"limited reasoning depth due to small parameter count, may
struggle with complex multi-step planning"*.

**Biện pháp chống rò rỉ.** P1 khẳng định model card **không chứa** kết quả đánh giá downstream (không
có success rate ALFWorld) và **không chứa** nhãn oracle dạng `prefers_concise`; tập rollout dùng để
tóm tắt weakness **rời** với tập đánh giá.

### 3.3 Hàm phần thưởng

```
R(F, S, e) = SR(F, S, e) − λ · NHR(F, S, e),      λ ∈ [0, 1]
```

với `SR ∈ {0,1}` là success của episode `e`, và `NHR` là **nothing-happens rate** — tỉ lệ bước mà
sau đó trạng thái môi trường không đổi, đóng vai trò biến đại diện cho hiện tượng *skill-induced
stalling* (agent phát ra hành động vô hiệu hoặc không hợp lệ). Mục tiêu tối ưu:

```
S*_F = argmax_S  E_{e ~ D} [ R(F, S, e) ]
```

Nhận xét về thiết kế: nếu chỉ tối ưu `SR`, thủ tục tìm kiếm không phân biệt được thất bại do skill
kém với thất bại do skill khiến agent lặp vòng vô ích cho tới khi cạn ngân sách bước. `NHR` chuyển
dạng thất bại thứ hai thành tín hiệu quan sát được, không cần gradient.

### 3.4 Giai đoạn 1 — general skills bằng hill climbing (P1, Thuật toán 1)

Lựa chọn hill climbing xuất phát từ ràng buộc tính toán: đánh giá **một** ứng viên general skill đòi
hỏi chạy agent trên **toàn bộ** task suite và tổng hợp phản hồi trên nhiều môi trường, khiến tìm
kiếm vét cạn bất khả thi.

Một vòng lặp gồm bốn bước:

1. **Rollout.** Chạy `F` với general skill tốt nhất hiện tại trên mọi loại tác vụ; tính phần thưởng
   trung bình.
2. **Analysis.** Teacher thu thập quỹ đạo thất bại và sinh **structured failure attribution**, tập
   trung vào khiếm khuyết hành vi ở cấp tổng quát thay vì lỗ hổng thủ tục theo tác vụ.
3. **Rewrite.** Teacher nhận đồng thời: skill hiện tại, failure attribution, `M_F`, và **K = 5 skill
   set có phần thưởng cao nhất trong toàn bộ lịch sử tìm kiếm**. Thành phần cuối cho phép teacher học
   từ toàn bộ quỹ đạo tối ưu hoá thay vì chỉ từ lần thất bại gần nhất.
4. **Accept/Reject.** Chấp nhận ứng viên khi và chỉ khi phần thưởng trung bình trên toàn task suite
   **vượt nghiêm ngặt** giá trị tốt nhất hiện tại.

Tham số (Phụ lục E.1): số vòng tối đa `I = 10`, patience `p = 3` (dừng sớm sau ba vòng liên tiếp
không cải thiện), kích thước lịch sử `K = 5`.

### 3.5 Giai đoạn 2 — task-specific skills bằng UCB tree search (P1, Thuật toán 2)

Khác biệt về bản chất so với general skill: trong cùng một loại tác vụ có thể tồn tại **nhiều chiến
lược khác biệt về cấu trúc** đều hiệu quả. Điều này đòi hỏi tìm kiếm dạng cây, cho phép khảo sát
nhiều nhánh thay vì cam kết vào một đường tinh chỉnh đơn.

Chạy **một cây độc lập cho mỗi loại tác vụ** `c` (các cây song song hoá hoàn toàn). Mỗi node lưu một
ứng viên `S^{T_c}`; mỗi cạnh tương ứng một lần teacher rewrite. Vòng lặp bốn bước:

- **Selection.** Chọn đệ quy leaf node theo UCB1:

  ```
  UCB1(n) = R̄(n) + C · sqrt( ln N_parent / N_n ),      C = 1.4
  ```

  với `R̄(n)` là phần thưởng điều chỉnh trung bình của `n` và toàn bộ hậu duệ, `N_n` là visit count.
- **Expansion.** Rollout `F` trên tác vụ loại `c` với skill set của node đã chọn; teacher thu quỹ đạo
  thất bại, sinh failure attribution, xuất `S^{T_c}` đã sửa đổi thành node con.
- **Evaluation.** Đánh giá node con trên tác vụ loại `c`, `N = 100` episode mỗi node.
- **Backpropagation.** Lan truyền phần thưởng từ node mới về root, cập nhật visit count và value
  estimate dọc đường.

Tham số: `J = 10` vòng cho mỗi loại tác vụ.

Hai giai đoạn chạy **tuần tự**: `S^{G*}` thu được từ Giai đoạn 1 giữ cố định trong suốt Giai đoạn 2.
Đầu ra cuối cùng là skill library đặc thù cho mô hình `S*_F = (S^{G*}_F, {S^{T_c*}_F}_{c∈C})`.

**Teacher LLM.** DeepSeek-V4-Pro.

### 3.6 MASA-Rewriter — chưng cất tìm kiếm thành một lượt forward

Hạn chế của pipeline tiến hoá: đòi hỏi hàng trăm đến hàng nghìn full-environment rollout **và** một
tín hiệu phần thưởng do môi trường cung cấp. Rewriter học chính **chính sách viết lại** mà pipeline
thực thi ngầm:

```
( M_F , S_Fin , d )  ⟶  S*_F
```

với `d` là mô tả tác vụ, `S_Fin` là skill set đầu vào (general hoặc task-specific), và đầu ra luôn
lấy từ các skill set điểm cao của pipeline tiến hoá.

**Thiết kế dữ liệu huấn luyện.** Để rewriter cải thiện được skill bất kể chất lượng đầu vào, `S_Fin`
được lấy mẫu chủ ý từ bốn nguồn phủ dải chất lượng rộng:

1. *search intermediates* — giai đoạn đầu, giữa và cuối của pipeline tiến hoá;
2. *cross-model transfers* — skill tối ưu của một backbone khác;
3. *one-shot teacher rewrites* — không qua tìm kiếm lặp;
4. *augmented variants* — nhiễu (noise ratio 0.3), khuyết (keep ratio 0.6), hoặc dài dòng.

**Cấu hình huấn luyện (Phụ lục F).** Qwen3-4B, full-parameter SFT, độ chính xác BF16. AdamW, lr
`1e-5`, cosine schedule, warmup 0.1, gradient checkpointing, effective batch size 4 (per-device 1 ×
gradient accumulation 4), 5 epoch, max sequence length 4096. Chọn checkpoint theo hội tụ training
loss. Hàm mất mát:

```
L = − E_{D_train} [ log p_θ ( S*_F | M_F , S_Fin , d ) ]
```

**Quy mô dữ liệu.** Rewriter kết hợp: **769 mẫu** (ALFWorld Pick/Look/Pick2 + WebShop + Search, loại
trừ các loại tác vụ held-out). Rewriter đặc thù môi trường: **499 mẫu** (WebShop + Search).

**Suy luận.** `S*_F = f_θ(M_F, S_Fin, d)` — một lượt forward, không tương tác môi trường, không tìm kiếm.

---

## 4. Thiết lập và kết quả thực nghiệm

### 4.1 Môi trường và baseline

**Ba môi trường.**

- **ALFWorld** — embodied household dạng văn bản, sáu loại tác vụ (Pick, Look, Clean, Heat, Cool, Pick2).
- **WebShop** — mô phỏng mua sắm trực tuyến: điều hướng, tìm sản phẩm, so sánh thuộc tính, quyết định
  mua theo đặc tả ngôn ngữ tự nhiên.
- **Search-augmented QA** — bảy benchmark: single-hop NQ, TriviaQA, PopQA; multi-hop HotpotQA, 2Wiki,
  MuSiQue, Bamboogle. Tiến hoá skill **chỉ thực hiện trên NQ và HotpotQA**; các benchmark còn lại là
  out-of-domain.

**Ba baseline.** (1) `No Skill` — backbone thuần. (2) `Base Skill` — library gốc của SkillRL dùng
chung cho mọi backbone, đồng thời là điểm khởi tạo `S^{G_0}` và `S^{T_c0}` của MASA.
(3) `DS-Adapter` — teacher rewrite **một lần**, đã điều kiện hoá trên model card, **không** tìm kiếm lặp.

`DS-Adapter` là baseline có giá trị chẩn đoán cao vì nó đã mang tính model-aware; chênh lệch
MASA − DS-Adapter do đó cô lập đóng góp của **thủ tục tìm kiếm lặp**.

Toàn bộ backbone Qwen3 vận hành ở non-thinking mode, phản ánh kịch bản triển khai có ràng buộc độ
trễ và ngân sách token, đồng thời bảo đảm khác biệt hiệu năng quy về thiết kế skill.

### 4.2 Kết quả trên ALFWorld (P1, Bảng 1)

| Mô hình | No Skill | Base Skill | DS-Adapter | **MASA** | Δ so với baseline mạnh nhất | Steps ↓ |
|---|---|---|---|---|---|---|
| Qwen3-4B | 17.1 | 20.0 | 27.1 | **31.4** | +4.3 | 44.6 → **38.4** |
| Qwen3-8B | 32.1 | 25.0 | 27.1 | **57.9** | **+25.8** | 39.1 → **29.2** |
| Qwen3-14B | 37.9 | 42.1 | 44.3 | **64.3** | +20.0 | 36.7 → **25.7** |
| Qwen3-32B | 36.4 | 41.4 | 45.0 | **65.7** | +20.7 | 37.0 → **24.3** |

Với Qwen3-14B và Qwen3-32B, MASA đứng đầu **đồng thời trên cả sáu loại tác vụ**, cho thấy cải thiện
tổng thể không đánh đổi bằng suy giảm độ phủ. Mức tăng ở 4B khiêm tốn hơn, được giải thích bằng trần
năng lực nội tại của backbone giới hạn mức hữu ích mà skill guidance có thể mang lại.

### 4.3 Kết quả trên WebShop và chẩn đoán nguyên nhân

| Mô hình | No Skill | Base Skill | DS-Adapter | **MASA** |
|---|---|---|---|---|
| Qwen3-4B | 23.0 | 19.4 | 19.2 | **26.4** |
| Qwen3-8B | 4.6 | 6.0 | 4.8 | **28.6** |
| Qwen3-14B | 2.8 | 1.6 | 2.0 | **29.2** |
| Qwen3-32B | 6.6 | 7.2 | 3.6 | **34.6** |

Điều kiện baseline bộc lộ một nghịch lý: **mô hình lớn hơn cho hiệu năng kém hơn rõ rệt** (Qwen3-14B
`No Skill` đạt 2.8% so với Qwen3-4B đạt 23.0%). Phụ lục G.1 định lượng cơ chế (P1, Bảng 7):

| Mô hình | Tỉ lệ bước có preamble suy luận | Độ dài hành động trung bình |
|---|---|---|
| Qwen3-4B | **0%** | 73 ký tự |
| Qwen3-8B | 57% | 1.021 ký tự |
| Qwen3-14B | **97%** | 574 ký tự |
| Qwen3-32B | 66% | 491 ký tự |

Các mô hình lớn sinh preamble suy luận dài trước mỗi lệnh hành động, làm cạn ngân sách bước cố định
vào hoạt động cân nhắc thay vì tương tác môi trường; agent khảo sát nhiều phương án nhưng không hoàn
tất đủ hành động mua để đạt success. Skill model-agnostic không được thiết kế để xử lý pattern hành
vi đặc thù mô hình này, và trong một số trường hợp còn làm hiệu năng xấu thêm.

MASA đảo ngược pattern nói trên đồng thời cải thiện hiệu suất: các baseline thành công cần trung bình
12–13 bước, trong khi MASA đạt success rate cao hơn chỉ trong 7–8 bước, riêng Qwen3-8B giảm còn
**4.7 bước**.

### 4.4 Kết quả trên Search-augmented QA (P1, Bảng 2)

Success rate trung bình trên bảy dataset: 4B 32.9 → **36.9**; 8B 31.3 → **37.2**; 14B 37.6 → **39.0**;
32B 38.1 → **41.5**. MASA đạt success rate trung bình cao nhất trên **cả bốn** backbone.

Vì tiến hoá skill chỉ thực hiện trên NQ và HotpotQA, các mức tăng trên benchmark out-of-domain có ý
nghĩa chẩn đoán: trên Qwen3-4B, Bamboogle tăng từ 12.9 (baseline mạnh nhất) lên **61.3**. Trên
backbone 32B, MASA đứng đầu 5 trên 7 dataset. Kết quả cho thấy skill đã tiến hoá nắm bắt được chiến
lược truy hồi và suy luận có tính khả chuyển, thay vì overfit vào dataset dùng trong tiến hoá.

### 4.5 Nghiên cứu loại bỏ (ablation)

**Cấu trúc tìm kiếm hai giai đoạn (P1, Bảng 3a).** Thay skill đã tiến hoá của từng giai đoạn bằng
teacher rewrite một lần:

| Biến thể | 4B | 8B | 14B | 32B |
|---|---|---|---|---|
| ALFWorld — pipeline đầy đủ | 31.4 | 57.9 | 64.3 | 65.7 |
| ALFWorld — bỏ task-specific | 25.0 | 32.9 | 63.6 | 50.0 |
| ALFWorld — bỏ general | 25.0 | 50.0 | 47.9 | 64.3 |
| WebShop — pipeline đầy đủ | 26.4 | 28.6 | 29.2 | 34.6 |
| WebShop — bỏ task-specific | 22.4 | 25.6 | 24.2 | 31.8 |
| WebShop — bỏ general | 23.4 | 7.2 | 10.2 | 9.6 |

Cả hai giai đoạn đều đóng góp, song tầm quan trọng tương đối phụ thuộc **môi trường và quy mô**.
Trên ALFWorld, loại bỏ tìm kiếm task-specific gây tổn thất lớn nhất ở Qwen3-8B (−25.0) và Qwen3-32B
(−15.7); loại bỏ general skill ảnh hưởng nặng nhất tới Qwen3-14B (−16.4). Trên WebShop, loại bỏ
general skill là tổn thất nghiêm trọng ở 8B/14B/32B (success rate rơi xuống một chữ số), trong khi
loại bỏ task-specific có tác động vừa phải. Bất đối xứng này phản ánh bản chất môi trường: WebShop
đòi hỏi chiến lược quyết định cấp cao nhất quán (do general skill mã hoá), còn ALFWorld đòi hỏi chuỗi
thủ tục chi tiết (do task-specific skill mã hoá).

**Model card trong MASA-Rewriter (P1, Bảng 3b).** Loại bỏ model card khỏi đầu vào của rewriter;
kết quả là success rate trung bình trên ba loại tác vụ ALFWorld held-out (Clean/Heat/Cool):

| Biến thể | 4B | 8B | 14B | 32B |
|---|---|---|---|---|
| Cross-env, có model card | 32.4 | 32.4 | 38.2 | 51.5 |
| Cross-env, không model card | 14.7 | 23.5 | 33.8 | 39.7 |
| Cross-task, có model card | 32.5 | 38.2 | 48.5 | 55.9 |
| Cross-task, không model card | 8.8 | 30.9 | 20.6 | 42.5 |

Loại bỏ model card làm suy giảm hiệu năng **nhất quán trên mọi cấu hình**, với khoảng cách rõ rệt
nhất ở các backbone nhỏ (Qwen3-4B, cross-task: 32.5 → 8.8). Model card do đó mang tín hiệu điều kiện
hoá thực chất.

### 4.6 Khả năng tổng quát hoá của rewriter (P1, §4.3, Hình 3)

Thiết kế: giữ lại ba loại tác vụ ALFWorld (Clean, Heat, Cool) khỏi tập huấn luyện của rewriter, yêu
cầu rewriter sinh task-specific skill cho chúng; general skill giữ nguyên. Baseline bổ sung:
`4B-Rewrite` — Qwen3-4B dùng làm rewriter **không qua huấn luyện**, tức cùng kiến trúc nhưng thiếu
năng lực viết lại đã học.

| Backbone | Base Skill | 4B-Rewrite | DS-Adapter | Cross-env | Cross-task |
|---|---|---|---|---|---|
| Qwen3-4B | 22.0 | 16.2 | 30.9 | 32.4 | **32.5** |
| Qwen3-8B | 30.9 | 25.0 | 29.4 | 32.4 | **38.2** |
| Qwen3-14B | 30.9 | 26.5 | 35.3 | 38.2 | **48.5** |
| Qwen3-32B | 44.1 | 41.2 | 48.5 | 51.5 | **55.9** |

**Chuyển giao xuyên môi trường (Cross-env).** Rewriter huấn luyện *độc quyền* trên evolution trace từ
Search và WebShop, sau đó áp dụng cho ALFWorld mà không có bất kỳ dữ liệu in-environment nào. Vẫn
vượt DS-Adapter trên cả bốn backbone (+1.5 / +3.0 / +2.9 / +3.0), bất chấp khoảng cách môi trường
đáng kể về action space và định dạng observation. Kết quả cho thấy chính sách viết lại đã học nắm bắt
pattern thích ứng **đặc thù mô hình**, khả chuyển qua môi trường.

**Chuyển giao xuyên tác vụ (Cross-task).** Bổ sung trace ALFWorld Pick/Look/Pick2 vào tập huấn luyện
(loại trừ các loại tác vụ đánh giá) mang lại cải thiện nhất quán, với mức tăng so với DS-Adapter là
+8.8 (8B), +13.2 (14B), +7.4 (32B), và so với Cross-env là +5.8 (8B), +10.3 (14B).

Hai quan sát bổ sung. Thứ nhất, **MASA-Rewriter với 4B tham số vượt nhất quán DS-Adapter vận hành
bằng DeepSeek-V4**, tức một mô hình nhỏ đã huấn luyện vượt teacher lớn hơn nhiều bậc ở một phần nhỏ
chi phí suy luận. Thứ hai, `4B-Rewrite` (chưa huấn luyện) **thấp hơn cả `Base Skill`** trên mọi
backbone, xác nhận năng lực viết lại là kết quả huấn luyện chứ không sẵn có ở mô hình 4B.

### 4.7 Phân tích định tính (Phụ lục I, Bảng 9)

Nghiên cứu trường hợp trên subtask có tỉ lệ thất bại cao nhất của WebShop — quyết định khớp màu sản
phẩm. Với **cùng một** subtask, pipeline tiến hoá sinh ra **bốn chiến lược khác biệt về bản chất**,
mỗi chiến lược nhắm vào failure mode chủ đạo của backbone tương ứng:

| Backbone | Failure mode chủ đạo | Chiến lược trong skill đã tiến hoá |
|---|---|---|
| Qwen3-4B | đoán các màu trông tương tự | **Khớp nhị phân nghiêm ngặt**: khớp chính xác chuỗi hoặc rời ngay. *"'navy blue' ≠ 'light blue' ≠ 'navy'. Do NOT select an approximate or similar color."* |
| Qwen3-8B | từ bỏ sản phẩm quá sớm, nhận điểm 0 | **Khớp linh hoạt**: mua phương án gần nhất để nhận partial credit. *"Even if the product doesn't perfectly match—BUY IT."* |
| Qwen3-14B | tiêu hao bước vào cân nhắc sản phẩm không phù hợp | **Thoát nhanh một bước**: không có exact match thì `back to search` ngay. *"Takes 1 step, not 5."* |
| Qwen3-32B | xử lý sai tên màu dạng mã (`d01green`) | **Khớp chuỗi con có luật ưu tiên**: *"'green' matches 'a1-green' or 'd01green' (contains)… prefer the one that matches more of the full color name."* |

Kết quả này là bằng chứng định tính cho luận điểm trung tâm: thích ứng có điều kiện theo mô hình diễn
ra ở cấp **chiến lược quyết định**, không phải ở cấp diễn đạt câu chữ; cùng một vấn đề đòi hỏi các
giải pháp khác biệt về bản chất tuỳ theo dạng thất bại của từng backbone.

Phân tích theo hạng mục (Phụ lục G.2, Bảng 8) cho thấy cải thiện của MASA phân bố rộng trên phần lớn
hạng mục WebShop ở mọi backbone, chứ không tập trung vào một hạng mục dễ nhằm nâng giá trị trung bình.

---

## 5. Hạn chế

### 5.1 Hạn chế do tác giả nêu

- Bằng chứng thực nghiệm giới hạn ở họ **Qwen3** (4B–32B) với kiểm chứng bổ trợ trên Gemma3. Mở rộng
  sang Llama, Mistral, GPT-o3, Claude đòi hỏi khối lượng tính toán lớn hơn đáng kể.
- Với mô hình closed-source, pipeline tiến hoá đòi hỏi **hàng trăm rollout qua API trả phí**, khiến
  chi phí tìm kiếm trên mỗi backbone cao hơn rõ rệt so với mô hình tự host. Rewriter là biện pháp
  giảm nhẹ một phần thông qua khấu hao chi phí khi đã có trace từ một số backbone.
- Pipeline phụ thuộc vào môi trường cung cấp **tín hiệu success/failure tự động**. Với tác vụ web mở
  hoặc ứng dụng thực tế thiếu reward tích hợp, cần thiết kế bộ đánh giá ngoài hoặc chú giải thủ công.
- Khía cạnh đạo đức: skill đã tiến hoá có thể kế thừa thiên lệch hoặc heuristic không an toàn từ quỹ
  đạo và phản hồi dùng trong tối ưu; cần kiểm duyệt trước khi triển khai. Việc nâng năng lực agent
  trong môi trường tương tác cũng đòi hỏi giám sát bổ sung trong bối cảnh an toàn trọng yếu.

### 5.2 Nhận xét phản biện bổ sung

1. **Không báo cáo phương sai hay số seed.** Thủ tục tìm kiếm dựa trên rollout ngẫu nhiên, teacher
   sinh văn bản ở nhiệt độ khác 0, và quy tắc accept/reject dựa trên so sánh phần thưởng điểm. Công
   bố không cung cấp error bar hay số lần lặp độc lập. Mức tăng +25.8 điểm (8B) đủ lớn để không phụ
   thuộc vào yếu tố này; mức tăng +4.3 điểm (4B) thì không.
2. **Khả năng rò rỉ gián tiếp qua trường weaknesses.** Trường này do teacher tóm tắt từ preliminary
   rollout. Tuyên bố về tính rời của tập rollout và việc không đưa nhãn oracle là hợp lệ, song nếu
   weakness được tóm tắt từ rollout trên **cùng môi trường đánh giá**, nó vẫn mang thông tin gián
   tiếp về môi trường đó. Công bố không có ablation cho biến thể "model card chỉ gồm metadata tĩnh".
3. **Chi phí tìm kiếm không được lượng hoá.** Riêng Giai đoạn 2 có `J = 10` vòng × `N = 100` episode
   cho mỗi loại tác vụ; với sáu loại tác vụ ALFWorld, con số là bậc 6.000 episode cho **một** backbone,
   chưa kể Giai đoạn 1. Không có bảng chi phí tính toán, dù đây chính là động cơ tồn tại của rewriter.
4. **Không nêu giá trị `k_G` và `k_T`.** Số skill truy hồi tại mỗi bước ảnh hưởng trực tiếp tới hiệu
   ứng granularity (truy hồi nhiều skill `Detailed` tạo tải token lớn), nhưng không được nêu giá trị
   cụ thể lẫn ablation.
5. Tác giả khai báo LLM chỉ được dùng cho hiệu đính ngữ pháp và văn phong (Phụ lục J).

---

## 6. Các yếu tố phương pháp có giá trị tái sử dụng

1. **Thiết kế thí nghiệm cô lập biến của §2** — giữ cố định nội dung nguyên lý và pipeline truy hồi,
   chỉ biến thiên biểu đạt. Đây là mẫu thiết kế áp dụng được cho mọi khẳng định dạng "yếu tố X ảnh
   hưởng tới hiệu năng agent".
2. **`NHR` như một số hạng reward shaping không cần gradient** — chi phí thấp, quan sát trực tiếp từ
   môi trường, và phân biệt được dạng thất bại mà success rate không phân biệt.
3. **Chưng cất thủ tục tìm kiếm vào một mô hình nhỏ**, với dữ liệu đầu vào lấy mẫu phủ dải chất lượng
   rộng (bao gồm nhiễu, khuyết và chuyển giao xuyên mô hình). 769 mẫu đủ để student 4B vượt teacher,
   với điều kiện dữ liệu phủ đúng phổ điều kiện đầu vào sẽ gặp khi triển khai.
