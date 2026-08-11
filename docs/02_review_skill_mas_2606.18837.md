# P2 — Skill-MAS: Evolving Meta-Skill for Automatic Multi-Agent Systems

**Định danh.** arXiv 2606.18837v2 · cs.MA · nộp 24/06/2026 · 32 trang (8 trang chính, 24 trang phụ lục).
**Tác giả.** Hehai Lin (HKUST Quảng Châu), Qi Yang (Ant Group), Chengwei Qin (HKUST Quảng Châu, liên hệ).
**Tài nguyên.** Project page, Gallery & Live Demo, mã nguồn (liên kết tại trang đầu bản PDF).
**Tài trợ.** Ant Group, thông qua CCF-Ant Research Fund (CCF-AFSG RF20250502).

---

## 1. Bối cảnh và định vị

### 1.1 Automatic-MAS

*Automatic-MAS* chỉ hướng nghiên cứu tự động sinh và tối ưu kiến trúc multi-agent system, thay cho
việc thiết kế thủ công từng vai trò agent, topology giao tiếp và workflow. Thiết kế thủ công tốn công
và không mở rộng được cho các tác vụ có mức dị biệt cao.

### 1.2 Thế lưỡng nan giữa năng lực mô hình và lưu giữ kinh nghiệm

P2 phân loại literature hiện có thành hai nhánh, mỗi nhánh mang một đánh đổi:

| | Nhánh 1: Inference-time MAS | Nhánh 2: Training-time MAS |
|---|---|---|
| Cơ chế | Meta-agent là frontier LLM **đóng băng**, ghép với thuật toán tìm kiếm lặp | Fine-tune một LLM nhỏ (thường ~7B) làm orchestrator trên dataset orchestration đã tuyển chọn |
| Đại diện | AFlow (MCTS trên không gian workflow), EvoAgent (toán tử tiến hoá), AOrchestra (điều phối động), MAS-Zero (self-reflective feedback loop) | MAS² (bộ ba agent tự sinh, tự hiệu chỉnh), MAS-Orchestra (function-calling RL, GRPO), ScoreFlow (DPO) |
| Ưu điểm | Năng lực suy luận ở mức tiên tiến nhất | Nội hoá kinh nghiệm qua cập nhật gradient |
| Nhược điểm | **Experience-agnostic**: meta-agent không có cơ chế bộ nhớ tích luỹ, lặp lại nguyên quy trình tìm kiếm ở mọi lần chạy, không chuyển giao được kinh nghiệm chẩn đoán từ thất bại trước | **Trần năng lực thấp** của mô hình nhỏ; đòi hỏi khối dữ liệu chất lượng cao lớn; **không mở rộng được lên frontier model >100B** do chi phí fine-tune/RL |

Câu hỏi nghiên cứu: có tồn tại con đường thứ ba **tách rời việc lưu giữ kinh nghiệm khỏi việc cập
nhật tham số**, cho phép một frontier meta-agent đóng băng tiến bộ dần về chuyên môn orchestration?

### 1.3 Khe hở trong paradigm Skill

Paradigm *Skill* — trừu tượng hoá năng lực vận hành thành tài liệu ngôn ngữ tự nhiên có cấu trúc,
sau đó tiến hoá tài liệu này — đang được phát triển tích cực: MemSkill (kỹ năng quản lý bộ nhớ),
Trace2Skill (trích thủ tục theo tác vụ từ lịch sử tương tác), Skill0, D2Skill, SKILLRL (tích hợp
khám phá skill với tối ưu policy), CoEvoSkills và EvoSkill (tinh chỉnh vai trò của agent thực thi),
cùng các nghiên cứu tổ chức skill của sub-agent qua hierarchical routing.

Điểm chung của toàn bộ nhóm trên: phân tích skill **bó chặt ở cấp sub-agent**, tức tập trung vào việc
các agent thực thi tác vụ bên trong hệ được trừu tượng hoá thành skill ra sao. Quan sát của P2:
**hành vi orchestration cấp cao của chính Meta-agent cũng mô hình hoá được như một evolvable skill.**
Đây là đóng góp ý tưởng của công bố.

---

## 2. Hình thức hoá Meta-Skill

Meta-Skill được biểu diễn dưới dạng một tệp `SKILL.md`, gồm YAML frontmatter và ba module — cấu trúc
mà P2 gọi là *tripartite formulation*:

```yaml
---
name: unified_meta_agent_skill
description: "A foundational meta-agent skill for generating Multi-Agent Systems (MAS).
  It systematically drives the process from conceptual task decomposition to agent
  engineering, and finally to workflow orchestration."
tags: [meta-agent, task-decomposition, agent-engineering, workflow-orchestration]
inputs: [user_query]
---
```

| Module | Đối tượng | Nội dung quy định |
|---|---|---|
| **1. Task Decomposition** | *the "what"* | Phân tích ý định và phạm vi; phân rã thành sub-task rời rạc, gắn kết logic; ánh xạ quan hệ phụ thuộc (prerequisite / parallel / iterative — **thứ tự logic, không phải dataflow hệ thống**); tiêu chí thành công đánh giá được cho từng sub-task |
| **2. Agent Engineering** | *the "who"* | Role profiling (mỗi sub-agent một định danh và vai trò chuyên biệt); instruction design (system prompt, mục tiêu, biên hành vi, kỳ vọng đầu ra); input context framing |
| **3. Workflow Orchestration** | *the "how"* | Architectural topology (sequential pipeline / router-based / hierarchical / blackboard); dataflow và state management (ánh xạ I/O, quản lý global memory); sinh mã orchestration khả thi hành |

Cấu trúc ba module phục vụ **mục đích kép**. Tại thời điểm suy luận, nó cung cấp cho meta-agent một
thủ tục có nguyên tắc để sinh MAS. Tại thời điểm tối ưu, nó cho phép phase chẩn đoán **quy lỗi về
đúng một module xác định**, nhờ đó cập nhật skill mang tính cục bộ và diễn giải được, thay vì viết
lại toàn bộ tài liệu.

Meta-Skill khởi tạo `S^(1)` được sinh bằng cách dùng LLM tóm tắt tài liệu **Anthropic (2026),
"Building multi-agent systems: when and how to use them"**; toàn văn `S^(1)` trình bày tại Hình 5 của
phụ lục.

---

## 3. Phương pháp: vòng lặp tối ưu đóng

Ký hiệu: `T = {t_1, …, t_N}` là validation set, `S^(r)` là Meta-Skill hiệu lực tại vòng
`r ∈ {1, …, R}`. Mỗi vòng gồm hai giai đoạn:

```
S^(r) ──► Giai đoạn 1: Multi-Trajectory Rollout ──► D^(r), {u_i}, {d_i}
                                                        │
                                                        ▼
          Giai đoạn 2: Selective Reflection
          ├─ 2a. Priority-Driven Task Selection      (elbow detection)
          ├─ 2b. Hierarchical Trajectory Reflection  (within-task → cross-task) ──► E
          └─ 2c. Skill Optimization                  (prune trước, bổ sung sau) ──► S^(r+1)
```

Sau `R` vòng, skill đạt validation performance cao nhất được chọn làm `S*` và đánh giá trên test set
tách biệt.

### 3.1 Giai đoạn 1 — Multi-Trajectory Rollout

Với mỗi tác vụ `t_i ∈ T`, lấy mẫu **`K` quỹ đạo độc lập** dưới Meta-Skill hiện tại. Mỗi rollout đi
trọn pipeline ba module và sinh một kết quả cuối kèm execution trace đầy đủ, ghi nhận dưới dạng quỹ
đạo chuẩn hoá:

```
τ_{i,k} = ( id_i , k , s_{i,k} , S^(r) , Φ_{i,k} )
```

với `s_{i,k} ∈ [0,1]` là điểm chuẩn hoá và `Φ_{i,k}` lưu kiến trúc MAS cùng các kết quả trung gian.
Corpus của vòng `r` là `D^(r) = {τ_{i,k}}_{i=1..N, k=1..K}`.

Từ `D^(r)`, hai thống kê per-task được rút ra để đặc trưng hoá phân bố hành vi:

```
uncertainty:   u_i = sqrt( (1/K) Σ_k ( s_{i,k} − s̄_i )² ),      s̄_i = (1/K) Σ_k s_{i,k}
difficulty:    d_i = − s̄_i
```

`u_i` lượng hoá mức thiếu nhất quán khi cùng một skill điều phối cùng một tác vụ qua nhiều lần chạy;
giá trị lớn báo hiệu skill mơ hồ hoặc đặc tả thiếu tại một điểm nào đó. `d_i` là điểm trung bình lấy
dấu âm, sao cho tác vụ khó nhận giá trị lớn hơn (các giá trị âm thô được ánh xạ về [0,1] bằng min–max
normalization ở §3.2).

Ý nghĩa phương pháp: một quỹ đạo đơn chỉ cho kết quả pass/fail, không phân biệt được khiếm khuyết hệ
thống của skill với dao động ngẫu nhiên của quá trình thực thi. Lấy mẫu `K` quỹ đạo chuyển mỗi tác vụ
từ một kết quả nhị phân thành một **đặc trưng phân bố**, nhờ đó phase phản tỉnh sau đó phân biệt được
*systematic skill deficiency* với *transient execution noise*.

### 3.2 Giai đoạn 2a — Priority-Driven Task Selection

Phản tỉnh đồng đều trên toàn bộ `N` tác vụ sẽ pha loãng tín hiệu chẩn đoán. Hai thống kê được min–max
chuẩn hoá trong phạm vi vòng hiện tại:

```
ṽ_i = ( v_i − min_j v_j ) / ( max_j v_j − min_j v_j ),      v ∈ {u, d}
```

rồi hợp nhất thành một điểm ưu tiên duy nhất:

```
p_i = ½ ( ũ_i + d̃_i )
```

ưu tiên đồng thời các tác vụ **dao động mạnh** và **khó một cách hệ thống**. Sắp giảm dần thu được
dãy `p_(1) ≥ … ≥ p_(N)`.

Vì dãy ưu tiên là rời rạc, sai phân hữu hạn được dùng làm bản rời rạc của đạo hàm nhằm phát hiện điểm
gãy tự nhiên của đường cong. Sai phân bậc một `δ_j = p_(j) − p_(j+1)` tương ứng đạo hàm bậc một; chỉ
số elbow xác định tại vị trí sai phân bậc hai lớn nhất về trị tuyệt đối:

```
j* = argmax_{j ∈ {1, …, N−2}} | δ_j − δ_{j+1} | + 1
```

Tập phản tỉnh là `T_sel = { t_(1), …, t_(j*) }`. Cơ chế này khiến **số tác vụ được chọn thích ứng
theo từng vòng**, thay cho một ngưỡng top-k cố định.

### 3.3 Giai đoạn 2b — Hierarchical Trajectory Reflection

**Pha 1 — phân tích tương phản trong tác vụ (within-task contrastive analysis).** Với mỗi
`t_i ∈ T_sel`, `K` quỹ đạo được phân hoạch thành tập điểm cao
`H_i = { τ_{i,k} : s_{i,k} ≥ median({s_{i,k}}_k) }` và tập điểm thấp `L_i`. Một LLM reflector đối
chiếu execution snapshot `Φ_{i,k}` giữa hai nhóm để chẩn đoán:

- **divergence points** — bước, phase hoặc quyết định cụ thể tại đó hai nhóm bắt đầu tách nhau, kèm
  phân loại nguyên nhân (exploration randomness, reasoning error, constraint violation, khác);
- **success factors** — hành vi và chiến lược dẫn tới thành công ở nhóm điểm cao;
- **failure modes** và **root causes** — wrong reasoning, premature termination, constraint
  violation, inefficient search, capability gap; kèm phân loại recoverable hay fundamental;
- **volatility root cause** và **difficulty root cause** tách biệt.

Đầu ra là báo cáo `R_i` kèm **candidate patch `δ̂_i`** — một sửa đổi cụ thể, khả thi hành, nhắm vào
đúng module skill bị liên đới (gồm target phase, constraint/rule, implementation mechanism và
expected impact).

Prompt vận hành (Hình 19 của phụ lục) áp đặt các ràng buộc cụ thể về chất lượng chẩn đoán, ví dụ yêu
cầu thay các phát biểu mơ hồ dạng *"trajectory fails due to poor reasoning"* bằng phát biểu định vị
được dạng *"trajectory fails at step 5 because it incorrectly assumes X when the constraint requires
Y"*, và bắt buộc đầu ra là JSON nghiêm ngặt không kèm markdown fence.

**Pha 2 — tổng hợp xuyên tác vụ (cross-task synthesis).** Nhằm tránh overfitting, tập báo cáo `{R_i}`
được phân tích chung để rút các pattern vượt lên trên từng tác vụ đơn lẻ: các **systematic weakness**
(xếp hạng theo severity × frequency) và các **systematic strength** — chiến lược orchestration bền
vững cần được bảo toàn. Kết quả là **structured evidence package `E`**, một prioritized repair list
xếp hạng các patch `{δ̂_i}` theo tích của expected impact và implementation feasibility.

### 3.4 Giai đoạn 2c — Skill Optimization

`E` cùng `S^(r)` dẫn hướng optimizer sinh `S^(r+1)`. Trình tự thao tác được quy định chặt:

1. **Cắt tỉa trước khi bổ sung.** Đối chiếu skill hiện tại với `E` để nhận diện các guidance
   **không hiệu quả hoặc phản tác dụng**, xoá hoặc viết lại chúng trước khi đưa vào quy tắc mới.
   Ràng buộc định lượng trong prompt: xoá **tối đa một element cho mỗi stage section trong mỗi vòng**,
   kèm chỉ dẫn *"When in doubt, keep it. Only delete when the Step2 evidence directly contradicts the
   guidance."*
2. **Bổ sung có giới hạn.** Tối đa **một substantive conceptual upgrade cho mỗi module trong mỗi
   vòng**; không dồn nhiều thay đổi không liên quan vào một lượt.
3. **Abstraction firewall.** Mỗi sửa đổi phải được trừu tượng hoá thành một **nguyên tắc orchestration
   tổng quát hoá được**, không phải bản vá cho một tác vụ. Prompt tối ưu (Hình 21) áp ba điều cấm
   tuyệt đối: không ví dụ đặc thù miền (*"DO NOT mention delivery, bookings, math formulas"*), không
   chi tiết cú pháp lập trình (*"DO NOT mention Python, AST parsing, variable names, snake_case,
   await, or state dictionaries"*), không heuristic cứng (*"DO NOT say 'if text contains and,
   split it'"*).
4. **Structural validity check** trước khi chấp nhận; bắt buộc bảo toàn scaffold ba module và các
   khoá YAML.

Ba ràng buộc *prune-before-add*, *≤1 upgrade mỗi module mỗi vòng* và *abstraction firewall* cấu thành
cơ chế kiểm soát tăng trưởng của tài liệu skill. Thiếu chúng, skill có xu hướng tích tụ thành tập
quy tắc đặc thù miền — dạng thất bại điển hình của các thủ tục prompt evolution.

### 3.5 Tham số (P2, Bảng 5)

| Tham số | Giá trị |
|---|---|
| `R` — số vòng tiến hoá | 10 |
| `K` — số rollout mỗi tác vụ | 5 |
| temperature (mọi lời gọi LLM) | 1.0 |
| max_tokens | 32.768 |

---

## 4. Thiết lập thực nghiệm

### 4.1 Benchmark và phân chia dữ liệu (P2, Bảng 4)

| Benchmark | Validation | Test | Đặc điểm |
|---|---|---|---|
| **DeepResearchBench** (DRB) | 16 | 84 | 100 tác vụ nghiên cứu trình độ tiến sĩ trên 22 lĩnh vực; đánh giá bốn chiều: comprehensiveness, insight, instruction-following, readability |
| **Humanity's Last Exam-Math** (HLE) | 32 | 168 | Toán học trình độ chuyên gia; lấy mẫu 200 câu từ subset MATH của bộ 2.500 câu |
| **BrowseComp-Plus** (BCP) | 32 | 168 | Truy vấn multi-hop đòi hỏi tìm kiếm lặp; corpus cố định với supporting document đã kiểm chứng bởi người và negative sample khó; lấy mẫu 200 câu |
| **VitaBench** (VITA) | 16 | 84 | Tác vụ tương tác đời thường (giao đồ ăn, mua tại chỗ, dịch vụ du lịch trực tuyến); môi trường mô phỏng gồm **66 tool**; chấm theo rubric |
| Multi-task Learning | 48 | — | Gộp một nửa validation set của mỗi dataset |

Mọi chỉ số chuẩn hoá về [0, 100%]. Báo cáo gồm hiệu năng trung bình và **chi phí suy luận trung bình
(USD)** trên test set. Chi phí tiến hoá bị loại khỏi Bảng 1 và trình bày riêng tại Bảng 6 (xem §4.5).

**Meta-agent khảo sát.** Gemini-3.1-Flash và GPT-5.4-Nano (cả hai đặt reasoning effort ở mức "low"),
Qwen3.5-Plus và DeepSeek-V4-Flash (dùng bản chuẩn, không bổ sung reasoning overhead). **Cùng một LLM
được dùng cho mọi cấu phần bên trong một automatic-MAS**, bảo đảm so sánh công bằng.
**LLM-as-a-judge mặc định: Gemini-3.1-Flash.**

**Baseline.** Inference-time: EvoAgent, AOrchestra, AFlow (giới hạn 10 vòng tìm kiếm, đánh giá
validation ba lần mỗi vòng). Training-time: MAS², MAS-Orchestra. Các baseline còn lại tuân thủ
nguyên thiết lập gốc.

### 4.2 Kết quả chính (P2, Bảng 1)

`Skill-MAS-init` dùng Meta-Skill khởi tạo `S^(1)`; `Skill-MAS-optimized` dùng `S*`.

**Gemini-3.1-Flash**

| Phương pháp | DRB | HLE | BCP | VITA | Avg.Perf ↑ | Avg.Cost ↓ |
|---|---|---|---|---|---|---|
| EvoAgent | 40.76 | 8.93 | 16.07 | 10.71 | 19.12 | 8.20 |
| AOrchestra | 39.49 | 7.14 | 14.88 | 14.29 | 18.95 | 2.31 |
| AFlow | 42.31 | 5.95 | 17.86 | 19.05 | 21.29 | **0.44** |
| MAS² | 38.03 | 2.33 | 11.39 | 8.33 | 15.02 | 0.91 |
| MAS-Orchestra | 40.40 | 17.26 | 8.90 | 9.52 | 19.02 | 2.48 |
| Skill-MAS-init | 36.10 | 14.88 | 19.05 | 16.67 | 21.68 | 2.63 |
| **Skill-MAS-optimized** | **44.71** | **21.43** | **23.21** | **28.60** | **29.49** | 2.82 |

**GPT-5.4-Nano**

| Phương pháp | DRB | HLE | BCP | VITA | Avg.Perf | Avg.Cost |
|---|---|---|---|---|---|---|
| EvoAgent | **52.91** | 14.29 | 23.81 | 8.33 | 24.83 | 6.26 |
| AOrchestra | 44.68 | 8.93 | 22.62 | 4.76 | 20.25 | 2.45 |
| AFlow | 44.29 | 11.31 | 18.45 | 2.38 | 19.11 | **0.69** |
| MAS² | 43.20 | 10.42 | 10.71 | 2.38 | 16.68 | 1.27 |
| MAS-Orchestra | 43.83 | 14.88 | 7.74 | 9.52 | 18.99 | 1.84 |
| Skill-MAS-init | 40.45 | 12.50 | 19.64 | 5.95 | 19.64 | 4.24 |
| **Skill-MAS-optimized** | 48.90 | **18.45** | **27.38** | **15.48** | **27.55** | 5.22 |

**Qwen3.5-Plus**

| Phương pháp | DRB | HLE | BCP | VITA | Avg.Perf | Avg.Cost |
|---|---|---|---|---|---|---|
| EvoAgent | 47.02 | 27.38 | 15.48 | 34.52 | 31.10 | 7.91 |
| AOrchestra | 44.05 | 12.50 | 19.05 | 40.48 | 29.02 | 1.91 |
| AFlow | 46.17 | 25.00 | 12.50 | 45.24 | 32.23 | **1.45** |
| MAS² | 40.87 | 16.56 | 11.52 | 22.62 | 22.89 | 4.04 |
| MAS-Orchestra | 40.43 | 21.43 | 10.12 | 35.71 | 26.92 | 1.35 |
| Skill-MAS-init | 48.88 | 24.40 | 20.24 | 36.90 | 32.61 | 1.83 |
| **Skill-MAS-optimized** | **51.28** | **29.76** | **23.81** | **48.80** | **38.41** | 2.43 |

**DeepSeek-V4-Flash**

| Phương pháp | DRB | HLE | BCP | VITA | Avg.Perf | Avg.Cost |
|---|---|---|---|---|---|---|
| EvoAgent | 49.10 | 16.67 | 16.67 | 61.90 | 34.30 | 3.33 |
| AOrchestra | 41.03 | 19.64 | 15.48 | 58.33 | 34.36 | 2.69 |
| AFlow | 49.93 | 21.43 | 15.48 | 55.95 | 35.70 | 1.45 |
| MAS² | 35.31 | 6.67 | 4.92 | 33.33 | 20.06 | **0.63** |
| MAS-Orchestra | 40.63 | 20.24 | 15.72 | 47.62 | 31.05 | 1.98 |
| Skill-MAS-init | 44.99 | 20.83 | 15.48 | 53.57 | 33.72 | 1.14 |
| **Skill-MAS-optimized** | **51.69** | **26.79** | **22.62** | **63.10** | **41.05** | 2.12 |

**Ba nhận xét từ bảng kết quả.**

1. `Skill-MAS-optimized` đạt Avg.Perf cao nhất trên **cả bốn** meta-agent. Trong toàn bộ 16 ô
   (4 benchmark × 4 meta-agent), **ngoại lệ duy nhất** là DRB với GPT-5.4-Nano, nơi EvoAgent giữ lợi
   thế (52.91 so với 48.90).
2. `Skill-MAS-init` — chưa qua bất kỳ vòng tiến hoá nào — đã cạnh tranh được: trên Gemini-3.1-Flash
   nó vượt baseline mạnh nhất AFlow (21.68 so với 21.29), trên Qwen3.5-Plus nó dẫn đầu toàn bộ
   baseline (32.61 so với 32.23). Điều này quy một phần đáng kể hiệu năng cho **bản thân tripartite
   formulation**, độc lập với vòng lặp tối ưu.
3. Về đánh đổi chi phí–hiệu năng (Hình 1d), ba nhóm phương pháp phân bố theo pattern khác biệt.
   Training-time MAS có chi phí thấp nhất nhưng hiệu năng kém nhất; nguyên nhân được P2 quy cho việc
   orchestrator nhỏ sinh MAS đơn giản tại thời điểm test, thiếu khả năng tổng quát hoá theo độ khó
   biến thiên giữa các miền. Inference-time MAS đạt hiệu năng cao hơn nhưng chi phí suy luận cao nhất
   do phải tái tối ưu trên mỗi mẫu. Skill-MAS sinh MAS **một lần** thay vì tối ưu lặp cho từng mẫu,
   nhờ đó đạt hiệu năng cao nhất ở mức chi phí trung bình.

### 4.3 Khả chuyển của Meta-Skill (P2, Bảng 2 và Hình 3-trái)

Thiết kế: Meta-Skill được tiến hoá tại một cặp (LLM, tác vụ) — *Skill Source* — rồi đánh giá tại một
cặp khác — *Test Setting*. `Δ` đo so với `Skill-MAS-init`.

| Kịch bản | Δ ghi nhận | Diễn giải |
|---|---|---|
| **No Transfer** (Source ≡ Test) | +7.74, +7.14, +9.53, +9.53 | Mức cải thiện lớn nhất, chiếm ưu thế trên đường chéo heatmap |
| **Panel A — cross-LLM, cùng tác vụ** | +2.97, +4.76, **+9.53**, +8.34 | Khả chuyển cao; do tác vụ nền không đổi, việc phân tích và tinh chỉnh quỹ đạo cho ra Meta-Skill tương tự bất kể LLM cụ thể |
| **Panel B — cross-task, cùng LLM** | +7.15, +3.57, +5.95, +5.35 | Vẫn cạnh tranh; **kiểm chứng trực tiếp cho abstraction firewall**: do prompt buộc meta-agent tránh thủ thuật đặc thù miền và tập trung vào pattern tổng quát, Meta-Skill tối ưu học được chiến lược task-agnostic và giữ hiệu lực trên dataset chưa gặp |
| **Panel C — cross-LLM và cross-task** | +2.38, +1.19, +3.57, +2.98 | Yếu nhất, phản ánh độ khó của việc chuyển giao đồng thời qua hai phân bố |

Trường hợp mạnh nhất ở Panel A: Meta-Skill tiến hoá trên (GPT-5.4-Nano, VitaBench) áp cho
(DeepSeek-V4-Flash, VitaBench) đạt 63.10 với Δ = +9.53, bằng đúng mức của kịch bản không chuyển giao.

### 4.4 Nghiên cứu loại bỏ

**(a) Số rollout `K`** (Hình 3-phải, `K ∈ {3, 5, 7}`). Tương quan dương rõ rệt: hiệu năng tăng đơn
điệu theo số quỹ đạo lấy mẫu. Tuy nhiên có hiện tượng **lợi ích giảm dần** — mức tăng từ 3 lên 5 lớn
hơn mức tăng từ 5 lên 7 — trong khi chi phí tiến hoá tăng tuyến tính theo `K`. Giá trị mặc định
`K = 5` là kết quả của đánh đổi này.

**(b) Selective Reflection** (P2, Bảng 3). Mặc định, Skill-MAS dùng ground-truth label để chấm quỹ đạo
và xếp ưu tiên. Hai biến thể **label-free** được khảo sát: `Full-Validation` (chọn toàn bộ mẫu) và
`Half-Validation` (chọn ngẫu nhiên 50%).

| Biến thể | GPT/BCP | GPT/VITA | DS/BCP | DS/VITA |
|---|---|---|---|---|
| **Ours** | **27.38** | 15.48 | **22.62** | 63.10 |
| Full-Validation | 22.02 | 13.10 | 19.64 | 59.52 |
| Half-Validation | 21.43 | 9.52 | 17.26 | 58.33 |
| Multi-task Learning | 20.83 | **16.67** | 22.02 | **64.29** |

Cả hai biến thể label-free đều suy giảm so với cấu hình mặc định nhưng vẫn vượt phần lớn baseline.
Kết quả có hai hàm ý: (i) adaptive priority selection đóng góp thực chất; (ii) Skill-MAS **suboptimal
trong thiết lập không có nhãn** — chính tác giả nêu đây là hướng phát triển quan trọng, với đề xuất
thay ground-truth bằng self-confidence score của meta-agent.

**(c) Multi-task Learning.** Tiến hoá Meta-Skill trên pool gộp cả bốn dataset rồi đánh giá trên từng
test set tương ứng. Kết quả không thuần nhất: cải thiện nhẹ trên VitaBench (64.29 so với 63.10) nhưng
suy giảm trên BrowseComp-Plus (22.02 so với 22.62). Giải thích của tác giả: Skill-MAS không được tối
ưu tường minh cho multi-task; thiếu cơ chế cô lập mẫu theo miền, hệ khó rút pattern cải thiện chung
mà lại hấp thụ nhiễu đặc thù miền.

### 4.5 Chi phí tiến hoá (P2, Bảng 6)

Chi phí trung bình (USD) để tiến hoá Meta-Skill trên validation set, tính bình quân qua bốn benchmark:

| Gemini-3.1-Flash | GPT-5.4-Nano | Qwen3.5-Plus | DeepSeek-V4-Flash |
|---|---|---|---|
| 9.35 | 31.36 | **59.06** | 24.54 |

Đây là chi phí **một lần**: sau khi tiến hoá, `S*` được tái sử dụng cho mọi truy vấn đầu vào, và chi
phí suy luận mới là nút thắt trong triển khai thực tế. Lập luận này biện minh cho việc loại chi phí
tiến hoá khỏi Bảng 1, song con số 59.06 USD cho **một** cặp (LLM, benchmark) là đại lượng cần tính
đến khi lập ngân sách.

### 4.6 Quỹ đạo tiến hoá của Meta-Skill (P2, Hình 4 và §5.3)

Trace trên cặp (DeepSeek-V4-Flash, BrowseComp-Plus), năm vòng, vòng 5 cho kết quả tốt nhất:

| Vòng | Nâng cấp được đưa vào | Module |
|---|---|---|
| 1 | Evidence Weighting | 1 — Decomposition |
| 2 | Parallel Fan-out cho tác vụ đa ràng buộc | 1 |
| 3 | Backtracking and Dynamic Replanning · bổ sung Link-verification task | 3 · 1 |
| 4 | Weighted-satisfaction protocol | 2 — Agent Engineering |
| **5** | **Merge-Node Re-execution Authority** | 3 — Orchestration |

Quỹ đạo có mạch logic nhất quán mà chính công bố chỉ ra: từ **cách đóng khung evidence**, qua **cách
phán xử evidence**, tới **cách khắc phục khi thất bại**. Module 1 dựng epistemic scaffolding —
constraint prioritization và fan-out biến phân rã từ phân hoạch phẳng thành relational search plan có
chỉ dấu cấu trúc. Trên nền đó, Module 2 thay các binary check giòn bằng đánh giá đã hiệu chỉnh, cụ
thể là weighted constraint satisfaction có partial-evidence fallback. Module 3 nâng năng lực điều
phối: cross-entity bridging và merge-node re-execution chuyển giai đoạn tích hợp từ **passive
aggregation** sang **active evidence recovery**.

Pattern chung xuyên bốn dataset là sự bổ sung các **reliability-oriented control**: constraint-aware
reasoning, structured output contract, verification gate, backtracking. Các cơ chế này nhắm vào các
dạng thất bại phổ biến của MAS — premature commitment, error propagation giữa các stage, và handoff
không ổn định. Mặc dù mỗi dataset nhấn mạnh khía cạnh khác nhau (hiệu chỉnh diễn giải với toán học,
đánh trọng số evidence với truy hồi), chúng hội tụ về cùng một lợi thế: một policy orchestration bền
vững hơn, kiểm toán được và khả chuyển hơn.

### 4.7 Đối chiếu cấu trúc MAS sinh ra (P2, Bảng 7 và 8)

**BrowseComp-Plus** — truy vấn chứa năm gợi ý độc lập cần thoả mãn đồng thời:

| Phương pháp | Cấu trúc sinh ra | Khiếm khuyết |
|---|---|---|
| EvoAgent | Ba expert retrieve song song | Cùng truy vấn cho cả ba, không xét decomposition và ràng buộc toàn cục |
| AOrchestra | MainAgent + delegate/sub-agent có thể lặp tìm kiếm | Tương đương multi-turn search thông thường, không đưa thêm yếu tố agentic |
| AFlow | Search ×3 + self-consistency ensemble | Verification ở mức answer-level, không ở mức entity-link |
| MAS² | — | **Thất bại: không sinh được executable MAS code** |
| MAS-Orchestra | Debate đa nhánh (6 nhánh) + reflexion | Chỉ thiết kế một lượt search, dẫn tới thiếu thông tin |
| Skill-MAS-init | Lọc gợi ý tuyến tính | Chỉ giữ một evidence path; mismatch sớm khó khôi phục |
| **Skill-MAS-optimized** | `parse_and_plan` phân thành **năm retriever theo từng clue**, sau đó `link_verification` **cưỡng chế nhất quán xuyên clue**, cuối cùng `merge_and_decide` | — |

**VitaBench** — truy vấn đời thường chứa ba chuỗi giao dịch độc lập (đặt đồ ăn, mua trà tại book bar,
mua vé tàu):

- `Skill-MAS-init` sinh một luồng nén bốn bước `order_meal → find_book_cafe → purchase_voucher →
  book_train_ticket`, **trộn lẫn retrieval, screening và transaction trong một chuỗi tuyến tính**,
  do đó nhạy với error propagation, text contamination và vi phạm ràng buộc.
- `Skill-MAS-optimized` dùng `constraint_extraction` để dispatch **ba nhánh explore → evaluate → order
  tường minh** (meal / bookbar / train), và `final_summary` chỉ tổng hợp **sau khi mọi nhánh đã hoàn
  tất quyết định**.

Đây là biểu hiện cụ thể nhất của nội dung "Meta-Skill được tiến hoá": chuyển từ **linear pipeline**
sang **branched workflow có verification gate**.

---

## 5. Hạn chế

### 5.1 Hạn chế do tác giả nêu

Skill-MAS thể hiện hiệu năng bền vững trong điều kiện có giám sát với ground-truth label, song hiệu
quả **có thể suy giảm trong thiết lập weakly-supervised hoặc unsupervised** khi thiếu phản hồi chất
lượng cao. Hướng khắc phục đề xuất: tích hợp cơ chế đánh giá tự giám sát, chẳng hạn LLM-as-a-judge,
để dẫn hướng phase selective reflection mà không phụ thuộc nhãn ngoài. Ablation tại Bảng 3 là bằng
chứng định lượng cho chính hạn chế này.

### 5.2 Nhận xét phản biện bổ sung

1. **Xung đột vai trò của judge.** Gemini-3.1-Flash vừa là LLM-as-a-judge mặc định, vừa là một trong
   bốn meta-agent được đánh giá. Trên DRB và VitaBench — hai benchmark chấm bằng LLM hoặc rubric —
   đây là rủi ro self-preference. Không có kiểm tra chéo bằng judge độc lập.
2. **Không báo cáo phương sai hay số seed.** Điều này mâu thuẫn với chính luận điểm phương pháp của
   công bố, rằng phải lấy mẫu `K` quỹ đạo mới tách được nhiễu khỏi tín hiệu; các số tại Bảng 1 lại là
   giá trị điểm không kèm error bar. Với test set 84 câu (DRB, VITA), một câu tương ứng ≈ 1.19 điểm;
   chênh lệch giữa `Skill-MAS-init` và baseline mạnh nhất trên Gemini là 0.39 điểm — **nhỏ hơn giá trị
   của một câu**. Kết luận về `Skill-MAS-optimized` (chênh +8.2 điểm) không bị ảnh hưởng, nhưng khẳng
   định "`init` đã cạnh tranh" thì thiếu cơ sở thống kê.
3. **Quy mô validation set nhỏ.** DRB và VITA chỉ có 16 tác vụ validation. Với `K = 5`, mỗi vòng sinh
   80 quỹ đạo, và elbow selection chọn ra một số ít tác vụ. Việc chọn `S*` theo validation score trên
   16 tác vụ tiềm ẩn rủi ro selection noise.
4. **Ranh giới label-free chưa rạch ròi.** Selective Reflection cần label để chấm quỹ đạo, nhưng bản
   thân việc chấm điểm trên DRB và VITA cũng do một LLM judge thực hiện. Thiết kế ablation thiếu một
   biến thể thứ ba cần thiết: giữ nguyên adaptive priority selection nhưng thay ground-truth score
   bằng judge score.
5. **Khả năng diễn giải kết quả của MAS².** Ghi nhận tại Bảng 7 rằng MAS² *thất bại trong việc sinh
   executable MAS code* khiến điểm số của baseline này tại Bảng 1 khó diễn giải: một phần hiệu năng
   thấp có thể do lỗi kỹ thuật của baseline chứ không do bản chất paradigm training-time. Cần thận
   trọng khi trích dẫn công bố này để lập luận rằng training-time MAS kém.
6. **Phạm vi của build contract.** `MAS_BUILD_CONTRACT` (Hình 17–18) chứa các ràng buộc kỹ thuật rất
   cụ thể: *"prefer triple-single-quoted raw strings"*, *"`state['final_output']` must be exactly one
   assignment"*, *"any loop must be capped at maximum 3 iterations"*, *"do NOT emit `from __future__`
   import annotations"*. Các ràng buộc này nằm **bên ngoài** Meta-Skill nên không vi phạm abstraction
   firewall về mặt hình thức, song chúng hàm ý một phần đáng kể độ ổn định của hệ đến từ khối
   engineering constraint thủ công này — và khối đó không được ablate.

---

## 6. Các yếu tố phương pháp có giá trị tái sử dụng

1. **Multi-trajectory rollout để tách nhiễu khỏi khiếm khuyết hệ thống.** Hai thống kê
   `u_i = std(scores)` và `d_i = −mean(score)` có chi phí thấp và gần trực giao về mặt thông tin.
   Cơ chế này áp dụng được cho bất kỳ pipeline tự cải thiện nào, không giới hạn ở MAS.
2. **Elbow detection trên điểm ưu tiên** thay cho ngưỡng top-k cố định: chỉ đòi hỏi sai phân hữu hạn
   bậc hai, nhưng khiến kích thước tập phản tỉnh thích ứng theo trạng thái từng vòng.
3. **Ba ràng buộc kiểm soát tăng trưởng cho skill/prompt evolution**: prune-before-add, giới hạn một
   nâng cấp mỗi module mỗi vòng, và abstraction firewall cấm ví dụ đặc thù miền, chi tiết cú pháp
   lập trình và heuristic cứng.
