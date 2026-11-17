# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603279 - Nguyễn Minh Triết

> Chỉ ghi phần thay đổi so với v1.0. Mỗi ca khó có ảnh ví dụ và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 3 | Hàng nhô ra: box/mask xe có gồm hàng không? | **Không** gồm hàng nhô quá 20% chiều dài xe **hoặc** cao hơn điểm cao nhất của xe (gương / tay lái). Túi buộc nằm trong phạm vi thân xe thì để trong box (không tách được). Mask: chỉ phần xe nhìn thấy, không gồm hàng. | img_009 | CAL-01 |
| 4 | Người khuất (ngưỡng "≥ 20% nhìn thấy") có hợp lý không? | Ước lượng 20% bằng mắt không nhất quán → thay bằng tiêu chí kiểm được: **thấy rõ đầu (mặt hoặc gáy) kèm vai → gán `person`**; chỉ thấy tay / chân / mảng lưng lẻ không nhận ra người → không gán; ca giữa → `can_review`. | img_008 | CAL-02 |
| 1, 2, 5, 6, 7 | Xe đạp điện; khối xe máy dày gần camera; kính xe; box chồng ảnh đêm; conf tối thiểu | **Chưa chốt**: lô batch không có đủ ca rõ ràng để quyết (ví dụ chưa gặp xe đạp điện nhận ra chắc chắn). Giữ nguyên v1.0, đánh dấu `can_review` khi gặp. | — | — |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| img_002 | Xe đẩy hàng không động cơ | YOLOE-26s gán `three_wheeler` (FP) | Không gán lớp nào | Mục 2, lớp `three_wheeler`, cột "Không gồm": thêm **xe đẩy tay, sạp hàng có bánh** (CAL-03) |
| img_013, img_004 | Tuk-tuk | YOLO26m gán `car` / `truck` ở mọi ngưỡng | `three_wheeler` | Mục 4.3 đã đủ; khi duyệt **đếm bánh mọi box car/truck của máy** (CAL-04) |
| img_013 | Mask tuk-tuk có tài xế + khách ngồi trong khung, xe máy đỏ đỗ sát | SAM 2.1 base: mask dính người, lan sang xe máy | Mask xe không gồm người; tách mask hai xe | Mục 4.2.4: thêm **"người ngồi trong / trên xe là vật riêng, kể cả khi bị khung xe che; SAM không tách được thì sửa polygon tay"** (CAL-05) |
| img_018 | Cụm xe máy, người ở xa (8 box cao 14–19 px) | YOLO26m vẽ từng box nhỏ, tôi giữ ở lần duyệt đầu | Xoá box lẻ, vẽ 1 `ignore_region` | Mục 4.1.6: nhắc **box máy < 20 px phải xoá dù conf cao**; chạy `validate_labels.py` trước khi export |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt câu hỏi mở 3 (hàng nhô: thêm điều kiện chiều cao) và 4 (tiêu chí "thấy rõ đầu + vai"); thêm "xe đẩy tay" vào phần Không gồm của `three_wheeler`; thêm quy tắc mask người ngồi trong xe; nhắc luật < 20 px | CAL-01, CAL-02, CAL-03, CAL-05; img_018 |