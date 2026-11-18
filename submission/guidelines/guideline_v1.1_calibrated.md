# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603162 - Đặng Minh Huy

> Chỉ viết phần thay đổi so với v1.0. Mỗi ca khó có ảnh ví dụ và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Xe đạp điện / xe máy điện nhỏ: `bicycle` hay `motorbike`? | **Chưa chốt** — 24 ảnh batch + 5 ảnh mẫu QC không xuất hiện xe điện nào phân biệt được; giữ quy tắc tạm: thấy yên to + baga nhựa + không bàn đạp → `motorbike`, ngược lại `bicycle`; cần mẫu thật để xác nhận | (không có) | — |
| 2 | Khối xe máy đỗ dày đặc gần camera: từng xe hay `ignore_region`? | Từng xe riêng nếu phân biệt được ≥ 2 bánh/yên; chỉ gom `ignore_region` khi cụm ≥ 4 xe chồng lấp không đếm được từng chiếc (giữ nguyên ngưỡng vật < 20 px của v1.0) | img_006.jpg (khối xe đỗ trong hẻm, mỗi xe một box 0.3–0.85) | CAL-05 |
| 3 | Hàng nhô ra: box xe có gồm hàng không? | **Không.** Box phương tiện chỉ ôm khung + thùng; giỏ/rổ tre nhô quá ~20% chiều dài thân xe không kéo dài box. Mask SAM cắt theo khung xe (nhất quán mục 4.2 v1.0) | img_003.jpg (xe kéo hàng, sạp chuối hai đầu) | CAL-03 |
| 4 | Ngưỡng "còn thấy ≥ 20%" người ngồi trong xe có mui | Giữ 20% nhưng thêm điều kiện khách quan: thấy được đầu HOẶC thân qua khe mui thì đánh `person`; chỉ thấy tay/chân mờ → không đánh, ghi `can_review` | img_013.jpg (khách ngồi trong tuk-tuk thấy qua khung sắt) | CAL-04 |
| 5 | Kính chắn gió / cửa sổ xe thuộc mask xe? | **Có** — kính là bộ phận xe, mask liền khối qua kính; người sau kính là `person` riêng | img_004.jpg (kính trước tuk-tuk xanh) | CAL-04 |
| 6 | Box máy chồng nhau quá nhiều: gộp hay giữ? | Cùng lớp, IoU ≥ 0.6 → giữ box diện tích lớn hơn, xóa box nhỏ (CAL-01). Khác lớp mà là hai vật thật (hai xe cạnh nhau) → giữ cả hai (CAL-05) | img_003.jpg, img_014.jpg | CAL-01, CAL-05 |
| 7 | Mức tin cậy tối thiểu để chấp nhận box máy không cần sửa | **Không đặt trần cứng** — conf thấp vẫn có thể đúng (img_015 motorbike 0.26–0.28 là xe thật), conf cao vẫn sai lớp (img_002 car 0.89 là tuk-tuk). Checklist 4 bước mới là hàng rào | img_002.jpg | CAL-02 |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| img_003.jpg | Xe kéo hàng 2 bánh + mái che | YOLOE cho 2 box `three_wheeler` 0.91/0.88 chồng nhau | Giữ 1 box lớn, xóa box trùng; xe kéo thuộc `three_wheeler` theo nghĩa "xe ba gác chở hàng" | mục 2.9 (lớp phương tiện) |
| img_031.jpg | Person FP ở giá áo góc trái | YOLO26m `person` 0.33 | Xóa box | mục 4.1.2 (ngưỡng thấy người) |
| img_035.jpg | Xích lô chở khách | YOLOE `three_wheeler` 0.46 chỉ ôm khoang khách | Nới box phủ toàn xe (khung + bánh sau + bàn đạp) | mục 4.2 (ranh giới mask/box xe) |
| img_014.jpg | Hai xe máy sát nhau bị nghi trùng | (người kiểm nhầm, không phải máy) | Giữ cả hai; thêm quy tắc xác minh 2 vật thật trước khi xóa | mục 7 (QC) |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt 6/7 câu mở (câu 1 để ngỏ vì thiếu mẫu); thêm quy tắc chống box trùng cùng lớp (IoU ≥ 0.6), xóa person biên conf < 0.35 không rõ đầu/thân, box xe nguyên chiếc + hàng nhô không tính, xác minh 2 vật trước khi xóa box chồng | CAL-01 → CAL-05 |
