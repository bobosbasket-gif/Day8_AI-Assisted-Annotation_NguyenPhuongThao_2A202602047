# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Phương Thảo - 2A202602047

Công cụ gán nhãn đã dùng: CVAT

Các con số trong báo cáo được truy từ `reports/rounds_table.md`, `outputs/selection_round1.csv`,
`outputs/metrics_round*.json` hoặc `outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là
chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo hướng nào, và vì sao?

Pool và test được chia theo trục thời gian để tránh cùng một chiếc xe, hoặc các khung hình gần như giống nhau, xuất hiện ở cả hai tập. Camera cố định và ảnh được lấy cách nhau 0.4 giây; một chiếc xe có thể ở trong ảnh vài giây. Vùng đệm loại các ảnh gần đoạn test, nên test đo khả năng tổng quát sang thời điểm khác thay vì ghi nhớ cùng cảnh và cùng xe.

Nếu chia ngẫu nhiên, các khung hình gần nhau có thể rơi vào cả train và test, tạo rò rỉ dữ liệu. Khi đó AP50 và recall có xu hướng cao hơn thực tế vì model được đánh giá trên cảnh hoặc xe rất giống những gì nó đã thấy. Cách chia hiện tại dùng 20 ảnh test, 268 ảnh pool và 112 ảnh buffer; các ảnh pool gần test nhất cách ít nhất 4.4 giây.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

Dòng vòng 0 trong `reports/rounds_table.md` là: `0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561`.

Trên 20 ảnh test, cold start có 197 TP, 16 FP và 206 FN tại conf 0.25. `compare_round0.jpg` cho thấy model khớp tương đối tốt với xe lớn và một số xe vừa, nhưng bỏ sót nhiều xe nhỏ ở xa, xe tối chỉ còn đèn, xe bị che hoặc nằm sát nhau. Điều này phù hợp với recall theo kích thước: small `0.182` (12/66), medium `0.547` (162/296) và large `0.561` (23/41). Recall small thấp hơn rất nhiều so với hai nhóm còn lại.

Một trường hợp cần rà lại nhãn tham chiếu trước khi kết luận model sai là xe rất xa chỉ còn hai đốm đèn hoặc vùng đèn phản chiếu trên mặt đường. Có 14 box tham chiếu cao dưới 16 pixel được bỏ qua khi chấm; nhãn tham chiếu cũng do model tạo và chưa được người rà từng box. Vì vậy cần kiểm tra trực tiếp ảnh và box trước khi gọi đó là lỗi của cold start, đồng thời không sửa `data/test/labels/`.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không? Vì sao?

Notebook dùng `W_U=0.5`, `W_A=0.3`, `W_D=0.2`, nên `score = 0.5U + 0.3A + 0.2D`: độ bất định của dự đoán có trọng số lớn nhất, tiếp theo là tỷ lệ box mơ hồ và độ đa dạng theo thời gian. `MIN_GAP_S=2.0` yêu cầu hai ảnh được chọn cách nhau ít nhất 2 giây; với camera cố định, điều này giảm việc tốn công rà nhiều frame gần như trùng nhau.

Ba ví dụ trong `reports/SELECTION.md` là `frame_0182.jpg` (hạng 1, score `0.9591`, 18 box mơ hồ), `frame_0369.jpg` (hạng 2, score `0.9324`, 16 box mơ hồ) và `frame_0099.jpg` (hạng 8, score `0.9063`, U `0.9460`, 14 box mơ hồ). Chúng đại diện cho cảnh đông xe, xe nhỏ hoặc tối và các vị trí cần người xem lại. `frame_0380.jpg` có score cao `0.9170` nhưng gần thời điểm và bố cục của `frame_0369.jpg` (152.0 s so với 147.6 s), nên không ưu tiên cả hai nếu ngân sách chỉ đủ năm ảnh; nó vẫn là frame dự phòng để kiểm tra khi `frame_0369.jpg` có lỗi đặc thù.

Score bất định chỉ giúp ưu tiên chi phí gán nhãn. Nó không chứng minh ảnh đó chứa lỗi, không chứng minh model sẽ cải thiện sau fine-tune và không thay thế việc xem box. Kết quả thực nghiệm vòng 1, AP50 giảm, càng cho thấy không được suy ra chất lượng model chỉ từ score chọn mẫu.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

### Bảng số đo

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 278 | 0.419 | -0.353 | 1.000 | 0.112 | 0.201 | 0.000 | 0.078 | 0.537 |

Ở vòng 1, trong 169 box pre-label của 12 ảnh, có 129 box giữ nguyên, 23 box chỉnh sửa, 17 box xóa và 126 box thêm mới; sau sửa có 278 box, accept rate `76.33%`. Các số này truy được từ `outputs/round1_diff.md` và `round1_diff.json`. Log cho thấy các lỗi chính gồm xe bị bỏ sót ở vùng xa hoặc sát mép ảnh, box gộp hai xe, box không bao xe nào và box trùng.

So với cold start, AP50 giảm `0.353` xuống `0.419`; so với vòng trước cũng là giảm `0.353`. Precision tăng từ `0.925` lên `1.000` vì model chỉ tạo 45 TP và 0 FP, nhưng recall giảm từ `0.489` xuống `0.112` và FN tăng lên 358. Xe nhỏ giảm từ `0.182` xuống `0.000`, xe vừa từ `0.547` xuống `0.078`, còn xe lớn gần tương đương nhưng vẫn giảm nhẹ từ `0.561` xuống `0.537`. Vì vậy vòng 1 không cho thấy nhóm nào tốt lên rõ ràng; model trở nên quá thận trọng và bỏ sót nhiều xe.

Một ca rõ trong `compare_round1.jpg` là `frame_0050`: cold start có `TP 11, FP 2, FN 7`, còn round 1 có `TP 2, FP 0, FN 16`. Round 1 giảm box dự đoán, nên loại được FP nhưng bỏ sót nhiều xe ở xa và xe nhỏ. Nguyên nhân cần kiểm có thể là lô train nhỏ, phân bố chỉ 12 ảnh, chất lượng hoặc độ nhất quán của box sau sửa, cấu hình fine-tune và ngưỡng confidence; không thể kết luận chỉ từ một ảnh rằng người gán nhãn sai.

`BLIND_SCAN.md` là quan sát độc lập trước pre-label về 26 xe ở `frame_0099`, tập trung vào xe bị cắt ở cạnh phải và các xe nhỏ, tối ở làn xa. `REVIEW_LOG.csv` và `round1_diff.md` là bằng chứng về lỗi pre-label đã được sửa, ví dụ thêm các xe ở giữa/sát cạnh dưới, tách box gộp và xóa box không bao xe hoặc bị trùng. `metrics_round1.json` và `compare_round1.jpg` chỉ mô tả kết quả model sau train, không phải chất lượng của từng quyết định gán nhãn.

Một ca khó theo guideline là xe chỉ thấy đèn và thân xe tối: nếu vẫn đoán được đường viền và box cao hơn khoảng 16 pixel thì vẽ box quanh phần thân đoán được, không chỉ khoanh hai đốm đèn. Với xe bị cắt ở mép ảnh, chỉ vẽ phần nằm trong ảnh; hai xe sát nhau vẫn phải có hai box riêng.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh, có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Vòng 1 kém hơn cold start trên cùng 20 ảnh test: AP50 giảm từ `0.771` xuống `0.419`, recall giảm từ `0.489` xuống `0.112`, và recall xe nhỏ giảm về `0.000`. Tôi dừng việc kết luận rằng fine-tune đã cải thiện model và không nên tiếp tục train thêm chỉ bằng cách lặp lại cùng quy trình trước khi kiểm tra nguyên nhân.

Hai nhóm nên ưu tiên ở vòng sau là: (1) xe nhỏ hoặc tối ở các làn xa, vì recall small bằng `0.000` sau vòng 1 và đây cũng là điểm yếu đã quan sát trong blind scan; (2) xe bị che, sát mép ảnh hoặc nằm trong cảnh đông xe, vì các ca này có nguy cơ box gộp, box trùng và bỏ sót cao. Chi phí rà nhãn của mỗi ảnh là đáng kể, nên chọn các frame có bất định cao nhưng phải tránh các frame gần trùng; `frame_0380` chỉ nên chọn thêm nếu cần kiểm tra khác biệt so với `frame_0369`.

Tập test chỉ có 20 ảnh, nên thay đổi nhỏ có thể do phương sai mẫu. Luật bỏ qua xe quá nhỏ làm các xe khó nhất không đóng góp đầy đủ vào AP50. Nhãn tham chiếu do model tạo chưa được rà thủ công, nên AP50 đo mức khớp với bộ tham chiếu chứ không phải chân lý thực địa. Vì vậy kết luận hiện tại là round 1 không khớp tốt với benchmark này, chưa phải bằng chứng tuyệt đối rằng mọi box người sửa đều sai.

Nếu AP50 giảm, trước khi train thêm tôi sẽ: kiểm tra đủ 12 ảnh và 278 box, class id `0`, tọa độ YOLO và kích thước box; đối chiếu `REVIEW_LOG.csv` với `round1_diff.json`; xem các box thêm/xóa ở ảnh tối, xe nhỏ và xe sát nhau; kiểm tra không đưa ảnh test vào train; sau đó kiểm tra cấu hình fine-tune, ngưỡng confidence và dấu hiệu model chỉ dự đoán các xe lớn. Chỉ khi dữ liệu và pipeline hợp lệ mới thử vòng mới, đồng thời so sánh lại trên đúng 20 ảnh test và giữ nguyên nhãn test.
