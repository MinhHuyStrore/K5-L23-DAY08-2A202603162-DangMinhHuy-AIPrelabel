# Báo cáo Day 08 — 2A202603162 - Đặng Minh Huy

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26m (`--size m`) | mặc định 0.3 khi gộp | r01/r02: bắt chắc person/motorbike/car; `threshold_sweep.md` cho thấy không lớp nào đạt recall ≥ 90% ở ngưỡng ≤ 0.5 (person tối đa 81% với 31 vật, motorbike 75% với 20 vật, car 89% với 9 vật) → theo quy tắc mẫu nhỏ, giữ mặc định |
| three_wheeler | YOLOE-26s, prompt: `tuk-tuk`, `cyclo`, `cargo tricycle` (`--classes three_wheeler`) | 0.3 (giới hạn `--max-thr 0.5` không cho phép cao hơn vì recall tốt nhất chỉ 86% với 7 vật gold) | r03: YOLO26 hoàn toàn không có khái niệm này (tuk-tuk img_002 bị gọi `car` 0.89), YOLOE bắt đúng 0.97 |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên):
  `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
  (YOLOE đứng trước để box `three_wheeler` loại box `car`/`truck` nhận nhầm trên cùng chiếc tuk-tuk; không truyền `--thr` vì sweep không chốt được ngưỡng lớp nào — chuỗi gợi ý là `--thr ""`.)
- **Phiên bản SAM bạn chọn**: SAM 2.1 base (`sam2.1_b.pt`) cho mask chính và MobileSAM (`mobile_sam.pt`) để đối chiếu. Lý do: máy không có GPU (Intel HD 520, 16 GB RAM) — SAM 2.1 base cho viền tốt nhất trong các bản đã thử với ~55 s/ảnh khi kèm detector, MobileSAM nhanh hơn hẳn trên CPU để thử nghiệm tọa độ; SAM 3 cần token HuggingFace nên bỏ qua.
- Lỗi nào **không sửa được bằng đổi ngưỡng**: tuk-tuk img_002 bị YOLO26 gọi `car` với conf 0.89 — nâng ngưỡng từ 0.25 lên 0.5 (r01→r02) box sai vẫn còn nguyên vì nó điểm cao. Sửa bằng cách nạp song song YOLOE-26s (`--classes three_wheeler`) rồi gộp có ưu tiên (YOLOE thắng khi IoU ≥ 0.6), và duyệt tay các ca còn lại.
- Box **điểm cao mà sai**: img_002 `car` 0.89 (thực chất tuk-tuk), img_031 `person` 0.33 lại là giá áo (điểm thấp mà đúng thì có: img_015 `motorbike` 0.26–0.28 là xe thật ở rìa, r01).
- Giới hạn: chọn/chấm ngưỡng trên chính 4 ảnh gold nên precision/recall của máy trên gold là **lạc quan** (in-sample); lớp bus chỉ 1 vật, truck 0 vật trong gold — script đã tự từ chối chọn ngưỡng riêng. Toàn bộ lượt chạy dùng `--device cpu --imgsz 640` (giảm imgsz so với 1024 mặc định) nên recall vật nhỏ thấp hơn cấu hình chuẩn trong walkthrough.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 379 | 100.0% | 379 box máy (đã loại 24 đối tượng quanh 12 bẫy) |
| edited | 0 | 0% | 379 |
| deleted | 0 | 0% | 379 |
| added | 0 | 0% | 379 đối tượng sau duyệt |

- Lớp máy tệ nhất: **three_wheeler** — YOLO26 không có lớp này (img_002, img_013 gọi nhầm `car`/`truck`), YOLOE bắt được nhưng box lỏng và sinh box trùng (img_003 hai box 0.91/0.88 trên cùng một xe kéo). Kế đến là `person` nền xa: conf 0.25 sinh hàng loạt box 0.25–0.35 lẫn với sạp hàng (img_031 giá áo bị gọi person 0.33).
- So với gold (máy, tức `pre_traps` sau trừ bẫy): **Precision 76.9%** (40 TP / 12 FP, KTC 95% 64–86%), **Recall 53.3%** (35 FN, KTC 95% 42–64%), mẫu số 75 đối tượng gold trên 4 ảnh. Recall thấp một phần do `--imgsz 640` trên CPU làm rơi vật nhỏ.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: **12/12 = 100%** (KTC 95% Wilson 76–100%; ngưỡng đạt 80%); không loại nào lọt (loose 3/3, extra 3/3, missing 3/3, wrong_class 3/3).
- Accepted 100% đi kèm bắt bẫy 100% **không** chứng minh người duyệt tỉnh táo: hai con số đo trên tập khác nhau (bẫy bị loại khỏi bảng accepted). Cách duyệt của tôi là duyệt đối chiếu máy-sạch (diff lô gộp chuẩn với lô đã tiêm bẫy) — phát hiện mọi thay đổi nhân tạo nhưng sẽ KHÔNG phát hiện lỗi hệ thống mà cả hai bản cùng sai (vd. box trùng #1/#2 img_003 chỉ lộ ra ở bước soi ID của QC mẫu). Đó chính là lý do Pha 6 vẫn tìm ra 3 lỗi thật dù tỉ lệ bắt bẫy tuyệt đối.

## 4. Máy có đáng dùng không?

- Thời gian (`time_log.csv`): gán tay gold 30 phút / 4 ảnh = **7.5 phút/ảnh**; duyệt 24 ảnh batch 55 phút = **2.3 phút/ảnh** (≈ 3.3× nhanh hơn). Thiên lệch: tôi đã gán gold trước nên quen cảnh 4 ảnh đó, và lượt duyệt có bản đồ diff hỗ trợ — con số thật khi mù hoàn toàn sẽ cao hơn.
- Kết luận: **có, nhưng chỉ cho lớp phổ biến** (person/motorbike/car — accepted 100% ngoài tập bẫy); three_wheeler bắt buộc ghép YOLOE + duyệt tay; vật < 20 px và người biên conf thấp máy không đáng tin.
- **Ba quy tắc giảm thiên kiến của bạn**:
  1. Quét đếm người/xe trước khi nhìn khung màu (nút ẩn nhãn ở bảng Objects) — kiểm được bằng ghi chú `quét_10s/ảnh` trong `time_log.csv`, mỗi ảnh ít nhất 10 giây trước khi bật nhãn máy.
  2. Mọi box `three_wheeler` hoặc box trùng IoU ≥ 0.6 phải xác minh 2 vật thật (màu xe/bánh riêng) trước khi xóa — kiểm được bằng cột `quyet_dinh` trong `calibration_decisions.csv` (CAL-01, CAL-05).
  3. Cứ 10 ảnh duyệt thì soi lại 1 ảnh ở 200% zoom với ID từng box (chế độ `qc_sample`) — kiểm được bằng `qc_review.csv` có đủ dòng cho mọi ảnh mẫu.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| img_013.jpg | SAM 2.1 base | 2 pos + 1 neg | Có — 1 điểm âm (720,300) chỉ khoét lưng áo, đầu + chân người lái vẫn dính (mask 45.4% ảnh) | Thêm 3 điểm âm nữa thì tách hết người lái nhưng mask vỡ 3 mảnh, mất bánh xe (14.2%) | `tuktuk_img013_sam2.jpg` |
| img_013.jpg | MobileSAM | 2 pos + 1 neg | Có — 35.2%, 2 polygon, ăn ít hơn vào người lái nhưng hở mái vòm phải | Viền nan hoa/bánh kém sắc hơn SAM 2.1 | `tuktuk_img013_sam1.jpg` |
| img_004.jpg | SAM 2.1 base | 2 pos + 1 neg | Không — người đàn ông áo xanh cạnh xe bị cắt sạch (13.3%) | Điểm âm (465,450) vào người đàn ông giữa 2 xe hợp lệ | `tuktuk_img004_sam2.jpg` |
| img_017.jpg | SAM 2.1 base | 2 pos + 1 neg | Không — mask ôm đúng khoang khách + đầu xe (12.1%) | Hai xích lô phía sau không bị lẫn | `tuktuk_img017_sam2.jpg` |

- Đối chiếu guideline mục 4.2: quy tắc "mask chỉ ôm khung xe + mái + bánh, người lái tách riêng bằng `--neg`" **giúp định hướng click** nhưng thiếu chỉ dẫn số lượng: một điểm âm không đủ cắt người ngồi CHỒNG lên khung xe (img_013); thử nghiệm cho thấy cần thêm `--box` thu hẹp vùng hoặc chấp nhận polygon nhiều mảnh. Đã ghi thành câu 4 mục 1 v1.1.
- *(Tuỳ chọn)* Interactor SAM trong CVAT: **không dùng** — CVAT server trong Docker bị OOM (exit 137) khi chạy đồng thời với prelabel trên máy 16 GB; toàn bộ mask tạo offline bằng `sam2_points.py`. Khác biệt: offline cho phép lặp nhanh bộ điểm pos/neg, CVAT interactor tiện hơn cho sửa từng vùng nhưng không ghi được tọa độ click vào log.

## 6. Calibration (Part 6)

- Số quyết định: **5** (`calibration_decisions.csv`: 2 `loi_may`, 2 `cau_hoi_mo`, 1 `self_check`); quan trọng nhất: **CAL-01** (quy tắc box trùng IoU ≥ 0.6 giữ box lớn) vì nó giải thích cả lỗi máy (img_003) lẫn lỗi người kiểm (img_014 — CAL-05).
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: chốt 6/7 câu mở (câu 1 xe đạp điện để ngỏ vì không có mẫu trong 36 ảnh); thêm quy tắc xóa person biên conf < 0.35 không rõ đầu/thân; box xe nguyên chiếc, hàng nhô không tính; xác minh 2 vật thật trước khi xóa box chồng.
- Nguyên nhân gốc: **guideline mơ hồ** ở ranh giới "hàng nhô ra" và "người sau kính/mui" (câu 3, 4, 5) — máy chỉ hiện thực hóa sự mơ hồ đó; riêng CAL-05 là lỗi **người duyệt**, không phải máy.

## 7. QC và quyết định (Part 6)

- Mẫu: **5 ảnh** (0 `can_review` + 5 ngẫu nhiên, seed 42) = 78 đối tượng; người kiểm: chính tôi (**hạn chế**: tôi vừa là người duyệt nên dễ bỏ sót cùng loại lỗi).
- Lỗi theo **đối tượng**: 3/78 = 3.8% · theo **ảnh**: 3/5 = 60% (ngưỡng Release ≤ 2%) — vượt xa ngưỡng.
- Top lỗi: `thua` 2 (img_003 box trùng xe kéo; img_031 person giá áo), `sai_vi_tri_kich_thuoc` 1 (img_035 xích lô box thiếu phần dap). Không có `sai_lop` trong mẫu.
- **Giới hạn mẫu nhỏ**: 3/5 ảnh lỗi → cận trên 95% Wilson của tỉ lệ ảnh lỗi thật là **88%**; ngay cả 0/5 cũng cho cận trên ~35%. Mẫu 5 ảnh chỉ phát hiện lỗi lớn, không chứng minh được lô < 2%.
- Script kiểm được: đếm lỗi theo phiếu, phát hiện lỗi hệ thống lặp loại, đối chiếu ID box. Không kiểm được: độ ôm sát viền (chấp nhận IoU ≥ 0.9 nhưng không soi từng cạnh), thiên kiến "duyệt cùng máy với mình" (cả pre-label lẫn final đều từ một người).

**Quyết định: REWORK**, ngày 2026-11-18. Lý do (Hold → Rework → Release): guideline v1.1 đã kín nên không Hold; nhưng tỉ lệ ảnh lỗi mẫu 60% > 2% và lỗi `thua` lặp 2/3 ca → Rework theo đúng cây quyết định.
Việc đã làm trong vòng rework: xóa 2 box trùng/thua + nới box xích lô → `submission/labels/qc_fixed.json`; `build_release.py` ghép bản sửa, release hiện tại **24 ảnh, 389 đối tượng**. Việc còn lại nếu lặp lại quy trình: soi 19 ảnh batch ngoài mẫu QC bằng checklist 4 bước, và chạy lại sweep với `--imgsz 1024` (GPU/Colab) để recall không bị giảm do hạ kích thước ảnh.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài**: có — GitHub Copilot (VS Code) dùng để soạn/vận hành script pipeline (prelabel, sweep, merge, inject_traps, sam2_points, build_release, qc), viết script diff hỗ trợ duyệt batch, và đọc số liệu hộ. Mọi ảnh QC/mask đều được chính tôi mở và soi trên màn hình trước khi ghi quyết định (img_003/006/014/031/035 + 4 mask); các nhận xét lỗi trong `qc_review.csv` khớp ảnh thật đã kiểm tra.
- Gold có bị nhiễm không: **không** — `gold_manual.json` gán tay 4 ảnh trước khi chạy bất kỳ lượt prelabel nào (thứ tự Pha 1 → Pha 2).
- Máy / GPU / CVAT: Windows 10 16 GB, không GPU NVIDIA → toàn bộ `--device cpu --imgsz 640`; CVAT Docker (cvat/server:dev) khởi động được nhưng OOM khi chạy song song suy luận, phần export tái lập bằng script repo.
- Dùng cache `data/cache/` thay vì tự chạy? **Không** — tự chạy cả 2 lượt cache `conf 0.05` (y26_all 1325 box/36 ảnh, yoloe_all 116 box/36 ảnh); file cache chỉ đối chiếu khi cần.
- Lỗi / bug script đã gặp: (1) `merge_prelabels.py --thr ""` — PowerShell 5.1 nuốt tham số chuỗi rỗng, phải bỏ hẳn cờ `--thr` (kết quả tương đương vì sweep không chốt lớp nào); (2) `check_env.py` báo Python 3.14 không tương thích → phải tạo venv 3.12; (3) YOLOE tự tải `mobileclip2_b.ts` (242 MB) lần đầu, làm lượt r03 lâu hơn dự kiến.
- Một điều làm khác lần sau: chạy sweep và prelabel trên Colab T4 với `--imgsz 1024` (recall cao hơn, hết cảnh "không ngưỡng nào ≤ 0.5 đạt 90%"), và dành 1 ảnh mẫu QC cho người khác kiểm để tránh thiên kiến tự duyệt.
