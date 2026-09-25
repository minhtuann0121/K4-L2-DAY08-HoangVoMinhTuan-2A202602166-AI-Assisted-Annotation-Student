# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

| Ưu tiên rà | Frame | Hạng CSV | Điểm | Thời điểm | Lý do |
| ---: | --- | ---: | ---: | ---: | --- |
| 1 | `frame_0182.jpg` | 1 | 0.9591 | 72.8 s | Điểm cao nhất, 18/28 box dự đoán mập mờ; cần kiểm tra nhiều xe sáng đèn và xe ở xa trong ảnh. |
| 2 | `frame_0369.jpg` | 2 | 0.9324 | 147.6 s | 16/43 box mập mờ, cảnh có nhiều xe ở cả hai chiều; ưu tiên một thời điểm muộn của video. |
| 3 | `frame_0326.jpg` | 4 | 0.9155 | 130.4 s | 15/39 box mập mờ; bổ sung thời điểm khác với hai frame trên. |
| 4 | `frame_0099.jpg` | 8 | 0.9063 | 39.6 s | Điểm vẫn cao, 14/29 box mập mờ; bổ sung đoạn đầu video để ngân sách rà không tập trung vào đoạn 130–152 s. |
| 5 | `frame_0270.jpg` | 13 | 0.8878 | 108.0 s | 14/35 box mập mờ; thêm một khoảng thời gian riêng giữa `0182` và `0326`. |

Đây là thứ tự ưu tiên khi chỉ rà 5 ảnh, không phải thứ tự thời gian. Tôi không lấy `frame_0380.jpg` (hạng 3, 152.0 s) vì nó cách `frame_0369.jpg` 4.4 s và contact sheet cho thấy bối cảnh giao thông tương tự; với ngân sách ít, rà thêm `0099` và `0270` phủ được các đoạn khác. `frame_0331.jpg` (132.4 s) cũng chỉ cách `0326` đúng 2.0 s. Cả 50 dòng đầu đều có `empty=False`, nên không có ví dụ model không dự đoán box trong phạm vi này. Ở vòng đầu, `D=1.0` cho các frame này vì chưa có frame nào đã gán nhãn; sự đa dạng theo thời gian ở đây là quyết định rà thủ công.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: `frame_0182.jpg` (hạng 1, 72.8 s, 0.9591), `frame_0369.jpg` (hạng 2, 147.6 s, 0.9324) và `frame_0099.jpg` (hạng 8, 39.6 s, 0.9063) đều có `selected=True` trong CSV. Cả ba xuất hiện trong `outputs/selection_round1.jpg` với tên, thời điểm và điểm làm tròn: lần lượt ô thứ 3 hàng 1, ô thứ 4 hàng 2 và ô thứ 1 hàng 1.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: `frame_0372.jpg` xếp hạng 6, điểm 0.9101 tại 148.8 s nhưng có `selected=False`. Nó chỉ cách `frame_0369.jpg` 1.2 s, dưới khoảng cách tối thiểu 2.0 s của bộ chọn lô; rà cả hai dễ tốn ngân sách cho những cảnh gần nhau. Với ngân sách 5 ảnh, tôi giữ `0369` vì điểm cao hơn (0.9324).

Điều phép chọn này chưa chứng minh về chất lượng mô hình: Điểm chọn mẫu đo mức bất định của dự đoán và số box mập mờ, không đo trực tiếp độ đúng của box, số xe bị bỏ sót hay hiệu quả sau khi huấn luyện. Contact sheet chỉ cho thấy bối cảnh của ảnh, không phải nhãn chuẩn. Trên 20 ảnh test độc lập, sau vòng 1 AP50 giảm từ 0.7714 xuống 0.6763, recall tại conf 0.25 giảm từ 0.4888 xuống 0.1960 (`outputs/metrics_round0.json`, `outputs/metrics_round1.json`). Vì vậy lô được chọn có giá trị để rà nhãn, nhưng kết quả hiện có không chứng minh mô hình đã cải thiện.
