# Báo cáo Day 08 — 2A202603279 - Nguyễn Minh Triết


## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
|| person | YOLO26m | 0.15 | recall 93% (13/14 gold), precision 39%; ngưỡng chung 0.5 làm mất 35% box (r01 → r02) |
| motorbike | YOLO26m | 0.1 | recall 91% (10/11 gold), precision 43%; xe máy nhỏ / ở xa có conf thấp (r02) |
| bicycle | YOLO26m | 0.2 | recall 100% (5/5 gold), precision 83% |
| car | YOLO26m | 0.3 (mặc định) | không ngưỡng nào ≤ 0.5 đạt recall 90% (tối đa 88% = 7/8): 1 xe máy không bắt ở mọi ngưỡng |
| bus / truck | YOLO26m | 0.3 (mặc định) | gold chỉ có 1 bus, 0 truck (< 5 vật): không đủ để chọn ngưỡng riêng |
| three_wheeler | YOLOE-26s, prompt từ `taxonomy.json` (tuk-tuk, xích lô, ba gác) | 0.15 | recall 100% (7/7 gold), precision 78%; YOLO26 gọi tuk-tuk là truck/car (r01, r03) |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên): python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "person=0.15,bicycle=0.2,motorbike=0.1,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json
- **Phiên bản SAM bạn chọn** SAM 2.1 base và lý do: Mask bao trọn người và xe khi box đúng (img_004); 10.5 giây/ảnh trên CPU (r04) chấp nhận được vì Part 5 chỉ cần 3 vật; không cần quyền truy cập như SAM 3; máy không có GPU.
- Lỗi nào **không sửa được bằng đổi ngưỡng** (ví dụ tuk-tuk bị gọi `car`)? Sửa thế nào? YOLO26 không có lớp three_wheeler (COCO không có) nên gọi tuk-tuk là truck / car ở mọi ngưỡng (r01, r02, r04). Sửa bằng YOLOE gõ chữ cho riêng lớp three_wheeler, rồi gộp hai model bằng `merge_prelabels.py` (loại box trùng IoU 0.6).
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào): <<>>
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh) nên số đo máy trên gold là lạc quan: lớp ít vật (bus 1, truck 0, bicycle 5) thì ngưỡng chỉ là ước lượng thô. Ngưỡng thấp (0.1–0.15) cho person / motorbike đổi recall lấy precision < 45%: người duyệt phải xoá nhiều box thừa.
## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 451 | 61.7% | 731 box máy |
| edited | 77 | 10.5% | 731 box máy |
| deleted | 203 | 27.8% | 731 box máy |
| added | 56 | 9.6% | 584 đối tượng sau duyệt |

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)? **three_wheeler**: chỉ accepted 5/40 box máy (12.5%), deleted 26/40 (65%), edited 9. YOLOE bắt nhầm xe đẩy / xe chở hàng không động cơ (giống img_002 ở Part 2) và box lỏng; ngưỡng 0.15 chọn trên gold (7 vật) cho recall cao nhưng thừa nhiều trên batch. Tiếp theo là **motorbike**: accepted 118/240 (49%), deleted 83/240 (35%) do ngưỡng 0.1 và cụm xe máy nhỏ ở xa (img_018: 8 box < 20 px phải gộp thành ignore_region). **car** accepted cao (42/52 = 81%) nhưng máy sót 18 xe (added), thường là xe bị người / xe khác che.
- So với gold: precision / recall của **máy** 53.1% (KTC 95%: 42–64%) / 93.5% (KTC 95%: 82–98%); TP 43, FP 38, FN 3 (mẫu số: 46 đối tượng gold, 4 ảnh). Lạc quan vì ngưỡng chọn trên chính 4 ảnh này.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: 11/12 = 91.7% (KTC 95%: 65–99%); loose 3/3, wrong_class 3/3, extra 3/3, missing 2/3. Lọt: 1 bẫy `missing` ở img_035 (motorbike). Vì sao: bẫy missing là vật **không có box** nên không có gì kéo mắt tới; ở img_035 tôi đã quét nhanh và có nhiều motorbike nên bỏ sót góc. Đây đúng là lỗi khó thấy nhất.
- Accepted 61.7% (không quá cao) cùng bắt bẫy 91.7% cho thấy tôi có sửa và xoá thật chứ không bấm Accept hàng loạt. Nhưng vẫn có dấu hiệu thiên kiến tự động hóa: lần export đầu còn 19 box < 20 px (8 cái ở img_018) mà tôi đã để nguyên vì box máy trông hợp lý; chỉ phát hiện nhờ cảnh báo của `validate_labels.py`, sau đó mới sửa.

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay 22 phút / 4 ảnh = 5.5 phút/ảnh (46 vật → 0.48 phút/vật) vs duyệt 180 phút / 24 ảnh = 7.5 phút/ảnh (584 vật → 0.21 phút/vật). Ảnh batch đông gấp đôi gold (24.3 vs 11.5 vật/ảnh) nên so theo **phút/vật** công bằng hơn. Thiên lệch: tôi gán gold trước nên đã quen kiểu cảnh và guideline khi duyệt batch → thời gian duyệt có lợi thế; ngược lại thời gian duyệt gồm cả sửa 19 box nhỏ sau khi validate.
- Kết luận: **có, nhưng chỉ cho một số lớp**. Đáng dùng cho person (accepted 278/380 = 73%) và car (81%, nhưng phải quét kỹ vật bị che). Motorbike dùng được nếu gộp cụm xa thành ignore_region trước. **Không đáng** cho three_wheeler (accepted 12.5%): sửa / xoá tốn hơn vẽ tay; cần prompt YOLOE tốt hơn hoặc ngưỡng cao hơn. Precision chung 53% nghĩa là gần nửa box máy phải xoá.
- **Ba quy tắc giảm thiên kiến của bạn**:
  1. **Ẩn nhãn máy và quét 15 giây trên mọi ảnh**, đếm nhẩm theo lớp trước khi hiện nhãn; ảnh > 30 vật thì quét theo 4 góc. Kiểm tra: bẫy `missing` phải bắt 3/3 (lần này 2/3).
  2. **Mọi box máy lớp three_wheeler / truck / car phải đếm số bánh trước khi giữ**, không dựa vào `conf`. Kiểm tra: bẫy `wrong_class` 3/3 và tỉ lệ edited/deleted của three_wheeler được ghi lại.
  3. **Chạy `validate_labels.py` sau mỗi ~10 ảnh** (export tạm), sửa hết box < 20 px trước khi tiếp tục. Kiểm tra: 0 cảnh báo khi export cuối.
## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| <<img_0xx>> | <<>> | <<>> | <<>> | <<>> | <<>> |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu? <<>>
- *(Tuỳ chọn)* Interactor SAM trong CVAT: làm được / không (lỗi gì, đã thử gì); khác gì offline: <<>>

## 6. Calibration (Part 6)

- Số quyết định: <<>> (`calibration_decisions.csv`); quan trọng nhất (dẫn `CAL-xx`): <<>>
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: <<>>
- Nguyên nhân gốc: <<guideline mơ hồ / chưa nắm quy tắc / máy>>

## 7. QC và quyết định (Part 6)

- Mẫu: <<n>> ảnh (<<m>> ảnh `can_review` + <<k>> ngẫu nhiên, seed 42); người kiểm: bạn (**hạn chế**: bạn vừa là người duyệt).
- Lỗi theo **đối tượng**: <<.../... = ...%>> · theo **ảnh**: <<.../... = ...%>> (ngưỡng Release ≤ 2%)
- Top lỗi (trong: bỏ sót, thừa, sai lớp, sai vị trí/kích thước, sai thuộc tính, sai mask) + một ảnh ví dụ mỗi loại: <<>>
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): <<>>
- Script kiểm được / không kiểm được (ví dụ ôm sát, cùng bỏ sót với máy, theo checklist thật): <<>>

**Quyết định: <<Release / Hold / Rework>>**, ngày <<YYYY-MM-DD>>. Lý do (đi theo Hold → Rework → Release): <<>>
Nếu Rework / Hold, việc cần làm tiếp: <<>>

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào; "không dùng" nếu không): <<>>
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)? <<không / có, như sau: ...>>
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: <<>>
- Dùng cache `data/cache/` thay vì tự chạy? <<không / có, vì ...>>
- Lỗi / bug script đã gặp (nếu có): <<>>
- Một điều bạn làm khác lần sau: <<>>
