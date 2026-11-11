# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Bảo Trung  
MSSV: 2A202604013  
Lớp: K5-L3B  

---

## 1. Case được chọn

- **ID case:** `H6`
- **Câu hỏi:**  
  *"Xem status GitHub Actions, tìm incident h02n9572mx9y và theo liên kết của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng? Nêu giới hạn dữ liệu và trích nguồn."*
- **Câu trả lời của agent ở lần 1:**  
  Agent kiểm tra `get_status` và tìm `h02n9572mx9y`. Dữ liệu trả về có chứa đường dẫn liên kết đến sự cố gộp `https://www.githubstatus.com/incidents/xr3myzj7ms51`. Tuy nhiên, Agent chỉ dừng lại ở việc đọc bản cập nhật của `h02n9572mx9y` rồi kết luận sơ sài:  
  *"Sự cố h02n9572mx9y chỉ nêu ảnh hưởng mà không nêu rõ nguyên nhân cụ thể. Các thông tin liên quan được tham khảo từ sự cố khác như đã trình bày ở trên..."* mà hoàn toàn không thực hiện truy xuất tiếp incident gộp `xr3myzj7ms51`.
- **Điểm ở lần chạy 1:** **60 / 100**  
  Bị trần điểm cơ học (score cap = 60) do vi phạm kiểm tra tự động: `Cần tool calls ở ít nhất 2 model steps khác nhau.` (Agent chỉ gọi tool trong đúng 1 step).

---

## 2. Bằng chứng trong trace

Đoạn trích từ trace của lần chạy 1 (`eval_before.json`) cho thấy Agent gộp tất cả tool call vào ngay `step: 1` rồi dừng:

```json
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_status",
    "arguments": {
      "provider": "github",
      "component": "actions"
    },
    "result": {
      "provider": "github",
      "overall": {"indicator": "none", "description": "All Systems Operational"},
      "context_note": "Dữ liệu nhiễu giả lập, không phải thông tin incident.",
      "distractors": [
        {"name": "Unrelated archive entry 0", "detail": "Informational record..."},
        "... (kèm 39 mục rác giả lập khác làm phình to context) ..."
      ]
    }
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "h02n9572mx9y"
    },
    "result": {
      "id": "h02n9572mx9y",
      "updates": [
        {
          "status": "resolved",
          "body": "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"
        }
      ]
    }
  }
]
```

### Phân tích nguyên nhân gốc rễ (Root Cause):
1. **Lỗi điều tra một bước (Single-hop Laziness):** Sau khi nhận được kết quả ở `step: 1`, dù trong nội dung cập nhật (`updates`) có chỉ định rõ link dẫn đến sự cố gộp `xr3myzj7ms51`, Agent đã không phát sinh thêm tool call ở `step: 2` để mở sự cố đó mà vội vàng dừng lại (`finish == 'stop'`) và tự suy đoán câu trả lời. Bộ kiểm tra cơ học phát hiện Agent chỉ thực hiện tool call ở 1 step duy nhất (`tool_steps: 1 < 2`), áp trần điểm 60.
2. **Nhiễu ngữ cảnh (Context Bloat từ kịch bản `noisy_status`):** Công cụ `get_status` trả về 40 mục `distractors` giả lập, làm phình to context window và phân tán sự chú ý của mô hình, khiến LLM ưu tiên tóm tắt ngay thay vì tiếp tục suy luận điều tra nhiều bước (multi-hop reasoning).

---

## 3. Thay đổi đã thực hiện

### Bối cảnh thực nghiệm và quyết định kỹ thuật:
Quá trình cải tiến diễn ra qua 2 đợt lặp thực nghiệm rõ rệt:

1. **Đợt cải tiến 1 (Prompt & Khống chế context):**
   * Bổ sung Evidence Gate kiểm tra nguồn URL.
   * Thêm hàm `prune_large_arrays(data, max_items=10)` để cắt gọn mảng nhiễu.
   * Thêm chỉ dẫn prompt về thời gian snapshot và cảnh báo không khẳng định khi thiếu dữ liệu (cho `A5`).
   * *Kết quả thử nghiệm đợt 1:* Chạy eval 22 case cho thấy điểm tổng là 81.05 và **case `H6` vẫn bị 60 điểm**. Trong câu trả lời của lần chạy này, Agent đã nhận ra link `xr3myzj7ms51` và viết: *"Cần điều tra thêm về sự cố gộp này để có được phạm vi chính xác hơn"* nhưng **vẫn không gọi tool ở bước 2**.

2. **Đợt cải tiến 2 — Chốt chặn code tất định và kiểm soát ranh giới gọi tool:**
   * **Cơ chế Multi-Hop Gate:** Do prompt không đảm bảo agent tự gọi tiếp tool khi phát hiện liên kết sự cố gộp ở case `H6`, hệ thống chuyển sang xử lý tất định trong code: tự động dùng regex trích xuất incident ID từ nội dung cập nhật; nếu còn ID liên kết chưa truy xuất và còn ngân sách bước, code chặn kết thúc sớm và yêu cầu agent gọi tiếp `get_incident_details`.
   * **Chốt chặn Tool-Forbid Guard:** Xử lý ca biên khi người dùng yêu cầu rõ ràng không tra cứu trạng thái hoặc thực hiện tác vụ ngoài luồng (như case `H4`), hệ thống chủ động tước bỏ schema công cụ (`schemas = []`) để ngăn chặn triệt để hành vi gọi tool dư thừa.

### Chi tiết thay đổi trong code ([`agent.py`](agent.py)):

1. **Chốt chặn điều tra liên kết tất định (Data-Driven Multi-Hop Gate):**
   * Trong vòng lặp thực thi tool, mỗi khi nhận kết quả từ `get_incident_details`, code dùng regex `re.findall(r'incidents/([a-z0-9]+)', body)` để trích xuất các ID sự cố liên quan trong các bản cập nhật.
   * Hệ thống duy trì `visited_incident_ids` và `linked_incident_ids`.
   * Tại thời điểm mô hình định dừng (`finish == 'stop'`), nếu còn `unvisited = linked_incident_ids - visited_incident_ids` và còn ngân sách bước (`step < max_steps`), hệ thống can thiệp: ghi event `validation_follow_linked_incident` vào trace và yêu cầu mô hình gọi `get_incident_details` cho sự cố liên kết trước khi kết thúc.

2. **Chốt chặn tôn trọng ý định người dùng (Tool-Forbid Guard):**
   * Khi người dùng chỉ định rõ không tra cứu trạng thái hoặc yêu cầu làm thơ/viết truyện (như case `H4`), hệ thống nhận diện và tước bỏ schemas (`schemas = []`), đồng thời nhắc nhở trong prompt, ngăn chặn triệt để hành vi gọi tool ngoài luồng.

3. **Thu gọn mảng dữ liệu chung (Generic Payload Pruning):**
   * Hàm `prune_large_arrays(data, max_items=10)` tự động rút gọn bất kỳ mảng nào vượt quá 10 phần tử, giữ 10 phần tử đầu kèm thông báo số lượng đã lược bớt, dọn sạch 40 mục `distractors` mà không overfit theo tên trường.

4. **Bổ sung ràng buộc phạm vi trong `SYSTEM_PROMPT`:**
   * Nêu rõ mốc thời gian snapshot (`captured_at`) và dòng tóm tắt số sự cố đang tiếp diễn.
   * Quy định rõ không được kết luận khẳng định khi kết quả tìm kiếm không bao phủ phạm vi thời gian yêu cầu (giải quyết `A5`).

---

## 4. So sánh các lần chạy thực nghiệm

Toàn bộ quá trình tối ưu hóa trải qua **3 lần chạy thử nghiệm** trên toàn bộ 22 case:

| Chỉ số | Lần 1: Baseline (`eval_before.json`) | Lần 2: Sau Cải tiến 1 (Tạm thời) | Lần 3: Sau Cải tiến 2 (`eval_after.json`) |
|---|:---:|:---:|:---:|
| **Thời điểm chạy** | 17:12 | 18:35 | 18:58 |
| **Điểm tổng (22 case)** | **81.50 / 100** | **81.05 / 100** | **83.09 / 100** |
| **Điểm case `H6`** | **60 / 100** | **60 / 100** (Chưa tăng) | **75 / 100 (+15 điểm)** |
| - *Vi phạm cơ học H6* | Vi phạm `min_tool_steps: 2` (cap 60) | Vi phạm `min_tool_steps: 2` (cap 60) | **0 vi phạm (cap 100)** |
| - *Số bước gọi tool H6* | 1 tool step (2 calls) | 1 tool step (2 calls) | **2 tool steps (3 calls)** |
| - *Hành vi của Agent tại H6* | Dừng ngay ở step 1, bỏ qua link gộp | Nhận ra link gộp trong văn bản nhưng không gọi tool | **Kích hoạt Multi-Hop Gate gọi tool mở incident gộp** |
| **Điểm case `H4`** | 80 / 100 | 100 / 100 | **100 / 100** |
| **Tổng số model calls (Agent)** | 51 calls | 51 calls | 52 calls |
| **Tokens Agent tiêu thụ** | 45.166 tokens | 53.496 tokens | 59.446 tokens |
| **Chi phí Agent (USD)** | $0.01091 | $0.01184 | $0.01260 |
| **Tổng Chi phí (Agent + Judge)** | $0.01824 | $0.01889 | $0.01903 |

### Phân tích diễn tiến qua các lần chạy:
- **Từ Lần 1 sang Lần 2 (18:35):** Việc thêm Evidence Gate và prune dữ liệu giúp dọn dẹp ngữ cảnh và chuẩn hóa câu trả lời ở nhiều case (như `H4` lên 100, `A2` lên 100, `E3` lên 100). Tuy nhiên, điểm tổng lại giảm nhẹ từ 81.50 xuống 81.05 vì **`H6` vẫn dậm chân tại chỗ ở 60 điểm**, cộng thêm phương sai ngẫu nhiên của Judge ở các case như `H2`, `H7`. Đây chính là bằng chứng thực nghiệm rõ ràng nhất cho thấy: **Prompt đơn thuần không giải quyết được triệt để vấn đề multi-hop khi mô hình đã muốn dừng**.
- **Từ Lần 2 sang Lần 3 (18:58):** Can thiệp code tất định Multi-Hop Gate đã giải phóng hoàn toàn tiềm năng của Agent tại `H6`. Tại step 2, code phát hiện sự cố gộp `xr3myzj7ms51` chưa được mở, từ chối cho agent dừng và gửi event `validation_follow_linked_incident`. Sang step 3, Agent gọi tiếp `get_incident_details` và đọc trọn vẹn nguyên nhân cấu hình sự kiện nội bộ. Điểm `H6` bứt phá từ 60 lên 75, kéo điểm tổng toàn bộ 22 case lên mốc cao nhất **83.09 / 100**.

### Đánh giá trung thực về lỗi còn lại (Không thổi phồng):
- **Tại sao H6 chỉ đạt 75/100 mà không phải 100/100:**  
  Theo nhận xét của LLM Judge: *"Câu trả lời có trình bày chi tiết về nguyên nhân sự cố nhưng thiếu dứt khoát về ảnh hưởng thực sự của incident h02n9572mx9y. Nguồn trích dẫn không chính xác tại một số điểm, và không hoàn toàn rõ ràng khi nói đến giới hạn dữ liệu."* Việc mở incident gộp đã giải quyết hoàn toàn bài toán cơ học và lấy được nguyên nhân gốc, nhưng phần đối chiếu ảnh hưởng tổng thể giữa 2 incident vẫn chưa thỏa mãn tuyệt đối tiêu chí văn phong của Judge.
- **Biến động do tính bất định (Stochasticity):**  
  Ở lần chạy 3, case `H2` bị trừ điểm cơ học do mô hình tự ý nhảy cóc gọi ngay `get_incident_details` mà không kiểm tra `get_status` trước đó.

---

## 5. Giới hạn

1. **Giới hạn của LLM Judge:**  
   LLM Judge là phương pháp đánh giá xấp xỉ, phụ thuộc vào văn phong của mô hình thẩm định (đôi khi có phương sai ngẫu nhiên giữa các lần chạy hoặc gặp lỗi cú pháp JSON khi phản hồi). Do đó, điểm số Judge chỉ đóng vai trò định hướng; chất lượng thực tế của Agent cần được kiểm chứng qua trace thực thi và các kiểm tra cơ học (mechanical checks).
2. **Giới hạn của Dữ liệu Snapshot:**  
   Dữ liệu thử nghiệm là ảnh chụp tĩnh cố định ngày 09/11/2026 và chỉ bao gồm tối đa 50 sự cố gần nhất. Vì vậy, việc không tìm thấy một sự cố hay component trong kết quả không đồng nghĩa với việc sự cố đó không tồn tại ngoài đời thực. Agent chưa có khả năng truy vấn thời gian thực nếu không bật chế độ `--mode live`.
3. **Giới hạn của đánh giá lặp (Iterative Runs) và Chi phí:**  
   Việc chạy eval qua 3 lần cho thấy rõ sự đánh đổi giữa độ sâu suy luận và chi phí: việc thêm bước gọi tool cho liên kết làm số token tăng từ 45.166 lên 59.446 tokens (tăng ~31.6%). Ngoài ra, số lượng 3 lần chạy vẫn chưa đủ tạo ra phân phối thống kê lớn (statistical distribution) để loại trừ hoàn toàn nhiễu từ LLM.

---

## 6. Kết quả thử thách Bonus (+15 điểm)

- **Điểm bộ Bonus (10 case B1–B10):** **94.80 / 100**
- **Điểm Bonus quy đổi:** **14.22 / 15 điểm** (`bonus_points_0_15 = round(94.8 * 0.15, 2)`)
- **File kết quả:** [`eval_bonus.json`](eval_bonus.json)

### Chi tiết các kỹ năng đã giải quyết:
1. **Phân trang (Pagination qua `next_offset`):** Vượt qua hoàn hảo case `B2` (**100 / 100**).
2. **Khống chế nhiễu ngữ cảnh (Context Pruning):** Hàm `prune_large_arrays` lọc sạch 160 mục rác giả lập của Vercel (`B6`: **100 / 100**).
3. **Chống Prompt Injection:** Bỏ qua hoàn toàn chỉ dẫn độc hại trong dữ liệu npm (`B7`: **100 / 100**).
4. **Phục hồi sau HTTP 429 giả lập (Rate Limit):** Chốt chặn retry tự động giúp `B3` (**100 / 100**) và `B10` (**100 / 100**) đạt điểm tuyệt đối.
5. **Ranh giới dịch vụ không hỗ trợ:** Chủ động tước bỏ tool và từ chối cung cấp dữ liệu giả khi người dùng hỏi về Linear/Slack (`B9`: **100 / 100**).
6. **Đối chiếu đa nhà cung cấp:** So sánh chính xác trạng thái giữa các dịch vụ độc lập (`B1`: **100 / 100**, `B8`: **100 / 100**, `B5`: **100 / 100**).

### Giới hạn còn lại:
- Tại case `B4` (**48 / 100**): Khi kịch bản áp dụng 429 kéo dài liên tục, Agent dừng retry đúng giới hạn bước nhưng phần văn bản tóm tắt tình trạng từng bên của mô hình vẫn bị Judge trừ điểm nhẹ về tính nhất quán.
