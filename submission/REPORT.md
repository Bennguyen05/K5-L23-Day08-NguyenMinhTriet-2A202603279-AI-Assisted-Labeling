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
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào): **Điểm cao mà sai**: img_008 #124 `three_wheeler` 0.90 và #125 0.86 là xe đẩy hàng trái cây, không phải xe ba bánh (sai theo CAL-03, bị QC bắt); tuk-tuk bị YOLO26 gán `truck` với điểm cao ở pilot (r01). **Điểm thấp mà đúng**: img_017 #176 `three_wheeler` 0.36 và #177 0.41 là tuk-tuk thật (bị người đứng trước che một phần); img_008 #111 `person` 0.32 là người thật đứng sau xe
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

- Thời gian (từ `time_log.csv`): gán tay 22 phút / 4 ảnh = 5.5 phút/ảnh (46 vật → 0.48 phút/vật) vs duyệt 180 phút / 24 ảnh = 7.5 phút/ảnh (584 vật → 0.31 phút/vật). Theo vật, duyệt nhanh hơn gán tay ~35% ((0.48 − 0.31) / 0.48); theo ảnh thì chậm hơn (7.5 vs 5.5) vì ảnh batch đông gấp đôi gold (24.3 vs 11.5 vật/ảnh). Thiên lệch: tôi gán gold trước nên đã quen kiểu cảnh và guideline khi duyệt batch → thời gian duyệt có lợi thế; ngược lại thời gian duyệt gồm cả sửa 19 box nhỏ sau khi validate.
- Kết luận: **có, nhưng chỉ cho một số lớp**. Đáng dùng cho person (accepted 278/380 = 73%) và car (81%, nhưng phải quét kỹ vật bị che). Motorbike dùng được nếu gộp cụm xa thành ignore_region trước. **Không đáng** cho three_wheeler (accepted 12.5%): sửa / xoá tốn hơn vẽ tay; cần prompt YOLOE tốt hơn hoặc ngưỡng cao hơn. Precision chung 53% nghĩa là gần nửa box máy phải xoá.
- **Ba quy tắc giảm thiên kiến của bạn**:
  1. **Ẩn nhãn máy và quét 15 giây trên mọi ảnh**, đếm nhẩm theo lớp trước khi hiện nhãn; ảnh > 30 vật thì quét theo 4 góc. Kiểm tra: bẫy `missing` phải bắt 3/3 (lần này 2/3).
  2. **Mọi box máy lớp three_wheeler / truck / car phải đếm số bánh trước khi giữ**, không dựa vào `conf`. Kiểm tra: bẫy `wrong_class` 3/3 và tỉ lệ edited/deleted của three_wheeler được ghi lại.
  3. **Chạy `validate_labels.py` sau mỗi ~10 ảnh** (export tạm), sửa hết box < 20 px trước khi tiếp tục. Kiểm tra: 0 cảnh báo khi export cuối.
## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| img_010 (xích lô) | SAM 2.1 base | 5 (4 dương, 1 âm) | Không gồm người đạp, không gồm khách | Mask đủ mui, khung, ghế, 2 bánh; còn vài đốm trên vai áo người đạp | `xichlo_img010.jpg` |
| img_009 (xe máy chở chuối) | SAM 2.1 base | 4 (3 dương, 1 âm) | Không gồm chuối, không gồm túi treo | Click âm lên chuối không đổi kết quả (bản không click âm giống hệt); hụt tay lái, gương; thân xe bị túi che thì mask đứt đoạn | `moto_chuoi_img009.jpg`, `moto_chuoi_img009_nohneg.jpg` |
| img_013 (tuk-tuk) | SAM 2.1 base | 12 (6 dương, 6 âm) | Không gồm tài xế / khách (sau 4 lần chạy) | 1 âm: dính tài xế + khách; 4 âm / 2 dương: mask co còn mui + mảng vỡ; 6 dương / 4 âm: đủ xe nhưng lan sang xe máy đỏ đỗ sát (vi phạm 4.2.4) và dính tay, chân tài xế; lần cuối: hết lan sang xe máy, còn vài đốm ở tay tài xế| `tuktuk_img013.jpg` |

- Đối chiếu guideline mục 4.2: luật "người là vật riêng" và 4.2.4 "vùng chạm thuộc vật phía trước" giúp quyết định click âm lên người lái (img_010, img_013). Luật 4.2.2 về lỗ thủng giúp chấp nhận khoảng hở giữa khung xích lô. **Còn thiếu:** v1.0 không nói mask xe có gồm **hàng hoá / túi buộc trên xe** không (img_009, câu hỏi mở 3), và không nói cách xử lý **người ngồi bên trong xe** (khách trong tuk-tuk) khi họ bị khung xe che một phần. SAM chỉ theo biên nhìn thấy, nên quyết định gồm hay không phải nằm trong guideline, không để model tự quyết.
- *(Tuỳ chọn)* Interactor SAM trong CVAT: không thử (CVAT Docker chưa cài serverless/Nuclio); dùng `sam2_points.py` offline để mô phỏng click.

## 6. Calibration (Part 6)

- Số quyết định: 6 (`calibration_decisions.csv`: 2 `cau_hoi_mo`, 2 `loi_may`, 1 `cau_hoi_mo` về mask, 1 `bay_lot`); quan trọng nhất: **CAL-03** (xe đẩy tay không phải `three_wheeler`): chính quy tắc này làm lộ 3/4 lỗi QC ở img_008, và giải thích vì sao `three_wheeler` chỉ accepted 5/40 box máy.
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: chốt câu hỏi mở 3 (hàng nhô: thêm điều kiện chiều cao, img_009) và 4 (tiêu chí "thấy rõ đầu + vai", img_008); "xe đẩy tay, sạp có bánh" vào phần Không gồm của `three_wheeler` (img_002); mask xe không gồm người ngồi trong khung (img_013); nhắc luật < 20 px (img_018). Câu 1, 2, 5, 6, 7 chưa chốt (chưa đủ ca).
- Nguyên nhân gốc: chủ yếu **guideline mơ hồ** (v1.0 không nói xe đẩy hàng là gì, không nói hàng chồng cao) cộng **máy** (YOLO26 không có lớp ba bánh, YOLOE nhận nhầm xe đẩy); một phần do **tôi** (bẫy lọt ở mép ảnh img_035, giữ box < 20 px ở lần duyệt đầu).

## 7. QC và quyết định (Part 6)

- Mẫu: 6 ảnh (1 ảnh `can_review` + 5 ngẫu nhiên, seed 42), 115 đối tượng; người kiểm: tôi (**hạn chế**: tôi vừa là người duyệt, nên dễ bỏ qua đúng những lỗi mình đã bỏ qua lúc duyệt).
- Lỗi theo **đối tượng**: 4/115 = 3.5% · theo **ảnh**: 2/6 = 33.3% (ngưỡng Release ≤ 2%)
- Top lỗi: **thừa** 4/4: img_008 #124, #125, #110 (xe đẩy hàng gán `three_wheeler`, sai theo CAL-03); img_003 #10 (box motorbike trùng #7, conf 0.16). Không thấy lỗi bỏ sót / sai lớp / sai vị trí trong mẫu.
- **Giới hạn mẫu nhỏ**: 2/6 ảnh lỗi, cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là 70%; ngay cả khi mẫu sạch 0/6, cận trên vẫn ~40% (quy tắc 3/n ≈ 50%), nên ngưỡng 2% không thể chứng minh bằng 6 ảnh.
- Script kiểm được: tên file, lớp hợp lệ, box trong ảnh, box < 20 px, accepted/edited/deleted/added, bắt bẫy. **Không** kiểm được: box có ôm sát vật thật, lớp có đúng thực tế (vd xe đẩy vs ba gác), vật cả tôi lẫn máy cùng bỏ sót ngoài gold, và tôi có thật sự theo checklist không.

**Quyết định: Rework**, ngày 2026-11-17. Lý do (đi theo Hold → Rework → Release): không Hold vì các lỗi tìm thấy đều đã có quy tắc rõ trong v1.1 (CAL-03) và dữ liệu không hỏng; **Rework** vì ảnh lỗi 2/6 = 33.3% > 2% và lỗi mang tính **hệ thống** (cùng loại `thua`, cùng nguồn: box `three_wheeler` của YOLOE trên xe đẩy hàng), không phải lỗi lẻ.
Việc cần làm tiếp: (1) duyệt lại **mọi box `three_wheeler`** trong 24 ảnh batch theo CAL-03, không chỉ ảnh trong mẫu; (2) xoá box trùng conf thấp (< 0.2) chồng lên box cùng lớp; (3) export các ảnh đã sửa → `submission/labels/qc_fixed.json`, chạy lại `build_release.py` và `qc_sample.py --force` cho vòng QC 2; (4) với lô sau, tăng ngưỡng YOLOE `three_wheeler` hoặc thêm prompt âm ("handcart", "pushcart").

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài**: có. Dùng Claude (chat) để: hướng dẫn cài đặt và sửa lỗi môi trường (SSL, Docker, đổi tên ảnh); giải thích các luật của guideline v1.0 khi gặp ca khó; gợi ý tọa độ click SAM; soạn nháp nội dung REPORT, `model_log.csv`, `calibration_decisions.csv`, guideline v1.1; chỉ ra các box đáng nghi trong mẫu QC. Tôi tự gán nhãn, tự duyệt 24 ảnh, tự chạy mọi lệnh, tự xác nhận từng lỗi QC trên ảnh và sửa các nhận xét cho đúng với điều tôi quan sát.
- Gold có bị nhiễm không? Không nhìn pre-label trước khi gán xong gold. Có hỏi AI cách hiểu luật (người trên xe vẽ 2 box; vật che > 80% / quá nhỏ) trong lúc gán, không đưa ảnh gold cho AI.
- Máy / GPU / CVAT dùng: Windows, 16 GB RAM, không GPU (PyTorch 2.15 +cpu), CVAT v2.81.0 Docker (WSL 2, giới hạn 7 GB). Vướng mắc: Python báo `CERTIFICATE_VERIFY_FAILED` khi tải model (sửa bằng `SSL_CERT_FILE` = certifi); tên file zip có dấu cách làm `Expand-Archive` lỗi; 2 ảnh gold upload thành `img_0xx (1).jpg` (sửa `file_name` trong JSON).
- Dùng cache `data/cache/`? Không, tự chạy YOLO26m và YOLOE-26s trên CPU.
- Lỗi / bug script đã gặp: không có lỗi script; `validate_labels.py` chỉ in 12 cảnh báo đầu.
- Một điều tôi làm khác lần sau: quét riêng 4 mép ảnh trước khi hiện nhãn máy, và chạy `validate_labels.py` sau mỗi ~10 ảnh thay vì đợi đến cuối.