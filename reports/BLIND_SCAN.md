# Quét độc lập trước khi xem pre-label

Frame: frame_0270.jpg

Số xe nhìn thấy bằng mắt: 26 xe

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:
- Vị trí làn đường bên phải phía xa: Một số xe con màu sẫm chỉ thấy cụm đèn hậu đỏ mờ, thân xe chìm vào vùng tối mép đường không có đèn chiếu sáng trực tiếp; AI rất dễ bỏ qua do thiếu đặc trưng biên dạng thân xe.
- Dải xe ở làn đối diện ngược chiều bên trái: Luồng đèn pha chiếu thẳng vào camera gây hiện tượng lóa quang học (glare); AI dễ nhận diện sai kích thước bao bọc thân xe hoặc nhầm vệt sáng trên mặt đường thành một xe độc lập.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
