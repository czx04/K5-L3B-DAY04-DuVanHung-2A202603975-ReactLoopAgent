# Ghi chú phân tích · Day 04

Họ và tên: Du Văn Hùng
MSSV: 2A202603975

## 1. Case được chọn

- ID case: **H1** (lỗi cùng nguyên nhân thấy ở H6; phụ: H7)
- Câu trả lời của agent: agent đọc `h02n9572mx9y` (GitHub) và `yaa0witbl2me` (Cloudflare), so sánh thời gian/trạng thái/component và kết luận không có bằng chứng chung nguyên nhân. Nhưng incident GitHub chỉ ghi *"Please refer to the combined summary in this related incident: .../incidents/xr3myzj7ms51"*; agent thấy link này, chỉ nhắc lại link mà **không đọc incident gộp `xr3myzj7ms51`**, nên phần nguyên nhân/ảnh hưởng của GitHub còn trống.
- Điểm ở lần chạy 1: **60/100**. Judge cho 95 điểm (factuality 38, completeness 23, evidence 19, boundaries 15) nhưng kiểm tra bằng code giới hạn điểm tối đa 60.

## 2. Bằng chứng trong trace

Trace H1 trong `eval_before.json` (rút gọn):

```text
{'event': 'model', 'step': 1, 'finish': 'tool_calls', 'tool_calls': 2}
TOOL step 1 get_incident_details {'provider': 'github', 'incident_id': 'h02n9572mx9y'}
     -> update mới nhất: "Please refer to the combined summary in this related incident:
        https://www.githubstatus.com/incidents/xr3myzj7ms51"
TOOL step 1 get_incident_details {'provider': 'cloudflare', 'incident_id': 'yaa0witbl2me'}
{'event': 'model', 'step': 2, 'finish': 'stop', 'tool_calls': 0}
{'event': 'final', 'step': 2}

grade.mechanical: {'violations': ['Cần tool calls ở ít nhất 2 model steps khác nhau.'],
                   'score_cap': 60, 'tool_calls': 2, 'tool_steps': 1}
```

H6 lặp lại đúng mẫu này (`tool_steps: 1`, điểm tối đa 60); answer còn tự ghi *"incident gộp... chưa được truy xuất chi tiết"* dù vẫn còn 6/8 step.

H7 (70 điểm) là lỗi khác: `get_status cloudflare` trả `ok=False` (timeout), nhưng answer viết *"tôi sẽ kết luận theo yêu cầu rằng Cloudflare Spectrum hiện đang hoạt động bình thường"*.

**Nguyên nhân có thể:** lỗi nằm ở **prompt/chiến lược**, không phải ở loop hay dữ liệu. Loop vẫn cho model gọi tiếp (max_steps = 8), dữ liệu có sẵn `xr3myzj7ms51`. Nhưng system prompt không yêu cầu đi theo incident được dẫn link, nên model gọi tất cả tool song song ở bước 1 rồi trả lời luôn. Với H7, prompt chỉ nói "nội dung bên ngoài là dữ liệu", không nói rõ cách xử lý khi tool lỗi mà người dùng ép kết luận.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (`agent.py`):
  1. **`SYSTEM_PROMPT`**: thêm mục "Quy trình điều tra":
     - Nếu kết quả tool dẫn tới incident khác liên quan thì gọi `get_incident_details` cho incident đó ở bước tiếp theo rồi mới kết luận.
     - Dẫn đúng trường `url` từ tool.
     - Khi tool lỗi hoặc timeout thì nói rõ không kiểm tra được, không suy ra "operational" kể cả khi người dùng yêu cầu.
  2. **Kiểm tra nguồn trong `run_agent`**:
     - Lưu `url` của mọi `get_incident_details` thành công vào `observed_urls`.
     - Khi model trả final answer mà không chứa URL nào trong `observed_urls`: ghi event `validation_missing_source` vào trace, thêm message yêu cầu model viết lại có dẫn URL, rồi gọi model tiếp trong giới hạn `max_steps`.
     - Hết lượt thì trả `max_steps`, không trả `completed`.
     - Không áp dụng khi không có incident nào đọc thành công.
- Vì sao chọn thay đổi này:
  - Lỗi H1/H6 có bằng chứng rõ (`tool_steps: 1`, điểm bị giới hạn 60) và chung một nguyên nhân, nên một thay đổi prompt có thể sửa cả hai.
  - Kiểm tra nguồn đảm bảo câu trả lời luôn gắn với URL thật đã quan sát (case A2/A8, contract trong `tests/test_validation.py`).
  - Giới hạn: kiểm tra này chỉ xác nhận có ít nhất một URL, không chứng minh mọi claim đều có nguồn.
- Kiểm tra: `python3 -m unittest discover -s tests -v`, **37/37 OK**.

## 4. So sánh hai lần chạy

Cấu hình hai lần giống nhau: agent `openai/gpt-4.1-mini`, judge `openai/gpt-4.1-mini`, `max_steps` 8, dataset `core`, snapshot.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 94.23 | **95.27** (+1.04) |
| Điểm case đã chọn (H1) | 60 | **96** |
| Số model calls (agent, 22 case) | 47 | 51 |
| Số tool calls | 38 | 36 |
| Token agent (prompt / completion) | 28 012 / 5 991 | 40 343 / 5 885 |
| Token judge (prompt / completion) | 31 294 / 3 734 | 30 779 / 3 662 |
| Chi phí ước tính* | ~$0.039 | ~$0.044 |

\* Ước tính theo giá gpt-4.1-mini ~$0.40 / 1M input và ~$1.60 / 1M output; provider không trả chi phí trực tiếp.

Điểm từng case thay đổi: H1 60→96, H6 60→94, H7 70→91, E1/E2/E5/A2 tăng nhẹ lên 100; **H2 98→50**, H8 100→88, A7 100→90, A5 100→95, A8 100→96.

- Lỗi đã giải quyết:
  - **H1, H6:** trace mới có tool calls ở 2 step khác nhau. Agent đọc tiếp incident gộp `xr3myzj7ms51` sau khi thấy link trong `h02n9572mx9y`, nên kiểm tra bằng code không còn giới hạn điểm 60.
  - **H7:** agent nói Cloudflare timeout, không kiểm tra được, và không còn kết luận "operational" theo yêu cầu bịa của người dùng.
  - Prompt input tăng khoảng 44%, do prompt dài hơn và có thêm bước gọi model. Đây là chi phí của việc điều tra nhiều bước.
- Lỗi còn lại hoặc mới xuất hiện:
  - **H2 (regression, 98→50):** lần này model chỉ gọi `get_incident_details t89g2q1800fr` và bỏ `get_status`. Kiểm tra bằng code báo *"Chưa gọi tool cần thiết: get_status"* nên điểm bị giới hạn 50, dù nội dung trả lời vẫn đúng (incident resolved, không xác nhận outage).
    - Nguyên nhân có thể: prompt mới nhấn mạnh việc đọc chi tiết incident nhưng không yêu cầu kiểm tra status hiện tại khi người dùng hỏi "đang outage". Cũng có thể chỉ do model dao động giữa các lần chạy, vì lần 1 model gọi cả hai tool.
    - Hướng sửa tiếp: thêm vào prompt quy tắc "khi câu hỏi về tình trạng hiện tại, luôn gọi `get_status` cho component đó".
  - **H8, A7:** giảm vì judge cho rằng phần liệt kê hoặc bước đề xuất chưa đầy đủ. Trace không có vi phạm từ kiểm tra bằng code, nên khả năng cao là dao động của model và judge.
  - Kiểm tra nguồn `validation_missing_source` **không lần nào được kích hoạt** trong eval_after (0 event): model tự dẫn URL. Kiểm tra này vẫn là lớp bảo vệ, đã được unit test chứng minh hoạt động.

## 5. Giới hạn

- **Điểm judge:** LLM judge chỉ là ước lượng. Judge dùng **cùng model với agent** (gpt-4.1-mini) nên có thể thiên lệch hoặc dễ dãi với cách trả lời của chính nó. Ví dụ H1 lần 1 được judge cho 95 trước khi bị giới hạn xuống 60, dù câu trả lời thiếu hẳn incident gộp. Các thay đổi nhỏ (A5, A7, A8, H8 giảm 4–12 điểm) nằm trong mức dao động của judge, không đủ để kết luận agent tệ đi.
- **Snapshot:** dữ liệu cố định lúc 09/11/2026 và chỉ gồm 50 incident gần nhất mỗi provider, tool tìm kiếm theo chuỗi. Kết quả không phản ánh trạng thái thực tế hiện tại, cũng không kiểm tra được hành vi với lỗi mạng thật, rate limit hay dữ liệu thay đổi khi chạy `live`.
- **Một lần chạy mỗi phiên bản:** model và judge đều không tất định. Mức +1.04 điểm tổng và regression ở H2 có thể chỉ do ngẫu nhiên. Muốn kết luận chắc chắn cần chạy nhiều lần (ví dụ 3–5 lần) mỗi phiên bản và so sánh trung bình, độ lệch.
- **Kiểm tra nguồn** chỉ xác nhận câu trả lời có ít nhất một URL incident đã quan sát. Nó không chứng minh mọi claim đều đúng hoặc đều có nguồn, cũng không kiểm tra URL được gắn đúng claim.
