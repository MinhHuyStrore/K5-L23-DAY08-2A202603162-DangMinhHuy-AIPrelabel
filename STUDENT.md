# Thông tin học viên

- Họ và tên: Đặng Minh Huy
- MSHV: 2A202603162
- Lớp (L2 / L3 / khác): khác (K5-L23)
- Tài khoản GitHub: MinhHuyStrore
- Repo bài nộp (URL): https://github.com/MinhHuyStrore/K5-L23-DAY08-2A202603162-DangMinhHuy-AIPrelabel
- Commit chốt bài (hash): d7f78d3
- Máy: Windows 10, 16 GB RAM, không GPU NVIDIA (Intel HD 520) — chạy toàn bộ CPU, `--device cpu --imgsz 640`
- CVAT (Docker): cvat/server:dev (nhánh 2.81); server OOM giữa buổi nên phần export được tái lập bằng pipeline script trong repo

## Kết quả tóm tắt

| Hạng mục | Số liệu |
|---|---|
| Gold 4 ảnh | 76 đối tượng, validate OK |
| Tour model (model_log) | 5 dòng, 3 họ model + 2 lượt SAM |
| Ngưỡng theo lớp | Không lớp nào đạt recall 90% ≤ 0.5 → giữ mặc định 0.3 |
| Pre-label gộp | 443 box → tiêm 12 bẫy → duyệt 24 ảnh (391 box) |
| Bắt bẫy | 12/12 = 100% (KTC Wilson 76–100%) |
| Mask SAM | 4 tệp (sam2.1_b vs mobile_sam so sánh trên img_013) |
| QC mẫu | 5 ảnh/78 đối tượng, 3 ảnh lỗi → REWORK → sửa qua qc_fixed.json |
| Release | 24 ảnh, 389 đối tượng (final_det.coco.json) |
