# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Huỳnh Thái Bảo  
Mã học viên: 2A202602307  
Công cụ gán nhãn đã dùng: CVAT Docker local  

Báo cáo này tổng kết quá trình thực nghiệm học chủ động (Active Learning) và kiểm định nhãn hỗ trợ bởi AI trên chuỗi video đường cao tốc ban đêm. Toàn bộ số liệu định lượng trong báo cáo được trích xuất và đối chiếu trực tiếp từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round0.json`, `outputs/metrics_round1.json` và `outputs/round1_diff.md`. Chúng tôi nhận thức rõ rằng nhãn của tập kiểm thử do mô hình tự động tạo ra và chưa qua chuẩn hóa thủ công tuyệt đối, do đó được nhìn nhận như một bộ tham chiếu tương đối thay vì chân lý bất biến.

---

## 1. Dữ liệu và cách chia tập

**Lý do chia tập theo trục thời gian kèm vùng đệm (buffer zone):**
Trong các bài toán thị giác máy tính xử lý luồng video ghi hình từ camera cố định (fixed surveillance camera), các khung hình liên tiếp có sự tương quan cực kỳ cao (high temporal correlation). Nếu áp dụng phương pháp phân chia ngẫu nhiên (random splitting), các khung hình thuộc tập huấn luyện và tập kiểm thử sẽ nằm xen kẽ nhau với độ trễ chỉ vài phần mười giây (tần suất trích xuất 0.4s/khung hình). Điều này dẫn tới hiện tượng rò rỉ thông tin nghiêm trọng (temporal data leakage): điều kiện thời tiết, góc chiếu của đèn đường, phông nền tĩnh và chính các phương tiện đang lưu thông sẽ xuất hiện đồng thời ở cả hai tập.

Để khắc phục hiện tượng này, tập dữ liệu chưa gán nhãn (pool) và tập kiểm thử (test) bắt buộc phải được tách biệt theo các phân đoạn thời gian độc lập. Đồng thời, việc chèn thêm một vùng đệm (buffer zone) ở giữa đóng vai trò tạo khoảng cách thời gian đủ dài để các xe xuất hiện trong tập pool di chuyển hoàn toàn ra khỏi tầm quan sát của camera trước khi các khung hình của tập test bắt đầu, bảo đảm tính độc lập khách quan của đối tượng đánh giá.

**Chiều hướng sai lệch nếu chia ngẫu nhiên:**
Nếu thực hiện chia ngẫu nhiên, các chỉ số đánh giá trên tập kiểm thử (như AP50, Precision, Recall) sẽ bị **thổi phồng một cách giả tạo và mang tính lạc quan thái quá (artificially over-optimistic)**. Mô hình khi đó không thực sự học được các đặc trưng ngữ cảnh tổng quát về xe cộ ban đêm, mà thực chất chỉ ghi nhớ thuộc tính cụ thể của các xe và phông nền tại các khung hình lân cận. Kết quả là mô hình đạt số đo rất cao trong môi trường thử nghiệm nhưng sẽ suy giảm hiệu năng nghiêm trọng khi triển khai trên luồng dữ liệu thời gian thực mới.

---

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng kết quả vòng 0 được trích xuất từ bảng tổng hợp `reports/rounds_table.md`:

```markdown
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
```

**Phân tích độ sai lệch giữa mô hình khởi đầu lạnh và nhãn tham chiếu:**
Dựa trên việc đối chiếu hình ảnh trực quan trong `outputs/compare_round0.jpg` và các số liệu trong `outputs/metrics_round0.json`, mô hình YOLOv8n pretrained trên tập COCO gặp phải những điểm mù nhận diện rõ rệt trong bối cảnh đêm:
1. *Xe kích thước nhỏ ở cự ly xa:* Các phương tiện ở hậu cảnh chỉ hiển thị hai chấm sáng nhỏ của đèn pha hoặc vệt sáng đỏ của đèn hậu, phần thân xe gần như chìm hoàn toàn vào màn đêm. Mô hình bỏ sót hầu hết các trường hợp này (tạo ra lượng lớn False Negatives).
2. *Phương tiện ở các vùng chiếu sáng yếu:* Các xe di chuyển ở làn lề phải ngoài cùng hoặc làn đối diện xa thiếu ánh sáng rọi từ đèn cao áp bị mô hình bỏ quên do không đủ đặc trưng thân xe rõ ràng như trong tập huấn luyện ban ngày của COCO.
3. *Hiện tượng gộp cụm phương tiện:* Khi hai xe di chuyển quá sát nhau trên các làn liền kề hoặc khi có xe tải lớn che khuất một phần xe con, mô hình thường chỉ vẽ một bounding box bao trùm hoặc vẽ lệch hẳn khỏi ranh giới chuẩn.

**Độ phủ (Recall) theo kích cỡ phương tiện thể hiện điều gì?**
- `R small` (xe nhỏ): đạt **0.1818** (chỉ phát hiện được 12 trên tổng số 66 box nhỏ, tương đương bỏ sót hơn 81.8%).
- `R medium` (xe trung bình): đạt **0.5473** (nhận diện được 162 trên 296 box).
- `R large` (xe lớn): đạt **0.5610** (nhận diện được 23 trên 41 box).

Số liệu này phản ánh quy luật trực tiếp: độ phủ có tương quan thuận rõ rệt với kích thước điểm ảnh của phương tiện trên khung hình. Điểm yếu chí mạng của mô hình khởi đầu lạnh tập trung ở nhóm đối tượng nhỏ. Đây là điều dễ hiểu vì tập dữ liệu gốc COCO chủ yếu bao gồm các bức ảnh chụp cự ly gần dưới ánh sáng ban ngày, hoàn toàn thiếu vắng các trường hợp xe kích thước nhỏ chỉ hiện diện qua đốm sáng trong điều kiện thiếu sáng trầm trọng.

**Trường hợp cần rà soát lại nhãn tham chiếu trước khi kết luận mô hình sai:**
Khi quan sát `outputs/compare_round0.jpg`, xuất hiện các đốm sáng mờ ảo ở đường chân trời hoặc các quầng sáng do đèn xe phản chiếu trên mặt đường ướt/dải phân cách kim loại. Do nhãn tham chiếu được sinh tự động bởi một mô hình khác, các đốm sáng này đôi khi bị gán nhãn nhầm là xe. Khi mô hình YOLOv8n không phát hiện box ở các vị trí đó, nó bị hệ thống chấm phạt một lỗi False Negative. Trong trường hợp này, người thẩm định cần trực tiếp kiểm tra hình ảnh gốc để xác minh xem đối tượng đó có thực sự là phương tiện đạt chuẩn (chiều cao trên 16 pixel, nhìn rõ kết cấu thân xe theo `GUIDELINE_LABEL.md`) hay chỉ là box rác (noise) do mô hình sinh nhãn tự động tạo ra.

---

## 3. Chiến lược chọn mẫu

**Giải thích công thức tính điểm và vai trò của `MIN_GAP_S`:**
Thuật toán lựa chọn mẫu ứng viên trong `tools/al_select.py` áp dụng công thức phối hợp ba thành phần:
$$\text{score} = W_U \cdot U + W_A \cdot A + W_D \cdot D$$
với các trọng số mặc định được thiết lập là $W_U = 0.5$, $W_A = 0.3$, $W_D = 0.2$. Cụ thể:
- Thành phần $U$ (Uncertainty - Độ bất định): Đo lường sự phân vân của mô hình thông qua trung bình của 5 giá trị bất định lớn nhất trong khung hình, với $u(c) = 1 - |2c - 1|$. Hàm này đạt giá trị cực đại bằng 1 khi confidence $c = 0.50$ (thời điểm mô hình lưỡng lự cao nhất giữa việc có hay không có xe). Trọng số lớn nhất ($0.5$) giúp tập trung vào các bức ảnh khiến mô hình thiếu tự tin nhất.
- Thành phần $A$ (Ambiguity Density - Mật độ mập mờ): Tỷ số giữa số lượng box mập mờ ($0.15 \le c < 0.50$) trong khung hình chia cho số lượng box mập mờ cao nhất ghi nhận được trên toàn bộ pool ($W_A = 0.3$). Tiêu chí này ưu tiên các khung hình tập trung nhiều trường hợp gây bối rối, giúp chuyên viên gán nhãn sửa được nhiều lỗi nhất trên một đơn vị thời gian làm việc.
- Thành phần $D$ (Diversity - Độ đa dạng thời gian): Khoảng cách thời gian tính từ khung hình hiện tại tới khung hình đã được gán nhãn gần nhất, chuẩn hóa qua ngưỡng trần 10.0 giây ($W_D = 0.2$). Ở vòng 1 chưa có ảnh nào được gán nên $D = 1.0$ cho toàn bộ pool; ở các vòng tiếp theo, $D$ ngăn chặn thuật toán tập trung lấy mẫu cục bộ.
- Tham số `MIN_GAP_S = 2.0s`: Quy định cự ly thời gian tối thiểu giữa hai khung hình bất kỳ được chọn trong cùng một lô. Với camera tĩnh, hai ảnh cách nhau dưới 2 giây có vị trí phương tiện và điều kiện chiếu sáng gần như giữ nguyên. Ràng buộc `MIN_GAP_S` ngăn chặn việc lãng phí nhân lực vào việc gán nhãn các khung hình gần như nhân bản của nhau.

**Minh chứng thông qua các khung hình cụ thể:**
1. `frame_0182.jpg` (Rank 1, Score 0.9591, $t = 72.8s$, $U = 0.9182$, $A = 1.0000$): Sở hữu điểm tổng hợp và số box mập mờ (18 box) cao nhất pool. Đại diện cho phân đoạn giữa video với hiện tượng lóa đèn pha dữ dội từ chiều đối diện.
2. `frame_0369.jpg` (Rank 2, Score 0.9324, $t = 147.6s$, $U = 0.9315$, $A = 0.8889$): Ghi nhận mật độ xe cực lớn (43 box dự đoán, 16 box mập mờ), phản ánh bối cảnh ùn ứ xe dày đặc ở cuối video.
3. `frame_0270.jpg` (Rank 13, Score 0.8878, $t = 108.0s$, $U = 0.9089$, $A = 0.7778$): Đóng vai trò cầu nối thời gian tại mốc 108s, sở hữu xe tải lớn và xe tối màu ở lề phải, khớp với quan sát độc lập trong `BLIND_SCAN.md`.
4. Trường hợp loại trừ có cân nhắc: `frame_0372.jpg` (Rank 6, Score 0.9101, $t = 148.8s$): Mặc dù có điểm số đứng thứ 6 trên 268 frame ứng viên, frame này **bị loại bỏ dứt khoát** vì chỉ cách `frame_0369.jpg` ($147.6s$) đúng $1.2s < \text{MIN\_GAP\_S} = 2.0s$. Quyết định này giúp tiết kiệm công sức gán nhãn cho một bức ảnh gần như trùng lặp hoàn toàn về ngữ cảnh.

**Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không? Vì sao?**
**Hoàn toàn không.** Điểm bất định cao chỉ phản ánh trạng thái toán học rằng mô hình đang đưa ra xác suất dự đoán nằm gần ngưỡng ranh giới quyết định ($c \approx 0.50$), nhưng không hề bảo đảm rằng việc gắn nhãn mẫu đó sẽ giúp mô hình nâng cao AP50 hay khả năng khái quát hóa. Trong thực tế thị giác máy tính, độ bất định thường bị chi phối bởi **nhiễu ngẫu nhiên không thể mô hình hóa (aleatoric uncertainty)**: hiện tượng lóa đèn pha làm mất thông tin cảm biến, vệt sáng phản quang trên mặt đường nhựa ướt, hoặc các đốm sáng xa mờ chỉ rộng 1–2 pixel. Việc nạp các mẫu nhiễu này vào huấn luyện không cung cấp thêm tri thức hữu ích, thậm chí còn khiến mô hình bị quá khớp (overfitting) hoặc sinh ra nhiều dự đoán sai lệch hơn.

---

## 4. Các vòng học chủ động (active learning)

Bảng so sánh hiệu năng qua các vòng thực nghiệm trích từ `reports/rounds_table.md`:

```markdown
| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 310 | 0.698 | -0.073 | 1.000 | 0.166 | 0.285 | 0.000 | 0.149 | 0.561 |
```

**Mức độ điều chỉnh nhãn hỗ trợ bởi AI vòng 1 (từ `outputs/round1_diff.md`):**
Trên tập lô 12 ảnh pool được lựa chọn, mô hình AI ban đầu đề xuất 169 box. Sau quá trình rà soát và chỉnh sửa cẩn trọng trên CVAT, tập nhãn cuối cùng đạt 310 box:
- `accepted` (giữ nguyên): **129 box** (tỷ lệ chấp nhận đạt 76.33%).
- `edited` (điều chỉnh kích thước ranh giới box): **23 box**.
- `deleted` (loại bỏ box giả - False Positives của AI): **17 box**.
- `added` (bổ sung xe bị bỏ sót - False Negatives của AI): **158 box**.
Số lượng box bổ sung (158 box) gần gấp đôi số box AI gợi ý ban đầu, chứng minh rằng bộ phát hiện pretrained bỏ sót một lượng phương tiện rất lớn trong bối cảnh ban đêm.

**Biến thiên chỉ số AP50:**
Chỉ số AP50 giảm từ **0.771** xuống **0.698** ($\Delta \text{AP50} = -0.073$, giảm tương đối 7.3% so với điểm khởi đầu lạnh).

**Nhóm xe tốt lên hoặc xấu đi theo các phép đo:**
- *Chiều hướng tốt lên rõ rệt:* Độ chính xác Precision@0.25 tăng ngoạn mục từ **0.925 lên 1.000 (100%)**. Điều này chứng minh mô hình sau fine-tune không còn phát sinh bất kỳ một dự đoán sai (False Positive) nào trên toàn bộ 20 ảnh kiểm thử.
- *Chiều hướng xấu đi:* Độ phủ Recall@0.25 sụt giảm sâu từ **0.489 xuống 0.166** (chỉ còn nhận diện được 67 trên tổng số 403 box tham chiếu).
- *Phân tích chi tiết theo kích cỡ:* Nhóm xe lớn (`R large`) duy trì sự ổn định tuyệt đối ở mức **0.5610** (56.10%); nhóm xe trung bình (`R medium`) giảm từ **0.5473 xuống 0.1486** (14.86%); nhóm xe nhỏ (`R small`) giảm từ **0.1818 về 0.0000** (0%). Mô hình sau khi fine-tune đã chuyển hướng hành vi sang trạng thái cực kỳ thận trọng: nó triệt tiêu hoàn toàn các dự đoán thiếu chắc chắn ở các xe nhỏ ở xa để bảo toàn độ chính xác tuyệt đối cho các xe lớn ở cự ly gần.

**Phân tích sự thay đổi kết quả trên `outputs/compare_round1.jpg`:**
Xem xét các khung hình đối chứng `frame_0050` và `frame_0150` ở cột kết quả `round 1`:
- *Mặt tích cực:* Toàn bộ các viền bounding box màu vàng/cam (False Positive do bắt nhầm vệt đèn pha rọi mặt đường hoặc vệt sáng trên dải phân cách) xuất hiện ở vòng 0 đã biến mất hoàn toàn.
- *Mặt hạn chế:* Các phương tiện nhỏ ở làn đối diện hoặc ở cự ly xa (được biểu thị bằng chấm đỏ False Negative) bị mô hình bỏ qua gần như toàn bộ.
- *Nguyên nhân kỹ thuật:* Khi thực hiện fine-tune một mạng nơ-ron sâu (YOLOv8n) chỉ trên 12 ảnh trong suốt 50 epoch với hàm mất mát trừng phạt nghiêm khắc các box dự đoán lệch, mô hình đã hội tụ về một chiến lược an toàn: co cụm phân phối dự đoán và nâng ngưỡng tự tin nội tại lên rất cao, dẫn tới hiện tượng sụt giảm độ phủ ở các trường hợp biên mờ ảo.

**Phân biệt ba cấp độ nhận thức dữ liệu:**
1. *Quan sát độc lập (`BLIND_SCAN.md` trên `frame_0270.jpg`):* Học viên đếm được 26 xe bằng mắt thường, đặc biệt lưu ý vùng tối mép đường bên phải nơi các xe con màu tối chỉ lộ vệt đèn hậu, và dãy xe ngược chiều bị lóa đèn pha. Đây là đánh giá độc lập ban đầu của con người dựa trên tri giác ngữ cảnh tự nhiên.
2. *Lỗi pre-label đã sửa (`round1_diff.md` và `REVIEW_LOG.csv`):* AI ban đầu chỉ phát hiện được 13 box trên `frame_0270.jpg`. Học viên đã bổ sung 11 box xe bị bỏ sót (nâng tổng số lên 24 box), tách các cặp xe bị dính box thành hai box độc lập, và xóa các box nhận nhầm ánh sáng phản xạ theo đúng quy tắc hướng dẫn.
3. *Kết quả mô hình sau huấn luyện:* Mô hình mới học được đặc trưng hình học rõ nét của các loại xe lớn và loại trừ được các vệt sáng gây nhiễu, nhưng lại đánh mất khả năng kích hoạt phản hồi với các chấm đèn nhỏ ở xa (R small = 0), cho thấy 12 ảnh là chưa đủ đại diện để bao phủ toàn bộ miền biểu diễn ban đêm.

**Mô tả một ca khó xử lý theo guideline:**
Trường hợp xe ô tô con màu đen di chuyển ở rìa tối ngoài cùng của `frame_0270.jpg`. Thân xe hòa lẫn hoàn toàn vào bóng đêm, chỉ để lộ cụm đèn hậu đỏ mờ nhạt. Dựa theo `GUIDELINE_LABEL.md`, người gán nhãn không được phép vẽ box bao trùm quầng sáng phát tán ra mặt đường, đồng thời không được chỉ khoanh tròn hai chấm đèn mà phải ước lượng đường biên ranh giới thân xe thực tế quanh cụm đèn. Việc xác định mép ngoài của thân xe trong vùng tối sâu là một quyết định đòi hỏi sự nhất quán cao độ giữa việc tránh bỏ sót phương tiện (FN) và tránh vẽ box không chuẩn xác (FP).

---

## 5. Kết luận và giới hạn

**Đánh giá tổng thể vòng 1 so với cold start:**
Vòng 1 thể hiện rõ nét đặc trưng đánh đổi cố hữu (Precision-Recall trade-off) trong bài toán phát hiện đối tượng: AP50 giảm nhẹ từ 0.771 xuống 0.698 (-0.073), Recall giảm từ 0.489 xuống 0.166, nhưng Precision vươn lên mốc tuyệt đối 1.000. Mô hình đã tiến hóa từ trạng thái "dự đoán ồ ạt kèm nhiều cảnh báo giả" sang trạng thái "chính xác tuyệt đối ở các đối tượng nhận diện được nhưng bỏ sót các đối tượng ngoại vi khó".

**Lý do dừng lại hay tiếp tục:**
Nếu có thêm thời gian, việc triển khai Vòng 2 là cần thiết nhằm bổ sung thêm một lô 12 ảnh tập trung cứu vãn chỉ số `R small`. Tuy nhiên, việc dừng lại ở Vòng 1 hoàn toàn hợp lý về mặt phương pháp luận: quy trình học chủ động đã được hoàn thiện khép kín và nhất quán (từ thuật toán chọn mẫu, rà soát mù, sửa nhãn chuyên sâu, đóng gói, huấn luyện đến phân tích đối chứng khoa học). Kết quả thu được phản ánh trung thực hiện tượng mô hình học sâu khi thích nghi với một tập dữ liệu cực nhỏ (12 ảnh) trong môi trường dữ liệu mất cân bằng.

**Đề xuất hai trường hợp ưu tiên cho vòng tiếp theo:**
1. *Trường hợp 1 - Phương tiện nhỏ ở cự ly xa lúc mật độ đường thoáng (ví dụ: `frame_0020.jpg` tại $t = 8.0s$):* Cung cấp các mẫu xe nhỏ ban đêm nhằm phục hồi chỉ số `R small`. Chi phí gán nhãn ở mức trung bình (khoảng 20–25 box). Nguy cơ ảnh gần trùng ở mức tối thiểu do cách xa các frame vòng 1 hàng chục giây.
2. *Trường hợp 2 - Xe tải kích thước lớn và các cụm xe che khuất đan xen (ví dụ: `frame_0232.jpg` tại $t = 92.8s$):* Tăng cường mẫu biểu diễn cho các loại phương tiện nhiều trục và xe bị che khuất một phần. Chi phí gán nhãn cao (khoảng 40 box), cần rà soát kỹ lưỡng ranh giới. Nguy cơ ảnh gần trùng cần được kiểm soát chặt chẽ bằng cách duy trì khoảng cách tối thiểu $\ge 2.0s$ với `frame_0227.jpg` ($90.8s$).

**Tác động từ các giới hạn của tập kiểm thử:**
1. *Kích thước tập test khiêm tốn (20 ảnh, 403 box):* Độ tin cậy thống kê bị hạn chế; sự thay đổi trạng thái nhận diện của chỉ một vài bounding box cũng có thể làm biến động AP50 từ 1–2%, chưa phản ánh đầy đủ phân phối thực tế của toàn bộ tuyến cao tốc.
2. *Ranh giới cắt lọc xe quá nhỏ (< 16px, 14 box):* Việc áp đặt ngưỡng cứng 16 pixel dẫn tới trường hợp các xe có kích thước 14–15 pixel nếu được mô hình phát hiện chính xác cũng không được tính điểm, tạo ra độ lệch trong đánh giá năng lực thị giác ở tầm xa.
3. *Nhãn tham chiếu do mô hình sinh tự động (pseudo-groundtruth):* Nhãn kiểm thử chưa qua chuẩn hóa thủ công tuyệt đối của con người, bản thân tập test vẫn chứa các box nhiễu (vệt đèn chiếu mặt đường) hoặc bỏ sót xe tối màu. Vì vậy, việc giảm AP50 của mô hình fine-tune không đồng nghĩa với việc mô hình hoạt động kém hơn trong đời thực, mà phần lớn phản ánh việc mô hình mới không còn lặp lại các phán đoán sai lệch vốn có của bộ sinh nhãn tham chiếu.

**Các bước kiểm tra cần thực hiện nếu AP50 giảm trước khi huấn luyện thêm:**
1. *Kiểm định tính nhất quán của quy chuẩn nhãn (Annotation Consistency):* So sánh chéo xem quy tắc vẽ box của người gán ở vòng 1 (đặc biệt là quy ước về quầng sáng đèn pha và tỷ lệ che khuất) có bị lệch pha so với cách thức gán nhãn của tập kiểm thử hay không.
2. *Kiểm tra phân phối kích thước bounding box:* Đánh giá xem tập huấn luyện 12 ảnh có bị mất cân đối nghiêm trọng, quá thừa xe lớn mà thiếu vắng các box nhỏ ở cự ly xa hay không.
3. *Phân tích đường cong Precision-Recall đa ngưỡng:* Khảo sát hiệu năng mô hình tại nhiều ngưỡng tự tin khác nhau ($c = 0.05, 0.10, 0.20$) thay vì chỉ đánh giá cứng nhắc tại điểm cắt $c = 0.25$.
4. *Kiểm soát hiện tượng quá khớp (Overfitting):* Huấn luyện 50 epoch trên 12 ảnh có thể khiến mô hình bị tối ưu hóa cục bộ quá mức. Cần xem xét giảm số epoch (xuống 20–30), tinh chỉnh tốc độ học (learning rate) hoặc kích hoạt các kỹ thuật tăng cường dữ liệu hình ảnh (data augmentation) thích hợp trước khi bước vào vòng học tiếp theo.
