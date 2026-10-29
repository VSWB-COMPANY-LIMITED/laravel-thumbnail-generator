# Changelog

## Quy tắc nhánh

- `main` và `v7.x` dùng chung một lịch sử (CMS >=7.x, namespace `Dev\\`).
- `lte.6x`, `lte.6x-is_dev`, `lte.6x-is_platform` phục vụ phiên bản cũ / phiên bản đặc thù — **không merge** với `main`/`v7.x` và không merge qua lại với nhau. Nhánh này: `lte.6x-is_platform`.

## 2026-10-29 — sửa lỗi đã xác nhận

- `convertToWebP()`: chỉ xoá file JPG/PNG gốc sau khi cập nhật DB thành công (trước đây xoá bên trong transaction, DB lỗi thì mất file).
- Route `thumbnail.resize`: escape dấu chấm trước phần mở rộng.
- Tìm/lưu bản `.webp` chỉ thay phần mở rộng cuối, không thay `.jpg` trong tên thư mục (`resize()`, `convertToWebP()`, `convertThumbnailsToWebP()`).
- `url_tmp()`: sửa lời gọi dùng biến chưa khai báo thành `parent::url($path)`.
- Các mục cần điều tra / cải tiến trong báo cáo audit vẫn chưa làm.

## 2026-10-29

- Thêm `docs/AUDIT-2026-10-29-thumbnail-generator-v6.md`: báo cáo audit tĩnh dòng v6, đề xuất nâng cấp logic và tốc độ (chỉ đề xuất, chưa sửa code).
- Ghi quy tắc nhánh vào `README.md` và `CHANGELOG.md`.
