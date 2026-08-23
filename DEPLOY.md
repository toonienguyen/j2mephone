# Deploy phoneME Web lên Cloudflare Pages (không cần terminal)

Emulator cần biên dịch `Core/` sang WebAssembly trước khi deploy. Cả 2 cách
dưới đây đều **không cần mở terminal hay GitHub Desktop** — chỉ thao tác trên
trình duyệt (github.com và dash.cloudflare.com):

- **Luồng 1 — GitHub Actions** (khuyên dùng): dán 1 file `.yml` vào repo qua
  web GitHub một lần. Từ đó, mỗi lần sửa code và push (kể cả sửa trực tiếp
  trên github.com), GitHub tự động build và deploy thẳng lên Cloudflare
  Pages — không cần quay lại Dashboard nữa.
- **Luồng 2 — Google Colab + Direct Upload**: chạy Colab (trên web) để build
  ra `dist/`, tải zip về máy, giải nén, kéo thả thủ công lên Cloudflare Pages
  mỗi lần muốn cập nhật.

Cả 2 luồng đều bỏ qua phần `socket://`/websockify proxy — chỉ cần nếu chơi
game J2ME có kết nối online (multiplayer, chat...). Chơi offline thì không
cần quan tâm `Dockerfile.websockify`, `websockify-entrypoint.sh`,
`host_port_token.py`.

---

## Luồng 1: GitHub Actions

### Bước 1 — Tạo API Token trên Cloudflare

1. [dash.cloudflare.com](https://dash.cloudflare.com) → avatar góc phải →
   **My Profile** → tab **API Tokens** → **Create Token**
2. Chọn **Create Custom Token** (không dùng template có sẵn — template "Edit
   Cloudflare Workers" không đủ quyền upload cho Pages)
3. Đặt tên bất kỳ, ví dụ `phoneme-pages-deploy`
4. Ở **Permissions**, chọn đúng 3 ô theo thứ tự:
   `Account` → `Cloudflare Pages` → **`Edit`** (không để mặc định "Read")
5. Ở **Account Resources**: `Include` → chọn account chứa project Pages của
   bạn
6. Phần **TTL** để trống (không bắt buộc chọn ngày hết hạn — để trống nghĩa
   là token không tự hết hạn, phù hợp cho deploy tự động dài hạn; nếu muốn an
   toàn hơn có thể đặt hạn, nhưng phải nhớ gia hạn kẻo Actions sẽ lỗi
   authentication khi token hết hạn)
7. **Continue to summary** → **Create Token**
8. Copy chuỗi token hiện ra (chỉ hiện đúng 1 lần)

### Bước 2 — Lấy Account ID

Trang chủ Dashboard → nhìn cột phải của bất kỳ trang tổng quan domain/account
nào — thấy **Account ID**, copy lại.

### Bước 3 — Tạo project Pages (nếu chưa có)

Nếu bạn đã từng **Connect to Git** trên Cloudflare Pages trước đó rồi, project
đã tồn tại — bỏ qua bước này, chỉ cần nhớ đúng **tên project** (xem trong
breadcrumb ở đầu trang project, ví dụ `phoneme-ios`).

Nếu chưa có project nào:
1. Dashboard → **Workers & Pages** → **Create** → tab **Pages** →
   **Connect to Git** → chọn repo `phoneME-iOS`
2. Đặt **Project name** tuỳ ý (nhớ tên này để dùng ở Bước 5)
3. Build command/output không quan trọng — GitHub Actions sẽ deploy thay,
   không dùng Git integration này để build
4. Sau khi tạo xong, vào **Settings → Build → Branch control**, tắt
   **Automatic deployments** để tránh Cloudflare Git integration tự build
   song song với GitHub Actions (nó sẽ lỗi vì thiếu Emscripten, gây nhiễu log)

### Bước 4 — Lưu 2 secret vào GitHub repo

1. Repo `phoneME-iOS` trên github.com → **Settings** → menu trái
   **Secrets and variables** → **Actions**
2. **New repository secret**, tạo lần lượt:

| Name | Giá trị |
|---|---|
| `CLOUDFLARE_API_TOKEN` | token đã copy ở Bước 1 |
| `CLOUDFLARE_ACCOUNT_ID` | account ID đã copy ở Bước 2 |

### Bước 5 — Sửa `web/wrangler.toml` cho khớp tên project thật

Mở `web/wrangler.toml` trong repo (bấm bút chì để sửa), đảm bảo `name` khớp
đúng tên project Cloudflare thật (xem lại Bước 3), ví dụ:

```toml
name = "phoneme-ios"
pages_build_output_dir = "./dist"
compatibility_date = "2026-08-08"
```

Commit thẳng vào `main`.

### Bước 6 — Tạo file workflow

1. Repo → **Add file** → **Create new file**
2. Đặt tên file: `.github/workflows/deploy.yml` (gõ dấu `/`, GitHub tự tạo
   thư mục con)
3. Dán nội dung:

```yaml
name: Build và Deploy Cloudflare Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: macos-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Cài Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Cài CMake
        run: brew install cmake

      - name: Cài Emscripten
        uses: mymindstorm/setup-emsdk@v14
        with:
          version: latest

      - name: Kiểm tra emcc
        run: emcc --version

      - name: Cài npm packages
        working-directory: web
        run: npm install

      - name: Build và Deploy (build:wasm → tsc → vite build → wrangler)
        working-directory: web
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
        run: |
          npm run build
          npx -y wrangler@4.120.0 pages deploy dist --branch=main
```

4. **Commit changes...** → commit thẳng vào `main`

Workflow chạy ngay sau commit. Theo dõi tiến trình ở tab **Actions**.

### Vì sao workflow viết như vậy (đã đụng lỗi thật, giữ lại để tránh lặp)

- **`runs-on: macos-latest`** (không phải `ubuntu-latest`): script build gọi
  lệnh `sips` (Scriptable Image Processing System) để convert icon PWA — công
  cụ hệ thống chỉ có trên macOS. Chạy Linux sẽ lỗi
  `ENOENT ... spawnSync /usr/bin/sips`. Không tốn phí vì **repo public được
  GitHub Actions cho chạy không giới hạn phút, kể cả macOS runner**.
- **`brew install cmake`** (không phải `apt-get`): vì runner đã đổi sang
  macOS, phải dùng trình quản lý gói của macOS.
- **`node-version: 22`** (không phải 20): Wrangler `4.120.0` (khoá cứng trong
  `web/package.json`) yêu cầu tối thiểu Node.js 22, báo lỗi
  `Wrangler requires at least Node.js v22.0.0` nếu dùng bản thấp hơn.
- **Gọi `wrangler pages deploy dist` trực tiếp** (không dùng `npm run deploy`
  có sẵn trong `package.json`): để kiểm soát rõ project name khớp thực tế,
  tránh phụ thuộc vào tên `phoneme` mặc định trong `wrangler.toml` nếu project
  thật trên Cloudflare có tên khác.
- **Token phải là Custom Token với quyền `Cloudflare Pages: Edit`**: dùng
  quyền `Read` hoặc template có sẵn sẽ gặp lỗi
  `Authentication error [code: 10000]` khi Wrangler cố upload.
- **`functions/api`** nằm trong `web/functions/api`. Vì lệnh `wrangler pages
  deploy` chạy trong `working-directory: web`, Wrangler tự nhận diện
  `functions/` cạnh `dist/` và deploy kèm Pages Function — không cần cấu hình
  thêm.

### Từ giờ về sau, mỗi lần muốn cập nhật:

- Sửa file trong repo qua giao diện web github.com (bút chì → sửa →
  **Commit changes**)
- Theo dõi tab **Actions** (mất khoảng 8-10 phút vì phải cài Emscripten mỗi
  lần trên macOS runner)
- Build xong tự động deploy, không cần làm gì thêm

---

## Luồng 2: Google Colab + Direct Upload (build tay từng lần)

Phù hợp nếu muốn kiểm soát từng lần deploy, không muốn tự động hoá. Nhược
điểm: mỗi lần cập nhật code phải lặp lại toàn bộ các bước dưới, và Colab xoá
hết dữ liệu khi đóng tab.

### Bước 1 — Mở Colab và chạy build

[colab.research.google.com](https://colab.research.google.com) → **New
notebook**. Dán từng khối lệnh vào từng ô (cell) riêng, chạy lần lượt bằng
nút play (▶):

**Ô 1 — Clone repo:**
```python
!git clone https://github.com/phd051199/phoneME-iOS.git
%cd phoneME-iOS
```

**Ô 2 — Cài Emscripten (mất vài phút):**
```python
!git clone https://github.com/emscripten-core/emsdk.git /content/emsdk
%cd /content/emsdk
!./emsdk install latest
!./emsdk activate latest
```

**Ô 3 — Nạp biến môi trường Emscripten:**
```python
import os
emsdk_env = !bash -c "source /content/emsdk/emsdk_env.sh && env"
for line in emsdk_env:
    if '=' in line:
        k, v = line.split('=', 1)
        os.environ[k] = v
!emcc --version
```
Thành công nếu ô in ra số phiên bản `emcc`.

**Ô 4 — Cài npm packages và build:**
```python
%cd /content/phoneME-iOS/web
!npm install
!npm run build
```
Build `Core/` sang Wasm → copy vào `web/public/wasm` → type-check → build
frontend vào `web/dist`. Mất khoảng 5-15 phút.

> Lưu ý: Colab chạy Linux, cũng sẽ gặp lỗi `sips` giống GitHub Actions ở trên
> nếu script build gọi lệnh đó — Colab không có sẵn `sips` (chỉ macOS mới
> có). Nếu Ô 4 lỗi vì `sips`, đây là giới hạn không tránh được của Colab; lúc
> đó nên dùng Luồng 1 (chạy trên `macos-latest`) thay vì Colab.

**Ô 5 — Nén kết quả và tải zip về máy:**

Game online của bạn dùng HTTP bridge (`/api/http`) chứ không phải raw
`socket://`, nên vẫn cần giữ `functions/`:
```python
%cd /content/phoneME-iOS
!zip -r phoneme-build.zip web/dist web/functions

from google.colab import files
files.download('phoneme-build.zip')
```

### Bước 2 — Giải nén trên máy Windows

Chuột phải vào `phoneme-build.zip` → **Extract All...** — sẽ có
`phoneme-build/web/dist/` (file tĩnh: HTML, JS, CSS, wasm, `_headers`) và
`phoneme-build/web/functions/` (HTTP bridge cho game online).

### Bước 3 — Direct Upload lên Cloudflare Pages

1. [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages**
2. Chưa có project: **Create** → tab **Pages** → **Upload assets** → đặt tên
   project tuỳ ý
3. Đã có project từ trước: vào project đó → tab **Deployments** →
   **Create deployment** → **Upload assets**
4. Kéo thả **nội dung bên trong** thư mục `dist` (không kéo chính thư mục
   `dist`) vào vùng thả file
5. **Deploy site**

> **Quan trọng vì bạn chơi game online:** Direct Upload qua kéo-thả trên
> Dashboard **không có cách chuẩn để đính kèm `functions/`** — giao diện chỉ
> nhận 1 thư mục static assets. Nếu deploy theo cách này, HTTP bridge
> (`/api/http`) sẽ **không hoạt động**, game online có thể lỗi kết nối dù
> UI vẫn load bình thường. Với nhu cầu chơi online, **nên dùng Luồng 1
> (GitHub Actions)** thay vì Direct Upload, vì `wrangler pages deploy` chạy
> từ dòng lệnh (kể cả trong CI) tự nhận diện và deploy kèm `functions/` đúng
> cách — điều Direct Upload qua trình duyệt không đảm bảo được.

---

## Nên chọn luồng nào?

| | Luồng 1: GitHub Actions | Luồng 2: Colab + Direct Upload |
|---|---|---|
| Cần terminal? | Không | Không |
| Setup ban đầu | Nhiều bước hơn (token, secrets, sửa `wrangler.toml`) | Ít bước hơn |
| Cập nhật lần sau | Chỉ sửa file trên github.com, tự động deploy | Làm lại toàn bộ từ Colab mỗi lần |
| Rủi ro lỗi `sips` | Không (chạy `macos-latest`) | Có (Colab là Linux) |
| Phù hợp khi | Sẽ cập nhật code thường xuyên | Chỉ build 1-2 lần |

Nếu không chắc, chọn **Luồng 1** — set 1 lần, sau đó chỉ cần sửa code qua web.

---

## Phần socket (kết nối online của game)

Game của bạn đang chơi được online là nhờ **HTTP bridge** (`web/functions/api`),
xử lý các request `http://`/`https://` từ MIDlet qua Pages Function cùng
origin — không phải qua websockify/Docker proxy. Bridge này đã được deploy
kèm tự động ở Luồng 1, nên không cần làm gì thêm.

Phần `Dockerfile.websockify`/`websockify-entrypoint.sh`/`host_port_token.py`
gần như chắc chắn **không liên quan đến bản web** bạn đang deploy. Lý do:
trình duyệt không cho phép mở raw TCP/UDP socket trực tiếp — đây là giới hạn
nền tảng của web, không phải của phoneME. Nên phần proxy này tồn tại trong
repo chủ yếu để phục vụ trường hợp hiếm: giả lập `socket://` trên bản web
bằng WebSocket + proxy phía server.

Socket thật sự (raw TCP qua hệ điều hành) chỉ có ý nghĩa với **bản app iOS**
(`phoneME.xcodeproj`), vì Core C++23 chạy trực tiếp trên thiết bị, gọi thẳng
socket API của iOS mà không bị giới hạn như trình duyệt. Vì bạn chỉ deploy
bản web lên Cloudflare Pages, khả năng cao bạn sẽ **không bao giờ cần** đến
phần websockify/Render này — trừ khi gặp đúng một game hiếm dùng `socket://`
trên web và không có cách thay thế qua HTTP.
