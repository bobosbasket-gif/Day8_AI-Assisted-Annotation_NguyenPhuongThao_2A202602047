# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét ảnh gần trùng hoặc trường hợp model không dự đoán được box:

- `frame_0182.jpg` - hạng 1, điểm 0.9591, thời điểm 72.8 s: điểm cao nhất, 28 box và 18 box mơ hồ; ưu tiên rà các xe nhỏ hoặc bị che khuất.
- `frame_0369.jpg` - hạng 2, điểm 0.9324, thời điểm 147.6 s: có 43 box và 16 box mơ hồ, là cảnh đông xe nên có nhiều nguy cơ box trùng hoặc bỏ sót.
- `frame_0326.jpg` - hạng 4, điểm 0.9155, thời điểm 130.4 s: 39 box và 15 box mơ hồ; bổ sung một thời điểm khác thay vì chỉ tập trung vào cụm cuối video.
- `frame_0331.jpg` - hạng 5, điểm 0.9154, thời điểm 132.4 s: điểm gần bằng `frame_0326.jpg` và cùng đoạn thời gian; chỉ chọn khi cần kiểm tra sự thay đổi cảnh, nếu hai ảnh quá giống nhau thì dành ngân sách cho frame khác.
- `frame_0099.jpg` - hạng 8, điểm 0.9063, thời điểm 39.6 s: điểm thấp hơn một số ứng viên nhưng khác đoạn thời gian, có 29 box và 14 box mơ hồ; giúp tránh dùng cả năm lượt rà cho các ảnh gần trùng ở cuối video.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

- `frame_0182.jpg` thuộc lô 12, hạng 1, `score=0.9591`, `U=0.9182`, `A=1.0`, `D=1.0`, `n_ambiguous=18`.
- `frame_0369.jpg` thuộc lô 12, hạng 2, `score=0.9324`, `U=0.9315`, `A=0.8889`, `D=1.0`, `n_ambiguous=16`; contact sheet cho thấy nhiều xe ở cả làn gần và xa.
- `frame_0099.jpg` thuộc lô 12, hạng 8, `score=0.9063`, `U=0.9460`, `A=0.7778`, `D=1.0`, `n_ambiguous=14`; đây là ảnh có độ bất định `U` cao nhất trong ba frame và có xe nhỏ, đèn sáng dễ bị nhầm với phản chiếu.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

`frame_0380.jpg` có điểm 0.9170, hạng 3, 40 box và 15 box mơ hồ nhưng không nằm trong năm frame ưu tiên. Trên contact sheet, nó rất gần `frame_0369.jpg` về thời điểm (152.0 s so với 147.6 s) và bố cục giao thông; nếu ngân sách chỉ có năm ảnh, rà cả hai có thể lặp lại cùng một cảnh. Tuy vậy, đây là ứng viên dự phòng quan trọng nếu `frame_0369.jpg` cho thấy lỗi nhãn đặc thù.

Điều mà phép chọn này chưa chứng minh được về chất lượng mô hình:

Điểm `score` chỉ là tín hiệu ưu tiên từ chiến lược uncertainty, được kết hợp từ các chỉ số `U`, `A` và `D` trong `selection_round1.csv`. Việc một ảnh có nhiều box mơ hồ hoặc được chọn không chứng minh rằng model dự đoán đúng hay sai, cũng không chứng minh AP50 sẽ tăng sau khi sửa nhãn. Các ảnh liền kề có thể gần trùng nên cần cân bằng giữa độ bất định và độ đa dạng thời điểm. Chất lượng sau fine-tune phải được đánh giá riêng trên cùng 20 ảnh test bằng các file metrics và compare, trong khi nhãn tham chiếu test do model tạo chưa được người rà toàn bộ.
