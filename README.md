# ProtoVault-Breach-OffSec

# Chuẩn bị

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

Đề bài lab (Q3 & Q4)


<img width="1384" height="46" alt="image" src="https://github.com/user-attachments/assets/a10c0701-0523-4a63-bd28-92c3b4da6916" />

Hỏi “địa chỉ public của leak” và yêu cầu tải về để xác minh (trích hash Naomi).
→ Dump phải nằm ở một URL công khai. Trường hợp hay gặp nhất: S3 object bị public/misconfigure (hoặc endpoint kiểu blob storage).

Thói quen/cấu trúc repo thực tế


Dự án Python/Flask thường có thư mục util/ chứa các tiện ích như script backup.


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



Như vậy, file này là 1 file rất nhạy cảm, nếu như được public rất dễ bị lợi dụng, mình đã tìm trong src code nhưng không có nên có thể đội dev cũng đã nhận ra được việc này và đã gỡ nó đi, vậy nên đáp án sẽ là:
```js
app/util/backup_db.py
```
<img width="1558" height="134" alt="image" src="https://github.com/user-attachments/assets/94908e73-f13b-467b-8ed2-70cb5991ea93" />

# FLAG 3



