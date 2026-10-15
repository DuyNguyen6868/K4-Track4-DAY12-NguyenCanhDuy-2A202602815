# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Nguyễn Cảnh Duy` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602815` |
| GitHub | `DuyNguyen6868` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `3103 / 4253` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `3` (các seed 4253, 4254, 4255) |

## Kết quả cuối

Số lấy từ `evidence/closedloop.json`, `openloop.json`, `gate.json`. Với mô hình học, cột va chạm ghi **dải qua 3 seed**; quyết định cổng dựa trên seed tệ nhất.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.264 | 20.0 | BLOCK (rớt R2, R3, R4, R6) |
| IDM | 1.522 | 0.1 | (baseline) |
| EgoMLP | 0.765 | 15.6 .. 24.2 | BLOCK (rớt R2, R3, R4, R5, R6) |
| PercMLP | 0.623 | 0.5 .. 0.9 | BLOCK (rớt R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.827 | 0.2 .. 0.3 | BLOCK (rớt R4, R6; qua R1, R2, R3, R5, R7) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Bước 0 cho thấy ba vấn đề. (1) Cột `Acc_LV` không dùng được: tương quan của nó với đạo hàm `Speed_LV` chỉ 0.045, trong khi cột tương ứng của xe sau `Acc_FAV` là 0.929. Vì vậy tôi không dùng `Acc_LV` mà dùng gia tốc xe dẫn suy ra từ `Speed_LV` đã làm mượt (`logs.a_lead`) để khai thác ca phanh. (2) Có 383 quỹ đạo trùng hệt nhau ở các ID cách xa nhau, nên 9993 quỹ đạo thô còn 9610 sau khi loại trùng. Nếu không loại, một bản có thể rơi vào train và bản kia vào test, mô hình được chấm trên đúng thứ nó đã thấy và điểm test bị thổi phồng. (3) Tôi chia theo **quỹ đạo**, không theo khung hình, vì hai khung cách nhau 0.1 s gần như giống nhau: chia theo khung thì "bản sinh đôi" của mỗi khung test luôn nằm trong train. Kết quả là 2883 quỹ đạo giữ lại để chấm, tách hẳn khỏi tập huấn luyện.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã cho 0/600 đoạn khác bit. Bản refactor float32 khác bit ở **600/600** đoạn nhưng vượt dung sai ở **0** đoạn, chênh lệch lớn nhất chỉ 4.0e-04 so với dung sai 1e-3 m/s². Nguyên nhân là phép dấu phẩy động không có tính kết hợp ((0.1+0.2)+0.3 khác 0.1+(0.2+0.3)) và float32 làm tròn thô hơn, nên so sánh bằng nhau tuyệt đối sẽ báo sai ở mọi đoạn; phải so theo dung sai chọn theo ý nghĩa vật lý của tín hiệu. Thay đổi AEB làm lệch **8/600** đoạn (58, 140, 187, 230, 234, 316, 519, 535). Mở đoạn 58 (số liệu dưới đây quan sát trên màn hình khi mở đoạn, không nằm trong JSON): chỉ 4 trong 106 khung lệch (khung 98 đến 101), ở đó TTC là 1.43 đến 1.49 s (dưới ngưỡng 1.5 s của AEB) và lệnh đổi từ khoảng -5.44 .. -5.91 thành -6.0 m/s². Ở mọi khung có TTC lớn hơn, AEB không kích hoạt nên lệnh giữ nguyên từng bit. Phát nhanh nhất cho 0 khung trễ và 0 khung rớt trên 59571 khung vì không có hạn chót; phát theo thời gian thật lộ ra **581 khung trễ và 576 khung rớt**, làm 371/600 đoạn đổi. Một khung xử lý quá khung 0.1 s bị trễ, và khung kế tiếp đến khi bộ điều khiển còn bận bị rớt, nên lỗi về thời hạn chỉ thấy được khi phát đúng nhịp.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới của tôi dùng tốc độ cruise 14 m/s (do MSSV quyết định), khoảng cách 2.8 đến 59.3 m (p2 .. p98 của log), tốc độ tiến gần 0 đến 14 m/s, tổng 2700 ô (60 x 45). Vùng **không ai tránh được** là 3.6 % lưới khi độ trễ phản ứng 0 s và 10.2 % khi trễ 0.6 s. Vùng đâm của IDM đi đúng theo biên đó: 3.6 % rồi 10.2 %, tức diện tích "đâm dù tránh được" của IDM là 0.0 % ở cả hai độ trễ, và kiểm chứng bộ mô phỏng cho 0 ô lệch ở cả hai độ trễ. Nhờ biên này tôi tách được "ai cũng đâm" khỏi "bộ điều khiển đâm": constant-velocity đâm tới 70.4 % lưới ở vùng tránh được (trễ 0.6 s). Log không phủ vùng khó: người lái trong log phanh mạnh nhất chỉ 3.6 m/s² (p99.9) trong khi bộ chấp hành làm được 9 m/s², và **12.7 % lưới** đòi phanh mạnh hơn mức đó. Chỉ 614 ca phanh khai thác được (6.4 % trong 9610 quỹ đạo, giảm tốc đỉnh trung vị -2.5 m/s²), nên mô hình học từ log chưa từng thấy các tình huống này.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng L2 | Va chạm trên log (%) | Hạng va chạm |
|---|---|---|---|---|
| constant-velocity | 1.264 | 3 | 20.0 | 3 |
| IDM | 1.522 | 4 | 0.1 | 1 |
| EgoMLP | 0.765 | 2 | 15.6 .. 24.2 | 4 |
| PercMLP | 0.623 | 1 | 0.5 .. 0.9 | 2 |

Bốn cặp đổi thứ tự giữa hai cách chấm: (constant-velocity, IDM), (constant-velocity, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất (1.522 m) mà lại ít đâm nhất vì L2 đo mức "giống người lái", còn IDM là luật cố định không học theo người (người giữ headway trung vị 1.87 s). Trong vòng kín, IDM nhìn khoảng cách và tốc độ tiến gần rồi phanh theo đó, nên chỉ đâm 0.1 % (2 trong 2883 quỹ đạo) và phanh trung vị khoảng 1.0 m/s². EgoMLP có L2 tốt hơn constant-velocity (0.765 so với 1.264 m) vì ngoại suy lịch sử tốc độ của người lái khá trơn, nhưng nó **không thấy xe dẫn**, nên không biết lúc nào phải phanh: phanh đỉnh trung vị chỉ khoảng 0.07 m/s² (seed tệ nhất), và trên các quỹ đạo có xe dẫn phanh nó đâm 46.9 % (seed tệ nhất). Vòng hở còn không cho mô hình gánh hậu quả của hành động chính nó: sai số không tích luỹ vì trạng thái luôn do người lái tạo ra.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Tỷ lệ va chạm của EgoMLP dao động từ **15.6 % đến 24.2 %** giữa ba seed (21.7 %, 15.6 %, 24.2 % cho seed 4253, 4254, 4255). Khoảng Wilson 95 % của seed tốt nhất là [14.3, 16.9] và của seed tệ nhất là [22.6, 25.8]: hai khoảng **không chồng nhau**, nên sự khác biệt đến từ huấn luyện chứ không phải nhiễu lấy mẫu. Nếu chỉ chạy seed 4254 tôi sẽ báo EgoMLP đâm 15.6 %, thấp hơn rõ rệt constant-velocity (20.0 %, khoảng [18.6, 21.5]) và kết luận "EgoMLP tốt hơn constant-velocity"; chạy seed 4255 thì ra 24.2 % và kết luận ngược lại. Quyết định phát hành dựa trên một lần chạy vì thế có thể do may rủi, nên cổng dùng seed tệ nhất và có quy tắc R5 về độ ổn định qua seed: EgoMLP rớt R5 (15.6 .. 24.2 %), còn PercMLP và PercMLP+ qua R5 (0.5 .. 0.9 % và 0.2 .. 0.3 %).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Tôi hiệu chỉnh xe sau theo IDM với thời khoảng cách T = 1.4 s. Khoảng cách phân bố headway (Wasserstein W1) so với người lái trong log là 0.35, trong khi một xe không bao giờ phản ứng có W1 = 0.85, nên xe sau phản ứng chân thực hơn nhiều (người lái giữ headway trung vị 1.87 s). Kết quả ở độ trễ 0.6 s, tỷ lệ lưới có va chạm bất kỳ, xe sau phát lại so với phản ứng:

| Bộ điều khiển | Phát lại (%) | Phản ứng (%) | Va chạm phía sau: phát lại / phản ứng (%) |
|---|---|---|---|
| constant-velocity | 80.6 | 80.6 | 0.0 / 0.0 |
| IDM | 80.9 | 10.2 | 70.6 / 0.0 |
| EgoMLP | 82.4 | 82.4 | 0.0 / 0.0 |
| PercMLP | 77.1 | 33.8 | 43.3 / 0.0 |
| PercMLP+ | 81.0 | 12.3 | 70.6 / 0.0 |

Khi có xe cắt vào, IDM phanh mạnh, đúng việc phải làm; xe sau *phát lại* cứ chạy theo log, không biết xe thử vừa phanh, nên húc vào đuôi: đó là "va chạm giả" (70.6 % lưới với IDM) và nó biến mất (0.0 %) khi xe sau phản ứng. Constant-velocity và EgoMLP không phanh nên không bị húc đuôi (0.0 %), vì chúng đâm vào xe phía trước trước. Vì thế trong thế giới phát lại, IDM (80.9 %) trông ngang constant-velocity (80.6 %) và PercMLP (77.1 %) còn trông **tốt hơn** IDM; khi xe sau phản ứng thì IDM chỉ còn 10.2 % (đúng bằng vùng bất khả thi) còn PercMLP là 33.8 %. Loại tác nhân phản ứng đã đảo ngược kết luận về PercMLP và IDM.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi bốn quy tắc, từ hai nguyên nhân gốc. **Kịch bản log không chứa** (R4, R6): trên lưới cắt vào, PercMLP đâm dù tránh được ở 23.6 % lưới (IDM là 0.0 %, ngưỡng baseline + 2 điểm), và với xe sau phản ứng là 33.8 % so với 10.2 % của IDM. Điều này khớp với Câu 3: 12.7 % lưới đòi phanh mạnh hơn mọi thứ có trong log. **Sai số tích luỹ khi tự lái** (R2, R3): ngay trên log, seed tệ nhất đâm 0.9 % (cận trên 1.3 %) so với ngưỡng 0.1 % + 1 điểm, và trên lát cắt có xe dẫn phanh là 2.8 % (cận trên 6.4 %) so với 0.6 % + 5 điểm; R5 thì qua (0.5 .. 0.9 %). Tôi sửa bằng hai cách: thêm các cửa sổ huấn luyện từ 20000 cut-in mô phỏng (chữa nguyên nhân 1, vì đây đúng là họ kịch bản log thiếu) và 2 vòng DAgger (chữa nguyên nhân 2, vì mô hình học ở chính các trạng thái nó tự đi vào, với nhãn là việc IDM làm ở đó). Hiệu quả: va chạm trên log giảm còn 0.2 .. 0.3 %, diện tích lưới 23.6 % xuống 2.0 %, nên R2 và R3 chuyển sang qua. Nhưng **PercMLP+ vẫn bị BLOCK**: R4 còn 2.0 % (vượt ngưỡng 0.0 % + 2 điểm một chút; tính theo seed tệ nhất, các seed là 0.15, 0.0, 2.04 %) và R6 là 12.3 % so với 10.2 % + 2 điểm. Nó **không tốt hơn** IDM: nhãn của nó đến từ IDM nên giỏi lắm chỉ bằng thầy, và mọi chỉ số trên đều kém hơn hoặc bằng IDM (va chạm trên log 0.2 .. 0.3 % so với 0.1 %, lưới 2.0 % so với 0.0 %). Ở tốc độ cruise 20 m/s chưa từng gặp khi huấn luyện, PercMLP+ vẫn đâm dù tránh được ở 0.4 .. 3.1 % lưới (PercMLP là 17.0 .. 30.4 %, IDM là 0.0 %). L2 vòng hở của PercMLP+ (0.827 m) còn **xấu hơn** PercMLP (0.623 m) trong khi vòng kín tốt hơn nhiều, thêm một bằng chứng điểm vòng hở không phải thước đo để phát hành. Đề xuất: không thay IDM, giữ IDM làm baseline; trước khi quyết định lại thì mở rộng dải tốc độ cut-in khi huấn luyện, thêm seed, chạy shadow mode trên xe thật, và đặt tiêu chí "tốt hơn IDM" rõ ràng chứ không chỉ "không kém hơn".

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) **Thế giới mô phỏng là giả định của chúng ta**: chỉ một chiều dọc, phanh tối đa 9 m/s², độ trễ 0.6 s, bộ chấp hành lý tưởng, và xe dẫn trong log không phản ứng với xe thử. Tôi chỉ thử nhiễu cảm biến Gauss đơn giản (khoảng cách 1 m, tốc độ 0.5 m/s), và PercMLP+ đâm 0.2 .. 0.3 % khi không nhiễu và 0.24 .. 0.35 % khi có nhiễu; nhiễu thật có trễ, mất dấu, nhận nhầm. Kết luận BLOCK hay RELEASE chỉ đúng trong thế giới này. (2) **Dữ liệu hẹp**: một bộ Argoverse 2 (9610 quỹ đạo sạch, 70 đến 109 bước ở 10 Hz, tức đoạn ngắn 7 đến 11 s, đô thị), người lái phanh tối đa 3.6 m/s². Log không có vùng phanh gấp, nên không thể biết mô hình xử lý ra sao ở 12.7 % lưới đó ngoài đời, và PercMLP+ chỉ được huấn luyện cho cruise tới 16 m/s trong khi tôi thử ở 20 m/s. (3) **Thống kê sự kiện hiếm và ít seed**: 2883 quỹ đạo giữ lại với 179 ca xe dẫn phanh, không đủ để chứng minh tỷ lệ va chạm cỡ một trên hàng triệu km mà xe thật cần; chỉ 3 seed, và R4 của PercMLP+ dao động 0.15, 0.0, 2.04 % giữa các seed. (4) **Ngưỡng cổng là lựa chọn của đội**: PercMLP+ trượt R4 và R6 sát ngưỡng (R4: 2.0 % so với ngưỡng 0.0 % + 2 điểm; R6: 12.3 % so với 10.2 % + 2 điểm), nên kết luận nhạy với ngưỡng, và qua cổng cũng không có nghĩa là an toàn. Để thu hẹp khoảng cách đến thế giới thật cần: mô phỏng giàu hơn (nhiều tác nhân phản ứng, cảm biến thật, chuyển làn), dữ liệu phanh gấp và cut-in thật, nhiều seed hơn, kiểm thử phần cứng trong vòng lặp và chạy shadow mode trên xe thật trước khi cho điều khiển.
