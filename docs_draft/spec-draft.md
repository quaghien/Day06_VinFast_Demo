# SPEC — AI Product Hackathon

**Nhóm:** 30-403-Day05

**Track:** VinFast

**Problem statement (1 câu):** *Khách hàng tìm hiểu mua xe VinFast gặp khó khăn khi tra cứu thông số, chính sách và thường phải chờ đợi lâu ngoài giờ hành chính mới được hỗ trợ; Web app tư vấn AI độc lập hoạt động 24/7, dùng Tavily real-time search kết hợp LLM reasoning (gpt-5.4) để trả lời chính xác, tự phân loại và chặn câu hỏi không phù hợp qua guardrails (gpt-5.4-mini), và điều hướng người dùng đặt lịch lái thử.*

---

## 1. AI Product Canvas

|   | Value | Trust | Feasibility |
|---|-------|-------|-------------|
| **Câu hỏi** | User nào? Pain gì? AI giải gì? | Khi AI sai thì sao? User sửa bằng cách nào? | Cost/latency bao nhiêu? Risk chính? |
| **Trả lời** | *Khách muốn mua xe. Pain: Chờ đợi CSKH lâu, tự đọc web khó. AI giải: Trả lời ngay tức khắc (giá, thông số, ưu đãi) & Lên lịch lái thử.* | *AI có thể báo sai giá. Giải pháp: AI kèm disclaimer và nếu không chắc sẽ fallback "Để lại SĐT cho CSKH tư vấn chuyên sâu".* | *Cost: ~$0.005/query. Latency <3s. Risk chính: Tư vấn sai lệch chính sách bán hàng hoặc so sánh toxic với xe đối thủ.* |

**Automation hay augmentation?** ☐ Automation · ☑ Augmentation
Justify: *Augmentation — AI đảm nhiệm khâu lọc "phễu đầu" (tư vấn cơ bản, xin thông tin, lên lịch), quyết định chốt sale và tư vấn tài chính chuyên sâu vững nhất vẫn là nhân sự (Sales/CSKH) tiếp nhận phần Context sau đó.*

**Learning signal:**

1. User correction đi vào đâu? *Feedback (rất hài lòng/không hài lòng) cho luồng chat và tỷ lệ Rớt (Drop-off) hội thoại đi vào Dashboard quản lý để cập nhật lại hệ RAG.*
2. Product thu signal gì để biết tốt lên hay tệ đi? *Tỷ lệ chuyển đổi Conversation -> Lead (đặt lịch thành công) tăng, tần suất xuất hiện keyword "Gặp tư vấn viên" giảm.*
3. Data thuộc loại nào? ☐ User-specific · ☑ Domain-specific · ☐ Real-time · ☐ Human-judgment · ☐ Khác: ___
   Có marginal value không? (Model đã biết cái này chưa?) *Có. Base model (ví dụ GPT-4o) không tự biết chính xác cấu hình mới nhất, giá lăn bánh tính toán cho từng tỉnh và các chính sách pin khuyến mãi thay đổi liên tục của VinFast.*

---

## 2. User Stories — 4 paths

### Feature: *Chatbot tư vấn xe & Đặt lịch lái thử*

**Trigger:** *User vào trang chủ VinFast, bấm vào widget Chat và hỏi "Tư vấn cho mình VF 8"*

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | User thấy gì? Flow kết thúc ra sao? | *AI chào hỏi, tóm tắt VF 8 kèm giá, hỏi user quan tâm bản Eco hay Plus. User chọn đặt lịch, AI thu thập SĐT & giờ, call Tool đặt lịch thành công.* |
| Low-confidence — AI không chắc | System báo “không chắc” bằng cách nào? User quyết thế nào? | *User hỏi “Sạc ở trạm bên thứ 3 có được bảo hành thay pin không?", LLM self-score confidence thấp, gọi `suggestConsultation(reason)`. AI giải thích rõ lý do: “Chính sách này phụ thuộc vào hợp đồng bảo hành riêng nên em chưa có tạm thời thông tin chính xác” → Hỏi: “Anh/chị có muốn em kết nối tư vấn viên không?” → Hiện nút [Gọi tư vấn viên]. User tự chọn bấm hoặc bỏ qua.* |
| Failure — AI sai | User biết AI sai bằng cách nào? Recover ra sao? | *Tavily trả về nguồn cũ, AI báo sai giá VF 3 là 500 triệu. User bắt lỗi: “Trên web báo 240 triệu mà?”. AI xin lỗi, truyền lại link nguồn đã dùng để user tự kiểm tra, flag phiên chat.* |
| Correction — user sửa | User sửa bằng cách nào? Data đó đi vào đâu? | *User bấm nút “Báo lỗi”, nhập câu trả lời đúng. Admin xem log session trên Dashboard, xác nhận và lưu vào Verified Answer Cache.* |

---

## 3. Eval metrics + threshold

**Optimize precision hay recall?** ☑ Precision · ☐ Recall
Tại sao? *Vì thông số kỹ thuật, bảo hành, và đặc biệt là giá bán xe mang tính pháp lý/thương hiệu rất cao. Thà AI báo "Xin phép chuyển CSKH" (Low Recall) còn hơn là "Bịa ra mức giá không có" (Low Precision) gây khủng hoảng truyền thông.*

| Metric | Threshold | Red flag (dừng khi) |
|--------|-----------|---------------------|
| *Answer Accuracy (Độ chính xác nội dung so với Fact)* | *≥ 95%* | *< 85% trong 1 tuần liên tục (Cần Freeze để fix DB)* |
| *Booking Conversion Rate (Số lượng chat tạo ra Lead)* | *≥ 10%* | *< 2% (Logic luồng gọi API Tool bị fail)* |
| *Hallucination / Toxic Rate* | *≤ 1%* | *> 5% (Nguy cơ khủng hoảng thương hiệu)* |

---

## 4. Top 3 failure modes

| # | Trigger | Hậu quả | Mitigation |
|---|---------|---------|------------|
| 1 | *User gài bẫy: (a) prompt injection / hỏi về hệ thống / API key / dữ liệu người dùng, (b) so sánh tiêu cực xe đối thủ, (c) câu hỏi ngoài chủ đề dịch vụ.* | *(a) Lộ system prompt hoặc thông tin nội bộ; (b) AI nói xấu đối thủ hoặc chê hãng; (c) Chatbot trả lời lan man mất focus.* | *Guardrail layer riêng dùng gpt-5.4-mini phân loại mỗi input thành: SENSITIVE (injection/system/data) → "Câu hỏi thuộc loại nhạy cảm, em không thể hỗ trợ"; COMPETITOR → lịch sự bẻ hướng VinFast; OFF_TOPIC → "Em chỉ hỗ trợ tư vấn xe VinFast"; PASS → xử lý bình thường. Tất cả phản hồi đều lịch sự.* |
| 2 | *Tavily API trả về kết quả cũ hoặc từ nguồn không chính thống (báo lá cải, forum) — LLM tổng hợp thành câu trả lời sai thông số/giá.* | *Tư vấn sai giá, sai thông số cho khách hàng; LLM confidence score tự chấm vẫn cao dù nguồn sai.* | *Prompt LLM bắt buộc trích dẫn URL nguồn kèm câu trả lời. Giới hạn truy vấn Tavily xoay quanh VinFast. Nếu không có kết quả từ domain chính thức → tự động set confidence thấp → gọi function suggestDirectConsultation().* |
| 3 | *Hệ thống API gọi CRM (Save lead) bị timeout hoặc down server.* | *Khách nhập số điện thoại nhưng AI báo lỗi thất bại, làm khách cụt hứng.* | *AI fail-safe, trả lời gracefully: "Hệ thống đang quá tải tí xíu, nhưng em đã ghi nhớ thông tin của anh/chị rồi. Sẽ có CSKH gọi lại xác nhận nhé." Fallback đẩy log offline lưu vào CSV đợi retry.* |

---

## 5. ROI 3 kịch bản

|   | Conservative | Realistic | Optimistic |
|---|-------------|-----------|------------|
| **Assumption** | *500 user/ngày, 5% đặt lịch* | *2000 user/ngày, 8% đặt lịch* | *5000 user/ngày, 12% đặt lịch* |
| **Cost** | *$3/ngày (Inference LLM)* | *$10/ngày* | *$25/ngày* |
| **Benefit** | *Tiết kiệm 4h trực chat (~$12), thêm 25 Lead* | *Giảm 16h CSKH, thêm 160 Lead* | *Bao trọn 80% L1 Support, thêm 600 Lead* |
| **Net** | *Dương nhẹ + Lead* | *Dương rất lớn* | *Đóng góp trực tiếp làm thay đổi Data Sale của hãng* |

**Kill criteria:** *Có hiện tượng chụp màn hình bóc phốt lan truyền MXH vì AI tư vấn sai luật (Critical Brand Harm). Hoặc sau 2 tháng Cost duy trì AI > Benefit lượng Lead mang về (Ví dụ: khách vào trêu đùa quá đông nhưng không ai đặt lịch).*

---

## 6. Mini AI spec (1 trang)

**1. Sản phẩm & Giải pháp:**
VinFast Auto-Agent là Web app tư vấn xe AI độc lập (giả lập — không phải website chính thức của VinFast). Sử dụng kiến trúc 2 lớp LLM: **Guardrail layer** (gpt-5.4-mini) phân loại và lọc input trước, **Reasoning layer** (gpt-5.4) xử lý câu hỏi hợp lệ với 2 tools: (1) **search(query)** — Tavily API tìm kiếm thông tin real-time trích xuất sẵn, (2) **suggestConsultation(reason)** — LLM giải thích lý do cụ thể và hỏi user có muốn gặp tư vấn viên không, hiện nút [Gọi tư vấn viên] để user tự quyết định (không tự động gọi).

**2. Khách hàng/User:** 
Khách hàng tiềm năng đang trong giai đoạn tìm hiểu (Consideration) muốn tra cứu thông số, so sánh giá cả siêu tốc và lên lịch lái thử không bị làm phiền; Khách hàng hiện tại tìm kiếm trợ giúp kỹ thuật xe.

**3. Bản chất sản phẩm (Auto/Aug):**
Augmentation (Tiền tuyến - L1 Support). Tự động chặn và giải quyết nhanh 80% câu hỏi FAQ (cấu hình, giá cọc, so sánh bản Plus/Eco...). Ngay khi có mùi "Khó", "Tài chính phức tạp" hoặc "Khiếu nại", lập tức Freeze và đề nghị chuyển ngữ cảnh chat cho tư vấn viên để chốt sale/chăm sóc. Đảm bảo user có Human touch ở chặng cuối.

**4. Metrics & Quality Trade-offs:**
Hệ thống ưu tiên tuyệt đối vào **Precision > Recall**. Chấp nhận việc AI có lúc thoái thác trả lời để nhường cho Human, tuyệt đối tránh hiện tượng bịa giá (đảo lộn bảng giá). 

**5. Cơ chế Data Flywheel:**
Càng nhiều khách hàng hỏi → Lộ ra Edge cases LLM không trả lời được (confidence thấp, Tavily không trả về nguồn chính thống) → Các cặp Q&A Low-confidence được tự động ghi vào Dashboard → Admin review, chỉnh sửa và xác nhận câu trả lời đúng ("verified") → Câu đó được lưu vào **Verified Answer Cache** (JSON/localStorage) → Lần sau cùng câu hỏi tương tự, hệ thống serve thẳng từ Cache mà không cần gọi Search (giảm latency + tránh kết quả sai nguồn) → Coverage Cache rộng dần, tỷ lệ Low-confidence giảm, Cost/query giảm theo. Toàn bộ log có thể export ra `.jsonl` để fine-tune LLM ở giai đoạn sau.
