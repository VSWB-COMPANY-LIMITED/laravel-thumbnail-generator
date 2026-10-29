# Changelog

## Quy tắc nhánh

- `main` và `v7.x` dùng chung một lịch sử (CMS >=7.x, namespace `Dev\\`). Thay đổi trên nhánh này phải được đưa sang nhánh kia.
- `lte.6x`, `lte.6x-is_dev`, `lte.6x-is_platform` phục vụ phiên bản cũ / phiên bản đặc thù — **không merge** qua lại với `main`/`v7.x`, và không merge qua lại giữa các nhánh `lte.6x*`.

## 2026-10-29

- Audit dòng v6: `docs/AUDIT-2026-10-29-thumbnail-generator-v6.md` được đặt riêng trên từng nhánh `lte.6x`, `lte.6x-is_dev`, `lte.6x-is_platform` (không có trên `main`/`v7.x`).

- Thêm `docs/AUDIT-2026-10-29-thumbnail-generator.md`: báo cáo audit tĩnh, đề xuất nâng cấp logic và tốc độ (chỉ đề xuất, chưa sửa code).
- Merge `v7.x` → `main` để hai nhánh đồng bộ.
- Ghi quy tắc nhánh vào `README.md` và `CHANGELOG.md`.
