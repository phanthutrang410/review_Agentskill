# Tổng quan thư mục — Literature review: Agent Skill Evolution

## 1. Phạm vi

Thư mục tổng hợp kết quả khảo cứu hai công bố tiền ấn phẩm về **tiến hoá skill cho LLM agent**,
tương ứng hai tệp PDF đặt tại thư mục cha `D:\CTV AGENT`.

| Mã hiệu nội bộ | Định danh arXiv | Nhan đề | Phân loại | Ngày nộp | Dung lượng |
|---|---|---|---|---|---|
| **P1 (MASA)** | 2605.30723v1 | *Skill is Not One-Size-Fits-All: Model-Aware Skill Alignment for LLM Agents* | cs.CL | 29/05/2026 | 20 tr. |
| **P2 (Skill-MAS)** | 2606.18837v2 | *Skill-MAS: Evolving Meta-Skill for Automatic Multi-Agent Systems* | cs.MA | 24/06/2026 | 32 tr. |

Khảo cứu được thực hiện trên **toàn văn**, bao gồm phần phụ lục của cả hai công bố (P1: Phụ lục A–J;
P2: Phụ lục A–F, gồm toàn bộ prompt và các Meta-Skill đã tối ưu).

## 2. Cấu trúc tài liệu

| Tệp | Nội dung | Đối tượng sử dụng |
|---|---|---|
| [`index.html`](index.html) | **Slide seminar, 32 trang** — bản dựng sẵn, phục vụ tại gốc GitHub Pages. Mỗi slide có URL riêng dạng `#/N` | Trình bày trước nhóm |
| [`docs/01_review_masa_2605.30723.md`](docs/01_review_masa_2605.30723.md) | Phân tích chi tiết P1: động cơ, thí nghiệm sơ bộ, phương pháp, kết quả, phản biện | Người cần nắm đầy đủ P1 |
| [`docs/02_review_skill_mas_2606.18837.md`](docs/02_review_skill_mas_2606.18837.md) | Phân tích chi tiết P2: định vị trong literature, formulation, vòng lặp tiến hoá, kết quả, phản biện | Người cần nắm đầy đủ P2 |
| [`docs/03_co_so_khai_niem.md`](docs/03_co_so_khai_niem.md) | Diễn giải các khái niệm nền tảng (skill library, retrieval, meta-agent, hill climbing, UCB, elbow detection) ở mức không giả định kiến thức trước | Người tiếp cận lần đầu |
| [`docs/04_phan_tich_doi_chieu.md`](docs/04_phan_tich_doi_chieu.md) | Đối chiếu hai công bố theo trục phương pháp luận; đánh giá độ tin cậy; xác định khe hở nghiên cứu | Người định vị hướng nghiên cứu |
| [`docs/05_dac_ta_tai_hien_thuc_nghiem.md`](docs/05_dac_ta_tai_hien_thuc_nghiem.md) | Đặc tả yêu cầu tái hiện: tham số, dữ liệu, hạ tầng, ước lượng chi phí, rủi ro | Người triển khai |

Thứ tự đọc khuyến nghị cho người tiếp cận lần đầu: `03` → `01` → `02` → `04` → `05`.

## 3. Tóm tắt điều hành

Hai công bố cùng giải quyết một bài toán tổng quát: **nâng cao năng lực của LLM agent mà không cập
nhật tham số mô hình**. Cả hai chọn cùng một phương tiện — *skill*, hiểu là tài liệu thủ tục viết
bằng ngôn ngữ tự nhiên, được nạp vào context tại thời điểm ra quyết định — và cùng một cơ chế cải
tiến: vòng lặp *rollout → quy lỗi thất bại → viết lại skill → chấp nhận nếu chỉ số tăng*. Điểm phân
biệt nằm ở **đối tượng được tối ưu**.

**P1 (MASA)** tối ưu skill library cho một agent đơn. Luận điểm trung tâm: skill **không** trung lập
với mô hình. Bằng chứng cô lập biến trên ALFWorld cho thấy cùng một library làm Qwen3-14B tăng điểm
song làm Qwen3-8B suy giảm dưới cả điều kiện đối chứng không skill (32.1 → 17.1–27.9). Giải pháp gồm
hai cấu phần: (i) pipeline tiến hoá skill hai tầng, điều kiện hoá trên *model card* của backbone
đích, dùng hill climbing cho general skill và UCB tree search cho task-specific skill; (ii) một
*rewriter* Qwen3-4B chưng cất pipeline nói trên thành một lượt forward duy nhất. Mức cải thiện lớn
nhất ghi nhận trên ALFWorld là +25.8 điểm success rate (Qwen3-8B).

**P2 (Skill-MAS)** tối ưu năng lực điều phối của meta-agent trong hệ multi-agent. Luận điểm trung
tâm: hành vi orchestration cấp cao có thể mô hình hoá như **một skill duy nhất** (Meta-Skill) gồm ba
module — phân rã tác vụ, thiết kế agent, điều phối workflow — và do đó tiến hoá được mà không cần
cập nhật tham số, khắc phục thế lưỡng nan giữa *model capability* (inference-time MAS) và
*experience retention* (training-time MAS). Đóng góp phương pháp đặc thù: lấy mẫu K quỹ đạo cho mỗi
tác vụ nhằm tách nhiễu thực thi khỏi khiếm khuyết hệ thống, và chọn tập tác vụ phản tỉnh bằng
elbow detection trên điểm ưu tiên tổng hợp từ độ bất định và độ khó. Phương pháp vượt toàn bộ năm
baseline trên cả bốn meta-agent được khảo sát.

## 4. Các quan sát định lượng trọng yếu

1. **Đảo ngược quan hệ quy mô — mô hình (P1, Bảng 5).** Qwen3-32B dưới skill `Detailed` đạt 42.9
   success rate, thấp hơn Qwen3-14B dưới cùng skill 4.6 điểm (47.5). Quy mô tham số lớn hơn không
   bảo đảm khai thác skill tốt hơn.
2. **Cơ chế của hiện tượng suy giảm theo quy mô (P1, Phụ lục G.1).** Trên WebShop, tỉ lệ bước có
   chèn đoạn suy luận dài trước lệnh hành động là 0% (4B), 57% (8B), 97% (14B), 66% (32B). Ngân sách
   bước bị tiêu hao vào suy luận thay vì tương tác môi trường; hệ quả là Qwen3-14B đạt 2.8% success
   rate trong khi Qwen3-4B đạt 23.0%.
3. **Hiệu quả của chưng cất tìm kiếm (P1, §4.3).** Rewriter Qwen3-4B huấn luyện trên **769 mẫu SFT**
   vượt DS-Adapter vận hành bằng teacher DeepSeek-V4 trên toàn bộ bốn backbone, ở chi phí suy luận
   nhỏ hơn nhiều bậc.
4. **Giá trị của formulation so với giá trị của tiến hoá (P2, Bảng 1).** `Skill-MAS-init` — chưa qua
   bất kỳ vòng tiến hoá nào — đã đạt hoặc vượt baseline mạnh nhất trên hai trong bốn meta-agent, cho
   thấy phần đáng kể hiệu năng đến từ bản thân cấu trúc ba module, độc lập với vòng lặp tối ưu.
5. **Tính khả chuyển của Meta-Skill (P2, Bảng 2).** Chuyển Meta-Skill qua LLM khác trên cùng tác vụ
   giữ được phần lớn mức cải thiện (Δ đến +9.53, bằng kịch bản không chuyển); chuyển đồng thời qua
   cả LLM và tác vụ suy giảm rõ (Δ +1.19 đến +3.57).

## 5. Hạn chế chung của cả hai công bố

Cả hai phương pháp đều đòi hỏi **tín hiệu phần thưởng tự động** — success flag từ môi trường (P1)
hoặc ground-truth label để chấm quỹ đạo (P2). P2 nêu hạn chế này trực tiếp trong mục Limitations và
định lượng bằng ablation: các biến thể label-free suy giảm trên toàn bộ bốn cấu hình khảo sát. Cả
hai công bố đều **không báo cáo phương sai hay số seed**, trong khi quy trình tối ưu của cả hai đều
dựa trên rollout ngẫu nhiên và so sánh chấp nhận/bác bỏ theo điểm số. Phân tích chi tiết về độ tin
cậy được trình bày tại [`docs/04_phan_tich_doi_chieu.md`](docs/04_phan_tich_doi_chieu.md), §3.
