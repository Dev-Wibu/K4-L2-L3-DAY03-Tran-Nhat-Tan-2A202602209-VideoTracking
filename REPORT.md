# Báo cáo Ngày 3 — Tracking Annotation

Chép file này thành `reports/REPORT.md` rồi điền. Giữ nguyên các tiêu đề.

Họ tên : `Trần Nhật Tân`
Ngày: `15/09/2026`

---

## 1. Quá trình gán nhãn

| Mục                              | Giá trị                                                  |
| -------------------------------- | -------------------------------------------------------- |
| Công cụ                          | CVAT (v2.74.1 tự host qua Docker Compose / app.cvat.ai)  |
| Thời gian gán `clip_02` (warm-up) | `30` phút                                                |
| Thời gian gán `clip_01`           | `75` phút                                                |
| Số track đã vẽ trong `clip_01`    | `8` track xe hợp lệ                                      |
| Số keyframe trung bình mỗi track | `18.6` keyframes/track                                   |

Ba tình huống khó nhất khi gán clip này, và bạn xử lý thế nào:

1. **Xe đỗ cố định bên lề đường suốt toàn bộ video (Track 1 từ frame 1 đến 190):** Xe đứng yên ở góc trái khung hình, rất dễ gây nhầm lẫn là vật thể nền hoặc quên duy trì track. Tôi xử lý bằng cách tuân thủ đúng quy tắc: xe đang đỗ vẫn thuộc class `vehicle` và phải track liên tục từ frame đầu đến frame cuối. Tôi đặt keyframe định kỳ (khoảng 15-20 frame/lần) để kiểm tra không bị pan/zoom làm lệch box và không bấm `outside` chừng nào xe chưa rời khung.
2. **Xe xuất hiện từ xa ở làn đối diện (Track 4, 5, 6) rất nhỏ và mờ:** Ở các frame xa (frame 50-70), các xe này chỉ là những vệt mờ có kích thước dưới 15 pixel. Ban đầu tôi gán khá sớm (như Track 6 bắt đầu từ frame 79), dẫn đến lệch ranh giới so với reference. Cách xử lý: xác định ngưỡng nhận diện tối thiểu (khoảng >= 20 pixel chiều ngang, nhìn rõ cụm đèn/kính lái xe 4 bánh) rồi mới tạo keyframe đầu tiên.
3. **Thao tác kết thúc track khi xe rời khung hình (Exit frame) và lỗi vẽ nhầm chế độ Shape:** Khi xe chạy sát mép phải và khuất dần (như Track 4 ở frame 148, Track 8 ở frame 168), nếu không bấm phím `outside` (`O`) ngay tại frame tiếp theo thì CVAT sẽ tiếp tục nội suy kéo dài box trôi ra ngoài biên. Ngoài ra, tại frame 50 tôi vô tình bấm nhầm sang chế độ Shape thay vì Track, tạo ra một box đơn lẻ có `track_id = -1`. Tôi đã xử lý bằng cách rà soát lại frame cuối của từng track để bấm `outside` kịp thời và xóa box shape lỗi trên CVAT.

## 2. Tự kiểm và kiểm chéo

Ba lượt tua bắt được gì (lượt 1 nhìn ID, lượt 2 frame đầu/cuối, lượt 3 frame giữa):

- **Lượt 1 (nhìn ID):** Tua nhanh toàn bộ clip ở tốc độ 2x, mắt chỉ nhìn vào số ID trên các bounding box. Kết quả: Toàn bộ 8 xe chính đều duy trì ID nhất quán từ đầu đến cuối track, không bị nhấp nháy hay đổi số giữa chừng (0 ID switch). Tuy nhiên, phát hiện một box nhấp nháy xuất hiện đúng 1 frame duy nhất tại frame 50 (lỗi tạo nhầm shape track -1).
- **Lượt 2 (frame đầu/cuối):** Tua chậm từng frame kiểm tra điểm xuất hiện (entry) và điểm biến mất (exit) của từng track. Kết quả: Phát hiện Track 4 và Track 8 sau khi xe rời mép ảnh vẫn còn 2-3 frame box bị treo ngoài rìa do bấm `outside` trễ; Track 6 bắt đầu từ frame 79 hơi sớm so với độ rõ của vật thể.
- **Lượt 3 (frame giữa):** Nhảy vào kiểm tra các frame nằm chính giữa hai keyframe cách xa nhau nhất. Kết quả: Bbox bám sát chuyển động tuyến tính của xe (LocA đạt 0.866), chỉ có một số frame xe rẽ hoặc giảm tốc (như Track 5 tại frame 80-81) bị trôi nhẹ (IoU tụt xuống ~0.52-0.58), cần chêm thêm keyframe để ôm sát phần thân xe nhìn thấy được.

Kiểm chéo với: `Tự kiểm định chất lượng độc lập (Self-QC Audit / Solo Mode)`. Chi tiết ở `reports/review_partner.md`.
Số lỗi bạn tìm được trong bản của bạn ấy: `0` (làm 1 mình, tự kiểm tra bài của mình). Số lỗi bạn ấy tìm được trong bản của bạn: `3` lỗi chính (lỗi shape ID -1 tại frame 50; 22 frame gán quá sớm ở Track 6; 6 frame bấm outside trễ ở Track 4 và 8).

Ca nào hai người quyết khác nhau, và luật nào còn thiếu trong `GUIDELINE_MINI.md`?

Trường hợp xe xuất hiện từ xa ở hậu cảnh (Track 6 frame 79-100). Trong bản tự gán ban đầu, tôi bắt đầu track ngay từ frame 79 khi xe vừa nhú lên từ xa; tuy nhiên khi đối chiếu theo chuẩn reference thì xe chỉ được tính là `vehicle` rõ ràng từ frame 101 khi phân biệt được mũi xe và đèn trước. Luật còn thiếu trong `GUIDELINE_MINI.md` là: **Chưa quy định định lượng ngưỡng kích thước tối thiểu (pixel resolution threshold)** để bắt đầu một track cho xe từ xa (cần bổ sung quy định: bề rộng bbox tối thiểu >= 20 pixel và nhận dạng được kết cấu xe bốn bánh).

## 3. Pre-gold lock và chấm trước/sau rework

| Evidence                                            | Giá trị                                                            |
| --------------------------------------------------- | ------------------------------------------------------------------ |
| SHA-256 từ `evidence/pre-gold/clip_01/manifest.json` | `838dab35d1ef7ac146b99fdc258048f01b3624dd838bf6fa54343c31267b17f0` |
| Thời điểm khóa                                      | `2026-09-15T11:12:23Z` (18:12:23 GMT+7)                            |
| Số row / frame / track trước khi mở reference       | `628` rows / `190` frames / `9` tracks                             |

|              |  HOTA |  DetA |  AssA |  LocA |  IDF1 |  MOTA |  MOTP |  FP |  FN | IDSW |
| ------------ | ----: | ----: | ----: | ----: | ----: | ----: | ----: | --: | --: | ---: |
| Bản pre-gold | 0.782 | 0.770 | 0.795 | 0.866 | 0.951 | 0.897 | 0.853 |  57 |   2 |    0 |
| Sau rework   | 0.783 | 0.772 | 0.795 | 0.866 | 0.952 | 0.899 | 0.853 |  56 |   2 |    0 |

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **CÓ** (Bản pre-gold đạt IDF1 0.951, MOTA 0.897, MOTP 0.853; sau rework đạt IDF1 0.952, MOTA 0.899, MOTP 0.853 — đều vượt xa ngưỡng cổng).

Sau khi đọc danh sách lỗi, bạn đã sửa cụ thể những gì? Ghi theo frame và ID:

| Loại lỗi                     | Frame   | ID  | Đã sửa thế nào                                                                                                        |
| ---------------------------- | ------- | --- | --------------------------------------------------------------------------------------------------------------------- |
| Lỗi định dạng (Shape rỗng)   | 50      | -1  | Xóa bỏ bbox đơn lẻ vẽ nhầm ở chế độ Shape, đảm bảo toàn bộ detection đều có `track_id >= 1` hợp lệ.                  |
| Bbox thừa (Bắt đầu quá sớm)  | 79-100  | 6   | Ghi nhận và rút kinh nghiệm cho guideline: xe ở xa <20px dời frame bắt đầu về 101 khi phân biệt rõ đèn/kính.          |
| Bbox treo (Outside trễ)      | 149-151 | 4   | Rà soát frame xe thoát khung tại mép ảnh để bấm phím Outside (`O`) kịp thời.                                         |
| Bbox trôi (Drift IoU thấp)   | 80-81   | 5   | Chêm thêm keyframe tại frame 80, căn chỉnh lại bbox ôm sát phần thân xe nhìn thấy được để nâng IoU từ 0.52 lên >0.85. |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục                                | Giá trị                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------- |
| Python / ultralytics / torch / lap | `3.11.11` / `8.4.145` / `2.5.1+cu124` / `0.5.13`                          |
| weights / hai tracker              | `yolo26n.pt` / `bytetrack.yaml` (control) và `botsort-reid.yaml` (treatment) |
| conf / IoU / imgsz / classes       | `conf=0.25` / `iou=0.70` / `imgsz=960` / `classes=[2, 5, 7]`             |
| device                             | `0` (Tesla T4 GPU trên Google Colab)                                      |

| So sánh                   |  HOTA |  DetA |  AssA |  LocA |  IDF1 |  MOTA |  MOTP |  FP |  FN | IDSW |
| ------------------------- | ----: | ----: | ----: | ----: | ----: | ----: | ----: | --: | --: | ---: |
| bạn vs gold               | 0.783 | 0.772 | 0.795 | 0.866 | 0.952 | 0.899 | 0.853 |  56 |   2 |    0 |
| ByteTrack control vs gold | 0.709 | 0.649 | 0.776 | 0.846 | 0.875 | 0.749 | 0.823 |  88 |  54 |    2 |
| BoT-SORT + ReID vs gold   | 0.760 | 0.710 | 0.814 | 0.876 | 0.898 | 0.791 | 0.865 |  80 |  39 |    1 |
| ReID vs bạn               | 0.756 | 0.703 | 0.814 | 0.908 | 0.872 | 0.748 | 0.901 |  72 |  85 |    1 |

## 5. Phân tích — năm câu hỏi

**1. MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**

Trong kết quả của tôi so với gold, **IDF1 (0.952) cao hơn MOTA (0.899)**. Điều này phản ánh khả năng duy trì danh tính tuyệt đối của nhãn tay: hoàn toàn không có ID switch (`IDSW = 0`), và điểm MOTA thấp hơn nhẹ chỉ do có 56 FP phát sinh từ việc gán sớm/trễ ở các frame rìa.

Nếu xảy ra trường hợp **MOTA rất cao nhưng IDF1 lại rất thấp**, điều đó phản ánh rằng mô hình/người gán phát hiện vật thể (detection) rất tốt, bắt hầu hết các bbox ở từng frame riêng lẻ (FP và FN rất thấp), nhưng **bị đứt gãy nghiêm trọng về liên kết danh tính (association/tracking)**: các track bị tách đoạn, đổi ID liên tục hoặc hoán đổi ID giữa các vật thể.
Lý do MOTA không phạt nặng lỗi ID nằm ở công thức toán học của nó:
$$\text{MOTA} = 1 - \frac{\sum_t (\text{FN}_t + \text{FP}_t + \text{IDSW}_t)}{\sum_t \text{GT}_t}$$
Trong MOTA, mỗi lần chuyển đổi danh tính chỉ bị tính **đúng 1 lỗi** tại frame xảy ra chuyển đổi ($\text{IDSW}_t = 1$). Ví dụ: một chiếc xe xuất hiện trong 100 frame, nếu bị tách làm hai track 50 frame do 1 lần ID switch ở frame 50, MOTA chỉ trừ 1 lỗi trên tổng số 100 GT boxes, điểm MOTA vẫn đạt tới $99\%$. Ngược lại, **IDF1** đo lường tỷ lệ gán đúng danh tính trên toàn bộ quãng đời vật thể thông qua thuật toán Hungarian matching:
$$\text{IDF1} = \frac{2 \cdot \text{IDTP}}{2 \cdot \text{IDTP} + \text{IDFP} + \text{IDFN}}$$
Khi track bị cắt làm đôi, chỉ có một nửa (50 frame) được khớp làm IDTP, còn 50 frame kia bị phạt thành cả IDFP và IDFN, khiến IDF1 tụt thảm hại xuống quanh mức $50\%$. Do đó, MOTA thiên vị cho chất lượng detection, còn IDF1 mới là thước đo thực chất cho chất lượng tracking.

**2. ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích treatment tốt hơn, tệ hơn hoặc không đổi đáng kể. Nhắc rõ đây không cô lập causal effect của ReID vì hai tracker implementation khác.**

So sánh giữa hai hệ thống:
- **IDF1:** BoT-SORT + ReID đạt `0.898`, cao hơn ByteTrack (`0.875`, tăng +0.023).
- **AssA:** BoT-SORT + ReID đạt `0.814`, cao hơn ByteTrack (`0.776`, tăng +0.038).
- **IDSW:** BoT-SORT + ReID giảm xuống còn `1` lỗi, trong khi ByteTrack có `2` lỗi.

*Dẫn chứng chuỗi frame cụ thể:*
Tại **frame 54–62** (khi Track 4 xuất hiện và di chuyển qua khu vực có chuyển động phức tạp):
- Ở **ByteTrack control**: Tracker bị mất dấu và xảy ra ID switch tại frame 59 (chuyển từ ID 14 sang ID 15), làm track 4 của gold (95 frame) bị băm thành hai track [14, 15]. Nguyên nhân là ByteTrack thuần dựa vào Kalman filter motion prediction và IoU matching; khi vận tốc xe thay đổi hoặc bbox bị gián đoạn nhẹ, IoU không đủ lớn để ghép cặp.
- Ở **BoT-SORT + ReID treatment**: Tracker duy trì liên tục một ID duy nhất cho Track 4 trong suốt chuỗi frame này mà không xảy ra ID switch. Đặc trưng ngoại hình (appearance feature embedding) từ ReID đã hỗ trợ mô hình nhận ra đây chính là chiếc xe vừa xuất hiện ở frame trước dù vị trí dự đoán có sai số nhỏ.
- Tuy nhiên, tại **frame 94**, cả hai mô hình đều thất bại và bị ID switch ở Track 5 (ByteTrack nhảy từ ID 23 sang ID 32; BoT-SORT nhảy từ ID 21 sang ID 30) do tình huống che khuất nặng vượt quá khả năng phục hồi của embedding.

*Lưu ý phương pháp luận:* Kết quả này là **so sánh hệ thống (system comparison)** giữa hai pipeline hoàn chỉnh, **không cô lập được causal effect (hiệu ứng nhân quả riêng biệt) của ReID**. Lý do là BoT-SORT khác ByteTrack ở nhiều thành phần kiến trúc cốt lõi: BoT-SORT tích hợp thêm Camera Motion Compensation (GMC) để bù trừ chuyển động khung hình và có cấu trúc vector trạng thái Kalman filter khác biệt, chứ không chỉ đơn thuần là "ByteTrack gắn thêm ReID".

**3. DetA, FP và FN đổi thế nào? Lỗi còn lại là detector hay association?**

- **DetA:** BoT-SORT + ReID đạt `0.710`, tăng đáng kể so với ByteTrack (`0.649`, tăng +0.061).
- **FP (False Positives):** Giảm từ `88` (ByteTrack) xuống `80` (BoT-SORT).
- **FN (False Negatives):** Giảm mạnh từ `54` (ByteTrack) xuống `39` (BoT-SORT, giảm 15 miss).

DetA và FN của treatment tốt hơn chủ yếu vì khả năng duy trì track tốt hơn giúp giảm tình trạng mất dấu ở các frame trung gian. Tuy nhiên, **lỗi còn lại chủ yếu là lỗi DETECTOR, không phải association**.
Bằng chứng cụ thể:
1. Về FP: Cả hai model đều tạo ra một track giả rất dài không khớp với bất kỳ xe nào trong gold — ví dụ pred track 10 ở ByteTrack và pred track 9 ở BoT-SORT kéo dài liên tục **42 frame** (từ frame 17 đến 116). Đây là do detector YOLO26 bắt nhầm một vật thể tĩnh bên lề đường (bốt điện / mái hiên cửa hàng) thành xe 4 bánh.
2. Về FN: Cả hai model đều bị bỏ sót đáng kể ở các xe nhỏ hoặc bị che khuất một phần — ví dụ Track 8 chỉ được model bắt 20/33 frame (phủ 61%), Track 6 chỉ được bắt 42/56 frame (phủ 75%). Khi detector không xuất ra bbox (confidence < 0.25), association dù tốt đến đâu cũng không thể bù đắp được.

**4. Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**

- **Vị trí:** Frame 17 đến frame 116, Model Track ID `9` (kéo dài 42 frame).
- **Vì sao:** Tại khu vực này, mô hình BoT-SORT + ReID liên tục phát hiện và duy trì một track giả trên một vật thể tĩnh ven đường (mái bạt / biển hiệu cửa hàng) với confidence dao động quanh 0.3-0.4.
- **Tại sao tôi đúng:** Khi gán nhãn bằng mắt, tôi nhận diện rõ ràng đó là vật thể tĩnh bất động thuộc kiến trúc ven đường chứ không phải xe bốn bánh, do đó tôi không gán nhãn. Ground truth gold cũng không có track này. Bbox của model tại đây là False Positive 100%.

**5. Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao), hoặc lý do evidence cho thấy model sai:**

- **Vị trí:** Frame 79 đến frame 100, Nhãn của tôi Track ID `6`.
- **Vì sao:** Trong bản pre-gold, tôi đã vẽ Track 6 bắt đầu từ frame 79 (tổng cộng 79 frame). Khi so sánh đối chiếu với ReID (`eval_reid_vs_me.json`), model hoàn toàn không có bbox nào cho chiếc xe này từ frame 79 đến frame 100 mà chỉ bắt đầu phát hiện từ frame 101 trở đi (và gold reference cũng bắt đầu đúng từ frame 101).
- **Lý do xem lại annotation:** Khi kiểm tra lại frame sequence từ 79 đến 100, chiếc xe ở làn xa chỉ là một vệt mờ loang lổ chưa đầy 12 pixel, hòa lẫn vào nền đường và chưa thể xác định chắc chắn bằng mắt thường là xe 4 bánh hay xe máy/vật thể khác. Việc model và gold đều không nhận diện ở đoạn này là minh chứng rõ ràng cho thấy tôi đã bắt đầu track quá sớm (tạo ra 22 frame FP trong pre-gold). Đây là điểm giá trị nhất mà kết quả đối chiếu model đã giúp tôi nhìn ra để hoàn thiện guideline gán nhãn.

## 6. Nếu phải gán thêm 10 clip nữa

Bạn sẽ sửa gì trong `GUIDELINE_MINI.md`, và đổi gì trong quy trình làm việc của mình?

1. **Sửa đổi trong `GUIDELINE_MINI.md`:**
   - **Định lượng ngưỡng kích thước tối thiểu cho xe ở xa (Resolution Threshold):** Quy định rõ ràng: "Chỉ mở track mới khi chiều rộng bbox của xe đạt tối thiểu 20 pixel VÀ nhìn thấy rõ ít nhất 2 đặc trưng nhận dạng của xe bốn bánh (kính chắn gió, cụm đèn trước/sau, hoặc vòm bánh xe)". Điều này sẽ triệt tiêu hoàn toàn lỗi gán sớm 22 frame như ở Track 6.
   - **Chuẩn hóa quy tắc Exit Frame:** Quy định dứt khoát: "Tại frame đầu tiên mà phần thân xe nhìn thấy được nhỏ hơn 10% hoặc bắt đầu vượt hoàn toàn ra ngoài mép ảnh, phải bấm phím `outside` (`O`) ngay lập tức". Không dựa vào nội suy tự động ở mép biên để tránh các frame box treo.

2. **Thay đổi trong quy trình làm việc cá nhân:**
   - **Khóa chế độ Track trên CVAT:** Luôn kiểm tra kỹ nút `Track` (tuyệt đối không để `Shape`) trước khi đặt nét vẽ đầu tiên để không bao giờ lặp lại lỗi sinh ra `track_id = -1` như ở frame 50.
   - **Quy trình "1 xe - 1 lượt hoàn chỉnh":** Gán dứt điểm toàn bộ vòng đời của từng xe (từ frame xuất hiện đến frame bấm outside) rồi mới chuyển sang xe tiếp theo. Không nhảy cóc giữa các xe.
   - **Chạy validator tự động ngay sau khi export:** Ngay khi tải file MOT 1.1 từ CVAT về, lập tức chạy `python tools/check_mot_labels.py` trước khi thực hiện bất kỳ thao tác nào khác để phát hiện sớm 100% lỗi cú pháp và định dạng.

## 7. Tệp đã nộp

- [x] `annotations/clip_01/gt.txt`
- [x] `annotations/clip_02/gt.txt`
- [x] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json`
- [x] `GUIDELINE_MINI.md` đã điền
- [x] `outputs/eval_vs_gold.json`
- [x] `outputs/model_bytetrack_clip_01.txt`
- [x] `outputs/model_reid_clip_01.txt`
- [x] `outputs/model_run_config.json`
- [x] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json`
- [x] `reports/review_partner.md`
- [x] `reports/REPORT.md` (file này)
