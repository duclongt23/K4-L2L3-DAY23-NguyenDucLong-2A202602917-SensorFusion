# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Nguyễn Đức Long
- MSSV: 2A202602917
- Email: duclongt23@gmail.com
- Link repo (fork): https://github.com/duclongt23/K4-L2L3-DAY23-NguyenDucLong-2A202602917-SensorFusion
- Commit hash nộp (`git rev-parse HEAD`): xem commit hash đã nộp trên LMS (commit cuối cùng trên `main`)

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: `compare`, `[0, 198]` (199 frame), `training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord`, `seed = 0`
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: precision = 0.9701, recall = 0.7004, tp = 519, fp = 16, fn = 222
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: rmse = 0.1503 m, matches = 502, sum_sq_err = 11.3436 m², ghost = 0, miss = 239, mean_confirmed = 2.5226
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: rmse = 0.1359 m, matches = 502, sum_sq_err = 9.2668 m², ghost = 0, miss = 239, mean_confirmed = 2.5226
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss:
  - Fused tốt hơn LiDAR-only: RMSE giảm từ 0.1503 m xuống 0.1359 m (~9.6%), cho thấy camera giúp cải thiện độ chính xác vị trí 3D.
  - Số lượng track không đổi: Cả hai mode đều có 502 matches, 0 ghost và 239 miss. Điều này phù hợp với thiết kế LiDAR quản lý vòng đời track, còn camera chỉ hỗ trợ EKF update.
  - Vẫn còn bỏ sót: Detector có 222 FN, là một nguyên nhân chính khiến số miss còn cao. Fusion chưa cải thiện khả năng bao phủ đối tượng.
  - Đạt yêu cầu rubric: Precision track = 100%, coverage = 96.7%, RMSE của cả hai mode đều dưới 0.45 m.
  - Giới hạn: Camera dùng phép đo mô phỏng từ ground truth có nhiễu, nên kết quả chưa phản ánh đầy đủ hiệu quả trên hệ thống thực tế.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. Khác biệt đo LiDAR 3D và Camera 2D trong EKF (`z`, `R`)?

LiDAR đo vị trí 3D `(x, y, z)` theo đơn vị mét, còn camera đo vị trí 2D `(u, v)` theo pixel. Vì vậy, hai sensor có `z`, `R` và mô hình đo khác nhau. LiDAR dùng mô hình đo tuyến tính, camera dùng phép chiếu pinhole phi tuyến nên cần Jacobian. Cả hai vẫn dùng chung EKF để cập nhật trạng thái 6D.

2. Vì sao cần gating Mahalanobis trước khi gán?

Mahalanobis tính khoảng cách có xét đến độ bất định của track và phép đo, thay vì chỉ xét khoảng cách Euclidean. Gating giúp loại các cặp không hợp lý trước khi ghép, tránh gán nhầm detection và làm sai trạng thái track.

3. Pipeline là track-then-fuse hay fuse-then-track?

Bài lab sử dụng **track-then-fuse**. Mỗi frame, EKF predict một lần, sau đó cập nhật bằng LiDAR rồi đến camera. LiDAR quyết định vòng đời track, camera chỉ điều chỉnh trạng thái. Trong log, hai mode có cùng 502 matches, 0 ghost và 239 miss, nhưng fused có RMSE thấp hơn.

4. Nếu camera lệch calibration, innovation/residual thay đổi thế nào?

Calibration sai khiến vị trí pixel dự đoán bị lệch so với phép đo thực, làm innovation có sai lệch hệ thống. Nếu lệch nhỏ, EKF có thể cập nhật sai vị trí; nếu lệch lớn, Mahalanobis gating sẽ loại các cặp camera. Điều này có thể làm chất lượng fusion giảm.

5. Vì sao cần truyền `sensor` ở frame rỗng? Vì sao LiDAR quyết định vòng đời track?

Khi không có measurement, hệ thống vẫn cần biết đang xử lý LiDAR hay camera. Frame LiDAR rỗng vẫn phải cập nhật score và kiểm tra xóa track, còn camera không được thay đổi vòng đời track. Trong lab, LiDAR là nguồn phát hiện 3D chính, camera chỉ hỗ trợ EKF update.

6. Điều kiện xác nhận, giữ và xóa track?

Track mới có score `1/6`, mỗi LiDAR hit cộng `1/6`, miss trong FOV trừ `1/6`. Track được confirmed khi score > 0.8 và không bị hạ trạng thái chỉ vì miss. Track bị xóa khi bất định vị trí vượt ngưỡng, confirmed có score < 0.6 hoặc track chưa confirmed có score ≤ 0. Camera không tham gia quyết định xóa track.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): Claude , ChatGPT
- Dùng cho phần nào (hàm, câu hỏi, debug): tóm tắt đề và lên kế hoạch; hướng dẫn code; debug môi trường Windows (xung đột OpenMP giữa torch và numpy, encoding UTF-8); giải thích nhiều câu hỏi.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): chạy `pytest student/tests -q` (128 passed, không còn failed/xfailed); chạy `fusion-run-lab --fusion compare --seed 0` trên frame 0–198; đối chiếu công thức EKF/Mahalanobis/pinhole với `docs/HUONG_DAN_KY_THUAT.md` và các gợi ý `# vi: TODO`.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [x] Part A–D: không bắt buộc sửa (hoặc ghi chú nếu bạn đã sửa)
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [x] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [x] Đã điền đủ file này, gồm khai báo AI
- [x] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [ ] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
