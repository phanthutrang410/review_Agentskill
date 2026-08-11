# Phân tích đối chiếu và đánh giá độ tin cậy

Tài liệu đối chiếu P1 (MASA, arXiv 2605.30723) và P2 (Skill-MAS, arXiv 2606.18837) theo trục phương
pháp luận, đánh giá độ tin cậy của các khẳng định thực nghiệm, và xác định các khe hở nghiên cứu còn
trống.

---

## 1. Đối chiếu cấu trúc

### 1.1 Điểm chung

Hai công bố độc lập về nhóm tác giả và cơ sở, cách nhau 26 ngày nộp, nhưng đồng quy về một **khuôn
mẫu phương pháp chung**:

```
tài liệu skill dạng văn bản
        │
        ▼
   rollout trên môi trường / benchmark
        │
        ▼
   thu quỹ đạo thất bại, quy lỗi có cấu trúc (LLM thực hiện)
        │
        ▼
   viết lại tài liệu skill (LLM thực hiện, có ràng buộc)
        │
        ▼
   chấp nhận nếu chỉ số tăng ──► lặp
```

Bốn tiền đề chung:

1. **Không cập nhật tham số.** Cả hai đóng băng LLM chính và chỉ tối ưu văn bản.
2. **Skill là văn bản, không phải code hay trọng số.** Do đó "học" quy về biên tập tài liệu.
3. **LLM đóng vai trò optimizer.** Bước viết lại do một LLM thực hiện, không phải thuật toán ký hiệu.
4. **Phần thưởng đến từ môi trường.** Quy tắc chấp nhận dựa trên một chỉ số đo được từ bên ngoài.

### 1.2 Điểm khác biệt

| Trục | P1 — MASA | P2 — Skill-MAS |
|---|---|---|
| Đối tượng tối ưu | Skill library truy hồi động, nhiều mục, cho **một agent** | Một tài liệu Meta-Skill đơn khối cho **meta-agent** |
| Cấp trừu tượng của skill | Execution-level (làm gì ở bước tiếp theo) | Architecture-level (thiết kế hệ ra sao) |
| Chiều thích ứng chủ đạo | **Theo backbone** — chủ đích model-specific | **Theo tác vụ** — chủ đích model-agnostic |
| Cấu trúc tìm kiếm | Hai giai đoạn: hill climbing (general) + UCB tree search (task-specific) | Một vòng lặp tuần tự `R = 10`, không phân nhánh |
| Cơ chế chống nhiễu | Số episode lớn khi đánh giá node (`N = 100`) | Lấy mẫu `K = 5` quỹ đạo, tách thống kê `u` và `d` |
| Chọn mục tiêu can thiệp | Toàn bộ task suite hoặc toàn bộ một loại tác vụ | Tập con thích ứng, chọn bằng elbow detection |
| Kiểm soát tăng trưởng skill | Không nêu ràng buộc tường minh | Ba ràng buộc tường minh (prune-before-add, ≤1 upgrade, abstraction firewall) |
| Cơ chế giảm chi phí triển khai | Chưng cất vào rewriter Qwen3-4B | Tái sử dụng `S*` cho mọi truy vấn |
| Backbone khảo sát | Qwen3 4B–32B (+ Gemma3 bổ trợ), toàn bộ open-weight | Gemini-3.1-Flash, GPT-5.4-Nano, Qwen3.5-Plus, DeepSeek-V4-Flash — hỗn hợp proprietary/open |
| Môi trường | ALFWorld, WebShop, Search-QA (7 dataset) | DeepResearchBench, HLE-Math, BrowseComp-Plus, VitaBench |

### 1.3 Đối lập đáng chú ý về triết lý thiết kế

Hai công bố đưa ra hai lập trường **trái ngược trực tiếp** về chiều thích ứng của skill.

P1 lập luận rằng skill **phải** đặc thù theo mô hình, và cung cấp bằng chứng định lượng cho tổn thất
khi bỏ qua chiều này (Bảng 3b: loại bỏ model card làm Qwen3-4B cross-task rơi từ 32.5 xuống 8.8).

P2 áp *abstraction firewall* cấm mọi chi tiết đặc thù, và xem tính **task-agnostic và model-agnostic**
là mục tiêu thiết kế; kết quả chuyển giao tại Bảng 2 Panel A được diễn giải như bằng chứng cho tính
đúng đắn của lựa chọn này.

Hai lập trường không mâu thuẫn về logic, do chúng thao tác ở hai cấp trừu tượng khác nhau:

- **Execution-level skill** mã hoá *cách hành động trong một tình huống cụ thể*. Hiệu quả của nó phụ
  thuộc thói quen sinh văn bản của backbone — mô hình sinh preamble dài cần luật khác với mô hình
  hành động trực tiếp. Đây là địa hạt của P1.
- **Architecture-level skill** mã hoá *cách phân rã vấn đề và bố trí kiểm chứng*. Nguyên tắc dạng
  "tách nhánh song song cho các ràng buộc độc lập rồi hợp nhất có verification gate" đúng bất kể
  backbone nào thi hành. Đây là địa hạt của P2.

**Giả thuyết rút ra:** tồn tại một *ranh giới trừu tượng* mà bên dưới nó skill cần điều kiện hoá theo
mô hình, còn bên trên nó skill khả chuyển. Không công bố nào định vị ranh giới này; đây là một khe hở
nghiên cứu (§4.1).

---

## 2. Mức độ bổ sung lẫn nhau

Ba cơ chế của P2 khả dụng trực tiếp cho bài toán của P1, và ngược lại.

**(a) Multi-trajectory rollout áp vào P1.** Quy tắc accept/reject trong hill climbing của P1 dựa trên
so sánh phần thưởng trung bình. Nếu bổ sung `u_i = std(scores)` như P2, quy tắc có thể phân biệt
"skill mới thực sự tốt hơn" với "skill mới may mắn ở lần đánh giá này" — điều mà giá trị trung bình
đơn thuần không cung cấp. Chi phí bổ sung bằng không, vì P1 vốn đã chạy `N = 100` episode mỗi node.

**(b) Elbow detection áp vào P1.** Giai đoạn 2 của P1 chạy tree search cho **mọi** loại tác vụ với
ngân sách đồng đều (`J = 10`, `N = 100`). Nhưng chính P1 ghi nhận biến thiên hiệu năng theo loại tác
vụ vượt 60 điểm (Kết luận 3). Phân bổ ngân sách theo điểm ưu tiên kiểu P2 sẽ tập trung tìm kiếm vào
các loại tác vụ có khiếm khuyết lớn nhất thay vì trải đều.

**(c) Model card áp vào P2.** P2 dùng cùng một Meta-Skill khởi tạo cho cả bốn meta-agent, và cùng một
quy trình tiến hoá. Kết quả Bảng 2 Panel A cho thấy Meta-Skill khả chuyển tốt qua LLM, nhưng
**không loại trừ** khả năng điều kiện hoá theo mô hình còn cải thiện thêm — đặc biệt khi lưu ý rằng
`Skill-MAS-init` đạt 21.68 trên Gemini nhưng chỉ 19.64 trên GPT-5.4-Nano, tức cùng một tài liệu cho
hiệu quả khác nhau theo backbone. Đây là dấu hiệu định lượng cho thấy chiều model-awareness chưa được
khai thác ở cấp architecture-level.

**(d) Chưng cất vào mô hình nhỏ áp vào P2.** Chi phí tiến hoá của P2 dao động 9,35–59,06 USD cho mỗi
cặp (LLM, benchmark). Cơ chế rewriter của P1 — huấn luyện một mô hình nhỏ trên các cặp
(input skill, output skill tối ưu) — về nguyên tắc áp dụng được cho Meta-Skill, với tiềm năng khấu
hao chi phí này qua nhiều miền.

---

## 3. Đánh giá độ tin cậy

### 3.1 Vấn đề chung: không báo cáo phương sai

Không công bố nào cung cấp error bar, khoảng tin cậy, hay số lần lặp độc lập, mặc dù quy trình của cả
hai đều ngẫu nhiên ở nhiều điểm: rollout của môi trường, sinh văn bản của teacher ở nhiệt độ khác 0,
và quy tắc chấp nhận dựa trên so sánh điểm.

Điểm này đặc biệt đáng lưu ý ở P2, vì đóng góp phương pháp trung tâm của chính công bố là luận điểm
rằng **một quỹ đạo đơn không đủ để phân biệt tín hiệu với nhiễu**. Nguyên tắc đó được áp cho vòng lặp
tối ưu bên trong nhưng không áp cho các con số báo cáo cuối cùng.

**Phân loại các khẳng định theo mức chịu ảnh hưởng:**

| Khẳng định | Biên độ | Đánh giá |
|---|---|---|
| P1: MASA cải thiện ALFWorld ở 8B/14B/32B | +20.0 đến +25.8 điểm | **Vững** — biên độ vượt xa mọi mức nhiễu hợp lý |
| P1: MASA cải thiện WebShop | 2.8 → 29.2 (14B) | **Vững** |
| P1: model card là cần thiết | 32.5 → 8.8 khi loại bỏ | **Vững** |
| P1: MASA cải thiện ALFWorld ở 4B | +4.3 điểm | **Không kết luận được** khi thiếu phương sai |
| P2: `Skill-MAS-optimized` vượt mọi baseline | +5.4 đến +8.2 điểm Avg.Perf | **Vững** |
| P2: `Skill-MAS-init` đã cạnh tranh | +0.39 điểm trên Gemini | **Không vững** — nhỏ hơn giá trị một câu trong test set 84 câu (≈1.19 điểm) |
| P2: cross-task transfer hiệu quả | Δ +3.57 đến +7.15 | **Vừa phải** — đủ lớn để nhiều khả năng thật, nhưng không kiểm định được |

### 3.2 Vấn đề riêng của P2: xung đột vai trò của judge

Gemini-3.1-Flash đồng thời là LLM-as-a-judge mặc định và là một trong bốn meta-agent được đánh giá.
Trên DeepResearchBench và VitaBench — hai benchmark chấm bằng LLM hoặc rubric — cấu hình này tạo rủi
ro self-preference. Không có kiểm tra chéo bằng judge độc lập.

Mức tác động khả dĩ: đáng chú ý là **cột DRB của Gemini là ô duy nhất mà `Skill-MAS-optimized` đạt
44.71**, cao hơn mọi baseline trên cùng cột, trong khi ở meta-agent GPT-5.4-Nano thì DRB lại là ô duy
nhất `Skill-MAS-optimized` **thua** baseline. Dữ liệu hiện có không đủ để quy kết quan hệ nhân quả,
nhưng đây là cấu hình đáng được kiểm chứng lại bằng judge thứ hai.

### 3.3 Vấn đề riêng của P2: quy mô validation set

DeepResearchBench và VitaBench chỉ có 16 tác vụ validation. Với `K = 5`, mỗi vòng sinh 80 quỹ đạo, và
elbow selection chọn ra một tập con nhỏ hơn nữa. Việc chọn `S*` — Meta-Skill cuối cùng — dựa trên
validation score tính trên 16 tác vụ tiềm ẩn rủi ro selection noise không nhỏ, đặc biệt khi có `R = 10`
ứng viên để chọn.

### 3.4 Vấn đề riêng của P1: rò rỉ gián tiếp qua trường weaknesses

Trường `weaknesses` trong model card do teacher LLM tóm tắt từ "preliminary rollout". P1 khẳng định
tập rollout này rời với tập đánh giá và không chứa nhãn oracle — hai biện pháp hợp lệ. Tuy nhiên nếu
weakness được tóm tắt từ rollout trên **cùng môi trường đánh giá**, nó vẫn mang thông tin gián tiếp về
môi trường đó, và ranh giới giữa "hồ sơ mô hình" với "quan sát về hiệu năng trên môi trường đích" trở
nên mờ.

Ablation cần thiết nhưng vắng mặt: biến thể model card **chỉ gồm metadata tĩnh** (kiến trúc + training
provenance + strengths từ release note), loại bỏ hoàn toàn phần weakness quan sát được. Nếu biến thể
này vẫn giữ phần lớn hiệu quả, khẳng định về model-awareness được củng cố đáng kể.

### 3.5 Vấn đề riêng của P1: chi phí tìm kiếm không được lượng hoá

Riêng Giai đoạn 2 gồm `J = 10` vòng × `N = 100` episode cho mỗi loại tác vụ. Với sáu loại tác vụ của
ALFWorld, con số là bậc 6.000 episode cho **một** backbone, chưa kể Giai đoạn 1 và các lời gọi teacher
DeepSeek-V4-Pro. Không có bảng chi phí tính toán trong công bố, mặc dù chính chi phí này là động cơ
tồn tại của MASA-Rewriter. Đối chiếu: P2 cung cấp bảng chi phí đầy đủ (Bảng 6).

### 3.6 Vấn đề riêng của P2: phạm vi của build contract

`MAS_BUILD_CONTRACT` (Hình 17–18) là một tài liệu ràng buộc kỹ thuật dài, chứa các quy định rất cụ
thể ở mức cú pháp: *"prefer triple-single-quoted raw strings"*, *"`state['final_output']` must be
exactly one assignment"*, *"any loop must be capped at maximum 3 iterations"*, *"do NOT emit
`from __future__ import annotations`"*. Tài liệu này nằm **bên ngoài** Meta-Skill nên không vi phạm
abstraction firewall về mặt hình thức. Nhưng hệ quả là một phần đáng kể độ ổn định của hệ — cụ thể là
tỉ lệ sinh ra MAS code chạy được — đến từ khối engineering constraint thủ công này, và khối đó không
được ablate.

Ghi nhận liên quan: baseline MAS² **thất bại trong việc sinh executable MAS code** trên BrowseComp-Plus
(Bảng 7). Điều này cho thấy tỉ lệ sinh code hợp lệ là một biến quan trọng và biến thiên mạnh giữa các
phương pháp. Khi trích dẫn P2 để lập luận rằng training-time MAS kém, cần tách bạch hai nguyên nhân:
hạn chế của paradigm và lỗi kỹ thuật của cài đặt baseline.

### 3.7 Hạn chế chung: phụ thuộc reward tự động

Cả hai phương pháp đòi hỏi một tín hiệu phần thưởng tự động:

| | Nguồn tín hiệu | Điều gì xảy ra khi thiếu |
|---|---|---|
| P1 | success/failure flag do môi trường cung cấp | Quy tắc accept/reject của hill climbing và value estimate của tree search đều mất cơ sở; P1 nêu cần thiết kế external evaluator hoặc chú giải thủ công |
| P2 | ground-truth label để chấm quỹ đạo | Định lượng tại Bảng 3: `Full-Validation` và `Half-Validation` suy giảm trên toàn bộ bốn cấu hình khảo sát |

Đây là ràng buộc nghiêm trọng nhất đối với khả năng áp dụng thực tế, vì phần lớn tác vụ agent trong
môi trường sản xuất không có success flag tự động.

---

## 4. Khe hở nghiên cứu

### 4.1 Định vị ranh giới giữa model-specific và model-agnostic skill

**Vấn đề.** P1 chứng minh execution-level skill cần điều kiện hoá theo mô hình; P2 chứng minh
architecture-level skill khả chuyển qua mô hình. Không công bố nào khảo sát vùng ở giữa.

**Thiết kế khả dĩ.** Xây một phổ skill từ cụ thể tới trừu tượng trên cùng một môi trường, ví dụ bốn
mức: (i) chuỗi lệnh cụ thể, (ii) thủ tục theo loại tác vụ, (iii) nguyên tắc chiến lược xuyên tác vụ,
(iv) nguyên tắc phân rã và kiểm chứng ở cấp kiến trúc. Với mỗi mức, đo mức suy giảm khi chuyển giao
qua backbone. Đại lượng cần ước lượng: **hệ số transfer theo cấp trừu tượng**.

**Giá trị.** Kết quả cho phép quyết định — với ngân sách hạn chế — nên đầu tư tìm kiếm ở cấp nào, và
cấp nào có thể xây một lần dùng chung.

### 4.2 Skill evolution không cần nhãn

**Vấn đề.** Cả hai công bố đều nêu đây là hạn chế; P2 nêu tường minh trong mục Limitations và đề xuất
hướng dùng self-confidence score của meta-agent.

**Quan sát mở đường.** P2 đã cung cấp một tín hiệu **không cần nhãn** ngay trong pipeline của mình:
`u_i = std(scores)`, độ bất định giữa các quỹ đạo. Đại lượng này tính được **hoàn toàn không cần
ground truth** — chỉ cần chạy cùng một tác vụ `K` lần và đo mức phân tán của kết quả. Tuy nhiên trong
P2 nó vẫn được tính từ điểm số có nhãn.

**Thiết kế khả dĩ.** Thay `s_{i,k}` bằng một đại lượng tự giám sát, ví dụ: mức đồng thuận giữa `K`
đầu ra (self-consistency), độ ổn định của cấu trúc MAS được sinh ra qua các lần chạy, hoặc tỉ lệ
verification gate nội bộ được thoả mãn. Sau đó giữ nguyên toàn bộ pipeline còn lại của P2 và đo mức
suy giảm so với biến thể có nhãn. Bảng 3 của P2 đã cung cấp sẵn hai điểm tham chiếu để so sánh.

### 4.3 Chưng cất Meta-Skill

**Vấn đề.** Chi phí tiến hoá của P2 là 9,35–59,06 USD cho mỗi cặp (LLM, benchmark), phải trả lại cho
mỗi miền mới.

**Thiết kế khả dĩ.** Áp trực tiếp cơ chế rewriter của P1: thu thập các cặp (Meta-Skill đầu vào, đặc tả
miền) → Meta-Skill tối ưu từ nhiều lần chạy tiến hoá, rồi SFT một mô hình nhỏ. P1 cho thấy 769 mẫu là
đủ ở execution-level; câu hỏi mở là quy mô dữ liệu cần thiết ở architecture-level, nơi mỗi mẫu là một
tài liệu dài hơn nhiều bậc.

**Rủi ro cần lường trước.** Meta-Skill tối ưu trong P2 dài hàng nghìn token (Hình 6–14). Chi phí thu
thập dữ liệu huấn luyện có thể vượt lợi ích khấu hao nếu số miền đích nhỏ.

### 4.4 Kết hợp hai cấp skill trong một hệ

**Vấn đề.** P2 tối ưu meta-agent nhưng để nguyên các sub-agent do meta-agent sinh ra. P1 tối ưu agent
thực thi nhưng không xét bối cảnh multi-agent. Chưa có công bố nào tối ưu đồng thời hai cấp.

**Câu hỏi nghiên cứu.** Hai cấp tối ưu có tương tác cộng hưởng, trung tính, hay triệt tiêu? Khả năng
triệt tiêu là thực: Meta-Skill có thể sinh ra một sub-agent với vai trò giả định rằng backbone hành
xử theo cách A, trong khi execution-level skill lại đang điều chỉnh backbone đó theo cách B.

**Ghi chú thiết kế.** Nếu triển khai, cần lịch tối ưu xen kẽ (alternating optimization) với ràng buộc
mỗi vòng chỉ đổi một cấp — tương tự nguyên tắc "≤1 upgrade mỗi module mỗi vòng" của P2, mở rộng lên
một bậc.

### 4.5 Kiểm định lại kết quả với thiết kế thống kê đầy đủ

**Vấn đề.** Như §3.1, các khẳng định biên độ nhỏ ở cả hai công bố không kiểm định được.

**Thiết kế khả dĩ.** Lặp lại các cấu hình có biên độ nhỏ (P1 ở Qwen3-4B; P2 ở `Skill-MAS-init`) với
tối thiểu 3–5 seed độc lập, báo cáo trung bình kèm độ lệch chuẩn. Đây là công việc chi phí thấp về
mặt trí tuệ nhưng có giá trị cao về mặt bằng chứng, và khả thi trên phần cứng hạn chế nếu giới hạn ở
các backbone nhỏ.

---

## 5. Kết luận đối chiếu

Hai công bố cùng thuộc một khuôn mẫu phương pháp — tiến hoá tài liệu văn bản dưới dẫn hướng của phản
hồi môi trường, không cập nhật tham số — nhưng thao tác ở hai cấp trừu tượng khác nhau và do đó đưa
ra hai kết luận biểu kiến trái ngược về chiều thích ứng của skill. Sự trái ngược này không phải mâu
thuẫn mà là chỉ dấu cho một câu hỏi chưa được trả lời: skill trở nên khả chuyển ở cấp trừu tượng nào.

Về đóng góp phương pháp, P1 mạnh hơn ở **thiết kế thí nghiệm** (§2 của P1 là một nghiên cứu cô lập
biến có thể tái sử dụng làm mẫu) và ở **cơ chế giảm chi phí triển khai** (chưng cất tìm kiếm vào mô
hình 4B). P2 mạnh hơn ở **xử lý nhiễu thống kê** (multi-trajectory rollout, tách `u` và `d`), ở
**phân bổ ngân sách tối ưu** (elbow detection), và ở **kiểm soát tăng trưởng của tài liệu skill** (ba
ràng buộc tường minh). Ba cơ chế sau của P2 áp dụng được trực tiếp cho bài toán của P1 với chi phí
bổ sung không đáng kể.

Về độ tin cậy, các khẳng định trung tâm của cả hai — MASA cải thiện đáng kể ALFWorld và WebShop ở
backbone ≥8B; `Skill-MAS-optimized` vượt toàn bộ baseline trên cả bốn meta-agent — có biên độ đủ lớn
để không phụ thuộc vào việc thiếu báo cáo phương sai. Các khẳng định phụ có biên độ nhỏ thì không.
Hạn chế nghiêm trọng nhất đối với ứng dụng thực tế, chung cho cả hai, là **sự phụ thuộc vào tín hiệu
phần thưởng tự động**, và đây đồng thời là khe hở nghiên cứu có giá trị cao nhất trong §4.
