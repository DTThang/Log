# 1. Giới thiệu 
- LogQL là ngôn ngữ truy vấn được phát triển cho loki
- Có 2 loại truy vấn logQL
  - Truy vấn log trả về nội dung ở dạng luồng
  - Truy vấn số liệu tính toán các giá trị dựa trên kết quả truy vấn 
- Một LogQL truy vấn bao gồm:
  - Bộ chọn lọc luồng log
  - Lọc biểu thức


# 2. Truy vấn log 
## 2.1 Toán tử số học  

Ký hiệu  | Phép toán 
---|---
+|phép cộng
-|phép trừ
*|phép nhân
/|phân công
%|modulo
^|lũy thừa / lũy thừa

Ví dụ:

- Nhân đôi tổng số log nhận vào của job varlogs trong 1phút:

        sum(count_over_time({job="varlogs"}[1m])) * 2

- Chia đôi tổng số log nhận vào của job varlogs trong 1phút:

        sum(count_over_time({job="varlogs"}[1m])) % 2

## 2.2 Toán tử logic và thiết lập

value | description
---|---
and|intersection
or|union
unless|complement


- `vector1 and vector2` xuất ra một vertor bao gồm các phần tử của vertor 1 trong đó có các phần tử của vertor2 có khớp các nhãn 
- `vector1 or vector2` xuất ra một vertor bao gồm các phần tử ban đầu (nhãn và giá trị) và thêm vào tất cả các phần tử của vector 2 không có nhãn phù hợp với vector1
`vector1 unless vector2` xuất ra một vertor bao gồm tất cả các phần tử của vertor1 mà không có phần tử nào của vertor 2 có các nhãn khớp. Tất cả các phần tử phù hợp giữa 2 vector đều bị loại bỏ  

Ví dụ  

        sum(count_over_time({job="varlogs"}[1m])) and  sum(count_over_time({filename="/var/log/auth.log"}[1m]))


## 2.3 Toán tử so sánh

Ký hiệu | giá trị
---|---
==| bình đẳng
!=|bất bình đẳng
\>|lớn hơn
\>=|lớn hơn hoặc bằng
<|ít hơn
<=|ít hơn hoặc bằng

- Toán tử so sánh định nghĩa giữa  các giá trị  scalar/scalar, vector/scalar, and vector/vector

- Ví dụ 

  - Lọc các luồng đã ghi ít nhất 50 dòng trong 1 phút qua
        ount_over_time({filename="/var/log/syslog"}[1m]) > 50

  - Đính kèm (các) giá trị 0/1 vào các luồng ghi ít hơn / nhiều hơn 10 dòng:

        count_over_time({filename="/var/log/syslog"}[1m]) > bool 10


## 2.4 Từ khóa on và ignoring

- Từ khóa  `ignoring` làm các nhãn chỉ định vị bỏ qua trong quá trình match. Cú pháp 
`<vector expr> <bin-op> ignoring(<labels>) <vector expr>`

- Ví dụ: 
  - Ví dụ này sẽ trả về các job có tổng số log trong 1 phút vừa qua vượt quá giá trị trung bình cho job

        max by(job) (count_over_time({filename="/var/log/syslog"}[1m])) > bool ignoring(job) avg(count_over_time({filename="/var/log/syslog"}[1m]))

- Từ khóa `on` làm giảm tập hợp các nhát được xét thành một danh sách chỉ định. Cú pháp: `<vector expr> <bin-op> on(<labels>) <vector expr>`

- Ví dụ :
  - Trả về tổng số job trong 1 phút trước trong job

        sum by(job) (count_over_time({filename="/var/log/syslog"}[1m])) / on() sum(count_over_time({filename="/var/log/syslog"}[1m]))


## 2.5 Lỗi pipe
- Lý do 
  - Một số bộ lọc nhãn có thể thật bại để thay đổi nhãn 
  - Chuyển đổi số liệu cho một nhãn có thể không thành công.
  - Một dòng nhật ký không phải là một tài liệu json hợp lệ.

- Sử dụng  `__error__` để loại bỏ lỗi. Cần đặt `__error__`ở  sau giai đoạn tạo ra lỗi     

# 3. Truy vấn số liệu
## 3.1 Tổng hợp phạm vi log 
- Là một loại truy vấn mà theo sau đó là một khoảng thời gian. 

- Chức năng


Giá trị | nghĩa  
---|---
rate(log-range)| tính số mục nhập mỗi giây
count_over_time(log-range)| đếm các mục nhập cho mỗi luồng nhật ký trong phạm vi đã cho.
bytes_rate(log-range)| tính toán số byte mỗi giây cho mỗi luồng.
bytes_over_time(log-range)| đếm số lượng byte được sử dụng bởi mỗi luồng nhật ký cho một phạm vi nhất định.
absent_over_time(log-range)| trả về một vectơ trống nếu vectơ phạm vi được truyền cho nó có bất kỳ phần tử nào và một vectơ 1 phần tử có giá trị 1 nếu vectơ phạm vi được truyền cho nó không có phần tử nào.


- Ví dụ 

  - Số lượng mục nhập trong 1 phút

        count_over_time({job="varlogs"}[1m])

  - Tỉ lệ bản ghi mỗi phút

        rate({job="varlogs"}[1m])

## 3.2 Unwrapped range aggregations
- Unwrapped range được sử dụng để trích suất nhãn như giá trị mẫu thay thế của dòng log. 
- Biểu thức `| unwrap label_identifier`,định danh nhãn là tên nhãn để sử dụng để trích suất các giá trị mẫu 
- Với các giá trị nhãn là string sẽ được chuyển đổi thành float, id nhãn có thể được bao bọc lại bởi hàm  `| unwrap <function>(label_identifier)`, hàm này sẽ cố gắng chuyển đổi giá trị nhãn từ một định dạng cụ thể.

- Một số hàm hỗ trợ:
  - `duration_seconds(label_identifier)` hoặc  `duration` sẽ chuyển đổi giá trị nhãn trong vài giây 
  - `bytes(label_identifier)` sẽ chuyển đổi giá trị nhãn thành byte thô áp dụng đơn vị byte 


## 3.3 Tían tử tổng hợp tích hợp sẵn 
- Một hàm tổng hợp chuyển đổi nhiều chuỗi vectơ thành một vectơ duy nhất
- Cú pháp  
       
        <aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
        
giá trị | ý nghĩa 
---|---
sum | tính tổng tất cả các vector
min | Hiển thị vector nhỏ nhất trong phạm vi thời gian
max | hiển thị vector lớn nhất  trong phạm vi thời gian
agv | trung bình của các vector trong phạm vi thời gian
stddev | tính độ lệch chuẩn của các giá trị từ tất cácr các vector trong phạm vi thời gian 
stdvar | tính phương sai tiêu chuẩn từ các giá trị từ tất cả các vector trong phạm vi thời gian 
count | đếm số phần tử tất cả các vector trong phạm vi thời gian  
bottomk | Chọn k giá trị thấp nhất trong tất cả vector trong phạm vi thời gian 
topk | Chọn k giá trị cao nhất trong tất các vector trong phạm vi thời gian 

> `bottomk` và `topk` không tạo ra vector mà là mọt chuỗi các vector có chứa k số vector trong khoảng thời gian 

Ví dụ 

- Tính tổng tất cả các vectơ trong dãy tại thời điểm

        sum(count_over_time({job="varlogs"}[1m]))
- Hiển thị giá trị nhỏ nhất từ ​​tất cả các vectơ trong phạm vi tại thời điểm

        min(count_over_time({job="varlogs"}[1m]))
- Hiển thị giá trị lớn nhất từ ​​tất cả các vectơ trong phạm vi tại thời điểm

        max(count_over_time({job="varlogs"}[1m]))
- Chỉ hiện  thị 2 giá trị từ tất cả các vertor trong phạm vi thời gian 

        topk(2, count_over_time({job="varlogs"}[1h]))  

---




# 2. Log Stream Selectors 

Operators | description
---|---
= | equals
!= | not equals  
=~ | regex matches
!~ | regex does not match

Ví dụ

- Lấy tất cả các dòng log cho job `varlog`

        {job="varlogs"}

- Lấy tất cả log trong filename `/var/log/syslog `

        {filename="/var/log/syslog"}

- Lấy tất cả log trong filename `/var/log/syslog ` và `/var/log/auth.log`

        {filename="/var/log/auth.log",job="varlogs"}

- Không lấy các dòng log chứa error

        {job="varlogs"} != "error"

- Lấy log chứa ký tự info và error sử dụng regex
        
        {job="varlogs"} |~ "error|info"
- Lấy log info nhưng không bao gồm info

        {job="varlogs"} |= "error" != "info"
- Lấy log không bao gồm dòng có info và error sử dụng regex

        {job="varlogs"} !~ "error|info"

- Lấy log bao gồm  ký tự `Invalid user` và ("bob" or "redis") sử dụng regex

        {job="varlogs"} |~ "Invalid user (bob|redis)"
- Lấy log bao chứa status 200 và 300

        {job="varlogs"} |~ "status [23]00"







# 5. Chức năng nhóm  

- Chuyển đổi một vectơ vô hướng thành một chuỗi các vectơ được nhóm theo tên tệp
- Ví dụ  
  - Nhóm một luồng log duy nhất theo tên tệp

        sum(count_over_time({job="varlogs"}[1m])) by (filename)

  - Nhóm nhiều luồng nhật ký theo máy chủ lưu trữ

        sum(count_over_time({job=~"varlogs"}[1m])) by (host)

  - Nhóm nhiều luồng nhật ký theo tên tệp và máy chủ  

        sum(count_over_time({job=~"varlogs"}[1m])) by (filename,host)






