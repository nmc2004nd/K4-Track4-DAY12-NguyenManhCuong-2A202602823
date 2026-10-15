# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Nguyễn Mạnh Cường |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602823 |
| GitHub | nmc2004nd |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 2464 / 2133 (các seed 2133..2137) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.253 | 20.4 (CI95 [19.0, 21.9]) | BLOCK (R2, R3, R4, R6) |
| IDM | 1.526 | 0.1 (CI95 [0.04, 0.3]) | (baseline) |
| EgoMLP | 0.767 ± 0.003 (0.763..0.771) | 19.9..30.9 | BLOCK (R2–R6) |
| PercMLP | 0.630 ± 0.003 (0.627..0.636) | 1.1..3.6 | BLOCK (R2–R6) |
| PercMLP+ (sau khi sửa) | 0.822 ± 0.014 | 0.2..0.3 (worst seed 0.35, CI95 [0.19, 0.64]) | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

- **`Acc_LV` không dùng được:** tương quan giữa cột `Acc_LV` và đạo hàm của `Speed_LV` chỉ là **0.045** (gần như 0), trong khi cột tương ứng của xe theo sau `Acc_FAV` tương quan **0.929**. Cột gia tốc xe dẫn là nhiễu, nên khi khai thác ca phanh tôi dùng `a_lead` suy ra từ `Speed_LV` **đã làm mượt** (phân vị p0.1 / p1 / p50 = −2.77 / −1.95 / 0.05 m/s²).
- **Quỹ đạo trùng:** 9993 quỹ đạo thô có **383** bản sao giống hệt (nằm ở các ID cách xa nhau), còn lại 9610. Nếu không loại, một bản nằm ở train và bản kia ở test sẽ làm rò dữ liệu và thổi phồng điểm test.
- **Chia theo quỹ đạo:** các khung liên tiếp của cùng một quỹ đạo tương quan rất mạnh (cửa sổ 2 s lịch sử / 3 s tương lai chồng lên nhau). Chia theo khung hình sẽ đưa gần như cùng một cửa sổ vào cả hai tập. Vì vậy tôi chia theo quỹ đạo (seed 2464): 2883 quỹ đạo giữ lại làm test, tách biệt hoàn toàn với tập train.
- Ngoài ra `Spatial_Gap` đúng bằng `Pos_LV − Pos_FAV − 4.5` (sai lệch lớn nhất 3.6e-11 m), nên có thể dùng gap trực tiếp.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

- **Refactor float32:** cả **600/600** đoạn đều không giống từng bit, nhưng **0/600** đoạn vượt dung sai (atol = rtol = 1e-3, sai lệch lớn nhất 1.2e-04). Phép cộng số thực không có tính kết hợp ((0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6), và float32 làm tròn khác float64, nên so `==` sẽ báo sai cho một thay đổi vô hại. Dung sai cỡ 1e-3 m/s² là hợp lý: nhỏ hơn nhiều so với độ phân giải có ý nghĩa của một lệnh gia tốc, nhưng lớn hơn sai số làm tròn khoảng 10 lần. Chạy lại cùng mã thì giống hệt (0/600).
- **AEB:** chỉ **4/600** đoạn lệch (166, 334, 500, 534). Mở đoạn 334: lệch ở khung 71, 72, 73, nơi gap 3.08 → 2.71 m, tốc độ tiến gần ≈ 2.07 m/s, **TTC 1.49 → 1.33 s < 1.5 s**. Lệnh đổi từ −4.58 / −5.28 / −5.93 sang −6.0. Đoạn 166 chỉ lệch ở khung 4 (TTC 1.49 s, −5.62 → −6.0). Luật AEB chỉ kích hoạt khi TTC < 1.5 s, và ở mọi khung khác lệnh IDM không đổi, nên sai khác dồn vào đúng các khung có TTC thấp.
- **Thời gian thật:** phát nhanh nhất cho 0 khung trễ, 0 khung rớt, vì không có hạn chót. Phát theo thời gian thật cho **582 khung trễ + 574 khung rớt** trên 59622 khung, làm **377/600** đoạn đổi lệnh. Phát nhanh nhất che mất các đột biến thời gian xử lý (140 ms > chu kỳ 100 ms), là lúc bộ điều khiển phải giữ lệnh cũ.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

- Lưới của tôi: cruise **14 m/s**, gap 2.76..59.3 m (p2..p98 của log), tốc độ tiến gần 0..14 m/s, 60 × 45 = **2700 ô**.
- Ô **không ai tránh được** (phanh 9 m/s² sau độ trễ): **3.6 %** với t_react = 0 và **10.2 %** với t_react = 0.6 s. Kiểm chứng bộ mô phỏng (FullBrake so với lời giải vật lý): **0** ô lệch xa biên ở cả hai độ trễ. Vùng đâm của IDM tăng 3.6 % → 10.2 %, trùng khít với vùng bất khả thi (đường đỏ nằm trên đường cyan trong `scenarios_idm.png`). Nghĩa là IDM chỉ đâm ở những ô **ai cũng đâm**; vùng "bộ điều khiển đâm" của IDM là 0.0 %. Không có biên này thì sẽ đổ lỗi nhầm cho IDM khoảng 10 % lưới.
- **Log không phủ vùng khó:** khai thác được 614 ca xe dẫn phanh (6.4 % số quỹ đạo), phanh đỉnh p50 −2.5, p99 −4.2 m/s². Người lái trong log phanh mạnh nhất khoảng **3.6 m/s²** (p99.9), trong khi bộ chấp hành làm được 9. **12.7 %** lưới đòi phanh mạnh hơn mức đó. Các chấm khai thác từ log chủ yếu ở vùng tiến gần chậm và gap trung bình, nên mô hình học từ log chưa từng thấy vùng cắt vào gấp.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín | Hạng |
|---|---|---|---|---|
| const-vel | 1.253 | 3 | 20.4 % | 3 |
| IDM | 1.526 | 4 | 0.1 % | 1 |
| EgoMLP | 0.767 | 2 | 24.3 % (TB; 19.9..30.9) | 4 |
| PercMLP | 0.630 | 1 | 1.9 % (TB; 1.1..3.6) | 2 |

Các cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP).
- **IDM** có L2 tệ nhất (1.526 m), vì nó không bắt chước người: nó giữ khoảng cách theo luật riêng (T = 1.2 s) nên quỹ đạo lệch khỏi người lái. Nhưng nó luôn phản ứng với gap và tốc độ tiến gần, nên trong vòng kín chỉ đâm 0.1 %. L2 đo "giống người", không đo "an toàn".
- **EgoMLP** có L2 tốt hơn const-vel (0.767 so với 1.253) vì nó học được quán tính của tốc độ. Nhưng nó **không nhìn thấy xe dẫn**: thiếu gap và tốc độ xe dẫn, nên khi xe dẫn phanh nó không biết (61.8 % va chạm trên lát cắt phanh). Trong vòng hở, lịch sử là của người lái, vốn đã phanh sẵn, nên EgoMLP "chép" được. Trong vòng kín, lịch sử là của chính nó, nên lỗi tích luỹ và nó đâm 24.3 %. Trên lát cắt phanh (4.5 % số cửa sổ), L2 tăng từ 0.767 lên 1.124: trung bình đã che đi chỗ hỏng.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

- EgoMLP dao động **19.9 % .. 30.9 %** va chạm qua 5 seed (2133..2137), dải rộng 11 điểm, trong khi L2 vòng hở gần như bằng nhau (0.767 ± 0.003).
- Khoảng Wilson 95 % của seed xấu nhất là **[29.3, 32.7] %**. Seed tốt nhất 19.9 % (k/n ≈ 574/2883) có khoảng cỡ [18.5, 21.4] %. **Hai khoảng không chồng nhau**, tức sai khác giữa các seed là thật, không phải nhiễu lấy mẫu. Khoảng tin cậy chỉ đo nhiễu của tập test, không đo nhiễu của việc huấn luyện.
- PercMLP cũng vậy: 1.1 .. 3.6 % (seed xấu nhất CI95 [3.0, 4.4] %).
- Nếu chỉ chạy một seed, tôi có thể báo "EgoMLP đâm 19.9 %" hoặc "30.9 %", hay "PercMLP chỉ 1.1 %" kèm một CI hẹp trông rất chắc chắn. Như vậy là chọn may mắn một seed rồi đưa ra kết luận sai về an toàn. Vì vậy cổng (R2, R5) dùng **seed xấu nhất** và dải qua seed.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng (t_react = 0.6 s, tỷ lệ lưới có va chạm bất kỳ):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm phía sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 80.6 % | 80.6 % | 0.0 % / 0.0 % |
| IDM | 80.9 % | 10.2 % | 70.6 % / 0.0 % |
| EgoMLP | 80.6 % | 80.6 % | 0.0 % / 0.0 % |
| PercMLP | 76.3 % | 31.6 % | 45.7 % / 0.0 % |

(Với t_react = 0: IDM 80.3 % → 3.6 %, phía sau 76.7 % → 0 %.)
- Người lái trong log giữ thời khoảng cách trung vị **1.88 s**. Tôi hiệu chỉnh `T = 1.4 s` (W1 = 0.35, tốt nhất), so với 0.88 của một xe không bao giờ phản ứng. Như vậy xe sau phản ứng gần với phân bố của người lái hơn nhiều.
- Khi xe sau chỉ phát lại, nó không biết xe thử phanh, nên một bộ điều khiển **phanh đúng** như IDM bị xe sau đâm vào: 70.6 % lưới là **va chạm giả**. Const-vel không phanh nên không bị đâm từ phía sau, nhưng tự đâm vào xe phía trước. Trong thế giới phát lại, IDM (80.9 %) trông **tệ ngang hoặc tệ hơn** const-vel (80.6 %). Trong thế giới phản ứng, IDM chỉ 10.2 %, đúng bằng vùng bất khả thi. Kết luận đảo ngược hoàn toàn, nên cần tác nhân phản ứng trước khi chấm.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

- **PercMLP bị chặn** bởi R2 (worst seed 3.61 %, cận trên 4.35 % > 0.10 % + 1), R3 (lát cắt phanh 10.1 %, cận trên 15.4 %), R4 (21.3 % lưới tránh được mà vẫn đâm, so với 0.0 % + 2), R5 (1.1..3.6 %, dải > 1 điểm) và R6 (31.6 % so với 10.2 % + 2). Có hai nguyên nhân gốc:
  1. **Kịch bản log không chứa:** R4/R6. Log không có cắt vào gấp và đòi phanh > 3.6 m/s² (12.7 % lưới), nên mô hình không biết phải phanh mạnh.
  2. **Sai số tích luỹ** (covariate shift) khi tự lái: R2/R3/R5. L2 vòng hở chỉ 0.630 nhưng vòng kín đâm 1.1..3.6 %, và dao động theo seed.
- **Sửa:** (1) thêm 197180 cửa sổ từ 20000 kịch bản cắt vào mô phỏng, nhãn là IDM, để chữa nguyên nhân 1. (2) 2 vòng **DAgger** gắn nhãn IDM ở chính các trạng thái mô hình tự đi vào, để chữa nguyên nhân 2. Kết quả: va chạm trên log 1.1..3.6 % → **0.2..0.3 %**, lưới 21.3 % → **0.3 %**, phản ứng 10.5 %, qua cả 7 quy tắc. Ở tốc độ cruise 20 m/s chưa từng thấy khi huấn luyện, PercMLP+ còn 0.1..2.7 % (PercMLP là 27.7..44.0 %, IDM 0.0 %).
- L2 vòng hở của PercMLP+ lại **tệ hơn**: 0.630 → 0.822 m, vì nó học theo IDM chứ không theo người. Điểm vòng hở không dự đoán được an toàn.
- **PercMLP+ không tốt hơn IDM.** Nó chỉ **không kém hơn** trong biên độ cổng cho phép: worst seed 0.35 % so với 0.10 % của IDM, lưới 0.3 % so với 0.0 %, phản ứng 10.5 % so với 10.2 %. Mọi số đều nhỉnh hơn IDM, và nó học từ chính IDM nên khó vượt thầy. Tôi **không** đề xuất thay IDM ngay: chạy shadow mode bên cạnh IDM, thử thêm kịch bản ngoài phân bố (tốc độ cao, nhiễu cảm biến, trễ), kiểm thử trên dữ liệu mới, và chỉ phát hành khi có lợi ích rõ ràng so với IDM (êm hơn, giống người hơn) mà không đánh đổi an toàn.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Thế giới mô phỏng 1 chiều, không cảm biến thật.** Chỉ có chuyển động dọc, đo đạc hoàn hảo (chỉ thêm nhiễu Gauss đơn giản: PercMLP 1.2..4.0 % khi có nhiễu). Không có chuyển làn, người đi bộ, sai số phát hiện hay mất dấu đối tượng. Con số 0.35 % chỉ đúng trong thế giới này, còn cảm biến thật có thể đẩy mô hình học ra khỏi phân bố nhanh hơn IDM.
2. **Xe dẫn trong log không phản ứng, và tác nhân phản ứng là giả định của chúng ta.** Xe sau là IDM với T = 1.4 s (W1 = 0.35, vẫn chưa khớp hẳn người lái). Kết luận R6 phụ thuộc vào mô hình tác nhân này: nếu người thật phanh chậm hơn hoặc mạnh hơn, kết quả đổi.
3. **Dữ liệu hẹp và nhãn chính là baseline.** Chỉ Argoverse 2, cặp 7–11 s, tốc độ p99 ≈ 17.8 m/s. Log không chứa phanh > 3.6 m/s². Kịch bản bổ sung và DAgger đều gắn nhãn bằng IDM, nên cổng thực chất đo "bắt chước IDM tốt đến đâu", không đo được trường hợp IDM sai.
4. **Thống kê:** 2883 quỹ đạo test và 178 ca phanh. Lát cắt phanh có CI rộng (cận trên 4.0 % cho PercMLP+), chưa đủ để chứng minh tỷ lệ va chạm cỡ 1e-6 mà xe thật cần. 5 seed cũng chưa đủ để biết đuôi phân bố.
- **Việc tiếp theo:** thêm dữ liệu từ nhiều nguồn (đêm, mưa, tốc độ cao), mô phỏng 2D có cảm biến và tác nhân đa dạng được hiệu chỉnh, kiểm thử trên đường đóng, rồi shadow mode trên xe thật trước khi cho mô hình điều khiển.
