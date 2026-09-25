# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Hoàng Võ Minh Tuấn

Công cụ gán nhãn đã dùng: CVAT

Sao chép file này thành `reports/REPORT.md` . Mọi con số phải truy được
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` hoặc
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Camera cố định ghi một cảnh liên tục, nên các frame sát nhau có cùng nền và thường chứa cùng chiếc xe. Nếu chia ngẫu nhiên, ảnh gần trùng hoặc cùng xe có thể xuất hiện trong cả pool được dùng để huấn luyện và tập test. Số đo test khi đó có xu hướng **cao hơn** khả năng khái quát thực tế vì mô hình đã thấy cảnh và đối tượng rất giống trong lúc train. Chia theo thời gian và để vùng đệm quanh các đoạn test làm giảm rò rỉ này, dù các ảnh vẫn chung bối cảnh của một camera.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Trong `outputs/compare_round0.jpg`, các box vàng tập trung ở xe nhỏ ở xa, cụm đèn khó tách và một số xe có thân tối hoặc bị che. Recall `small` chỉ 0.182, thấp hơn `medium` 0.547 và `large` 0.561; theo bộ tham chiếu này, xe nhỏ là nhóm bị bỏ sót nhiều nhất. Ở `frame_0350`, có box đỏ quanh một cụm đèn xe sát nhau ở nửa trái ảnh. Cần đối chiếu ảnh gốc và nhãn tham chiếu bằng mắt để biết đó thực sự là dự đoán thừa hay tham chiếu thiếu xe hoặc vẽ ranh giới khác; không thể chỉ dựa vào màu đỏ để kết luận mô hình sai.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

`U` biểu thị mức phân vân của model đối với các box đã dự đoán, `A` tăng khi ảnh có nhiều box mập mờ, còn `D` ưu tiên ảnh cách xa thời điểm đã gán nhãn. `W_U`, `W_A`, `W_D` là trọng số cộng ba thành phần thành điểm chọn mẫu. `MIN_GAP_S` ngăn chọn những frame quá sát nhau trong cùng lô, giảm công rà các cảnh gần trùng của camera cố định.

Trong `reports/SELECTION.md`, `frame_0182.jpg` (hạng 1, điểm 0.9591, 72.8 s, 18 box mập mờ), `frame_0369.jpg` (hạng 2, 0.9324, 147.6 s, 16 box mập mờ) và `frame_0099.jpg` (hạng 8, 0.9063, 39.6 s, 14 box mập mờ) đều được chọn theo CSV. `0099` giúp trải công rà sang đoạn sớm của video. Một frame khác là `frame_0372.jpg` (hạng 6, 0.9101, 148.8 s) có `selected=False`: nó chỉ cách `0369` 1.2 s nên dễ lặp cảnh và xe. Điểm bất định không đo các xe model bỏ sót hoàn toàn và không chứng minh việc gán ảnh đó sẽ cải thiện kết quả sau train.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

Tập test gồm 20 ảnh, 403 box tham chiếu được chấm; 14 box cao dưới 16 px bị bỏ qua. Ngưỡng IoU là 0.5; P/R/F1 tính tại conf 0.25. Bảng từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 336 | 0.676 | -0.095 | 1.000 | 0.196 | 0.328 | 0.000 | 0.196 | 0.512 |

Vòng 0 chưa có nhãn gợi ý được sửa và là mốc so sánh. Ở vòng 1, trên 12 ảnh, pre-label có 169 box: giữ nguyên 145, chỉnh 9, xoá 15 và thêm 182, thành 336 box (`outputs/round1_diff.md`). AP50 giảm 0.095 so với cold start; vì đây là vòng fine-tune đầu tiên nên mức giảm so với vòng trước cũng là 0.095. Recall theo cùng tập test giảm ở cả ba nhóm: `small` 0.182 xuống 0.000, `medium` 0.547 xuống 0.196 và `large` 0.561 xuống 0.512. Precision tăng 0.925 lên 1.000 tại conf 0.25 nhưng recall và F1 đều giảm, nên không thể coi riêng precision là cải thiện.

Hai ảnh `outputs/compare_round0.jpg` và `outputs/compare_round1.jpg` cho thấy ở `frame_0150`, cold start có TP 10 / FP 2 / FN 10, còn vòng 1 có TP 3 / FP 0 / FN 17. Nhiều xe xa khớp tham chiếu ở cold start đã thành box vàng bị bỏ sót sau fine-tune. Giả thuyết cần kiểm là confidence của các dự đoán xe khó giảm dưới ngưỡng chấm; cần xem confidence gốc trước khi kết luận nguyên nhân.

`reports/BLIND_SCAN.md` ghi quan sát độc lập **trước khi xem pre-label**: `frame_0099.jpg` có 26 xe nhìn thấy, trong đó hai xe ở góc trái chỉ lộ một đèn. Đây là ước lượng bằng mắt, không phải kết quả test. `outputs/round1_diff.md` cho biết riêng frame đó từ 13 box gợi ý thành 26 box sau rà (giữ 12, xoá 1, thêm 14); đây là sửa nhãn train, không phải mức cải thiện của mô hình. `reports/REVIEW_LOG.csv` ghi thêm box `car 189` ở `frame_0326.jpg`, thêm `car 164` và sửa `car 169` ở `frame_0312.jpg`. Tuy nhiên, diff của `frame_0312.jpg` tính `edited = 0`; cần đối chiếu log, file nhãn đã lưu và cách ghép box của báo cáo diff trước khi khẳng định thao tác sửa này được tính vào tổng. Kết quả sau train phải được đánh giá riêng trên tập test.

Ca khó là hai xe chỉ lộ một đèn ở mép trái `frame_0099.jpg`. Theo `GUIDELINE_LABEL.md`, nếu vẫn xác định được thân xe thì chỉ vẽ box quanh phần xe nhìn thấy trong ảnh, không kéo box ra ngoài mép và không tính vệt phản chiếu đèn trên mặt đường. Nếu chỉ thấy ánh sáng mà không xác định được xe, không nên gán nhãn như một xe chắc chắn.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Theo bộ tham chiếu hiện có, vòng 1 kém cold start: AP50 từ 0.771 xuống 0.676, recall từ 0.489 xuống 0.196, F1 từ 0.640 xuống 0.328. Tôi tạm dừng train thêm để kiểm tra nguyên nhân trước khi mở vòng tiếp theo. Cần rà tính nhất quán của 336 box train, nhất là box mới thêm và box sát mép; kiểm tra định dạng/toạ độ YOLO, ánh xạ lớp `car`, cấu hình suy luận và phân bố confidence. Cũng cần rà thủ công một phần nhãn tham chiếu test để hiểu sai lệch phép đo, nhưng không đưa ảnh test vào train.

Hai ca nên cân nhắc cho vòng sau là xe nhỏ ở xa chỉ còn cụm đèn, vì recall `small` của vòng 1 là 0.000, và xe bị che hoặc cắt mép như trong `frame_0099.jpg`. Ca thứ nhất tốn công phóng to và quyết định nhất quán có đủ dấu hiệu của xe để gán; ca thứ hai tốn công tách thân xe khỏi phản chiếu và đặt box chỉ trên phần nhìn thấy. Cả hai dễ lặp cùng xe nếu lấy nhiều frame sát nhau, nên phải cân nhắc độ trải thời gian trước khi trả thêm chi phí rà nhãn.

Kết luận bị giới hạn bởi chỉ 20 ảnh test của một camera. Việc bỏ qua 14 box cao dưới 16 px khiến số đo không phản ánh xe cực xa; nhãn tham chiếu do mô hình tạo chưa được rà từng box có thể thiếu xe hoặc lệch vị trí. Do AP50 giảm, cần kiểm tra dự đoán mất confidence hay lệch box trong các ảnh so sánh, chất lượng nhãn train và sai lệch tham chiếu test trước khi quy mức giảm cho chiến lược chọn mẫu hoặc tiếp tục fine-tune.
