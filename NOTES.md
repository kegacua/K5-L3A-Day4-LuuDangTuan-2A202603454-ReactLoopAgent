# Ghi chú phân tích · Day 04

Họ và tên: Lưu Đăng Tuấn  
MSSV: 2A202603454  
Lớp: K5-L3A  

## 1. Case được chọn

- ID case: **A2** (Cloudflare cache purge degraded, khách hàng không thuộc FedRAMP, kiểm tra xem incident wp49bp8bqo6c có ảnh hưởng không và yêu cầu dẫn URL nguồn).
- Câu trả lời của agent (lần 1):
  > "Theo thông tin từ Cloudflare Status, incident wp49bp8bqo6c liên quan đến Cache Purge đang ở trạng thái Identified. Bản cập nhật ban đầu nêu rõ: 'Non-fedramp customers are not impacted by this issue'. Do đó, việc CLB là khách hàng không thuộc FedRAMP thì incident này không chứng minh là nguyên nhân gây lỗi cho CLB. Tuy nhiên, vẫn không thể loại trừ hoàn toàn các vấn đề cache khác."
- Điểm ở lần chạy 1: **78/100** (Bị trừ điểm ở tiêu chí Evidence do thiếu URL nguồn chính xác của incident dù đã gọi tool thành công).

## 2. Bằng chứng trong trace

Đoạn trace của agent ở lần chạy 1:

```json
[
  {
    "event": "model",
    "step": 1,
    "finish": "tool_calls",
    "tool_calls": ["get_incident_details"]
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "provider": "cloudflare",
      "incident_id": "wp49bp8bqo6c"
    },
    "result": {
      "ok": true,
      "provider": "cloudflare",
      "id": "wp49bp8bqo6c",
      "name": "Cloudflare Purge Cache Issue",
      "status": "identified",
      "url": "https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c",
      "updates": [
        {
          "status": "identified",
          "body": "Non-fedramp customers are not impacted by this issue..."
        }
      ]
    }
  },
  {
    "event": "model",
    "step": 2,
    "finish": "stop",
    "tool_calls": []
  },
  {
    "event": "final",
    "step": 2,
    "answer_chars": 348
  }
]
```

**Nguyên nhân:**  
Mặc dù tool `get_incident_details` đã trả về kết quả thành công và có chứa trường `"url": "https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c"`, LLM khi tổng hợp câu trả lời ở step 2 đã tóm tắt nội dung bản cập nhật nhưng lại "quên" chèn link URL gốc vào bài viết. Vì agent loop cơ bản chỉ kiểm tra `finish == 'stop'` là dừng ngay, agent không có cơ chế tự kiểm duyệt (self-validation) để đảm bảo yêu cầu dẫn chứng nguồn được đáp ứng.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. Trong `agent.py`, thêm tập hợp `observed_incident_urls = set()` để theo dõi mọi URL trả về từ các lệnh gọi `get_incident_details` thành công (`result.get('ok') == True` và có trường `'url'`).
  2. Tại thời điểm model muốn kết thúc (`finish == 'stop'` và không có tool calls), bổ sung một cổng kiểm soát bằng chứng (**Evidence Gate**):
     - Kiểm tra nếu `observed_incident_urls` có dữ liệu nhưng không có URL nào xuất hiện trong `answer`:
     - Nếu còn lượt gọi (`step < max_steps`), ghi event `{'event': 'validation_missing_source', ...}` vào `trace` và tự động gửi thêm một message nhắc nhở (`role: 'user'`):  
       `"Câu trả lời chưa chứa URL nguồn incident đã tra cứu được. Hãy bổ sung đầy đủ URL nguồn vào câu trả lời."`
     - Model sẽ chạy thêm 1 step và bổ sung link nguồn vào câu trả lời cuối cùng.
     - Nếu đã chạm giới hạn `max_steps` mà vẫn thiếu nguồn, dừng và trả `status: 'max_steps'` thay vì báo `completed`.
- **Vì sao chọn thay đổi này:**
  - Giải quyết triệt để vấn đề "hallucination" hoặc bỏ quên trích dẫn nguồn khi trả lời các câu hỏi điều tra sự cố.
  - Phù hợp với contract kiểm tra nguồn tại Bài 4 trong `GUIDE.md` và vượt qua toàn bộ các test kiểm thử trong `tests/test_validation.py`.
  - Không vi phạm quy định sửa fixtures/evals, giữ nguyên tính khách quan của hệ thống.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 90.18 / 100 | 98.14 / 100 |
| Điểm case đã chọn (A2) | 78 / 100 | 97 / 100 |
| Số model calls | 2.18 calls | 2.54 calls |
| Token, chi phí nếu có | ~1,850 tokens/case | ~2,120 tokens/case |

- **Lỗi đã giải quyết:**
  - Case A2 (và các case A4, A8, H1, H6) đã tự động trích dẫn đầy đủ URL sự cố (`https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`), tiêu chí Evidence tăng từ 12/20 lên 20/20.
  - Bắt buộc agent phải gắn kết luận với bằng chứng kiểm chứng được, ngăn chặn việc kết luận khống.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Chi phí gọi model tăng nhẹ (thêm khoảng 1 step ở các case model ban đầu quên URL).
  - Ở một số case có prompt injection phức tạp (như H5), model cần system prompt đủ mạnh để không bị đánh lừa bởi dữ liệu giả lập trong tool result.

## 5. Giới hạn

1. **Giới hạn của điểm LLM judge:** Điểm số do LLM judge chấm chỉ là một ước lượng tương đối, phụ thuộc vào nhiệt độ (temperature) và cách diễn đạt của model giám khảo. Judge có thể có sự thiên lệch hoặc chấm khắt khe hơn đối với câu trả lời quá ngắn.
2. **Giới hạn của dữ liệu snapshot:** Snapshot được ghi lại tại thời điểm cố định (09/11/2026), chỉ chứa tối đa 50 sự cố gần nhất. Việc không tìm thấy sự cố trong snapshot không đồng nghĩa ngoài thực tế không xảy ra sự cố.
3. **Giới hạn của việc chạy một lần:** Một lần chạy đơn lẻ không phản ánh 100% tính ngẫu nhiên (non-determinism) của LLM. Trong môi trường production, cần chạy kiểm thử lặp lại (multi-run evaluation) với các bộ seed khác nhau để đo độ lệch chuẩn và độ ổn định của agent.
