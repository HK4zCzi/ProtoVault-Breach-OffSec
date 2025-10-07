# ProtoVault-Breach-OffSec

# Chuẩn bị


PASS: BloodBreathSoulFire


Đề bài cho ta 1 fize zip, giải nén sẽ được các folder trên, bài đang bảo rằng manh mối đầu tiên là email, hãy xem nó thử
<img width="663" height="90" alt="image" src="https://github.com/user-attachments/assets/53335be6-f605-4e71-89bf-b0b69a4fc13f" />

# FLAG 1
Ảnh trong email cho thấy một đoạn trích dữ liệu SQL:

```js
COPY public.item_types (id, name, description) FROM stdin;
1  Biological Samples  Preserved specimens of rare or hazardous organisms
2  Prototype Devices   Experimental technologies under development
3  Confidential Documents  Sensitive research reports
4  Hazardous Chemicals Toxic, corrosive, or otherwise dangerous substances
```
Mục đích là chứng minh rằng kẻ tấn công có quyền truy cập dữ liệu thực, để tăng tính thuyết phục cho lời đe dọa.

<img width="1073" height="833" alt="ransom_email" src="https://github.com/user-attachments/assets/8a7b15f9-7f3e-42ef-9a08-3840891a901d" />



Ảnh tống tiền cho thấy PostgreSQL dump (COPY public.item_types (id, name, description) FROM stdin;) => Trong repo chắc chắn có cấu hình/chuỗi kết nối PostgreSQL và có thể có script backup (dễ gây rò rỉ).

Ta không biết ở commit nào lộ bí mật. git grep + git rev-list --all cho phép tìm mọi commit.

<img width="1428" height="428" alt="image" src="https://github.com/user-attachments/assets/6cbec8f9-ab2e-4d3c-a182-7d79509d17e5" />

Ồ, ta có thể thấy gần đây thì mới dùng host sản xuất, user riêng, sslmode=verify-full (bắt buộc SSL và xác thực CN) còn trước đó thì lại để một chuỗi kém an toàn: không SSL, lộ full password, host 127.0.0.1. Điều này cho thấy repo từng chứa thông tin nhạy cảm -> tăng khả năng rò rỉ đến từ ứng dụng / quy trình của team.

Vậy nên sau khi tổng hợp lại thông tin, đáp án câu đầu tiên sẽ là
```js
Yes, postgresql://assetdba:8d631d2207ec1debaafd806822122250@pgsql_prod_db01.protoguard.local/pgamgt?sslmode=verify-full
```

<img width="1548" height="115" alt="image" src="https://github.com/user-attachments/assets/d2ada4d0-edb4-477b-92f6-c10b4d96b3ce" />

# FLAG 2
Xác định file trong mã nguồn có khả năng làm rò rỉ CSDL (có thể là từ backup → đưa lên public)

Ảnh tống tiền


<img width="648" height="282" alt="image" src="https://github.com/user-attachments/assets/f9fe42b6-80af-4b40-893c-b81b2c0eb110" />

Thể hiện PostgreSQL dump: COPY … FROM stdin; → xác nhận dữ liệu rò rỉ là dump PostgreSQL.
Câu khiêu khích: “Your encoding trick isn’t going to fool anyone.”
→ Đây không phải mã hóa thực sự, mà là một kiểu “mã hoá giả”/obfuscation. Với dự án Python, thủ phạm kinh điển là ROT13.

Ở đề bài lab (Q3 & Q4), ta thấy có 1 điểm đáng lưu tâm là


<img width="1384" height="46" alt="image" src="https://github.com/user-attachments/assets/a10c0701-0523-4a63-bd28-92c3b4da6916" />



Hỏi “địa chỉ public của leak” và yêu cầu tải về để xác minh (trích hash Naomi).
→ Dump phải nằm ở một URL công khai. Trường hợp hay gặp nhất: S3 object bị public/misconfigure (hoặc endpoint kiểu blob storage).


Thường thì dự án Python/Flask thường có thư mục util/ chứa các tiện ích như script backup.


Backup PostgreSQL thường dùng pg_dump.


Từ các thông tin trên, ta sẽ tìm dấu vết backup trong toàn bộ lịch sử, đề phòng file đã bị xóa.
```js
git grep -nI 'pg_dump\|ROT13\|rot_13\|db_backup' $(git rev-list --all)
```
<img width="993" height="724" alt="image" src="https://github.com/user-attachments/assets/f5208fd7-9c5c-459f-b5b3-34f6535c39f4" />

Từ các dòng grep, ta khôi phục được hành vi của script này:

Tạo bản sao database
```js
dump_cmd = f"pg_dump -U {DB_USER} {DB_NAME} > /tmp/{BACKUP_FILENAME}"
```

→ pg_dump xuất toàn bộ database vào file .sql.

Mã hóa ROT13
```js
encoded = codecs.encode(data, "rot_13")
```

→ Dữ liệu được “mã hóa” bằng ROT13 (chỉ đảo chữ cái, rất dễ đảo ngược).

Đặt tên file .xyz
```js
BACKUP_FILENAME = "db_backup.xyz"
```

→ Đổi đuôi file để che mắt, nhưng thực chất chỉ là dump text.

Upload lên S3
```js
S3_KEY = "db_backup.xyz"
```

→ File backup được upload lên Amazon S3 bucket.


Các bạn có thể tìm hiểu thêm về S3 ở [đây](https://viblo.asia/p/tim-hieu-s3-aws-amazon-simple-storage-service-gDVK2QGm5Lj)



Như vậy, file này là 1 file rất nhạy cảm, nếu như được public rất dễ bị lợi dụng, tôi đã tìm trong src code nhưng không có nên có thể đội dev cũng đã nhận ra được việc này và đã gỡ nó đi, vậy nên đáp án sẽ là:
```js
app/util/backup_db.py
```
<img width="1558" height="134" alt="image" src="https://github.com/user-attachments/assets/94908e73-f13b-467b-8ed2-70cb5991ea93" />

# FLAG 3

Như ở trên đã nêu, dev sử dụng S3 để upload lên, mô hình chung 1 đường dẫn truy cập công khai của S3 trông như này
```js
https://<bucket>.s3.<region>.amazonaws.com/<object>
```

Bây giờ ta sẽ tìm thử bucket và region trong đó
```js
git grep -nI 'S3_BUCKET\|S3_REGION|S3_bucket\|S3_region' $(git rev-list --all)
```

<img width="1062" height="712" alt="image" src="https://github.com/user-attachments/assets/2b6d23e8-79c9-41a7-b821-cf4374fefd9c" />

Ở đây ta nhận được
```js
S3_BUCKET = "protoguard-asset-management"
```
Nhưng lại không tìm thấy REGION, mày mò 1 lúc thì tôi đã thử truy vết commit gần nhất của backup_db.py để xem thêm
```js
git log -p -- app/util/backup_db.py | grep S3_REGION -A 2 -B 2
```
<img width="820" height="426" alt="image" src="https://github.com/user-attachments/assets/2926d605-7610-4fef-8089-32dd67adb091" />

Và chúng ta đã thấy được thứ chúng ta cần tất cả trong này, có vẻ tôi đã hơi dài dòng 1 chút rồi =)))

```js
S3_BUCKET = "protoguard-asset-management"
S3_KEY = "db_backup.xyz"
S3_REGION = "us-east-2"
```
Với những thông tin này ta có thể ghép lại thành 1 URL
```js
https://protoguard-asset-management.s3.us-east-2.amazonaws.com/db_backup.xyz
```
Và thành công có file này

<img width="1145" height="515" alt="image" src="https://github.com/user-attachments/assets/af65c980-059d-416f-83f1-c75f6322ba71" />

Nhiệm vụ tiếp theo là tìm password hash for Naomi Adler

Tôi đã viết 1 script để tự động hóa việc này
```js
<#
.SYNOPSIS
    Tải file dump từ S3, giải mã ROT13 và trích xuất password hash của Naomi Adler.
#>

param (
    [string]$Url = "https://protoguard-asset-management.s3.us-east-2.amazonaws.com/db_backup.xyz",
    [string]$Rot13File = "dump.rot13",
    [string]$DecodedFile = "dump.sql",
    [string]$TargetFirst = "Naomi",
    [string]$TargetLast = "Adler"
)

function Decode-Rot13($text) {
    return ($text.ToCharArray() | ForEach-Object {
        if ($_ -match '[A-Za-z]') {
            $base = if ($_ -cmatch '[A-Z]') { [int][char]'A' } else { [int][char]'a' }
            [char]((( [int][char]$_ - $base + 13) % 26) + $base)
        } else { $_ }
    }) -join ''
}

Write-Host "[1/4] Downloading leaked S3 object..." -ForegroundColor Cyan
Invoke-WebRequest -Uri $Url -OutFile $Rot13File -ErrorAction Stop
Write-Host "[+] Downloaded file: $Rot13File"

Write-Host "[2/4] Decoding ROT13..." -ForegroundColor Cyan
$content = Get-Content -Path $Rot13File -Raw -Encoding UTF8
$decoded = Decode-Rot13 $content
Set-Content -Path $DecodedFile -Value $decoded -Encoding UTF8
Write-Host "[+] Wrote decoded dump to: $DecodedFile"

Write-Host "[3/4] Searching for COPY users section..." -ForegroundColor Cyan
$lines = Get-Content -Path $DecodedFile

# Tìm dòng COPY liên quan đến users
$copyStart = ($lines | Select-String -Pattern 'COPY public\.users|COPY users' -CaseSensitive:$false).LineNumber
if (-not $copyStart) {
    # Nếu không có, tìm "COPY public.hfref" vì dump này bị ROT13
    $copyStart = ($lines | Select-String -Pattern 'COPY public\.hfref|COPY hfref' -CaseSensitive:$false).LineNumber
}

if (-not $copyStart) {
    Write-Host "[!] Could not find any user table COPY block." -ForegroundColor Red
    exit 1
}

# Lấy block dữ liệu
$block = @()
for ($i = $copyStart; $i -lt $lines.Count; $i++) {
    if ($lines[$i] -eq '\.') { break }
    $block += $lines[$i]
}

# Parse tên cột
$headerLine = $lines[$copyStart - 1]
$columns = ($headerLine -replace 'COPY public\.[^\(]+\(|\) FROM stdin;','') -split ',' | ForEach-Object { $_.Trim() }
$pwdIndex = [Array]::IndexOf($columns, 'password_hash')
if ($pwdIndex -lt 0) { $pwdIndex = [Array]::IndexOf($columns, 'cnffjbeq_unfu') }

Write-Host "[4/4] Looking for user $TargetFirst $TargetLast..." -ForegroundColor Cyan
foreach ($row in $block) {
    if ($row -match $TargetFirst -and $row -match $TargetLast) {
        $fields = $row -split "`t"
        if ($pwdIndex -ge 0 -and $pwdIndex -lt $fields.Count) {
            $hash = $fields[$pwdIndex]
            Write-Host "`n=== RESULT ==="
            Write-Host ("Password hash for {0} {1}:`n{2}" -f $TargetFirst, $TargetLast, $hash) -ForegroundColor Green
            exit 0
        } else {
            Write-Host "[!] Password hash column not found for that row." -ForegroundColor Red
            exit 1
        }
    }
}

Write-Host "[!] Could not find password hash for $TargetFirst $TargetLast." -ForegroundColor Red
exit 1
```
Ta có hash như sau
```js
pbkdf2:sha256:600000$YQqIvcDipYLzzXPB$598fe450e5ac019cdd41b4b10c5c21515573ee63a8f4881f7d721fd74ee43d59
```
<img width="1074" height="185" alt="image" src="https://github.com/user-attachments/assets/6f89f2a6-733d-436c-89fc-8d6ea2e363ee" />

Đáp án:

```js
https://protoguard-asset-management.s3.us-east-2.amazonaws.com/db_backup.xyz, pbkdf2:sha256:600000$YQqIvcDipYLzzXPB$598fe450e5ac019cdd41b4b10c5c21515573ee63a8f4881f7d721fd74ee43d59
```


<img width="1543" height="134" alt="image" src="https://github.com/user-attachments/assets/a247580b-51e1-4796-8131-a036f3e59550" />


# FLAG 4

Như Flag 3, địa chỉ public ấy sẽ là

```js
https://protoguard-asset-management.s3.us-east-2.amazonaws.com/db_backup.xyz
```

<img width="1543" height="120" alt="image" src="https://github.com/user-attachments/assets/83081619-bf04-40be-b763-6659326c9142" />





