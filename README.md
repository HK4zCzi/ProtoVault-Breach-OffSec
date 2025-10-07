# ProtoVault-Breach-OffSec

#Chuẩn bị

Đề bài cho ta 1 fize zip, giải nén sẽ được các folder trên, bài đang bảo rằng manh mối đầu tiên là email, hãy xem nó thử
<img width="663" height="90" alt="image" src="https://github.com/user-attachments/assets/53335be6-f605-4e71-89bf-b0b69a4fc13f" />


Ảnh trong email cho thấy một đoạn trích dữ liệu SQL:

COPY public.item_types (id, name, description) FROM stdin;
1  Biological Samples  Preserved specimens of rare or hazardous organisms
2  Prototype Devices   Experimental technologies under development
3  Confidential Documents  Sensitive research reports
4  Hazardous Chemicals Toxic, corrosive, or otherwise dangerous substances
Mục đích là chứng minh rằng kẻ tấn công có quyền truy cập dữ liệu thực, để tăng tính thuyết phục cho lời đe dọa.
<img width="1073" height="833" alt="ransom_email" src="https://github.com/user-attachments/assets/8a7b15f9-7f3e-42ef-9a08-3840891a901d" />

Ảnh tống tiền cho thấy PostgreSQL dump (COPY public.item_types (id, name, description) FROM stdin;) => Trong repo chắc chắn có cấu hình/chuỗi kết nối PostgreSQL và có thể có script backup (dễ gây rò rỉ).

Ta không biết ở commit nào lộ bí mật. git grep + git rev-list --all cho phép tìm mọi commit.
<img width="1428" height="428" alt="image" src="https://github.com/user-attachments/assets/6cbec8f9-ab2e-4d3c-a182-7d79509d17e5" />
Ồ, ta có thể thấy gần đây thì mới dùng host sản xuất, user riêng, sslmode=verify-full (bắt buộc SSL và xác thực CN) còn trước đó thì lại để một chuỗi kém an toàn: không SSL, lộ full password, host 127.0.0.1. Điều này cho thấy repo từng chứa thông tin nhạy cảm -> tăng khả năng rò rỉ đến từ ứng dụng / quy trình của team.

Sau khi tổng hợp lại thông tin, đáp án câu đầu tiên sẽ là
Yes, postgresql://assetdba:8d631d2207ec1debaafd806822122250@pgsql_prod_db01.protoguard.local/pgamgt?sslmode=verify-full
<img width="1548" height="115" alt="image" src="https://github.com/user-attachments/assets/d2ada4d0-edb4-477b-92f6-c10b4d96b3ce" />

