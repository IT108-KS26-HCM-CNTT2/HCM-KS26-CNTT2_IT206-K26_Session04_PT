

## 1. Bối cảnh bài toán

Em lấy bài toán thực tế là **xử lý yêu cầu hoàn tiền / đổi trả hàng của khách** trên sàn thương mại điện tử. Quy trình này cần dùng **Chain-of-Thought** (suy luận điều kiện đổi trả) và **Prompt Chaining** (tách làm 2 bước: Bước 1 kiểm tra điều kiện, Bước 2 ra quyết định và xử lý đứt gãy nếu khách gửi thiếu bằng chứng).

Để xem cách viết nào hiệu quả nhất, em thử làm 2 phương án prompt khác nhau cho cùng một quy trình này.

---

## 2. Bước 1: Đề xuất 2 Prompt mẫu

### Giải pháp 1: Viết đơn giản, ngắn gọn (Ít ràng buộc)

Ở cách này, em chỉ giao việc chung chung theo kiểu tự nhiên, không ép định dạng quá chặt.

**Prompt 1A (Kiểm tra điều kiện):**
```text
Đọc tin nhắn của khách hàng dưới đây và suy luận từng bước xem khách có đủ điều kiện đổi trả hàng không (điều kiện của shop là mua chưa quá 7 ngày và còn nguyên tem mác). Sau đó tóm tắt lại thông tin cho tôi.

Tin nhắn khách: "{{TIN_NHAN_KHACH}}"
```

**Prompt 1B (Phản hồi khách):**
```text
Dựa vào kết quả kiểm tra ở trên, hãy viết một tin nhắn trả lời khách hàng. Nếu đủ điều kiện thì đồng ý đổi trả, nếu thiếu thông tin thì hỏi thêm khách, nếu quá hạn thì từ chối khéo.

Kết quả bước 1: "{{KET_QUA_BUOC_1}}"
```

---

### Giải pháp 2: Viết chi tiết, có nhiều ràng buộc và chặn lỗi (Chain-of-Thought + JSON)

Ở cách này, em chia rõ từng bước suy luận, bắt buộc trả về JSON để nối chuỗi không bị đứt gãy, và thêm điều kiện chặn các trường hợp khách nói mập mờ.

**Prompt 2A (Phân tích logic & Bắt lỗi dữ liệu):**
```text
Bạn là nhân viên CSKH. Hãy đọc tin nhắn của khách và suy luận từng bước (Chain-of-Thought) để kiểm tra điều kiện đổi trả (Quy định: Mua <= 7 ngày VÀ còn nguyên tem mác).

Các bước suy luận bắt buộc:
- Bước 1: Tìm số ngày khách đã mua hàng. Nếu khách không nói rõ ngày thì ghi là "KHONG_RO".
- Bước 2: Tìm tình trạng tem mác. Nếu khách không nhắc tới thì ghi là "KHONG_RO".
- Bước 3: Chốt trạng thái:
  + "HOP_LE": Nếu <= 7 ngày VÀ còn tem.
  + "TU_CHOI": Nếu > 7 ngày HOẶC đã giật tem.
  + "THIEU_THONG_TIN": Nếu có ít nhất 1 mục bị "KHONG_RO" (xử lý đứt gãy).

Chỉ trả về đúng định dạng JSON sau, không viết thêm lời dẫn:
{
  "suy_luan": "giải thích ngắn gọn 3 bước trên",
  "so_ngay": "...",
  "tinh_trang_tem": "...",
  "trang_thai": "HOP_LE | TU_CHOI | THIEU_THONG_TIN",
  "can_hoi_them": "ghi rõ thông tin bị thiếu nếu có, không thì để trống"
}

Tin nhắn khách: "{{TIN_NHAN_KHACH}}"
```

**Prompt 2B (Xử lý chuỗi tiếp theo dựa trên JSON):**
```text
Nhận dữ liệu JSON từ bước trước và soạn câu trả lời cho khách theo đúng điều kiện:
- Nếu "trang_thai" == "HOP_LE": Báo khách gửi hàng về kho để đổi mới.
- Nếu "trang_thai" == "TU_CHOI": Xin lỗi và từ chối lịch sự, nêu rõ lý do dựa vào "suy_luan".
- Nếu "trang_thai" == "THIEU_THONG_TIN": Hỏi bổ sung đúng phần bị thiếu ở mục "can_hoi_them", giọng điệu nhẹ nhàng.

Dữ liệu JSON:
{{KET_QUA_JSON_BUOC_2A}}
```

---

## 3. Bước 2: Lập bảng so sánh Trade-off

Sau khi thử nghiệm cả 2 cách, em lập bảng so sánh sự đánh đổi (Trade-off) giữa chúng:

| Tiêu chí | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết) |
| :--- | :--- | :--- |
| **Độ dễ hiểu** | **Cao** (Viết tự nhiên như nói chuyện, ai đọc cũng hiểu ngay) | **Trung bình** (Dài dòng, có cấu trúc JSON và nhiều quy tắc) |
| **Tốn token** | **Ít** (Prompt ngắn nên tốn ít chi phí và chạy nhanh hơn) | **Nhiều hơn** (Tốn khoảng gấp 3 lần số token đầu vào) |
| **Chặn lỗi biên** | **Trung bình** (Khi khách nói thiếu thông tin, AI hay tự đoán mò hoặc trả lời lan man làm hỏng bước 2) | **Cực tốt** (Bắt buộc ghi `KHONG_RO` và ép ra JSON nên nối sang bước 2 rất mượt, không bị đứt gãy) |

---

## 4. Bước 3: Lựa chọn phương án tối ưu và Chạy thử

### Lựa chọn của em: Giải pháp 2 (Chi tiết, nhiều ràng buộc)

**Lý do chọn:**
Dù Giải pháp 2 tốn nhiều token hơn và viết hơi cực lúc đầu, nhưng vì bài toán này dùng kỹ thuật **Prompt Chaining** (lấy đầu ra của bước 1 làm đầu vào cho bước 2), nếu bước 1 trả về văn bản tự do như Giải pháp 1 thì rất dễ xảy ra tình trạng **đứt gãy chuỗi** hoặc AI tự suy diễn sai quy định của shop. Giải pháp 2 ép AI suy luận từng bước và xuất JSON giúp kiểm soát chặt các bẫy dữ liệu bị thiếu.

---

### Kết quả chạy thử Giải pháp 2

Em thử nhập một tin nhắn "bẫy" của khách: **Khách có nói ngày mua (3 ngày) nhưng cố tình không nhắc gì đến tem mác.**

* **Input đưa vào Prompt 2A:**
  > *"Shop ơi áo này mình mới nhận được 3 hôm trước nhưng mặc hơi chật, cho mình đổi sang size XL nhé."*

* **Kết quả AI trả về ở Prompt 2A:**
```json
{
  "suy_luan": "Bước 1: Khách nhận hàng 3 hôm trước (<= 7 ngày, đạt). Bước 2: Khách không nhắc tới tình trạng tem mác nên ghi KHONG_RO. Bước 3: Thiếu thông tin về tem mác nên chưa thể duyệt đổi ngay.",
  "so_ngay": "3 ngày",
  "tinh_trang_tem": "KHONG_RO",
  "trang_thai": "THIEU_THONG_TIN",
  "can_hoi_them": "Tình trạng tem mác của sản phẩm (còn nguyên tem mác hay đã cắt)"
}
```

* **Lấy JSON trên đưa vào Prompt 2B, kết quả AI trả về:**
  > *"Dạ chào bạn, shop đã nhận được yêu cầu đổi size XL cho đơn hàng của mình ạ. Vì đơn mình mới nhận 3 ngày nên vẫn nằm trong thời gian hỗ trợ đổi trả. Bạn cho shop hỏi thêm là áo hiện tại vẫn còn nguyên tem mác chưa cắt phải không ạ? Bạn xác nhận giúp shop để shop hướng dẫn mình gửi đổi hàng nhé!"*
