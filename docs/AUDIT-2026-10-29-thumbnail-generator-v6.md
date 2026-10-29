# Audit đề xuất nâng cấp — `thumbnail-generator` dòng v6 (`lte.6x*`)

- Ngày audit: 2026-10-29
- Nhánh được audit: `lte.6x`, `lte.6x-is_dev`, `lte.6x-is_platform` (cùng 1 tài liệu này được đặt trên cả 3 nhánh)
- Người/agent audit: Claude Code, đọc tĩnh mã nguồn
- **Loại tài liệu: ĐỀ XUẤT.** Lượt audit đầu không sửa code. Lượt sau (cùng ngày) chỉ sửa các lỗi đã chứng minh 100% — xem bảng dưới. Mọi mục khác vẫn để nguyên cho agent tích hợp tự điều tra và làm.

### Trạng thái sửa trên nhánh `lte.6x-is_dev` (cập nhật 2026-10-29)

| Mục | Trạng thái | Cách đã chứng minh |
|---|---|---|
| 2.1 `convertToWebP()` xoá file gốc trong `DB::transaction` | ✅ Đã sửa: chỉ xoá file gốc **sau khi** transaction trả về thành công. Khi chạy thành công kết quả không đổi; khi DB lỗi, file gốc được giữ | Đọc code: lệnh xoá file nằm trước `createOrUpdate()` trong cùng closure, rollback DB không khôi phục được file |
| 2.10 `url_tmp()` dùng biến chưa khai báo | — Không áp dụng: lỗi chỉ có ở `lte.6x-is_platform` | — |
| 2.12 (phần regex) route thiếu `\.` | ✅ Đã sửa `routes/web.php`. Phần chặn `../` vẫn chưa làm (cần điều tra) | Chạy `preg_match` với regex đúng như Symfony bọc: `fooXjpg`, `a_png` khớp regex cũ, không khớp regex mới |
| 2.16 (mới) Đường dẫn WebP thay `.jpg` cả trong tên thư mục | ✅ Đã sửa 3 chỗ: `PublicController::resize()`, `convertToWebP()`, `convertThumbnailsToWebP()` | `photos.jpg/a.jpg` → cũ ra `photos.webp/a.webp` (lưu/đọc sai thư mục); nay chỉ thay phần mở rộng cuối |
| Các mục còn lại | ⏳ Chưa làm — cần điều tra hoặc là cải tiến | — |

Các lỗi đã sửa **chưa được chạy trên site CMS thật** (repo không có app). Agent tích hợp vẫn phải tự kiểm lại bằng request/upload thật theo mục 0 — đặc biệt 2.1: giả lập `createOrUpdate()` ném lỗi, kiểm tra file gốc còn trên đĩa.
- Bản audit dòng v7 nằm ở nhánh `main`/`v7.x`: `docs/AUDIT-2026-10-29-thumbnail-generator.md`. Các nhánh v6 **không merge** với `main`/`v7.x` — mọi đề xuất ở đây phải được làm riêng trên từng nhánh v6.

---

## 0. BẮT BUỘC với agent thực thi — đọc trước khi làm bất cứ gì

Tài liệu này **không phải nguồn sự thật**. Agent thực thi phải tự đo, tự phản biện, tự kiểm chứng.

1. **Audit tĩnh, chưa chạy app.** Repo chỉ chứa package, không có mã nguồn class cha
   `Dev\Media\RvMedia` / `Platform\Media\RvMedia`, `ThumbnailService`, `UploadsManager`. Mọi nhận
   định về hành vi class cha (`getImageUrl()`, `url()`, `handleUpload()`, `getSizes()`,
   `getRealPath()`, `canGenerateThumbnails()`, `insertWatermark()`...) là **giả thuyết**. Agent phải
   mở source thật của plugin Media ở một site CMS v6 đang cài package này để xác nhận.
2. **Không có số đo nào trong tài liệu này.** Chữ "chậm", "tốn RAM" là suy luận từ đọc code. Trước khi
   sửa, đo baseline trên hạ tầng thật (storage driver thật, ảnh thật); sau khi sửa, đo lại bằng đúng
   cách đó. Không có số trước/sau → không được ghi là "đã tối ưu".
3. **Không có test.** Bước 0 của mọi thay đổi: viết test giữ lại hành vi hiện tại, chạy xanh, rồi mới
   sửa. Không tự tin "refactor không đổi behavior" chỉ vì đọc code thấy vậy.
4. **Số dòng có thể lệch.** Số dòng trong tài liệu lấy theo `lte.6x-is_dev`/`lte.6x-is_platform`.
   Trên `lte.6x` các file có thêm header bản quyền nên **lệch khoảng +20 dòng**. Luôn tự `grep` theo
   tên method trước khi sửa.
5. **Mỗi mục có "Cách tự kiểm chứng" — là yêu cầu bắt buộc**, không phải gợi ý. Chưa làm và ghi lại
   kết quả (log, response, số đo) thì mục đó chưa xong.
6. **Ba nhánh là ba sản phẩm riêng.** Sửa một nhánh không có nghĩa nhánh kia đã được sửa. Không
   cherry-pick mù giữa nhánh `Dev\` và `Platform\` — namespace khác nhau. Không merge sang `main`/`v7.x`.
7. **Package này ghi đè binding `RvMedia::class`** (`src/Providers/ThumbnailGeneratorServiceProvider.php:37-39`),
   tức là thay toàn bộ hành vi media của mọi site cài nó. Lỗi ở đây nhân với số site. Coi mọi thay đổi
   hành vi là rủi ro cao.
8. **Phản biện chính mình trước khi đánh dấu xong:** "Đã test khi `ENABLED_WEBP` tắt chưa? Với cloud
   storage chưa? Với ảnh không tồn tại chưa? Số đo lấy trên máy dev hay hạ tầng thật?" Không trả lời
   được bằng log/số liệu thật → chưa verified.

---

## 1. Khác biệt giữa 3 nhánh (đã kiểm bằng `git diff`)

| Nhánh | Namespace | Khác biệt so với `lte.6x-is_dev` |
|---|---|---|
| `lte.6x-is_dev` | `Dev\` | — (gốc so sánh) |
| `lte.6x` | `Dev\` | Thêm header bản quyền ở đầu file, format lại khoảng trắng. **Logic giống hệt.** `composer.json` giống hệt, kể cả `"dev-extensions/kernel": "dev-lte.6x-is_dev"` |
| `lte.6x-is_platform` | `Platform\` | Đổi namespace. Thêm **1 lỗi** trong `url_tmp()` (mục 2.10) |

So với v7, dòng v6 có thêm: override `handleUpload()`, `generateThumbnails()`, `convertToWebP()`,
`convertThumbnailsToWebP()`, cờ `env('ENABLED_WEBP')`, và hỗ trợ cấu trúc cache cũ có thư mục mtime.

---

## 2. Phát hiện & đề xuất

Mức độ: 🔴 correctness/security/mất dữ liệu · 🟠 hiệu năng · 🟡 dọn code/bảo trì.

### Nhóm A — Riêng dòng v6

#### 2.1 ✅ (đã sửa) `convertToWebP()` xoá file gốc bên trong `DB::transaction` — rollback DB không khôi phục được file

**Vị trí:** `src/ThumbnailMedia.php:716-751` (`convertToWebP`), đoạn `Storage::delete($originalFilePath)` dòng 730-732.

**Vấn đề:** file JPG/PNG gốc bị xoá **trước** khi cập nhật bản ghi `media_files`, và cả hai nằm trong
`DB::transaction`. Nếu `createOrUpdate()` ném lỗi, DB rollback về bản ghi cũ trỏ tới file `.jpg` —
nhưng file `.jpg` đã bị xoá khỏi đĩa, không rollback được. Kết quả: bản ghi media trỏ tới file không
tồn tại. Ngoài ra, mọi nội dung đã nhúng URL `.jpg` cũ (bài viết, block HTML, sitemap, link ngoài) sẽ
404 sau khi upload xong, vì file gốc bị xoá có chủ đích.

**Đề xuất:** đổi thứ tự — cập nhật DB trong transaction trước, chỉ xoá file gốc **sau khi commit
thành công** (`DB::afterCommit()` hoặc đặt lệnh xoá sau khối transaction). Cân nhắc giữ file gốc
(chỉ thêm `.webp` bên cạnh) nếu hệ thống có nội dung nhúng URL gốc — đây là quyết định sản phẩm, hỏi
người phụ trách, không tự quyết.

**Cách tự kiểm chứng (bắt buộc):**
- Trên staging, tạo test giả lập `createOrUpdate()` ném lỗi (mock repository) trong `convertToWebP()`;
  sau đó kiểm tra: bản ghi DB còn trỏ `.jpg` không, file `.jpg` còn trên đĩa không. Nếu file mất mà
  bản ghi còn trỏ tới nó → lỗi xác nhận.
- Sau khi sửa, chạy lại đúng test đó: file gốc phải còn khi DB lỗi.
- Grep nội dung DB (`posts.content`, `pages.content`, `meta_boxes`...) xem có URL ảnh `.jpg/.png` gốc
  của file đã convert không — để biết việc xoá file gốc đang làm vỡ bao nhiêu nội dung thật.

#### 2.2 🔴 Cờ `env('ENABLED_WEBP')` đọc lúc runtime — sai khi đã `config:cache`, và giá trị `"0"` bật nhầm tính năng

**Vị trí:** `src/ThumbnailMedia.php:56`, `:132`, `:211`.

**Vấn đề:**
- Laravel chỉ đảm bảo `env()` hoạt động trong file `config/*`. Khi chạy `php artisan config:cache`
  (thường có ở production), `.env` không được nạp → `env('ENABLED_WEBP', false)` trả về `false` → toàn
  bộ tính năng WebP **tắt âm thầm** trên production dù `.env` ghi bật.
- So sánh `=== false`: nếu `.env` ghi `ENABLED_WEBP=0` thì `env()` trả chuỗi `"0"`, `"0" === false`
  là sai → tính năng **bật** dù người cấu hình muốn tắt.
- `getImageUrl()` gọi `env()` ở mỗi lần render ảnh.

**Đề xuất:** đưa cờ vào file config của package (ví dụ `config/thumbnail-generator.php` với
`'enabled_webp' => env('ENABLED_WEBP', false)`), nạp bằng `loadAndPublishConfigurations()`, đọc bằng
`config(...)` và ép kiểu `(bool)` / `filter_var(..., FILTER_VALIDATE_BOOLEAN)`. Lưu ý skill
`botble-conventions` mục 6: tên file config không trùng core, và phải khai đúng tên trong
`loadAndPublishConfigurations()` — thiếu thì `config()` trả `null` âm thầm.

**Cách tự kiểm chứng (bắt buộc):**
- Trên staging: đặt `ENABLED_WEBP=true`, chạy `php artisan config:cache`, upload 1 ảnh JPG → xem có
  file `.webp` sinh ra không. Nếu không → lỗi xác nhận. Lặp lại với `config:clear` để so sánh.
- Đặt `ENABLED_WEBP=0`, `config:clear`, upload → xem WebP có bị sinh ra không.
- Hỏi/kiểm tra từng site production có chạy `config:cache` không trước khi kết luận mức ảnh hưởng.

#### 2.3 🔴 (cần xác minh) Rewrite URL sang route `/resize/` có thể không chạy trên v6 vì `url()` đã bị đổi tên thành `url_tmp()`

**Vị trí:** `src/ThumbnailMedia.php:121-196` (`url_tmp`) và `:118` (`return $this->url($url)`).

**Vấn đề:** comment ghi rõ `url()` gây redirect vòng lặp nên được đổi tên tạm thành `url_tmp()`, và
không nơi nào trong package gọi `url_tmp()` (đã grep cả 3 nhánh). Hệ quả: `getImageUrl()` gắn
`?w=300&h=200` vào đường dẫn rồi gọi `$this->url()` của **class cha**, không phải logic rewrite
`storage/` → `resize/storage/`. Nếu class cha không tự rewrite, URL trả ra có dạng
`/storage/news/image.jpg?w=300&h=200` → webserver trả file gốc nguyên kích thước, bỏ qua query →
**route resize không bao giờ được gọi**, ảnh không được thu nhỏ, trang nặng hơn dự kiến.

**Đề xuất:** không sửa trước khi xác minh. Nếu xác nhận đúng, tìm nguyên nhân gốc của vòng lặp
redirect (ghi trong comment) rồi mới khôi phục logic, không chỉ đổi tên ngược lại.

**Cách tự kiểm chứng (bắt buộc — ưu tiên cao nhất nhóm A):**
- Trên site v6 thật, render 1 trang có ảnh gọi `getImageUrl($url, '300x200')`, xem `src` thật trong
  HTML: có chứa `/resize/` không.
- `curl -I` URL đó: kích thước trả về (`Content-Length`) và `Content-Type` có khớp ảnh đã thu nhỏ/WebP
  không, hay là file gốc.
- Đọc `RvMedia::url()` thật của core v6 đang cài.

#### 2.4 🟠 `handleUpload()` là bản sao (fork) của core — dễ lệch khi core nâng cấp; giải mã toàn bộ ảnh chỉ để lấy chiều rộng

**Vị trí:** `src/ThumbnailMedia.php:205-375`.

**Vấn đề:**
- Method chép lại gần như toàn bộ `handleUpload()` của core, chỉ thêm kiểm tra chiều rộng và gọi
  `convertToWebP()`. Khi core sửa lỗi/bảo mật trong `handleUpload()`, bản sao này không nhận được.
- `Image::make($fileUpload->getRealPath())` (dòng 264) giải mã toàn bộ ảnh vào RAM chỉ để đọc
  `width()`. `getimagesize()` đọc header file, rẻ hơn nhiều với ảnh lớn.
- Ảnh rộng hơn 1920px bị **từ chối upload** (dòng 267-271, hardcode). Cần đối chiếu với setting core
  `media_reduce_large_image_size` / `media_image_max_width` (nếu core v6 có) — có thể đang chặn upload
  trong khi core có sẵn cơ chế tự thu nhỏ.

**Đề xuất:** thay `Image::make()->width()` bằng `getimagesize()`; đưa ngưỡng 1920 vào config. Về lâu
dài, thay vì chép `handleUpload()`, gọi `parent::handleUpload()` rồi xử lý WebP trên kết quả (giống
cách bản v7 làm) — **chỉ khi** đã xác nhận core v6 trả về đủ dữ liệu để làm vậy.

**Cách tự kiểm chứng (bắt buộc):**
- `diff` method này với `RvMedia::handleUpload()` của đúng phiên bản core v6 đang cài trên từng site,
  liệt kê những thay đổi core mà bản sao đang thiếu.
- Đo `memory_get_peak_usage()` khi upload 1 ảnh lớn (ví dụ 6000×4000) với `Image::make()` vs
  `getimagesize()`. Ghi số liệu cụ thể.

#### 2.5 🟠 Sinh thumbnail rồi giải mã lại để chuyển WebP — mã hoá 2 lần mỗi kích thước

**Vị trí:** `src/ThumbnailMedia.php:389-429` (`generateThumbnails`), `:760-839` (`convertThumbnailsToWebP`).

**Vấn đề:** với mỗi kích thước, `ThumbnailService` lưu file JPG/PNG; sau đó `convertThumbnailsToWebP()`
đọc lại từng file đó từ đĩa, giải mã, mã hoá WebP, lưu, rồi xoá JPG. Tức mỗi kích thước tốn 2 lần
encode + 1 lần decode + thêm I/O. Ngoài ra `getRealPath($file->url)` được gọi lại trong mỗi vòng lặp
(dòng 401) dù giá trị không đổi.

**Đề xuất:** đưa `getRealPath($file->url)` ra ngoài vòng lặp. Nếu `ThumbnailService` (hoặc Intervention
Image v2) cho phép chọn định dạng output, mã hoá thẳng WebP một lần.

**Cách tự kiểm chứng (bắt buộc):**
- Đọc API thật của `ThumbnailService` v6 xem có chọn được định dạng output không. Không có → chỉ làm
  phần đưa `getRealPath` ra ngoài vòng lặp.
- Đo tổng thời gian upload 1 ảnh (với số kích thước thật trong `getSizes()`) trước/sau, lặp ≥ 10 lần,
  ghi trung bình.

#### 2.6 🟠 Ghi log mức `info` nhiều lần cho mỗi lần upload

**Vị trí:** `src/ThumbnailMedia.php:617-689` (`convertToWebP`) — 4-5 lệnh `Log::info` mỗi upload; khi lỗi
còn ghi toàn bộ `gd_info()` và stack trace.

**Vấn đề:** log tăng nhanh trên site upload nhiều, tốn I/O, khó tìm lỗi thật.

**Đề xuất:** hạ xuống `Log::debug` hoặc bỏ các log theo dõi bước; chỉ giữ `Log::error` khi thất bại.

**Cách tự kiểm chứng:** đếm số dòng log sinh ra cho 1 lần upload trước/sau; kiểm tra cấu hình
`LOG_LEVEL` thật của site (nếu đã là `warning` thì mục này không có tác động thực tế — ghi nhận và bỏ qua).

#### 2.7 🟠 Xoá thumbnail quét cả cấu trúc cache cũ có thư mục mtime ở mỗi lần xoá file

**Vị trí:** `src/ThumbnailMedia.php:440-533` (`purgePhysicalThumbnails`), vòng lặp `File::directories($sizeDirectory)` dòng 495.

**Vấn đề:** với mỗi thư mục kích thước, code liệt kê thêm toàn bộ thư mục con để tìm thư mục mtime cũ.
Nếu `public/resize/` còn nhiều thư mục mtime từ thời cấu trúc cũ, mỗi lần xoá 1 file media phải quét
`số kích thước × số thư mục mtime`.

**Đề xuất:** dọn một lần toàn bộ thư mục mtime cũ trong `public/resize/` (bằng lệnh artisan hoặc script
có kiểm tra), sau đó bỏ nhánh tương thích ngược này.

**Cách tự kiểm chứng (bắt buộc):** đếm số thư mục mtime thật trên từng site
(`find public/resize -mindepth 2 -maxdepth 2 -type d -regex '.*/[0-9]+$' | wc -l`). Bằng 0 hoặc rất ít
→ mục này không đáng làm, ghi nhận và bỏ qua. Đo thời gian xoá 1 file media trước/sau nếu làm.

#### 2.8 🟡 Code chết và import thừa

**Vị trí:** `src/ThumbnailMedia.php`
- `url_tmp()` (dòng 121-196) không được gọi ở đâu.
- `convertToWebP()`: kiểm tra `mime_type === 'image/webp'` (dòng 634) không bao giờ đúng vì dòng 625
  đã loại mọi mime ngoài jpeg/png; biến `$fileExtension` (dòng 737-741) tính xong không dùng.
- Import không dùng: `FileExistsException`, `ResponseFactory`, `Http`, `Request`, `Response`,
  `JsonResponse`, `FileNotFoundException`, `MimeTypes`, `Throwable`... (tự kiểm bằng PHPStan/IDE).

**Đề xuất:** xoá sau khi đã có test (mục 2.15). Riêng `url_tmp()`: xử lý cùng mục 2.3, không xoá
trước khi kết luận 2.3.

**Cách tự kiểm chứng:** chạy PHPStan/Psalm trước/sau; chạy test baseline — phải xanh, không đổi.

#### 2.9 🟡 `composer.json` của `lte.6x` yêu cầu kernel `dev-lte.6x-is_dev`

**Vị trí:** `composer.json` nhánh `lte.6x`.

**Vấn đề:** file trùng hệt nhánh `lte.6x-is_dev`. Có thể là chủ đích (`lte.6x` là bản `Dev\` có header
bản quyền), cũng có thể là sót khi tách nhánh.

**Đề xuất:** người phụ trách xác nhận. Không tự đổi — đổi constraint có thể làm `composer update`
thất bại trên các site đang dùng.

**Cách tự kiểm chứng:** kiểm tra kho `dev-extensions/kernel` có nhánh `lte.6x` không, và các site đang
cài `thumbnail-generator` từ nhánh `lte.6x` đang resolve kernel từ nhánh nào (`composer show -a`).

#### 2.10 🔴 (chỉ `lte.6x-is_platform`) `url_tmp()` dùng biến chưa khai báo

**Vị trí:** `src/ThumbnailMedia.php:133` nhánh `lte.6x-is_platform`:
```php
return parent::getImageUrl($url, $size, $relativePath, $default);
```
`url_tmp(?string $path)` chỉ có tham số `$path`; `$url`, `$size`, `$relativePath`, `$default` chưa khai
báo. Hai nhánh `Dev\` đúng là `return parent::url($path);`.

**Mức độ thật:** hiện không gây lỗi vì `url_tmp()` không được gọi (mục 2.3). Nhưng nếu ai khôi phục
`url_tmp()` thành `url()` trên nhánh này mà không để ý, mọi lần gọi khi `ENABLED_WEBP` tắt sẽ lỗi.

**Đề xuất:** sửa cùng lúc với mục 2.3 trên nhánh `lte.6x-is_platform`.

**Cách tự kiểm chứng:** `php -l` không bắt được lỗi này. Viết unit test gọi `url_tmp('storage/a.jpg')`
với `ENABLED_WEBP=false` để thấy lỗi trước khi sửa, xanh sau khi sửa.

#### 2.11 🟡 Dấu vết tên vendor trong tên class core: `RvMedia`, `RvMediaFacade`

**Vị trí:** mọi `use ...\Media\RvMedia as AppMedia` / `RvMediaFacade`.

**Ghi nhận:** đây là tên class của core v6, package chỉ tham chiếu (đã alias thành `AppMedia`). Theo
skill `botble-conventions` mục 4: chỉ báo, không tự sửa hàng loạt; không đưa thêm tên này vào code mới,
comment, tên file.

### Nhóm B — Giống dòng v7 (cùng đoạn code, xem thêm bản audit v7 trên `main`)

#### 2.12 🔴 Regex route thiếu escape dấu chấm (✅ phần regex đã sửa); `$slug` không được chặn `../`

**Vị trí:** `routes/web.php:141` — `'storage/(.*).(jpg|jpeg|png|webp|gif)'`: dấu `.` trước phần mở rộng
khớp **mọi ký tự**, phải là `\.`. `src/Http/Controllers/PublicController.php:18` — `public_path($slug)`
không kiểm tra kết quả còn nằm trong `public/`.

**Đề xuất:** sửa regex thành `\.`; thêm kiểm tra `realpath()` của file nằm trong `realpath(public_path())`,
nếu không → 404.

**Cách tự kiểm chứng (bắt buộc, chỉ trên staging):** gọi `/resize/storage/testXjpg` và các biến thể
`../`, `%2e%2e%2f` bằng `curl -i`, ghi lại status/body trước và sau khi sửa. Nếu webserver đã chặn
`../` thì ghi nhận mức độ là "phòng thủ thêm", nhưng vẫn nên sửa vì chi phí gần như bằng 0.

#### 2.13 🔴 `w`/`h` do người gọi truyền không có giới hạn trên; GIF động bị ép sang WebP

**Vị trí:** `PublicController.php:82-83` (`$width`/`$height` không clamp khi client truyền), `:40`
(`$forceWebP` gồm cả `gif`). Request mỗi lần một cặp `w`/`h` khác nhau sẽ tạo file cache mới, tốn CPU,
RAM và đĩa. Ảnh GIF đi qua `fit()` + `encode('webp')` của Intervention Image v2 có khả năng chỉ còn
frame đầu.

**Đề xuất:** luôn `min()` theo giới hạn lấy từ config; bỏ `gif` khỏi danh sách ép WebP.

**Cách tự kiểm chứng (bắt buộc):** đo RAM/thời gian với `?w=10000&h=10000` so với `?w=300&h=200` trên
staging; đếm số frame của GIF trước/sau khi qua route (`identify -format %n`).

#### 2.14 🟠 `getImageUrl()`: gọi `getMimeType(getRealPath())` 2 lần; regex `[0-9|auto]` sai cú pháp; đổi tên file theo size và thêm query chạy chồng nhau

**Vị trí:** `src/ThumbnailMedia.php:78-108`.
- Dòng 81 và 91 cùng gọi `canGenerateThumbnails($this->getMimeType($this->getRealPath($url)))`; lần 2
  chạy trên `$url` **đã bị đổi tên** thành `image-300x200.jpg` (file có thể chưa tồn tại).
- Dòng 90: `[0-9|auto]` là tập ký tự đơn, không phải "số hoặc chữ auto"; pattern không neo `^$`, nên
  `"abcx1"` vẫn khớp và sinh `w=abc`.
- Khi `$size` vừa có trong `getSizes()` vừa khớp dạng `WxH`, cả 2 nhánh cùng chạy → URL có dạng
  `image-300x200.jpg?w=300&h=200`.
- Gọi `env()` ở đầu mỗi lần (mục 2.2).

**Đề xuất:** tính mime đúng 1 lần trên `$url` gốc; thay regex bằng `explode('x', $size, 2)` có kiểm tra
số; quyết định 2 nhánh có loại trừ nhau không **sau khi** đã xác minh mục 2.3 (vì trên v6, query có
thể không bao giờ tới route resize).

**Cách tự kiểm chứng (bắt buộc):** lập bảng input/output cho mọi giá trị `$size` thật đang dùng
(`grep -rn "getImageUrl(\|get_object_image(" ` trên site tiêu thụ + `getSizes()` thật); cách mới phải
cho kết quả giống 100% với cách cũ, trừ các trường hợp đã xác nhận là lỗi.

#### 2.15 🟡 Helper deprecated, không có test

- `helpers/helpers.php:10-43`: `apps_get_thumbnail_generation()` đánh dấu `@deprecated since 5.7`,
  gọi `get_object_image()` không có trong package. Grep trên từng site tiêu thụ trước khi xoá.
- `get_max_mimesizes()` decode JSON mỗi lần gọi — chỉ tối ưu nếu đo được gọi nhiều lần/request.
- Không có thư mục `tests/`. Viết test baseline là **bước 0** trước mọi mục khác.

---

#### 2.16 ✅ (đã sửa) Đường dẫn WebP thay phần mở rộng ở mọi vị trí trong path

**Vị trí:** `PublicController::resize()`, `ThumbnailMedia::convertToWebP()`, `ThumbnailMedia::convertThumbnailsToWebP()`.

**Lỗi cũ:** `str_replace('.' . $ext, '.webp', $path)` thay mọi chỗ `.jpg`/`.png` trong toàn đường dẫn, kể cả
tên thư mục (`photos.jpg/a.jpg` → `photos.webp/a.webp`). Ở `resize()` làm không tìm thấy bản WebP; ở 2
method convert làm **lưu file WebP vào thư mục sai** và bản ghi trỏ tới đường dẫn đó.

**Đã sửa:** `preg_replace('/\.[^.\/]+$/', '.webp', $path)` — chỉ thay phần mở rộng cuối. Đường dẫn thường cho
kết quả giống hệt cũ.

## 3. Không đề xuất (và vì sao)

- Không đổi cơ chế cache đĩa `public/resize/` hay `flock()` sang Redis — chưa có số đo cho thấy đây là
  điểm nghẽn. `flock()` chỉ sai khi chạy nhiều server; agent phải tự xác nhận hạ tầng trước.
- Không nâng Intervention Image v2 → v3 trên dòng v6 — thay đổi API lớn, ngoài phạm vi tối ưu.
- Không sửa core/vendor (`RvMedia`...) — skill `botble-conventions` mục 1.
- Không port tự động thay đổi giữa 3 nhánh v6 hay sang v7 — xem mục 0.6.

## 4. Thứ tự triển khai đề xuất (làm riêng trên từng nhánh)

1. Viết test baseline (2.15).
2. Xác minh 2.3 (resize có chạy trên v6 không) và 2.2 (`env()` + `config:cache`) trên site thật — hai
   mục này quyết định phần lớn các mục còn lại có ý nghĩa hay không.
3. Sửa 2.1 (mất file gốc), 2.12, 2.13, 2.10.
4. Hiệu năng: 2.4, 2.5, 2.14, 2.6, 2.7 — mỗi mục 1 commit, đo trước/sau từng mục.
5. Dọn dẹp: 2.8, 2.15, xác nhận 2.9.
