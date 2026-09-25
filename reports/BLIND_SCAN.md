# Quét độc lập trước khi xem pre-label

Frame: frame_0099.jpg
Số xe nhìn thấy bằng mắt: 26
Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:
1. Cạnh bên phải: một xe bị cắt bởi mép phải ảnh, chỉ thấy phần đầu xe và đèn. Cần vẽ box cho phần xe nằm trong ảnh.
2. Khu vực giữa phía trên, ở các làn xe đi xa: các xe tối và nhỏ nằm gần nhau dễ bị bỏ sót hoặc gộp thành một xe. Cần tách từng xe theo phần thân nhìn thấy.


Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
