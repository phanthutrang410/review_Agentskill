# Cơ sở khái niệm

Tài liệu này định nghĩa các khái niệm nền tảng cần thiết để đọc hai công bố P1 (MASA) và P2
(Skill-MAS), và không giả định kiến thức trước về LLM agent. Mỗi khái niệm được trình bày theo trình
tự: định nghĩa, cơ chế, và vai trò trong hai công bố.

---

## 1. LLM agent và vòng lặp quan sát–hành động

**Định nghĩa.** Một *LLM agent* là hệ thống trong đó một mô hình ngôn ngữ không sinh ra một câu trả
lời duy nhất, mà tham gia một vòng lặp nhiều bước với môi trường:

```
quan sát o_1 → hành động a_1 → quan sát o_2 → hành động a_2 → … → kết thúc (thành công hoặc thất bại)
```

**Ví dụ (ALFWorld).** Môi trường là một căn phòng mô phỏng bằng văn bản. Yêu cầu: *"làm nóng một quả
táo rồi đặt lên bàn"*. Agent phải phát ra tuần tự các lệnh dạng `go to fridge 1`, `open fridge 1`,
`take apple 1`, `go to microwave 1`, `heat apple 1 with microwave 1`. Sai thứ tự — chẳng hạn phát
lệnh `heat` trước khi đặt vật vào lò — dẫn tới thất bại.

**Ràng buộc quan trọng.** Mỗi episode có **ngân sách bước** hữu hạn. Cạn ngân sách trước khi hoàn tất
mục tiêu được tính là thất bại. Ràng buộc này giải thích vì sao hành vi "suy nghĩ dài dòng trước mỗi
hành động" lại gây tổn hại nghiêm trọng (xem §7).

**Chỉ số đánh giá.** *Success rate* (SR) — tỉ lệ episode hoàn thành mục tiêu. *Steps* — số bước trung
bình để hoàn thành; giá trị thấp hơn thể hiện hiệu suất cao hơn.

---

## 2. Skill và skill library

**Định nghĩa.** Một *skill* là một đoạn hướng dẫn thủ tục viết bằng ngôn ngữ tự nhiên, mô tả *cách
làm* một việc chứ không phải *sự kiện gì đúng*. Ví dụ:

> *"Principle: Search every plausible surface or container exactly once before revisiting; prioritize
> unopened or unseen locations to cover the whole room methodically.*
> *When to apply: Anytime the goal object count is not yet met and unexplored locations remain."*

**Skill library** là tập hợp nhiều skill như vậy. Điểm cần nhấn mạnh: **skill không phải là code và
không phải là trọng số mô hình** — nó chỉ là văn bản. Do đó việc "cải thiện agent" trong cả hai công
bố quy về việc **biên tập lại một tài liệu văn bản**, không đụng tới tham số của mô hình.

**Phân tầng library (P1).** Library được chia hai tầng theo phạm vi áp dụng:

| Tầng | Ký hiệu | Phạm vi | Ví dụ |
|---|---|---|---|
| General skill | `S^G` | Xuyên mọi loại tác vụ | *"always verify your action parsed correctly"* |
| Task-specific skill | `S^{T_c}` | Riêng loại tác vụ `c` | *"check the fridge before the counter"* |

---

## 3. Retrieval — cơ chế đưa skill vào context

Không thể nạp toàn bộ library vào context ở mỗi bước, vì context window hữu hạn và văn bản thừa gây
nhiễu. Cơ chế thực tế:

1. Mỗi skill được chuyển thành một vector số (*embedding*) bằng một mô hình mã hoá. P1 dùng
   **Qwen3-Embedding-0.6B**.
2. Tại mỗi bước, quan sát hiện tại `o_t` cũng được chuyển thành vector.
3. Hệ chọn `k` skill có **cosine similarity** cao nhất với `o_t` và chỉ nạp chúng vào context.

Ký hiệu hình thức trong P1:

```
Ŝ_t = TopK(S, o_t, k)
a_t ~ F( · | τ_<t , Ŝ_t )
```

với `τ_<t` là lịch sử tương tác và `F` là backbone LLM đã đóng băng.

**Hệ quả cần lưu ý.** Vì mỗi bước chỉ nạp `k` skill, độ dài của mỗi skill nhân với `k` quyết định tải
token thực tế. Đây là lý do biến *granularity* (§4) có tác động vật chất, không chỉ tác động về văn phong.

---

## 4. Granularity và phát hiện trung tâm của P1

**Định nghĩa.** *Granularity* là độ chi tiết của cách diễn đạt một skill, khi **nội dung nguyên lý
giữ nguyên**. P1 khảo sát ba mức trên cùng một nguyên lý:

| Mức | Độ dài | Đặc trưng |
|---|---|---|
| `Concise` | ≈ 370 token | Một câu nguyên lý |
| `Moderate` | ≈ 1.092 token | Nguyên lý + điều kiện áp dụng |
| `Detailed` | ≈ 2.737 token | Các bước đánh số + cú pháp lệnh cụ thể + trace ví dụ + cảnh báo lỗi thường gặp |

**Phát hiện.** Mức tối ưu **khác nhau theo từng backbone**, và sai lệch gây tổn hại đo được:

| Backbone | Mức tối ưu | Ghi nhận đáng chú ý |
|---|---|---|
| Qwen3-4B | Moderate | Cả `Concise` lẫn `Detailed` đều kém hơn mức trung gian |
| Qwen3-8B | **không skill** | Cả ba mức skill đều làm hiệu năng thấp hơn điều kiện đối chứng |
| Qwen3-14B | Detailed | 47.5 SR |
| Qwen3-32B | Detailed | 42.9 SR — **thấp hơn 14B 4.6 điểm** |

Cơ chế được P1 đưa ra cho trường hợp Qwen3-8B: mô hình này tự thân đã đi theo chuỗi hành động ngắn
và hiệu quả; skill sai lệch áp đặt một pattern suy luận thủ tục ghi đè lên chuỗi tự nhiên đó, gây
thăm dò dư thừa. Nói cách khác, hiệu quả của skill phụ thuộc vào **mức tương thích giữa cách diễn đạt
và chiến lược giải quyết vấn đề mặc định của mô hình**, không chỉ vào nội dung.

---

## 5. Model card

**Định nghĩa (trong P1).** Một hồ sơ có cấu trúc về backbone đích, gồm ba nhóm trường:

1. **Architecture metadata** — family, số tham số, num_layers, hidden_size, số attention head, kích
   thước context window, vocab size.
2. **Training provenance** — base hay instruct-tuned, quy trình alignment (ví dụ `SFT + DPO + GRPO`),
   quy mô dữ liệu huấn luyện, hỗ trợ đa ngữ.
3. **Capability profile** — điểm mạnh trích từ release note chính thức; điểm yếu do teacher LLM tóm
   tắt tự động từ một tập nhỏ rollout thử nghiệm.

**Vai trò.** Đây là tín hiệu điều kiện hoá cho phép teacher biết *đang viết skill cho ai*. Ablation
tại P1 Bảng 3b cho thấy loại bỏ model card làm suy giảm hiệu năng trên mọi cấu hình, nặng nhất ở
backbone nhỏ (Qwen3-4B, cross-task: 32.5 → 8.8).

**Lý do không dùng quy mô tham số làm biến đại diện.** Gemma3-4B đạt tối ưu tại `Concise` trong khi
Qwen3-4B đạt tối ưu tại `Moderate`. Cùng ngân sách tham số, phản ứng khác biệt về chất — nên biến
quyết định là toàn bộ đặc tính mô hình, không phải số tham số.

---

## 6. Hai thuật toán tìm kiếm dùng trong P1

### 6.1 Hill climbing (Giai đoạn 1, cho general skill)

Thủ tục leo dốc đơn giản:

```
Lặp tối đa I = 10 vòng:
    chạy thử skill hiện tại trên toàn bộ task suite  → điểm R_i
    nếu R_i > điểm tốt nhất:  chấp nhận ứng viên
    ngược lại:                bác bỏ, giữ nguyên
    nếu 3 vòng liên tiếp không cải thiện: dừng sớm
```

Lý do chọn hill climbing: đánh giá **một** ứng viên general skill đòi hỏi chạy agent trên **toàn bộ**
task suite, nên tìm kiếm vét cạn bất khả thi về chi phí tính toán.

### 6.2 UCB tree search (Giai đoạn 2, cho task-specific skill)

Với task-specific skill, cùng một loại tác vụ có thể có nhiều chiến lược khác biệt về cấu trúc đều
hiệu quả. Do đó cần tìm kiếm dạng cây: mỗi node là một phiên bản skill, mỗi cạnh là một lần viết lại.

Việc chọn nhánh để mở rộng dùng công thức UCB1:

```
UCB1(n) = R̄(n) + C · sqrt( ln N_parent / N_n ),      C = 1.4
```

Hai số hạng thể hiện hai xu hướng đối lập:

| Số hạng | Ý nghĩa | Xu hướng |
|---|---|---|
| `R̄(n)` | phần thưởng trung bình của nhánh | **exploitation** — ưu tiên nhánh đã tốt |
| `C · sqrt( ln N_parent / N_n )` | tăng khi `N_n` (số lần đã thử nhánh) nhỏ | **exploration** — ưu tiên nhánh chưa khảo sát |

Đây là công thức chuẩn của bài toán multi-armed bandit, cũng là thành phần chọn nhánh trong MCTS.
Tham số: `J = 10` vòng cho mỗi loại tác vụ, `N = 100` episode để đánh giá mỗi node. Các cây của các
loại tác vụ khác nhau độc lập hoàn toàn nên song song hoá được.

---

## 7. Nothing-happens rate (NHR)

**Định nghĩa.** Tỉ lệ bước mà sau khi agent phát ra hành động, **trạng thái môi trường không thay
đổi**. Nguyên nhân điển hình: hành động không hợp lệ, hành động vô hiệu, hoặc agent lặp lại một thao
tác đã thực hiện.

**Vai trò trong hàm phần thưởng của P1:**

```
R(F, S, e) = SR(F, S, e) − λ · NHR(F, S, e),      λ ∈ [0,1]
```

**Lý do cần thiết.** Nếu chỉ tối ưu SR, thủ tục tìm kiếm không phân biệt được hai dạng thất bại khác
nhau về bản chất: (i) skill kém dẫn tới quyết định sai, và (ii) skill khiến agent lặp vòng vô ích cho
tới khi cạn ngân sách bước. `NHR` chuyển dạng thất bại thứ hai thành một tín hiệu quan sát trực tiếp
từ môi trường, không cần gradient và có chi phí tính toán không đáng kể.

**Hiện tượng liên quan được đo tại P1, Phụ lục G.1.** Trên WebShop, tỉ lệ bước có chèn đoạn suy luận
dài trước lệnh hành động là 0% (Qwen3-4B), 57% (8B), 97% (14B), 66% (32B). Vì ngân sách bước cố định,
mô hình lớn tiêu hao ngân sách vào cân nhắc thay vì tương tác, dẫn tới nghịch lý mô hình 14B chỉ đạt
2.8% success rate trong khi mô hình 4B đạt 23.0%.

---

## 8. Multi-agent system (MAS) và meta-agent

**Định nghĩa MAS.** Thay vì một LLM giải trọn tác vụ, hệ chia tác vụ cho nhiều agent chuyên biệt, mỗi
agent có vai trò và prompt riêng, kết nối theo một topology xác định.

Ví dụ (BrowseComp-Plus, P2 Bảng 7) — truy vấn *"xác định vận động viên thoả mãn năm gợi ý sau…"*:

```
                  ┌─ retrieve_team_player_pair ──┐
                  ├─ retrieve_tournament_stat ───┤
parse_and_plan ──►├─ retrieve_pro_league_title ──┤──► link_verification ──► merge_and_decide
                  ├─ retrieve_national_and_goals ┤
                  └─ retrieve_national_appearance┘
```

**Định nghĩa meta-agent.** LLM **thiết kế ra sơ đồ trên**: quyết định chia thành mấy nhánh, mỗi nhánh
là agent gì, nối dây ra sao. Meta-agent không giải tác vụ; nó sinh ra hệ thống giải tác vụ.

**Phân biệt hai cấp skill.**

| | Đối tượng được cải thiện | Công bố |
|---|---|---|
| Execution-level skill | agent thực thi tác vụ | phần lớn literature hiện có, và P1 |
| **Meta-Skill** | **meta-agent thiết kế hệ** | **P2** |

---

## 9. Thế lưỡng nan mà P2 giải quyết

| | Inference-time MAS | Training-time MAS |
|---|---|---|
| Meta-agent | frontier LLM đóng băng + thuật toán tìm kiếm | LLM nhỏ (~7B) đã fine-tune |
| Năng lực suy luận | cao | bị giới hạn bởi trần năng lực mô hình nhỏ |
| Lưu giữ kinh nghiệm | **không có** — lặp lại nguyên quy trình tìm kiếm ở mọi lần chạy | có, qua cập nhật gradient |
| Mở rộng lên mô hình >100B | có | **không** — chi phí fine-tune/RL không khả thi |

**Giải pháp của P2.** Giữ meta-agent đóng băng (được năng lực), nhưng đưa kinh nghiệm vào **một tài
liệu văn bản tiến hoá được** thay vì vào trọng số (được lưu giữ kinh nghiệm). Tài liệu đó là
Meta-Skill, gồm ba module: phân rã tác vụ (*the what*), thiết kế agent (*the who*), điều phối workflow
(*the how*).

---

## 10. Multi-trajectory rollout và hai thống kê phân bố

**Vấn đề.** Chạy một tác vụ một lần cho kết quả pass/fail. Kết quả này không phân biệt được "skill có
khiếm khuyết hệ thống" với "lần chạy này gặp bất lợi ngẫu nhiên".

**Giải pháp của P2.** Chạy mỗi tác vụ `K = 5` lần, thu `K` điểm `s_{i,1..K} ∈ [0,1]`, rồi tính:

```
uncertainty:  u_i = std( s_{i,1..K} )      → skill điều phối tác vụ này không nhất quán
difficulty:   d_i = − mean( s_{i,1..K} )   → tác vụ này khó một cách hệ thống
```

Hai thống kê đặc trưng hai dạng vấn đề khác nhau và gần trực giao về mặt thông tin. Ví dụ minh hoạ:

| Tác vụ | 5 điểm | `u` | `d` | Chẩn đoán |
|---|---|---|---|---|
| A | 0.9, 0.9, 0.8, 0.9, 0.9 | thấp | thấp | ổn định và dễ — không cần can thiệp |
| B | 0.1, 0.1, 0.0, 0.1, 0.1 | thấp | **cao** | khó một cách hệ thống — thiếu năng lực nền |
| C | 0.9, 0.1, 0.8, 0.0, 0.9 | **cao** | trung bình | skill mơ hồ tại một điểm quyết định |

Tác vụ dạng C là mục tiêu ưu tiên: dao động lớn báo hiệu skill đặc tả thiếu, và đây là dạng khiếm
khuyết sửa được bằng biên tập văn bản.

---

## 11. Elbow detection trên điểm ưu tiên

Hai thống kê được min–max chuẩn hoá rồi hợp nhất: `p_i = ½( ũ_i + d̃_i )`. Sau khi sắp giảm dần, cần
quyết định lấy bao nhiêu tác vụ để phản tỉnh.

Thay cho ngưỡng top-k cố định, P2 dùng **elbow detection**: xác định điểm mà đường cong ưu tiên "gãy".
Vì dãy là rời rạc, sai phân hữu hạn đóng vai trò đạo hàm:

```
δ_j = p_(j) − p_(j+1)                                    (sai phân bậc một)
j*  = argmax_{j ∈ {1,…,N−2}} | δ_j − δ_{j+1} | + 1       (sai phân bậc hai lớn nhất)
```

Minh hoạ số học. Giả sử dãy đã sắp: `0.95, 0.92, 0.90, 0.45, 0.43, 0.40`. Sai phân bậc một:
`0.03, 0.02, 0.45, 0.02, 0.03`. Sai phân bậc hai đạt cực đại tại vị trí chuyển từ `0.02` sang `0.45`
— tức đúng chỗ đường cong gãy. Kết quả: chọn ba tác vụ đầu.

**Ưu điểm.** Số tác vụ được chọn **thích ứng theo từng vòng**: vòng đầu khi khiếm khuyết còn nhiều
thì chọn rộng, các vòng sau khi hệ đã ổn định thì chọn hẹp, tập trung ngân sách tối ưu vào nhóm thực
sự có vấn đề.

---

## 12. Ba ràng buộc kiểm soát tăng trưởng của skill

Vấn đề chung của mọi thủ tục prompt/skill evolution: sau nhiều vòng, tài liệu phình ra thành một tập
quy tắc vụn vặt đặc thù cho từng trường hợp đã gặp, mất khả năng tổng quát hoá. P2 áp ba ràng buộc
tường minh trong prompt tối ưu (Hình 21):

| Ràng buộc | Nội dung | Mục đích |
|---|---|---|
| **Prune-before-add** | Rà soát và xoá/viết lại guidance không hiệu quả **trước** khi bổ sung quy tắc mới; xoá tối đa một element mỗi section mỗi vòng | Ngăn tích tụ quy tắc chết |
| **Giới hạn bổ sung** | Tối đa một *substantive conceptual upgrade* cho mỗi module trong mỗi vòng | Bảo đảm quy lỗi được: mỗi vòng chỉ đổi một biến |
| **Abstraction firewall** | Cấm tuyệt đối ví dụ đặc thù miền, chi tiết cú pháp lập trình, và heuristic cứng | Bảo toàn tính khả chuyển của Meta-Skill |

Hiệu lực của ràng buộc thứ ba được kiểm chứng gián tiếp tại P2 Bảng 2, Panel B: Meta-Skill tiến hoá
trên một tác vụ vẫn mang lại cải thiện Δ = +3.57 đến +7.15 khi áp cho tác vụ khác trên cùng LLM.

---

## 13. Tóm tắt đối chiếu hai công bố

| | P1 — MASA | P2 — Skill-MAS |
|---|---|---|
| Đối tượng tối ưu | skill library của **một agent** | Meta-Skill của **meta-agent** |
| Cái gì được đóng băng | backbone LLM | meta-agent LLM |
| Tín hiệu điều kiện hoá | **model card** của backbone | không có; chủ đích task-agnostic |
| Thuật toán tìm kiếm | hill climbing + UCB tree search | vòng lặp reflect–rewrite tuần tự, `R = 10` |
| Xử lý nhiễu ngẫu nhiên | `N = 100` episode mỗi lần đánh giá node | `K = 5` quỹ đạo mỗi tác vụ, tách `u` và `d` |
| Chọn mục tiêu can thiệp | toàn bộ task suite | elbow detection trên điểm ưu tiên |
| Cơ chế triển khai chi phí thấp | rewriter Qwen3-4B (một lượt forward) | `S*` tái sử dụng cho mọi truy vấn |
| Phụ thuộc reward | success flag từ môi trường | ground-truth label để chấm quỹ đạo |
