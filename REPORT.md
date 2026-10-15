# REPORT — Lab Day 12

| Thông tin | Giá trị |
|---|---|
| Họ và tên | Hoàng Văn Nam |
| MSSV (đúng bằng `--student-id` đã chạy) | 2A202602853 |
| GitHub | [namhv521](https://github.com/namhv521/K4-Track4-Replay-CloseOpenLoop) |
| Seed chia dữ liệu / seed mô hình | 8554 / 7588, 7589, 7590, 7591, 7592 |
| Số seed huấn luyện (`--seeds`) | 5 |
| Môi trường thực nghiệm | Windows, Python 3.12.9, CPU, một luồng BLAS, không dùng Colab |

MSSV đầy đủ được chương trình chấp nhận; không cần dùng mã rút gọn `02853`.
Lệnh thực nghiệm đầy đủ: `python run_all.py --student-id 2A202602853` (mặc định 5 seed, không `--quick`).
Chỉ sửa thân 10 hàm TODO. Không sửa kiểm thử, simulator, dữ liệu gốc hoặc ngưỡng cổng; evidence được chương trình sinh.

## Kết quả cuối

L2 là trung bình L2 vị trí tại 1, 2 và 3 giây; bảng ghi dải qua các seed.
Va chạm là tỷ lệ quỹ đạo test có va chạm trong mô phỏng vòng kín; bộ điều khiển xác định chỉ có một giá trị.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| const-vel | 1.277–1.277 | 20.916–20.916 | chặn |
| IDM | 1.567–1.567 | 0.035–0.035 | baseline |
| EgoMLP | 0.770–0.773 | 17.725–23.517 | chặn |
| PercMLP | 0.631–0.637 | 0.555–3.989 | chặn |
| PercMLP+ | 0.794–0.847 | 0.139–0.208 | chặn |

Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [gate.json](evidence/gate.json).
Dải L2 của PercMLP+ được tính lại từ 5 model đã lưu, cùng tập test và hàm chấm; lưu ở [report_details.json](evidence/report_details.json).

## Câu 1 · Dữ liệu

Ba vấn đề chính là cột gia tốc xe dẫn không đáng tin, quỹ đạo trùng gây rò rỉ dữ liệu, và các khung hình cùng quỹ đạo phụ thuộc mạnh.
Audit có 992679 dòng, 9993 quỹ đạo; loại 383 bản trùng còn 9610 quỹ đạo trước khi chia.
Tương quan `Acc_LV` với đạo hàm tốc độ chỉ 0.045354, so với 0.929087 ở xe theo sau, nên không dùng `Acc_LV` để khai thác phanh.
Thay bằng `Speed_LV` làm mượt trung bình trượt 5 mẫu rồi lấy gradient theo bước thời gian 0.1 s, tránh khuếch đại nhiễu khi đạo hàm tốc độ thô.
Chia theo quỹ đạo với seed 8554: 6727 train và 2883 test, không chia ngẫu nhiên theo frame; nếu không, những cửa sổ gần như giống nhau có thể xuất hiện ở cả hai tập.
Nguồn: [data_audit.json](evidence/data_audit.json), [report_details.json](evidence/report_details.json).

## Câu 2 · Phát lại

Hai lần chạy cùng mã giống hệt ở cả 600 đoạn. Refactor float32 khác giá trị chính xác ở 600 đoạn nhưng 0 đoạn vượt dung sai; sai khác lớn nhất 0.00009733 m/s².
Dung sai kiểm tra là `atol + rtol * abs(golden)` với `atol = rtol = 0.001`; cộng số thực khác thứ tự hoặc đổi độ chính xác có thể đổi chữ số cuối mà chưa đổi hành vi đáng kể.
AEB làm lệch 12/600 đoạn. Đã mở đoạn 57: frame 98–101, TTC lần lượt 1.4743, 1.4333, 1.4311, 1.4896 s, đều dưới ngưỡng 1.5 s.
Ví dụ frame 98, gap 4.3033 m, tốc độ tiến gần 2.9189 m/s; AEB đổi lệnh từ -5.4363 thành -6.0 m/s². Đoạn khác không vượt ngưỡng hoặc IDM đã phanh ít nhất mạnh bằng AEB nên không đổi lệnh.
Fastest có 0 khung trễ, 0 khung rớt; realtime có 585 trễ và 583 rớt trong 60009 khung, làm đổi 374 đoạn. Realtime giữ lệnh cũ khi trễ/rớt nên lộ hậu quả deadline; chi phí xử lý ở đây là mô phỏng, không phải benchmark thời gian thật của máy.
Nguồn: [replay.json](evidence/replay.json), [replay_inspection.json](evidence/replay_inspection.json).

## Câu 3 · Kịch bản

Lưới có 60 khoảng cách × 45 tốc độ tiến gần = 2700 ô; cruise 14 m/s, gap 2.760–59.312 m, tốc độ tiến gần 0–14 m/s.
Biên bất khả thi là `gap < dv * t_react + dv² / (2 * b_max)`, với `b_max = 9 m/s²`: xe mất khoảng cách trong thời gian trễ rồi cần thêm quãng đường để triệt tiêu tốc độ tiến gần.
Tỷ lệ ô bất khả thi tăng từ 3.630% lên 10.222% khi trễ tăng từ 0 lên 0.6 s; vùng đâm IDM cũng tăng như vậy, thêm 6.593 điểm phần trăm.
Kiểm chứng phanh cực đại có 0 ô sai ngoài vùng cách biên ít nhất 5% ở cả hai độ trễ. Va chạm trong vùng bất khả thi không thể quy hết cho bộ điều khiển; cần xét phần tránh được nhưng vẫn đâm.
Khai thác được 614 sự kiện phanh (6.389% quỹ đạo), chủ yếu ở tốc độ tiến gần thấp và khoảng cách còn dương như các chấm trắng trong hình. Mức phanh người lái p99.9 là 3.625 m/s²; 12.667% lưới đòi hỏi mạnh hơn. Đây là phân vị, không phải cực đại tuyệt đối; log thiếu độ phủ các ca khó.
Nguồn: [scenarios.json](evidence/scenarios.json), hình [scenarios_idm.png](evidence/scenarios_idm.png).

## Câu 4 · Vòng hở và vòng kín

| Bộ điều khiển | L2 trung bình (m) | Hạng L2 | Va chạm trung bình (%) | Hạng va chạm | Dải va chạm (%) |
|---|---|---|---|---|---|
| const-vel | 1.277 | 3 | 20.916 | 4 | 20.916–20.916 |
| IDM | 1.567 | 4 | 0.035 | 1 | 0.035–0.035 |
| EgoMLP | 0.771 | 2 | 19.896 | 3 | 17.725–23.517 |
| PercMLP | 0.633 | 1 | 1.991 | 2 | 0.555–3.989 |

Các cặp đảo thứ tự: const-vel / IDM; IDM / EgoMLP; IDM / PercMLP. Hạng vòng kín ở đây dùng trung bình các seed như `ranking` trong evidence; cổng lại xét seed xấu nhất.
IDM không tối ưu để bắt chước vị trí người lái và giả định xe dẫn giữ tốc độ khi dự báo, nên L2 1.567 m không tốt bằng mô hình học; khi điều khiển, IDM cập nhật khoảng cách và tốc độ tương đối liên tục để phanh.
EgoMLP đạt L2 0.771 m, tốt hơn const-vel 1.277 m, nhưng không quan sát gap hoặc tốc độ xe dẫn; lịch sử tốc độ riêng không đủ nhận biết xe trước đang phanh.
Vòng hở cho mô hình lịch sử thật ở mỗi cửa sổ, còn vòng kín đưa hậu quả hành động trước vào trạng thái sau; sai số nhỏ tích luỹ có thể gây va chạm. Lát cắt sắp phanh chỉ chiếm 4.369% cửa sổ; L2 lát cắt của PercMLP 0.832 m so với toàn bộ 0.633 m cho thấy trung bình che lỗi.
Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), hình [openloop_vs_closedloop.png](evidence/openloop_vs_closedloop.png).

## Câu 5 · Một lần chạy chưa đủ

| Seed EgoMLP | Va chạm / số quỹ đạo | Tỷ lệ (%) | Wilson CI95 (%) |
|---|---|---|---|
| 7588 | 541/2883 | 18.765 | [17.382; 20.232] |
| 7589 | 563/2883 | 19.528 | [18.122; 21.015] |
| 7590 | 511/2883 | 17.725 | [16.374; 19.161] |
| 7591 | 678/2883 | 23.517 | [22.005; 25.100] |
| 7592 | 575/2883 | 19.945 | [18.526; 21.443] |

Trên cùng 2883 quỹ đạo test, EgoMLP dao động 17.725–23.517%, rộng 5.793 điểm phần trăm do seed huấn luyện.
Seed tốt nhất 7590 có CI95 [16.374; 19.161]%; seed xấu nhất 7591 có CI95 [22.005; 25.100]%; hai khoảng không chồng nhau.
Nếu chỉ chạy seed tốt có thể báo 17.725%, còn seed xấu cho 23.517%; chọn một lần thuận lợi sẽ che độ bất ổn. Cổng dùng cận trên Wilson của seed xấu và giới hạn độ dao động 1 điểm phần trăm.
Wilson mô tả bất định tỷ lệ trên tập quỹ đạo dưới giả định lấy mẫu phù hợp, không thay thế biến thiên do huấn luyện. Các seed dùng chung test nên không được cộng mẫu như các quan sát độc lập; CI chồng nhau cũng không tự chứng minh hai seed tương đương.
Nguồn: [closedloop.json](evidence/closedloop.json), mục `logs.EgoMLP.per_seed`.

## Câu 6 · Tác nhân phản ứng

Hai thế giới dưới đây cùng độ trễ ego 0.6 s; mỗi tỷ lệ là phần trăm diện tích lưới, không phải xác suất tai nạn ngoài đời.

| Bộ điều khiển | Bất kỳ va chạm khi xe sau phát lại (%) | Bất kỳ va chạm khi xe sau phản ứng (%) | Va chạm sau khi phát lại (%) | Va chạm sau khi phản ứng (%) |
|---|---|---|---|---|
| const-vel | 80.630 | 80.630 | 0.000 | 0.000 |
| IDM | 80.852 | 10.222 | 70.630 | 0.000 |
| EgoMLP | 82.222 | 82.222 | 0.000 | 0.000 |
| PercMLP | 75.630 | 43.593 | 32.519 | 0.000 |
| PercMLP+ | 80.370 | 15.111 | 69.815 | 0.000 |

Hiệu chỉnh trên tối đa 3000 quỹ đạo train chọn `T = 1.4` s; median headway của người lái 1.855 s, W1 của IDM hiệu chỉnh 0.343, nhỏ hơn tác nhân không phản ứng 0.883. W1 tốt hơn chỉ nói phân bố headway gần hơn, chưa chứng minh hành vi người lái đầy đủ.
Khoảng cách xe sau ban đầu 21.600 m, trễ phản ứng xe sau 0.6 s. Khi xe sau chỉ phát lại tốc độ cũ, nó không biết ego vừa phanh tránh xe cắt vào, nên IDM bị tính 70.630% va chạm phía sau so với 0.000% khi xe sau phản ứng.
Trong thế giới phát lại, IDM có tổng va chạm 80.852% so với const-vel 80.630%; khi xe sau phản ứng, các số tương ứng là 10.222% và 80.630%. Do đó xếp hạng chịu ảnh hưởng của mô hình tác nhân, không thể quy mọi cú đâm sau giả định này thành lỗi ego.
Nguồn: [agents.json](evidence/agents.json), mục `calibration` và `rear_comparison_t_react_0.6`. Với mô hình học, bảng là seed có tổng va chạm reactive lớn nhất, không phải trung bình mọi seed.

## Câu 7 · Cổng phát hành và sửa

| Ứng viên | Quyết định | Quy tắc rớt |
|---|---|---|
| const-vel | chặn | R2, R3, R4, R6 |
| EgoMLP | chặn | R2, R3, R4, R5, R6 |
| PercMLP | chặn | R2, R3, R4, R5, R6 |
| PercMLP+ | chặn | R4, R6 |

PercMLP có cận trên Wilson log 4.767% so với ngưỡng IDM 1.035%; phần lưới tránh được nhưng vẫn đâm 33.370% so với ngưỡng 2.000%; tổng va chạm reactive 43.593% so với ngưỡng 12.222%.
R2 phản ánh sai số tích luỹ khi tự lái, R3 cho thấy lỗi tập trung ở quỹ đạo xe dẫn phanh (tỷ lệ 15.957%, cận trên 21.866%, ngưỡng 5% vì IDM không đâm ở lát cắt này), còn R4/R6 làm lộ thiếu kịch bản cắt vào trong dữ liệu bắt chước. R5 rớt vì dải 0.555–3.989% rộng hơn 1 điểm phần trăm; không chỉ cần giảm L2 trên các trạng thái người lái từng đi qua.
Sửa bằng 20000 kịch bản cắt vào có nhãn IDM và 2 vòng DAgger cho mỗi seed: cho mô hình lái trên train, lấy trạng thái nó tự đi vào rồi dự báo IDM 3 giây ở đúng trạng thái đó làm nhãn. Không dùng test để tạo dữ liệu huấn luyện.
Các quy tắc chuyển từ rớt sang qua: R2, R3, R5. Sau sửa, dải va chạm log là 0.139–0.208%, cận trên Wilson seed xấu 0.453%, diện tích lỗi quy trách nhiệm trên lưới còn 4.889%.
L2 vòng hở trung bình đổi từ 0.633 thành 0.815 m; điểm bắt chước có thể xấu đi khi khả năng phục hồi và tránh va chạm tốt hơn. PercMLP+ chưa qua cổng hiện tại. Cụ thể R4 có diện tích lỗi 4.889% so với ngưỡng 2.000%; R6 có tổng va chạm reactive 15.111% so với ngưỡng 12.222%. Kết quả thực nghiệm này được giữ nguyên dù hướng dẫn mô tả trường hợp qua cổng; không chỉnh ngưỡng hoặc số bằng tay. Mô hình học nhãn IDM nên cũng chưa có bằng chứng nó tốt hơn IDM.
Ở cruise giữ lại 20 m/s, ngoài miền dữ liệu tổng hợp ≤16 m/s, diện tích lỗi là IDM 0.000–0.000%, PercMLP 22.556–57.963%, PercMLP+ 0.185–4.407%. Tôi chưa thay IDM trong hệ thống thật; cần mở rộng miền tốc độ, tác nhân và dữ liệu độc lập, rồi đánh giá shadow trước khi quyết định.
Nguồn: [gate.json](evidence/gate.json), [fix.json](evidence/fix.json), [closedloop.json](evidence/closedloop.json), [openloop.json](evidence/openloop.json).

## Câu 8 · Bài lab này không chứng minh được gì

Thứ nhất, mô phỏng chỉ có chuyển động dọc 1 chiều với bước 0.1 s, giới hạn phanh 9 m/s² và giả định vật lý cố định; chưa kiểm tra đổi làn, cua, mặt đường ướt hoặc sai số động lực học, nên cần simulator phong phú và thử nghiệm có kiểm soát.
Thứ hai, thế giới log phát lại không phản ứng với ego, còn xe sau reactive chỉ là IDM với `T = 1.4` s; đổi tác nhân đã đổi tỷ lệ va chạm, nên cần nhiều kiểu tài xế và tương tác hai chiều trước khi suy luận an toàn thực tế.
Thứ ba, lưới 2700 ô và 614 sự kiện phanh không đại diện phân bố giao thông; 12.667% lưới đã vượt mức phanh người lái p99.9, và miền huấn luyện tổng hợp chỉ tới 16 m/s. Cần thu thêm tình huống hiếm, miền tốc độ cao và một tập đánh giá ngoài nguồn hiện tại.
Thứ tư, 5 seed và 2883 quỹ đạo test chưa tạo bảo đảm an toàn ở mức rủi ro rất nhỏ; không có va chạm quan sát vẫn có cận trên Wilson dương. Cần tăng dữ liệu độc lập và báo bất định, không diễn giải qua cổng thành chứng nhận an toàn.
Thứ năm, nhiễu quan sát chỉ được mô phỏng với độ lệch chuẩn gap 1 m và tốc độ xe dẫn 0.5 m/s; deadline replay cũng là chi phí giả lập, chưa gồm camera, mất cảm biến hoặc phần cứng điều khiển thật. Cần kiểm thử toàn bộ pipeline perception và độ trễ trên thiết bị đích.
Nguồn tham chiếu: [scenarios.json](evidence/scenarios.json), [agents.json](evidence/agents.json), [closedloop.json](evidence/closedloop.json), [fix.json](evidence/fix.json), [replay.json](evidence/replay.json). Cổng là quyết định kiểm thử nội bộ trong mô hình giả định, chưa đủ để phát hành xe tự hành.

## Tái lập và kiểm tra

Chạy trong thư mục repo trên PowerShell:

```powershell
$env:PYTHONUTF8 = "1"
$env:OPENBLAS_NUM_THREADS = "1"
$env:OMP_NUM_THREADS = "1"
$env:MKL_NUM_THREADS = "1"
python -m pip install -r requirements.txt
python -m unittest discover -s tests -v
python run_all.py --student-id 2A202602853
python scripts/check_submission.py --save-verify
```

Bộ kiểm thử gốc có 25 test. Nhật ký chạy đầy đủ ở [run.log](evidence/run.log), kết quả kiểm tra nộp bài ở [verify.txt](evidence/verify.txt).
Các model được sinh trong `work/models/`; `data/` và `work/` không commit. Notebook Colab giữ nguyên bộ khung vì lần thực nghiệm này chạy local và quy định chỉ cho sửa 10 hàm TODO cùng báo cáo.
Lịch sử được chia thành 5 checkpoint: replay và kịch bản; L2 và Wilson; gate và DAgger; bằng chứng dữ liệu và replay; kết quả đầy đủ và báo cáo.
Repo nộp bài: [namhv521/K4-Track4-Replay-CloseOpenLoop](https://github.com/namhv521/K4-Track4-Replay-CloseOpenLoop). Remote `origin` được đặt về repo học viên thay cho repo lớp.
