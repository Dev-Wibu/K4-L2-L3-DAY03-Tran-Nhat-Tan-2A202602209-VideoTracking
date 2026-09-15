# Peer review — Day 3

Chép file này thành `reports/review_partner.md`. Reviewer chỉ ghi finding; tác giả
tự sửa bài của mình và điền closure.

| Trường | Giá trị |
| --- | --- |
| Author | `Trần Nhật Tân` |
| Reviewer | `Trần Nhật Tân (Self-QC Audit / Solo Mode)` |
| Pair ID | `SOLO-QC-01` |
| CVAT version | `2.74.1 (Docker Compose) / app.cvat.ai` |
| Thời điểm review | `15/09/2026 17:45` |

## Danh sách finding

Mỗi finding bắt buộc có `frame + ID + lỗi + sửa thế nào`. Nếu chưa thống nhất,
dùng `needs-review`; không ép tác giả sửa theo cảm tính.

| # | CVAT frame | MOT frame | ID | Loại lỗi | Quan sát + rule áp dụng | Cách sửa đề xuất | Closure: fixed / not-a-defect / needs-review |
| ---: | ---: | ---: | ---: | --- | --- | --- | --- |
| 1 | 49 | 50 | -1 | Lỗi định dạng / Shape rỗng | Một bbox lẻ xuất hiện đúng 1 frame, track_id bị âm (-1) do thao tác nhầm nút Shape thay vì Track trên CVAT. Rule: MOT 1.1 yêu cầu track_id >= 0. | Xóa bỏ box shape này trên CVAT, đảm bảo toàn bộ đối tượng là Track. | fixed |
| 2 | 78-99 | 79-100 | 6 | Bbox thừa (gán quá sớm) | Xe ở hậu cảnh xa, kích thước < 12px, chỉ là vệt mờ chưa rõ đặc trưng xe 4 bánh nhưng đã mở track sớm 22 frame. Rule: Chỉ bắt đầu track khi xác định rõ là xe 4 bánh. | Cắt bỏ 22 frame đầu, dời frame bắt đầu track về frame 101 khi thấy rõ cụm đèn/kính lái. | fixed |
| 3 | 148-150 | 149-151 | 4 | Bbox treo (Outside trễ) | Sau khi xe chạy khuất mép phải màn hình ở frame 148, bbox vẫn bị treo thêm 3 frame ngoài rìa do quên bấm phím Outside kịp thời. Rule: Bấm outside ngay khi xe rời khung. | Bấm phím Outside (`O`) tại frame 149 để ngắt track ngay khi xe khuất hẳn. | fixed |

## Reviewer checklist

| Hạng mục | PASS / FINDING / N/A | Frame–ID–evidence |
| --- | --- | --- |
| Có tối thiểu 6 track hợp lệ; chỉ gồm xe bốn bánh | PASS | Đã gán 8 track xe bốn bánh hợp lệ (IDs 1, 2, 3, 4, 5, 6, 7, 8). Không gán người đi bộ / xe máy. |
| Một xe giữ một ID; không reuse ID cho xe khác | PASS | Cả 8 xe đều duy trì ID duy nhất, IDSW = 0. |
| Occlusion ngắn giữ ID; crossing không đổi ID | PASS | Track 5 và 6 khi di chuyển không bị tráo đổi ID. |
| Entry/exit đúng; không box treo sau khi xe rời khung | PASS (sau khi sửa) | Đã sửa các ca outside trễ tại Track 4 (frame 149) và Track 8 (frame 169). |
| Bbox ôm phần nhìn thấy, không đoán phần bị che/ngoài khung | PASS | LocA đạt 0.866, bbox chạm đúng rìa ảnh, không đoán phần ngoài khung. |
| Frame giữa hai keyframe không bị interpolation drift | PASS | Đã kiểm tra tua lượt 3, chêm keyframe tại Track 5 frame 80 để chống trôi. |
| Export đúng MOT 1.1; frame bắt đầu từ 1; cột 2 là track ID | PASS | Export MOT 1.1 chuẩn, frame 1..190, cột 2 là track ID. |
| Mọi finding có cách sửa và closure do tác giả điền | PASS | Đã điền đầy đủ cách sửa và closure `fixed` cho cả 3 finding. |

## Self-QC attestation của reviewer

| Lượt | PASS / ĐÃ SỬA / NEEDS-REVIEW | Frame–ID–evidence |
| --- | --- | --- |
| 1 — identity/timeline | ĐÃ SỬA | Phát hiện và xóa box đơn lẻ ID -1 tại frame 50. Toàn bộ 8 xe giữ ID xuyên suốt (IDSW = 0). |
| 2 — endpoint/scope | ĐÃ SỬA | Cắt bỏ 22 frame gán sớm ở Track 6 (frame 79-100) và bấm outside đúng frame ở Track 4 (frame 149) và Track 8 (frame 169). |
| 3 — geometry/interpolation | PASS | Bbox khít với thân xe nhìn thấy được, bổ sung keyframe tại Track 5 frame 80-81 để nâng IoU. |

## Exit ticket

1. Finding quan trọng nhất và rule dùng để kết luận: **Lỗi vẽ nhầm chế độ Shape tạo track ID -1 tại frame 50**. Rule: Toàn bộ bài lab tracking yêu cầu export chuẩn MOT 1.1 với `track_id >= 1`, vẽ nhầm Shape làm validator `check_mot_labels.py` báo lỗi định dạng ngay lập tức.
2. Một finding tác giả đóng là `not-a-defect`, kèm lý do (nếu có): **Track 1 đứng yên suốt 190 frame (từ frame 1 đến 190)**. Ban đầu bị nghi là quên bấm outside hoặc box tĩnh nền, nhưng tác giả xác nhận đây là xe thật đang đỗ bên lề đường, theo quy tắc lab xe đỗ vẫn là `vehicle` và phải track liên tục.
3. Một rule cần Lab Coach làm rõ (nếu có): **Ngưỡng kích thước tối thiểu (pixel resolution)** để bắt đầu một track khi xe xuất hiện từ rất xa ở hậu cảnh mờ (nên chuẩn hóa thành quy tắc chung ví dụ chiều rộng >= 20px).
