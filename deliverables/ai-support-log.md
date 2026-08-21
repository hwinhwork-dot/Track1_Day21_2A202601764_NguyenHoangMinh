# AI Support Log

Người viết: Nguyễn Hoàng Minh, mã học viên 2A202601764.

## AI đã giúp em ở đâu

| Bước | Em nhờ AI làm gì | Em kiểm lại thế nào |
|---|---|---|
| Gate 1, bộ câu hỏi | Em nhờ AI viết lại 30 câu cho giống giọng học viên thật nói chuyện. Ba trục và 25 ô là nhóm em tự chọn trước. | Em đọc lại từng câu xem có đúng ô mình định test không, câu nào trùng ý thì em bỏ. |
| Gate 2, so nhãn | Em nhờ AI viết script gom ba file nhãn lại và đếm mức đồng thuận. | Em chạy `eval/agreement.py` có sẵn rồi đối chiếu, hai bên cùng ra 15 trên 30. |
| Gate 2, nhãn vàng | Em nhờ AI phân tích 15 câu ba đứa chấm lệch nhau và đề xuất nhãn cho từng câu. | Em bắt AI dẫn bằng chứng trong file kết quả cho từng câu, rồi em tự đọc lại cả 15 câu trước khi chốt. |
| Gate 3, rubric | Em nhờ AI gom các lý do fail trong cột note thành nhóm. | Bảy tiêu chí và chuyện tiêu chí nào là blocker do nhóm em quyết khi thảo luận. |
| Gate 4, judge prompt | Em nhờ AI soạn nháp prompt cho judge và nghĩ ví dụ near-miss. | Em chạy thật bốn vòng rồi so với nhãn vàng, vòng nào không nhích thì em đọc lý do judge đưa ra. |
| Gate 4, code check | Em nhờ AI viết hai hàm check mới theo mẫu có sẵn trong repo. | Em chạy trên cả 30 câu rồi tự tra tay vài câu xem rule có bắt oan không. |
| Vòng cải thiện | Em nhờ AI viết lại system prompt của tutor. | Em chạy lại cả 30 câu ba lần và so với đúng bộ ngưỡng nhóm em đã chốt từ trước. |

## AI sai, hời hợt hoặc làm mất coverage ở đâu

Lần chốt nhãn vàng đầu tiên, AI áp thẳng rubric mới lên cả 30 câu và ra 21 câu không đạt. Con số đó nghiêm hơn cả ba đứa tụi em, vì Minh chấm 12 câu không đạt, Hải chấm 7 câu và Đăng chấm 9 câu.

Khi phân tích hai câu bị tụt sau lúc sửa prompt, AI đoán rằng bộ slide vốn khó trích dẫn hơn tài liệu văn xuôi. Em bảo kiểm lại bằng số thì thấy ngược hẳn, ở lần chạy đầu trích từ slide chỉ sai 6 phần trăm còn trích từ văn xuôi sai tới 22 phần trăm.

Ở câu sc-13, AI nghi judge bị lộ ví dụ trong prompt vào phán quyết. Em bảo mở file kết quả ra xem thì câu trả lời của tutor có con số 92 phần trăm thật, nên judge đọc đúng chứ không lộ gì.

## Em đã tự sửa hoặc quyết định lại điều gì

Em bác cách chốt nhãn vàng đầu tiên và bắt làm lại, vì nhãn vàng phải phản ánh cách nhóm em chấm chứ không phải cách một rubric mới chấm. Cách làm lại là giữ nguyên 15 câu cả ba đứa đã đồng ý và chỉ phân xử 15 câu còn lại.

Em quyết định không dùng dấu ba chấm để loại một quote khỏi diện đạt, sau khi em phân loại 12 ca sai và thấy 5 ca chỉ là lược trích bình thường chứ không phải bịa. Em sửa lại rule thay vì sửa nhãn.

Em tự chấm 30 câu ở vòng độc lập mà không nhờ AI và không xem nhãn của Hải với Đăng. Các con số ngưỡng và verdict cuối là nhóm em ngồi bàn với nhau, dựa trên chuyện học viên chịu được lỗi gì chứ không dựa vào kết quả đã chạy.
