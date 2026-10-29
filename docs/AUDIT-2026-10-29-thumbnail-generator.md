# Audit đề xuất nâng cấp — `dev-extensions/thumbnail-generator`

- Ngày audit: 2026-10-29
- Branch: `v7.x`
- Người/agent audit: Claude Code, đọc tĩnh toàn bộ mã nguồn repo này
- Phạm vi: toàn bộ `src/`, `helpers/`, `routes/`, `resources/`, `composer.json`
- **Loại tài liệu: ĐỀ XUẤT.** Lượt audit đầu không sửa code. Lượt sau (cùng ngày) chỉ sửa các lỗi đã chứng minh 100% — xem bảng dưới. Mọi mục khác vẫn để nguyên cho agent tích hợp tự điều tra và làm.

### Trạng thái sửa (cập nhật 2026-10-29)

| Mục | Trạng thái | Cách đã chứng minh |
|---|---|---|
| 2.1 Regex route thiếu `\.` | ✅ Đã sửa (`routes/web.php`) | Chạy `preg_match` với regex đúng như Symfony bọc: `fooXjpg`, `a_png` khớp regex cũ, không khớp regex mới; `a.jpg`, `a-300x200.webp` khớp cả hai |
| 2.13 (mới) Backblaze CDN bật nhưng không có custom domain → URL mất host | ✅ Đã sửa (`src/ThumbnailMedia.php`, nhánh backblaze trong `url()`) | `str_replace($endpoint, '', Storage::url(...))` cho ra `/bucket/news/a.jpg` (đường dẫn tương đối, sai host). Nay trả `Storage::url($path)` |
| 2.14 (mới) Đường dẫn WebP thay `.jpg` cả trong tên thư mục | ✅ Đã sửa (`PublicController::resize()`) | `.../photos.jpg/news/a.jpg` → cũ ra `.../photos.webp/news/a.webp`; nay chỉ thay phần mở rộng cuối |
| Các mục còn lại (2.2–2.12) | ⏳ Chưa làm — cần điều tra hoặc là cải tiến | — |

Các lỗi đã sửa **chưa được chạy trên site CMS thật** (repo không có app). Agent tích hợp vẫn phải tự kiểm lại bằng request thật theo mục 0.

---

## 0. ĐỌC TRƯỚC KHI LÀM BẤT CỨ GÌ — bắt buộc với agent thực thi

Tài liệu này **không phải nguồn sự thật**. Đây là audit **tĩnh** (đọc code, không chạy được app
thật vì repo này là 1 package độc lập, **không có mã nguồn vendor** `Dev\Media\AppMedia`,
`Dev\Media\Services\ThumbnailService`, `Dev\Media\Services\UploadsManager` kèm theo — những class
này được `src/ThumbnailMedia.php` extends/gọi nhưng **không thể đọc được trong repo này**). Điều đó
có nghĩa:

1. **Mọi nhận định về hành vi của `AppMedia` (class cha)** — ví dụ `getSizes()`, `canGenerateThumbnails()`,
   `getMimeType()`, `getRealPath()`, `getDefaultImage()`, `isUsingCloud()`, `getMediaDriver()`,
   `imageManager()`, `handleUpload()` — đều là **suy luận từ cách gọi, chưa verify được bằng cách đọc
   implementation thật**. Agent thực thi phải tự mở package `dev-extensions/libs/media` (hoặc tên
   tương đương) ở project CMS thật đang cài plugin này, đọc source thật của `AppMedia`, rồi mới kết
   luận đúng/sai.
2. **Không benchmark nào trong tài liệu này là số đo thật** — mọi chữ "chậm", "redundant I/O", "tốn
   CPU" là suy luận logic từ đọc code, **không phải kết quả profiling**. Trước khi sửa, agent thực thi
   PHẢI tự đo (xem mục 6).
3. **Không có test nào trong repo** (`tests/` không tồn tại) — mọi đề xuất "an toàn" trong tài liệu
   này chưa được test nào xác nhận. Agent thực thi PHẢI viết test tái hiện hành vi hiện tại TRƯỚC khi
   sửa, để có baseline so sánh — không tự tin rằng refactor "chắc chắn không đổi behavior" chỉ vì đọc
   code thấy vậy.
4. **Không tự tin tuyệt đối vào số dòng (file:line) trong tài liệu này** — số dòng đúng tại thời điểm
   audit (`v7.x` tại commit audit); nếu code đã đổi, agent phải tự `grep`/mở lại file để tìm đúng vị
   trí trước khi hành động, không suy ra từ số dòng cũ.
5. **Mỗi đề xuất ở mục 2 đều có cột "Cách tự kiểm chứng"** — đây là YÊU CẦU BẮT BUỘC, không phải gợi ý.
   Agent thực thi không được đánh dấu một đề xuất là "đã áp dụng/đã xong" nếu chưa thực hiện đúng cách
   kiểm chứng đó và ghi lại kết quả đo được (số liệu cụ thể, không phải "có vẻ nhanh hơn").
6. Tài liệu này **không đề xuất sửa bất kỳ API/contract public nào** (`ThumbnailMedia::getImageUrl()`,
   route `thumbnail.resize`, facade `ThumbnailMedia`) mà không có ghi chú rõ "BREAKING" — vì package
   này được nhiều site CMS khác nhau tiêu thụ qua composer (`dev-extensions/thumbnail-generator`), đổi
   hành vi public mà không khảo sát call site ở từng site là rủi ro thật.
7. Repo này đứng ngoài convention `dev/plugins/<name>` mô tả trong skill `botble-plugin` — nó là
   **library/override** (`dev-extensions/libs/...`), tự ghi đè binding `AppMedia::class` trong service
   container (xem `src/Providers/ThumbnailGeneratorServiceProvider.php:37-39`). Vì vậy **đây gần như là
   code "nền tảng" cho mọi site dùng nó** — rủi ro của một lỗi ở đây nhân với số site cài package. Mọi
   thay đổi hành vi (không chỉ perf refactor thuần) nên coi là thay đổi rủi ro cao.

---

## 1. Tóm tắt điều hành

Package nhỏ, 2 class chính (`ThumbnailMedia` override `AppMedia`, `PublicController` xử lý route
resize động) + 1 ServiceProvider + 2 helper. Không có test, không có migration, không có config file
riêng (dùng `setting()` của plugin Media core).

Phát hiện quan trọng nhất **không phải tối ưu tốc độ** mà là **3 vấn đề correctness/security tiềm
ẩn** (mục 2.1–2.3) cần agent thực thi tự tái hiện bằng request thật trước khi tin. Sau đó là nhóm tối
ưu tốc độ thật (mục 2.4–2.8), và nhóm dọn code/maintainability (mục 2.9+).

---

## 2. Danh sách phát hiện & đề xuất

Ký hiệu mức độ: 🔴 Cao (correctness/security) · 🟠 Trung bình (performance có đo được) · 🟡 Thấp
(maintainability/code style).

### 2.1 ✅ (đã sửa) Route regex `resize/{slug}` không escape dấu chấm trước phần mở rộng

**Vị trí:** `routes/web.php:141`
```php
->where('slug', 'storage/(.*).(jpg|jpeg|png|webp|gif)')
```
**Vấn đề:** dấu `.` ngay trước `(jpg|jpeg|...)` trong regex PCRE là **"match 1 ký tự bất kỳ"**, không
phải dấu chấm literal (phải viết `\.`). Hệ quả: constraint này **không thực sự ép buộc có dấu chấm**
trước phần mở rộng — một slug như `storage/fooXjpg` (X là ký tự bất kỳ) cũng khớp pattern. Không gây
sập ngay, nhưng làm route nhận vào nhiều input hơn ý định ban đầu, mở rộng bề mặt tấn công cho vấn đề
2.2.

**Đề xuất:** đổi thành `'storage/(.*)\.(jpg|jpeg|png|webp|gif)'` (escape dấu chấm).

**🔴 Cách tự kiểm chứng (bắt buộc trước khi sửa và sau khi sửa):**
- Viết Feature test: gọi `$this->get('/resize/storage/testXjpg')` (không có dấu chấm thật) trên 1 CMS
  có cài plugin thật — so sánh response TRƯỚC/SAU khi sửa regex. TRƯỚC khi sửa, nếu route match (không
  404 ở tầng route) thì xác nhận bug là thật.
- Không kết luận chỉ từ đọc regex — PCRE behaviour với route compiler của Laravel (có escape ở layer
  khác không) cần test bằng request thật (`curl -i` hoặc `route:list` + `php artisan tinker` build URL
  rồi gọi). Đây là đúng tinh thần RULE "verify = request thật" của skill `botble-plugin` mục 5.

---

### 2.2 🔴 Khả năng path traversal / đọc file ngoài ý muốn qua `$slug`

**Vị trí:** `src/Http/Controllers/PublicController.php:19` (`public_path($slug)`) kết hợp với
route constraint lỏng ở mục 2.1.

**Vấn đề:** `$slug` lấy trực tiếp từ URL, không qua `realpath()`/kiểm tra nằm trong `public_path()`
trước khi `public_path($slug)` ghép đường dẫn. Laravel route thường tự decode URL nhưng **không tự
chặn `../`** trong một capture group dạng `(.*)`. Nếu `$slug` chứa `../../.env` + phần mở rộng ảnh giả
(ví dụ tận dụng lỗ hổng 2.1 để qua được constraint lỏng), `public_path($slug)` có thể trỏ ra ngoài
`public/`.

**Mức độ rủi ro thật cần xác nhận:** KHÔNG chắc chắn đây là lỗ hổng khai thác được — Laravel route
matching + webserver (Nginx/Apache rewrite rule cho route resize) có thể đã chặn `../` ở tầng trước khi
tới PHP. Đây là lý do mục 0.2 nhấn mạnh **không tin tuyệt đối** — đây là 1 giả thuyết cần kiểm chứng,
không phải kết luận.

**Đề xuất (chỉ áp dụng sau khi xác nhận lỗ hổng là thật):**
- Thêm guard: `realpath($original_img_path)` phải bắt đầu bằng `realpath(public_path())` trước khi
  dùng file, nếu không → `abort(404)`.
- Validate `$slug` không chứa `..` trước khi ghép path (`Str::contains($slug, '..')`).

**🔴 Cách tự kiểm chứng (bắt buộc):**
- Trên môi trường staging (KHÔNG production), thử request thật:
  `curl -i "https://staging.example.com/resize/storage/../../../../.env"` (và vài biến thể encode
  `%2e%2e%2f`) — xem response có leak nội dung `.env` hay không, hay bị 404/chặn ở webserver/router.
- Nếu không khai thác được, hạ mức độ phát hiện này xuống 🟡 ghi chú "defense-in-depth", không phải
  🔴 lỗ hổng xác nhận — nhưng **vẫn nên thêm guard vì chi phí code gần như 0** và không depend vào
  cấu hình webserver của từng site khác nhau.

---

### 2.3 🔴 `forceWebP` áp cho GIF phá animation; width/height không bị chặn trên (resource exhaustion)

**Vị trí:** `src/Http/Controllers/PublicController.php:43-45` và `:88-89`.

**Vấn đề A — GIF động bị convert thành WebP tĩnh:**
```php
$forceWebP = in_array($extension, ['jpg', 'jpeg', 'png', 'gif', 'webp']);
```
GIF animation thường dùng cho ảnh động (banner, sticker). Intervention Image khi `read()` + `cover()`
+ `encode(WebpEncoder)` một GIF nhiều frame, theo hiểu biết thông thường của driver GD/Imagick, **có
khả năng chỉ lấy frame đầu** → ảnh động thành ảnh tĩnh. Đây là **thay đổi hành vi hiển thị cho người
dùng cuối**, không chỉ là tối ưu — cần coi là bug chức năng nếu đúng.

**Vấn đề B — không có upper bound cho `w`/`h` do client tự truyền:**
```php
$width = max(1, (int) round((float) $request->w));
$height = max(1, (int) round((float) $request->h));
```
Giới hạn `1920`/`1800` (xem README) **chỉ áp dụng khi client KHÔNG truyền `w`/`h`** (nhánh
`blank($request->w)` ở dòng 71-79). Nếu client chủ động truyền `?w=20000&h=20000`, không có đoạn code
nào clamp lại → Intervention Image sẽ cố resize/crop ảnh xuống kích thước đó, tốn CPU/RAM tỉ lệ với
diện tích pixel, và tạo 1 file cache lớn trên disk. Nhiều request như vậy với tham số random (để né
cache — đổi để mỗi lần tạo file cache khác nhau) có thể là vector DoS nhẹ (CPU + disk).

**Đề xuất:**
- A: giới hạn `forceWebP` chỉ cho `['jpg', 'jpeg', 'png']` (bỏ `gif`), HOẶC nếu vẫn muốn ép GIF ra
  WebP, dùng encoder animated-aware và viết test xác nhận animation được giữ — không giả định.
- B: luôn `min($width, MAX_W)` / `min($height, MAX_H)` kể cả khi client truyền `w`/`h` tường minh, với
  `MAX_W`/`MAX_H` lấy từ config thay vì hardcode rải trong method.

**🔴 Cách tự kiểm chứng (bắt buộc):**
- A: Upload 1 GIF động thật, gọi `/resize/storage/.../animated.gif?w=200`, mở file output bằng tool
  xem ảnh/`exiftool`/`identify -format %n` (ImageMagick) để đếm số frame TRƯỚC và SAU khi qua route —
  không kết luận "mất animation" chỉ từ đọc code Intervention Image (version khác nhau hành vi có thể
  khác).
- B: Đo RAM/CPU thật bằng `w=10000&h=10000` trên staging (dùng `memory_get_peak_usage()` log lại, hoặc
  `top`/`htop` khi gọi), so với baseline `w=300&h=200`, để có số liệu cụ thể trước khi quyết định
  `MAX_W`/`MAX_H` hợp lý (không bịa con số).

---

### 2.4 🟠 `Storage::exists($webpPath)` gọi disk I/O không cache trong `ThumbnailMedia::url()`

**Vị trí:** `src/ThumbnailMedia.php:113-124`.

**Vấn đề:** mỗi lần `url()` được gọi (tức mỗi ảnh render trong 1 trang — có thể hàng chục lần/trang
danh sách bài viết) với local storage, code check `Storage::exists($webpPath)` — 1 lượt I/O đĩa (hoặc
`stat()` syscall) không được cache, trong khi `PublicController::resize()` ở cạnh đó đã có pattern
cache metadata qua `apps_cache_get`/`apps_cache_store` (dòng 51-65) cho đúng loại bài toán này. Đây là
thiếu nhất quán trong cùng package — một nơi đã tối ưu, một nơi quên.

**Đề xuất:** cache kết quả `Storage::exists()` theo key `webp_exists:{webpPath}:{mtime hoặc version}`,
TTL ngắn hơn (vài phút, không cần 30 ngày như metadata vì file có thể được convert webp sau), dùng lại
helper `apps_cache_get`/`apps_cache_store` đã có sẵn trong hệ thống.

**🟠 Cách tự kiểm chứng (bắt buộc):**
- KHÔNG tự tin "cache chắc nhanh hơn". Đo thật: dùng `Benchmark::measure()` của Laravel hoặc
  `microtime(true)` quanh lời gọi `ThumbnailMedia::url()` lặp 1000 lần trên 1 trang list thật (có N
  ảnh), so sánh tổng thời gian TRƯỚC/SAU khi thêm cache, trên môi trường storage thật đang dùng (local
  disk vs NFS vs mount network — chi phí `Storage::exists()` khác nhau rất nhiều tùy hạ tầng, đừng suy
  diễn từ máy dev).
- Nếu cache TTL ngắn làm stale (ảnh mới convert webp xong nhưng cache vẫn báo chưa có) → kiểm tra có
  gây regression UX không trước khi merge.

---

### 2.5 🟠 `getMimeType()`/`getRealPath()` (class cha) bị gọi 2 lần trong `getImageUrl()` cho cùng mục đích, trên 2 giá trị `$url` khác nhau

**Vị trí:** `src/ThumbnailMedia.php:51-54` và `:67`.

**Vấn đề:**
```php
// lần 1 — $url còn nguyên (chưa gắn suffix size)
$this->canGenerateThumbnails($this->getMimeType($this->getRealPath($url)))
...
$url = str_replace($fileName.'.'.$fileExtension, $fileName.'-'.$this->getSize($size).'.'.$fileExtension, $url);
...
// lần 2 — $url ĐÃ bị đổi thành "image-300x200.jpg" ở trên
if ($size && $this->canGenerateThumbnails($this->getMimeType($this->getRealPath($url))) && ...)
```
Hai điều cần agent thực thi tự xác nhận (không đọc code suy ra chắc được, vì phụ thuộc implementation
thật của `getRealPath()`/`getMimeType()` ở class cha, vốn không có trong repo này):
1. **Chi phí**: nếu `getRealPath()` đụng filesystem (`realpath()`/`file_exists()`), gọi 2 lần = 2 lượt
   I/O cho cùng 1 request render ảnh — tốn kém nếu nhân với số ảnh/trang.
2. **Correctness** (quan trọng hơn chi phí): lần gọi thứ 2 dùng `$url` đã bị nối thêm `-{size}` vào
   tên file (dòng 59-63) — tức đang check mime/tồn tại của 1 file **có thể chưa từng được sinh ra**
   (file thumbnail tĩnh dạng `image-300x200.jpg` chỉ tồn tại nếu `AppMedia`/`ThumbnailService` đã từng
   generate sẵn — static thumbnail convention). Nếu file đó chưa tồn tại, `getRealPath()` có thể trả
   về `false`/path gốc tùy implementation → `canGenerateThumbnails` nhận mime sai hoặc rỗng → nhánh
   query-string (`?w=&h=`) phía dưới (dòng 66-84) có thể không được thêm vào dù đáng ra phải có, hoặc
   ngược lại. **Đây có thể là nguồn gốc của lỗi URL kép mô tả ở mục 2.6** — cần verify cùng lúc.

**Đề xuất:** tính `$realPath = $this->getRealPath($url)` và `$mimeType = $this->getMimeType($realPath)`
đúng 1 lần, TRƯỚC khi `$url` bị đổi tên ở dòng 59-63, rồi dùng lại biến đó cho điều kiện thứ 2 — nhưng
**chỉ làm vậy nếu agent xác nhận được ở mục 2.6 rằng việc dùng `$url` gốc (chưa suffix) cho cả 2 lần
check không làm thay đổi hành vi hiện tại** (vì hiện tại lần 2 đang dùng `$url` đã suffix — có thể đó
là chủ đích, không phải bug, chỉ là code trùng lặp vô tình đúng).

**🟠 Cách tự kiểm chứng (bắt buộc, làm TRƯỚC khi sửa 2.5 và 2.6):**
- Đọc source thật `Dev\Media\AppMedia::getRealPath()`/`getMimeType()`/`canGenerateThumbnails()`/
  `getSizes()` ở 1 project CMS có cài plugin (không có trong repo audit này — xem mục 0.1).
- Viết unit test gọi `ThumbnailMedia::getImageUrl($url, '300x200')` với 1 file ảnh thật tồn tại và có
  `getSizes()` chứa key `'300x200'`, in ra URL kết quả, so với URL kỳ vọng theo README (mục 7, ví dụ
  2) — xác nhận hành vi HIỆN TẠI đúng như mô tả trước khi đụng vào, để có baseline revert nếu sai.

---

### 2.6 🔴 (cần verify) Nguy cơ URL bị biến đổi 2 lớp chồng chéo khi `$size` khớp cả `getSizes()` và regex `WxH`

**Vị trí:** `src/ThumbnailMedia.php:51-64` (nhánh đổi tên file theo size có sẵn trong `getSizes()`) +
`:66-84` (nhánh regex thêm query string `?w=&h=`) chạy **nối tiếp, không loại trừ nhau** — nếu `$size`
(ví dụ `"300x200"`) vừa khớp `array_key_exists($size, $this->getSizes())` VÀ vừa khớp regex `WxH`, cả
2 nhánh đều chạy trên cùng 1 request.

**Giả thuyết bug (CHƯA XÁC NHẬN — đây đúng là loại nhận định mà mục 0 yêu cầu không tin tuyệt đối):**
Kết quả cuối có thể là `storage/news/image-300x200.jpg?w=300&h=200` — vừa đổi tên file theo size tĩnh,
vừa thêm query string cho route resize động. Khi URL này tiếp tục qua `url()` (`:102-174`):
- Nếu file tĩnh `image-300x200.jpg`/`.webp` **đã tồn tại thật** (do `AppMedia` đã generate sẵn lúc
  upload) → phần `?w=300&h=200` vẫn bị dính vào URL cuối cùng một cách vô nghĩa (ảnh tĩnh không cần
  query resize) — không gây lỗi nhưng là URL thừa, có thể phá cache CDN theo query string.
- Nếu file tĩnh **chưa tồn tại** → `url()` thấy path có `?` → rewrite `storage/` → `resize/storage/`
  (dòng 159-165) → route `thumbnail.resize` nhận `$slug = "storage/news/image-300x200.jpg"` — nhưng
  **file nguồn thật trên đĩa là `image.jpg`, không phải `image-300x200.jpg`** → `PublicController::resize()`
  dòng 21 `file_exists()` trả `false` → fallback sang `default_image` → **người dùng thấy ảnh mặc định
  thay vì ảnh thật**, với kích thước bất kỳ đã khai báo trong `getSizes()`.

**Đề xuất:** KHÔNG đề xuất sửa code ở đây — đây là loại lỗi cần xác nhận bằng request thật trước, vì
sửa nhầm giả thuyết có thể làm hỏng 1 luồng đang hoạt động đúng mà audit tĩnh hiểu sai.

**🔴 Cách tự kiểm chứng (BẮT BUỘC, mức độ ưu tiên cao nhất của toàn bộ tài liệu):**
1. Trên CMS thật có cài plugin, tìm/tạo 1 size key nằm trong `getSizes()` của `AppMedia` (ví dụ xem
   `config/media.php` hoặc setting tương ứng) mà tên trùng định dạng `WxH` (ví dụ `"300x200"`).
2. Gọi `ThumbnailMedia::getImageUrl($fileUrlThật, '300x200')` với 1 ảnh **chưa từng được truy cập qua
   size này trước đó** (để chắc chưa có cache/file tĩnh nào được tạo sẵn).
3. In ra URL trả về, mở URL đó bằng `curl -i` thật, xem có trả ảnh đúng hay bị fallback default image.
4. Lặp lại bước 2-3 với 1 size key có format khác (ví dụ `"300xauto"`, không khớp `getSizes()`) để so
   sánh — xác nhận đúng giả thuyết lỗi chỉ xảy ra khi `$size` khớp ĐỒNG THỜI cả 2 điều kiện.
5. Chỉ sau khi có kết quả request thật xác nhận hoặc bác bỏ giả thuyết, mới quyết định có sửa logic ở
   `getImageUrl()` (ví dụ: 2 nhánh nên loại trừ nhau bằng `elseif`) hay không, và viết lại test case
   tái hiện trước khi sửa.

---

### 2.7 🟠 Regex `preg_match_all('/(.*[0-9|auto])x(.*[0-9|auto])/m', $size, ...)` — sai cú pháp character class, chạy regex engine cho input cố định dạng đơn giản

**Vị trí:** `src/ThumbnailMedia.php:66`.

**Vấn đề (correctness, không chỉ performance):** `[0-9|auto]` là một **character class**, nghĩa là
"1 trong các ký tự: `0`-`9`, `|`, `a`, `u`, `t`, `o`" — **không phải** "chữ số HOẶC từ auto" như ý
định. Với `"300x200"`/`"300xauto"`/`"autox200"` pattern vẫn match đúng chỉ vì ký tự cuối mỗi vế (`0`,
`o`) tình cờ nằm trong class. Ngoài ra pattern không neo `^...$`, nên input như `"300x200px"` hay
`"abcx1"` vẫn match một phần và sinh ra `w`/`h` không phải số (ví dụ `w=abc`). Kết luận: regex đúng
với input hợp lệ hiện tại là do trùng hợp, không phải do viết đúng — agent phải tự chạy bảng test để
xác nhận, không tin đoạn phân tích này.

**Đề xuất:** thay bằng parser đơn giản, nhanh hơn và rõ ý hơn regex:
```php
[$w, $h] = array_pad(explode('x', (string) $size, 2), 2, null);
// rồi so sánh $w !== 'auto', $h !== 'auto' trực tiếp
```
Loại bỏ hoàn toàn regex engine cho 1 bài toán tách chuỗi theo dấu `x` — rẻ hơn về CPU và dễ đọc hơn.

**🟠 Cách tự kiểm chứng (bắt buộc):**
- Viết bảng test input/output cho CẢ 2 cách (regex cũ vs `explode` mới) với toàn bộ size format đang
  dùng thật trong hệ thống (lấy từ `getSizes()` thật + mọi nơi gọi `getImageUrl()` trong codebase CMS
  tiêu thụ package này — `grep -rn "getImageUrl(" --include=*.blade.php --include=*.php`), xác nhận
  2 cách cho **CÙNG kết quả 100%** trước khi thay. Không suy diễn tương đương bằng mắt.
- Benchmark `explode` vs `preg_match_all` bằng `microtime()` lặp 100k lần — ghi số liệu cụ thể (dù
  khả năng chênh lệch rất nhỏ với 1 lần gọi/request, nên **đừng tự PR rằng đây là "tối ưu tốc độ lớn"**
  nếu số đo không chứng minh được; coi đây chủ yếu là sửa correctness + đọc code dễ hơn).

---

### 2.8 🟠 `url()` tự string-replace `storage/` → `resize/storage/` thay vì dùng route thật; trùng logic parse path với `getImageUrl()`

**Vị trí:** `src/ThumbnailMedia.php:148-166`.

**Vấn đề:** URL route resize bị **hardcode lại bằng string** (`str_replace(['storage/', '/storage/'], ['resize/storage/', '/resize/storage/'], ...)`) thay vì gọi `route('thumbnail.resize', ...)` — route này đã
có tên tường minh ở `routes/web.php:142` (`->name('thumbnail.resize')`). Nếu sau này đổi prefix route
(`resize/` → cái khác) hoặc đổi tên route, chỗ này sẽ âm thầm sai mà không có lỗi biên dịch nào báo.

**Đề xuất:** dùng `route('thumbnail.resize', ['slug' => $purePath])` kèm query string, thay cho string
replace thủ công. Giảm 1 điểm trùng lặp kiến thức (DRY) giữa route definition và nơi build URL.

**🟠 Cách tự kiểm chứng (bắt buộc):**
- Xác nhận `route('thumbnail.resize', [...])` sinh ra ĐÚNG y hệt string hiện tại cho mọi path test case
  thật (so sánh string-for-string), đặc biệt với path có/không có leading slash, có/không có query —
  vì route helper có thể encode ký tự khác với string replace thủ công (ví dụ encode `/` trong slug).
  Nếu khác, giữ string replace và chỉ record lại rủi ro trong code comment, không tự đổi.

---

### 2.9 🟡 Dead code: `return '';` sau `try { ... } catch (...) { return ...; }`

**Vị trí:** `src/ThumbnailMedia.php:99`, `:173`, `:200`.

**Vấn đề:** Các method `getImageUrl()`, `url()`, `deleteThumbnails()` đều có 1 dòng `return '';`
(hoặc `return false;`) ngay sau khối `try/catch` đã return đầy đủ ở cả 2 nhánh — dòng này **không thể
reachable**, static analyzer (PHPStan/Psalm) nếu được thêm vào CI sẽ report dead code.

**Đề xuất:** xóa 3 dòng dead code này. Rủi ro thay đổi hành vi = 0 (code không bao giờ chạy tới được).

**🟡 Cách tự kiểm chứng:** chạy PHPStan/Psalm mức cao trên file trước/sau để xác nhận không còn warning
unreachable code; chạy lại toàn bộ test suite hiện có (nếu đã được viết thêm theo mục 2.1-2.6) để
xác nhận 0 thay đổi hành vi.

---

### 2.10 🟡 `apps_get_thumbnail_generation()` đã `@deprecated since 5.7` nhưng vẫn ship, gọi hàm ngoài `get_object_image()` không tồn tại trong repo này

**Vị trí:** `helpers/helpers.php:10-43`.

**Vấn đề:** hàm bị đánh dấu deprecated, gọi `get_object_image()` — hàm này **không được định nghĩa ở
đâu trong repo này**, phải tồn tại ở core CMS tiêu thụ package. Giữ code chết/deprecated làm tăng diện
tích phải đọc khi audit, và nếu core đổi tên `get_object_image()`, hàm deprecated này sẽ fatal error
khi gọi (nhưng vì deprecated nên khả năng còn ai gọi là thấp).

**Đề xuất:** xác nhận không còn call site nào dùng `apps_get_thumbnail_generation()` trong toàn bộ các
site đang cài package (grep across mọi project CMS tiêu thụ, KHÔNG chỉ repo này), rồi xóa hẳn. Nếu còn
dùng, giữ nguyên, không tự xóa.

**🟡 Cách tự kiểm chứng (bắt buộc trước khi xóa):**
- `grep -rn "apps_get_thumbnail_generation" .` trên **từng site CMS thật** đang require package này
  (không chỉ repo package) — vì package này được nhiều project composer require riêng, repo package
  tự nó không có call site (đã grep ở đầu phiên audit: chỉ có definition, không có nơi gọi — nhưng đó
  chỉ là trong repo NÀY, không đại diện cho mọi site tiêu thụ).

---

### 2.11 🟡 `get_max_mimesizes()` decode JSON từ `setting()` mỗi lần gọi, không cache trong request

**Vị trí:** `helpers/helpers.php:49-54`.

**Vấn đề:** `json_decode(setting(...))` chạy lại mỗi lần hàm được gọi. Nếu `setting()` tự cache
(thường đúng với Botble — setting có cache layer riêng), chi phí thật chỉ là `json_decode` nhỏ, không
đáng kể. Đây là candidate tối ưu **thấp giá trị** — không nên ưu tiên trừ khi đo được hàm này gọi rất
nhiều lần/request (ví dụ trong loop render nhiều field).

**Đề xuất:** memoize bằng static variable trong function nếu đo được gọi > 1 lần/request thật.

**🟡 Cách tự kiểm chứng (bắt buộc — và đây là ví dụ mẫu cho "đừng tối ưu khi chưa đo"):**
- Grep toàn bộ call site `get_max_mimesizes()` trong cả package lẫn site tiêu thụ, đếm số lần gọi
  thực tế trong 1 request (ví dụ qua Laravel Debugbar hoặc Telescope query/log count), chỉ tối ưu nếu
  số lần gọi > 1 và `setting()` xác nhận KHÔNG tự cache ở layer dưới.

---

### 2.12 🟡 Thiếu test suite hoàn toàn

**Vị trí:** toàn repo — không có `tests/`.

**Vấn đề:** mọi đề xuất ở mục 2.1–2.8 đều thay đổi logic nhánh điều kiện phức tạp. Theo đúng tinh thần
skill `botble-plugin` ("Mỗi sửa tạo/filter/parse LOGIC → testcase"), áp dụng bất kỳ đề xuất nào ở trên
mà không có test tái hiện hành vi cũ trước là rủi ro cao cho 1 package được nhiều site dùng chung.

**Đề xuất:** thêm `tests/Unit/ThumbnailMediaTest.php` (test `getImageUrl()`/`url()` với bảng input/
output cho mọi combination size đang dùng thật) và `tests/Feature/PublicControllerResizeTest.php`
(test route thật: file tồn tại/không tồn tại, webp có/không, cache hit/miss, w/h tường minh/tự tính,
và 2 test case của mục 2.1/2.2 nếu xác nhận là lỗ hổng thật).

**🟡 Cách tự kiểm chứng:** không áp dụng — đây chính là cơ chế kiểm chứng cho các mục khác. Agent thực
thi nên coi việc viết test này là **bước 0** trước khi đụng vào mục 2.1–2.8, không phải việc làm sau
cùng.

---

### 2.13 ✅ (đã sửa) Backblaze CDN không có custom domain làm mất host của URL

**Vị trí:** `src/ThumbnailMedia.php`, nhánh `media_backblaze_cdn_enabled` trong `url()`.

**Lỗi cũ:** khi CDN Backblaze bật nhưng `media_backblaze_cdn_custom_domain` trống, code gọi
`str_replace($currentEndpoint, $customDomain, Storage::url($path))` tức là thay endpoint bằng chuỗi
rỗng → URL còn lại dạng `/bucket/...`, trình duyệt tải từ domain của site → ảnh lỗi.

**Đã sửa:** trả `Storage::url($path)` (URL gốc của Backblaze) khi không có custom domain.

**Agent tích hợp phải tự kiểm:** trên site dùng Backblaze, bật CDN, để trống custom domain, xem `src`
ảnh trả ra có đúng host Backblaze và tải được không.

### 2.14 ✅ (đã sửa) Đường dẫn WebP thay phần mở rộng ở mọi vị trí trong path

**Vị trí:** `src/Http/Controllers/PublicController.php`, `resize()`.

**Lỗi cũ:** `str_replace('.' . $ext, '.webp', $path)` thay mọi chỗ `.jpg` trong toàn đường dẫn, kể cả
tên thư mục (`photos.jpg/a.jpg` → `photos.webp/a.webp`) → không tìm thấy bản WebP có thật.

**Đã sửa:** `preg_replace('/\.[^.\/]+$/', '.webp', $path)` — chỉ thay phần mở rộng cuối cùng. Với
đường dẫn thường, kết quả giống hệt cũ.

## 3. KHÔNG đề xuất (và vì sao) — để tránh agent thực thi tự suy diễn thêm việc

- **Không đề xuất đổi cơ chế cache disk (`public/resize/...`) sang cache driver khác (Redis/S3)** —
  chưa có số đo nào cho thấy disk cache là bottleneck; đổi cơ chế cache là thay đổi kiến trúc lớn,
  ngoài phạm vi "tối ưu logic hiện có" mà user yêu cầu.
- **Không đề xuất đổi `flock()` sang cơ chế lock khác (Redis lock, etc.)** — cơ chế file lock hiện tại
  đúng cho single-server; nếu hệ thống đang chạy nhiều server (load balancer), đây MỚI là vấn đề thật
  (file lock không chia sẻ giữa nhiều máy) — nhưng agent thực thi phải tự xác nhận hạ tầng thật có
  multi-server hay không trước khi coi đây là việc cần làm, tài liệu này không giả định.
- **Không đề xuất sửa core/vendor** (`Dev\Media\AppMedia`...) — đúng RULE #1 skill `botble-conventions`,
  package này chỉ nên override qua class riêng (đang làm đúng), không chạm vendor.

---

## 4. Thứ tự triển khai đề xuất (nếu agent thực thi được giao làm thật, theo lượt sau)

1. Viết test baseline (mục 2.12) cho hành vi HIỆN TẠI của `getImageUrl()`, `url()`, `resize()`.
2. Kiểm chứng mục 2.6 bằng request thật — đây là phát hiện có khả năng ảnh hưởng user thật cao nhất,
   phải làm trước mọi tối ưu tốc độ.
3. Kiểm chứng + xử lý 2.1, 2.2, 2.3 (security/correctness).
4. Áp dụng 2.4, 2.5, 2.7, 2.8 (performance), đo trước/sau từng cái, không gộp nhiều thay đổi performance
   vào 1 commit để còn biết thay đổi nào gây hiệu quả gì.
5. Dọn 2.9, 2.10, 2.11 (cleanup, rủi ro thấp nhất).

## 5. Phản biện lại chính báo cáo này (agent thực thi phải tự làm lại bước này, không chỉ đọc)

Trước khi tin bất kỳ mục nào ở trên, tự hỏi: "Nếu tôi là người review PR áp dụng đề xuất này, tôi sẽ
hỏi gì để bắt bẻ?" — ví dụ: "Đã test trên ảnh KHÔNG tồn tại chưa?", "Đã test với `isUsingCloud() ===
true` chưa (để chắc nhánh cloud không bị ảnh hưởng bởi thay đổi ở nhánh local)?", "Số đo benchmark lấy
trên máy dev hay trên hạ tầng thật (storage driver thật, network latency thật)?". Nếu không tự trả lời
được bằng số liệu/log thật, đề xuất đó CHƯA được coi là verified, dù đã được ghi trong tài liệu này.
