# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

Nếu bị giới hạn nghiêm ngặt ở ngân sách 5 ảnh gán nhãn thủ công, người gán nhãn không thể lựa chọn máy móc 5 frame đứng đầu bảng xếp hạng (từ rank 1 đến rank 5). Nguyên nhân là các frame rank 4 (`frame_0326.jpg` tại 130.4s) và rank 5 (`frame_0331.jpg` tại 132.4s) nằm sát nhau (chênh lệch đúng 2.0s), tạo ra hiện tượng cụm cảnh thời gian (temporal clustering), khiến 40% chi phí gán nhãn bị dồn vào cùng một trạng thái giao thông. Để tối ưu hóa hiệu quả tri thức thu nhận, danh sách 5 frame ưu tiên dưới đây được chọn lọc nhằm dung hòa giữa độ bất định cục bộ, mật độ đối tượng gây bối rối và tính bao phủ trải rộng suốt trục thời gian:

1. **`frame_0182.jpg`** (Thứ tự ưu tiên: 1 | Rank: 1 | Điểm: 0.9591 | Thời điểm: 72.8s | U: 0.9182 | A: 1.0000 | 28 box dự đoán, 18 box mập mờ):
   - *Lý do:* Nắm giữ điểm tổng hợp cao nhất toàn bộ pool (0.9591), đồng thời đạt số lượng box mập mờ cực đại (A = 1.0000 với 18 box có confidence trong khoảng 0.15–0.50) và U rất cao (0.9182). Frame thuộc phân đoạn giữa video (72.8s), nơi luồng đèn pha ngược chiều gây chói mạnh, khiến mô hình gặp khó khăn nhất trong việc nhận định ranh giới thân xe.
2. **`frame_0369.jpg`** (Thứ tự ưu tiên: 2 | Rank: 2 | Điểm: 0.9324 | Thời điểm: 147.6s | U: 0.9315 | A: 0.8889 | 43 box dự đoán, 16 box mập mờ):
   - *Lý do:* Điểm cao thứ nhì pool với độ bất định U = 0.9315. Khung hình ghi nhận mật độ giao thông dày đặc nhất ở giai đoạn cuối video (147.6s) với 43 box dự đoán và 16 box mập mờ.
   - *Quyết định loại bỏ ảnh gần trùng:* Việc lựa chọn `frame_0369.jpg` đi liền với quyết định chủ động gạch bỏ `frame_0372.jpg` (Rank 6, score 0.9101, t = 148.8s, chỉ cách 1.2s) và `frame_0368.jpg` (Rank 9, score 0.9003, t = 147.2s, cách 0.4s). Do camera giám sát đặt cố định, khoảng cách 1.2s không tạo ra sự biến đổi đáng kể về vị trí phương tiện; loại bỏ ảnh trùng giúp bảo toàn ngân sách quý báu cho các cảnh quay khác.
3. **`frame_0099.jpg`** (Thứ tự ưu tiên: 3 | Rank: 8 | Điểm: 0.9063 | Thời điểm: 39.6s | U: 0.9460 | A: 0.7778 | 29 box dự đoán, 14 box mập mờ):
   - *Lý do:* Đại diện đắc lực cho phân đoạn đầu video (39.6s), giúp phân bổ tập huấn luyện đồng đều thay vì thiên lệch hoàn toàn về nửa sau clip. Frame này sở hữu độ bất định U cực cao (0.9460 - cao hơn cả rank 1 và rank 2), chứa 14 box gây dao động nhận thức cho mô hình.
4. **`frame_0326.jpg`** (Thứ tự ưu tiên: 4 | Rank: 4 | Điểm: 0.9155 | Thời điểm: 130.4s | U: 0.9310 | A: 0.8333 | 39 box dự đoán, 15 box mập mờ):
   - *Lý do:* Đại diện cho cao điểm ùn ứ phương tiện ở mốc 130s với 39 box dự đoán, 15 box mập mờ và U đạt 0.9310.
   - *Quyết định phòng ngừa trùng cụm:* Giữa cặp đôi `frame_0326.jpg` (130.4s) và `frame_0331.jpg` (rank 5, 132.4s, score 0.9154), ta kiên quyết chỉ lấy `frame_0326.jpg` và bỏ qua `frame_0331.jpg` cũng như `frame_0330.jpg` (rank 12, 132.0s). Dù 130.4s và 132.4s cách nhau đúng 2.0s theo ngưỡng thuật toán, nhưng với hạn mức 5 ảnh, giữ cả hai sẽ làm lãng phí 40% công sức vào một bối cảnh kẹt xe gần như tương đồng.
5. **`frame_0270.jpg`** (Thứ tự ưu tiên: 5 | Rank: 13 | Điểm: 0.8878 | Thời điểm: 108.0s | U: 0.9089 | A: 0.7778 | 35 box dự đoán, 14 box mập mờ):
   - *Lý do:* Tạo nhịp cầu nối đa dạng không gian - thời gian tại mốc 108.0s (nằm giữa khoảng trống lớn từ 72.8s đến 130.4s). Frame có U = 0.9089 và 14 box mập mờ. Việc chọn frame này giúp 5 ảnh trải đều trên toàn bộ dòng thời gian (~40s, ~73s, ~108s, ~130s, ~148s). Đồng thời, đây chính là frame được thực hiện quét độc lập tại `BLIND_SCAN.md`, nơi phát hiện nhiều xe tối màu và xe làn xa bị mô hình bỏ sót.

---

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

1. **`frame_0182.jpg`**:
   - *Minh chứng CSV:* Rank 1, score = 0.9591 (cao nhất pool), t_sec = 72.8s, U = 0.9182, A = 1.0000 (18 box mập mờ - kỷ lục toàn pool), n_boxes = 28, empty = False, selected = True.
   - *Minh chứng contact sheet (`selection_round1.jpg`):* Tọa lạc ở hàng 1, ô số 3. Đoạn đường ban đêm ghi nhận ánh đèn pha ngược chiều rọi thẳng vào thấu kính gây hiện tượng lóa quang học (glare). Nhiều xe ở làn giữa và làn biên bị bóng tối hoặc quầng sáng che khuất một phần, khiến mô hình phân vân ranh giới bao bọc (nhiều box có confidence rơi vào vùng lưỡng lự 0.15–0.50).
2. **`frame_0369.jpg`**:
   - *Minh chứng CSV:* Rank 2, score = 0.9324, t_sec = 147.6s, U = 0.9315, A = 0.8889 (16 box mập mờ), n_boxes = 43 (mật độ xe rất lớn), empty = False, selected = True.
   - *Minh chứng contact sheet (`selection_round1.jpg`):* Nằm ở hàng 2, ô số 4. Tình huống giao thông ùn tắc nghiêm trọng ở cuối video; các vệt đèn hậu đỏ bên phải và luồng đèn trước bên trái san sát nhau. Mô hình phát hiện tới 43 box nhưng gặp trở ngại lớn khi tách bạch các phương tiện bám sát đuôi nhau ở cự ly xa.
3. **`frame_0270.jpg`**:
   - *Minh chứng CSV:* Rank 13, score = 0.8878, t_sec = 108.0s, U = 0.9089, A = 0.7778 (14 box mập mờ), n_boxes = 35, empty = False, selected = True.
   - *Minh chứng contact sheet (`selection_round1.jpg`):* Nằm ở hàng 1, ô số 6. Phân cảnh có sự hiện diện của xe tải cỡ lớn di chuyển ở làn giữa, bên cạnh các chấm đèn đỏ đơn độc của xe con trong góc tối mép đường bên phải. Độ bất định U vượt 0.90 mang lại giá trị gia tăng lớn về mặt tri thức phân loại kích thước và điều kiện chiếu sáng.

---

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

- **Frame điểm cao nhưng KHÔNG chọn: `frame_0372.jpg`**
  - *Thông số định lượng:* Rank 6, score = 0.9101 (đứng thứ 6/268 frame ứng viên), U = 0.9202, A = 0.8333, n_boxes = 42, n_ambiguous = 15, nhưng trạng thái `selected = False`.
  - *Lý do loại trừ:* Thời điểm xuất hiện t_sec = 148.8s chỉ cách `frame_0369.jpg` (Rank 2, t = 147.6s, đã chọn trước đó) vẻn vẹn **1.2 giây**, vi phạm điều kiện khoảng cách tối thiểu `MIN_GAP_S = 2.0s`. Trong 1.2 giây với camera góc cố định, các xe hầu như chưa dịch chuyển vị trí đáng kể, góc phản xạ ánh sáng và bối cảnh hậu cảnh gần như trùng khít. Việc nạp frame này sẽ gây dư thừa thông tin (data redundancy) và tiêu tốn công sức gán nhãn mà không mang lại đột phá học tập cho mô hình.
- **Frame điểm thấp nhưng VẪN NÊN XEM: `frame_0000.jpg`**
  - *Thông số định lượng:* Rank 92, score = 0.7870, t = 0.0s, n_boxes = 24, n_ambiguous = 8, A = 0.4444.
  - *Lý do cần lưu tâm:* Điểm số bị kéo tụt do mật độ box mập mờ thấp (A thấp vì đường thoáng đãng, các xe giãn cách rộng). Tuy nhiên, đây là khung cảnh mở đầu video với các phương tiện di chuyển vận tốc cao, dễ xuất hiện hiện tượng nhòe chuyển động (motion blur) và cho phép quan sát rõ kết cấu thân xe thay vì chỉ thấy đèn pha/đèn hậu chói chang. Nếu hệ thống chỉ tập trung vào các frame điểm cao chứa ùn tắc ánh sáng lóa, mô hình sẽ bị thiên lệch phân phối (distribution bias) và đánh mất năng lực phát hiện xe trong điều kiện giao thông thông thoáng.

---

Điều phép chọn này chưa chứng minh về chất lượng mô hình:

1. **Độ bất định cao không đồng nghĩa với việc nâng cao hiệu năng mô hình:** Điểm bất định ($U$) chỉ thể hiện sự thiếu tự tin của mô hình hiện tại (confidence tiệm cận 0.50), hoàn toàn không phải lời khẳng định chắc chắn rằng khi con người gán nhãn và huấn luyện bổ sung frame đó thì AP50 hay năng lực tổng quát hóa sẽ tự động gia tăng.
2. **Nhiễu thị giác không thể học (Aleatoric Uncertainty):** Độ bất định cao có thể bắt nguồn từ các yếu tố nhiễu quang học thuần túy: chùm đèn pha chiếu trực diện gây lóa cảm biến diện rộng, mặt đường ướt phản quang thành các vệt sáng dài giống xe, hoặc các đốm sáng ở đường chân trời chỉ vỏn vẹn vài pixel không đủ thông tin nhận dạng. Việc bắt ép mô hình học từ các mẫu nhiễu này thậm chí có thể gây hại (overfitting hoặc làm tăng mạnh False Positive).
3. **Bản chất của hàm thu thập mẫu (Acquisition Function):** Công thức tính điểm `score = W_U·U + W_A·A + W_D·D` chỉ là một thuật toán heuristic phục vụ việc sàng lọc ứng viên dựa trên các giả định kỹ thuật, chứ không phải thước đo đánh giá chất lượng mô hình. Năng lực thực tế phải được đo đạc khách quan thông qua kiểm thử thực nghiệm trên tập test chuẩn với các chỉ số mAP, Precision, Recall độc lập.
4. **Giới hạn của tập kiểm nghiệm và nhãn tham chiếu:** Việc chọn lô ảnh này chưa chứng minh được mô hình sẽ vượt trội trong môi trường vận hành thực địa bởi tập test hiện tại chỉ có 20 ảnh, và nhãn tham chiếu do mô hình tự động sinh ra (chưa được rà soát thủ công 100%). Do đó, biến thiên chỉ số sau một vòng chỉ mang tính đối chiếu cục bộ, chưa thể khẳng định năng lực khái quát hóa toàn diện.
