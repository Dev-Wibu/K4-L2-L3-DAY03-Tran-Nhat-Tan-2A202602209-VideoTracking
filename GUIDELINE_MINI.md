# Mini annotation guideline — Ngày 3 (tracking)

> Điền file này **trong lúc gán nhãn**, không phải sau khi xong. Mỗi lần bạn dừng
> lại nghĩ "cái này tính sao nhỉ?" thì đó là một dòng phải ghi vào đây.
>
> Đây là tài liệu mà người gán nhãn tiếp theo sẽ đọc để làm giống bạn. Nếu hai
> người trong nhóm gán khác nhau, gần như luôn là vì file này chưa nói rõ — chứ
> không phải vì ai kém.

Nhóm / tên: `Trần Nhật Tân (Làm cá nhân - Solo)`
Clip: `clip_01`, `clip_02`

---

## 1. Phạm vi: gán cái gì, không gán cái gì

Một lớp duy nhất: **`vehicle`** — xe bốn bánh (xe con, van, xe buýt, xe tải).

| Gán | Không gán |
| --- | --- |
| xe con, SUV, taxi, xe bán tải | người đi bộ |
| van, minivan | xe đạp |
| xe buýt, minibus | **xe máy / mô tô** |
| xe tải, xe đầu kéo | xe trong ảnh quảng cáo, trong gương, dưới bóng nước |

Bổ sung của nhóm (nếu có): `Tuyệt đối không gán xe máy và người đi bộ dù xuất hiện rõ trong khung hình vì chúng nằm ngoài schema. Mọi bbox thừa sẽ bị phạt nặng vào False Positive (FP).`

## 2. Luật ID — phần quan trọng nhất

| Tình huống | Luật của nhóm | Vì sao |
| --- | --- | --- |
| Xe bị che một phần rồi hiện lại | giữ nguyên ID nếu bị che **dưới 25 frame** (mặc định của lab: 25 frame = 2 giây @ 12.5 fps) | Duy trì tính liên tục của identity theo thời gian thực (temporal continuity), tránh băm nhỏ track làm tụt IDF1. |
| Xe bị che lâu hơn ngưỡng trên | Mở track mới với ID mới | Quá 2 giây thì độ bất định về chuyển động và danh tính quá lớn, không thể khẳng định chắc chắn là cùng một xe. |
| Xe rời khung hình rồi quay lại | Mặc định: **track mới** | Một khi xe đã vượt ra ngoài biên ảnh thì track cũ chấm dứt tại frame đó; khi vào lại phải cấp ID mới. |
| Hai xe cắt nhau / chồng lên nhau | Mỗi xe giữ nguyên bbox và ID của chính mình, không hoán đổi | Bbox ôm phần nhìn thấy của từng xe; theo dõi quỹ đạo chuyển động mượt mà để chống ID switch. |

## 3. Luật bbox

| Tình huống | Luật của nhóm |
| --- | --- |
| Xe bị cắt bởi rìa ảnh | bbox chạm đúng rìa, không đoán phần ngoài ảnh |
| Xe bị xe khác che một phần | bbox ôm phần **nhìn thấy được** |
| Xe vừa xuất hiện, còn rất nhỏ / rất mờ | bắt đầu track từ frame đầu tiên xác định được là xe bốn bánh; ngưỡng nhóm chọn: **chiều rộng bbox >= 20px và nhận diện được cụm đèn/kính lái/bánh xe** |
| Xe đang đỗ, không di chuyển | Vẫn gán nhãn `vehicle` và duy trì track liên tục từ frame đầu đến frame cuối có mặt trong clip |
| Keyframe đặt dày ở đâu | Đặt dày (2–5 frame/keyframe) tại khúc cua, đổi hướng, tăng/giảm tốc, xuất hiện và biến mất; đặt thưa (15–20 frame) khi xe đi thẳng đều |

## 4. Ít nhất ba ca mơ hồ đã gặp thật

Ghi **frame cụ thể** và **ID cụ thể**, không ghi chung chung.

### Ca 1
- Clip / frame / ID: `clip_01 / frame 1..190 / ID 1`
- Tình huống: Chiếc xe con màu xám đỗ yên bên lề đường bên trái suốt toàn bộ 190 frame của video mà không dịch chuyển.
- Quyết định: Vẽ track liên tục từ frame 1 đến 190, đặt keyframe giãn cách đều đặn để chống trôi pan, không bấm Outside vì xe không rời khung.
- Lý do: Theo mục 1 của GUIDE.md, xe đang đỗ vẫn là `vehicle` và cần theo dõi suốt thời gian nó trong khung để detector và tracker đánh giá đúng độ phủ đối tượng tĩnh.

### Ca 2
- Clip / frame / ID: `clip_01 / frame 79..101 / ID 6`
- Tình huống: Chiếc xe xuất hiện từ rất xa ở làn đường đối diện. Tại frame 79, vật thể chỉ là một chấm sáng mờ <12 pixel, đến frame 101 mới hiện rõ hình khối ca bin và 2 cụm đèn trước.
- Quyết định: Ban đầu vẽ từ frame 79 (gây ra 22 frame FP thừa), sau khi rà soát thống nhất dời frame bắt đầu track về đúng frame 101.
- Lý do: Tránh gán nhãn đoán mò (over-annotation) khi vật thể chưa đạt ngưỡng phân giải tối thiểu để phân biệt giữa xe 4 bánh và các phương tiện khác.

### Ca 3
- Clip / frame / ID: `clip_01 / frame 148..151 / ID 4`
- Tình huống: Chiếc xe di chuyển thoát ra mép phải khung hình. Tại frame 148 chỉ còn một góc cản sau chạm mép ảnh; sang frame 149 xe đã hoàn toàn khuất khỏi tầm nhìn.
- Quyết định: Bấm phím Outside (`O`) ngay tại frame 149 để ngắt track, không kéo dài sang frame 150-151.
- Lý do: Nếu không bấm Outside ngay, CVAT sẽ tự động nội suy vẽ tiếp một bbox lơ lửng ngoài rìa tạo ra "ghost box" (bbox treo).

## 5. Sửa gì sau khi chấm với gold và sau khi kiểm chéo

Luật nào trong file này hoá ra còn thiếu hoặc còn mơ hồ? Viết lại cho rõ:

- **Bổ sung định lượng ngưỡng kích thước tối thiểu khi mở track:** Quy định rõ ràng trong mục 3: Chiều rộng bbox phải đạt ít nhất 20 pixel VÀ nhìn thấy cấu trúc đặc trưng xe (đèn, kính, bánh xe) thì mới bắt đầu track. Không mở track khi xe chỉ là đốm mờ ở chân trời.
- **Quy định dứt khoát về frame bấm Outside:** Khi phần nhìn thấy được của xe nhỏ hơn 10% và bắt đầu rời khỏi rìa ảnh, frame tiếp theo bắt buộc phải bấm phím `outside` (`O`) ngay lập tức, không để box trôi tự do theo interpolation.
- **Quy chuẩn công cụ gán nhãn:** Luôn kiểm tra giao diện CVAT ở chế độ `Rectangle Track`, không sử dụng chế độ `Shape` để tránh lỗi vô tình tạo ra bbox đơn lẻ không có ID hợp lệ (track_id = -1).
