# Thông tin học viên

- Họ và tên: Nguyễn Minh Triết
- MSHV: 2A202603279
- Lớp (L2 / L3 / khác): L2
- Tài khoản GitHub: Bennguyen05
- Repo bài nộp (URL): https://github.com/Bennguyen05/K5-L23-Day08-NguyenMinhTriet-2A202603279-AI-Assisted-Labeling>>
- Commit chốt bài (hash): d48dfdaf922ec4fe945069fefac85a63638a4717
- Máy: Windows, 16 GB RAM, không GPU (CPU, PyTorch 2.15 +cpu)
- CVAT (Docker): v2.81.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 451 / 77 / 203 (mẫu số: 731 box máy); added: 56 (mẫu số: 584 đối tượng)
- So với gold: precision / recall của **máy** 53.1% / 93.5% (46 đối tượng gold)
- Bắt bẫy: 11/12 = 91.7%
- Thời gian: gán tay 5.5 phút/ảnh (0.48 phút/vật); duyệt có pre-label 7.5 phút/ảnh (0.31 phút/vật)
- Lỗi QC: 33.3% theo ảnh (2/6), 3.5% theo đối tượng (4/115)
- **Quyết định:** Rework — lý do một dòng: 2/6 ảnh lỗi > 2%, lỗi hệ thống (xe đẩy hàng bị gán three_wheeler theo CAL-03)