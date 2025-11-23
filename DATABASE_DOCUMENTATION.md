# TÀI LIỆU MÔ TẢ CẤU TRÚC DATABASE
## Hệ Thống Quản Lý Bán Vé Xe Khách

---

## 📋 TỔNG QUAN

Database được thiết kế cho hệ thống quản lý bán vé xe khách với đầy đủ các chức năng:
- Quản lý người dùng và phân quyền
- Quản lý xe, tài xế, lộ trình
- Đặt vé online và tại quầy
- Thanh toán và hoàn tiền
- Đánh giá và thông báo
- Báo cáo thống kê

---

## 📊 CẤU TRÚC DATABASE

### 1. NHÓM BẢNG NGƯỜI DÙNG VÀ TÀI KHOẢN

#### 1.1. Bảng `Roles` - Vai trò
```
RoleID (PK)           - ID vai trò
RoleName              - Tên vai trò (Admin, Nhân viên, Tài xế, Khách hàng)
Description           - Mô tả vai trò
CreatedAt             - Thời gian tạo
```

**Quy tắc nghiệp vụ:**
- Mỗi người dùng chỉ có 1 vai trò duy nhất (QĐ4)
- Có 4 vai trò chính: Admin, Nhân viên bán vé, Tài xế, Khách hàng

#### 1.2. Bảng `Users` - Người dùng
```
UserID (PK)           - ID người dùng
FullName              - Họ và tên
Email (UNIQUE)        - Email (định danh tài khoản)
PhoneNumber (UNIQUE)  - Số điện thoại
Password              - Mật khẩu (mã hóa Hash)
RoleID (FK)           - Vai trò
Status                - Trạng thái (Hoạt động, Khóa)
EmailVerified         - Email đã xác thực chưa
LoyaltyPoints         - Điểm tích lũy
CreatedAt             - Thời gian tạo
UpdatedAt             - Thời gian cập nhật
```

**Quy tắc nghiệp vụ:**
- Email và SĐT không được trùng (QĐ2)
- Mật khẩu tối thiểu 6 ký tự, có ít nhất 1 chữ số (QĐ2)
- Tài khoản mới mặc định là vai trò "Khách hàng" (QĐ2)
- Tài khoản phải ở trạng thái "Hoạt động" mới đăng nhập được (QĐ1)
- Email đã xác thực không thể đổi (QĐ3)

---

### 2. NHÓM BẢNG QUẢN LÝ XE VÀ TÀI XẾ

#### 2.1. Bảng `VehicleTypes` - Loại xe
```
TypeID (PK)           - ID loại xe
TypeName (UNIQUE)     - Tên loại xe (Limousine, Giường nằm, Ghế ngồi)
TotalSeats            - Tổng số ghế
Description           - Mô tả
CreatedAt             - Thời gian tạo
```

#### 2.2. Bảng `Vehicles` - Xe
```
VehicleID (PK)        - ID xe
LicensePlate (UNIQUE) - Biển kiểm soát (duy nhất)
TypeID (FK)           - Loại xe
InsuranceNumber       - Số bảo hiểm
InsuranceExpiry       - Hạn bảo hiểm
Status                - Tình trạng (Hoàn thiện, Hư hại, Phế liệu)
CreatedAt             - Thời gian tạo
UpdatedAt             - Thời gian cập nhật
```

**Quy tắc nghiệp vụ:**
- Biển kiểm soát là duy nhất (QĐ7)
- Xe có 3 trạng thái: Hoàn thiện, Hư hại, Phế liệu (QĐ7)
- Loại xe quy định số ghế và sơ đồ ghế

#### 2.3. Bảng `Drivers` - Tài xế
```
DriverID (PK)         - ID tài xế
UserID (FK, UNIQUE)   - Liên kết với Users
DriverLicense (UNIQUE)- Số bằng lái
LicenseExpiry         - Hạn bằng lái
DateOfBirth           - Ngày sinh
Gender                - Giới tính
Salary                - Lương
CreatedAt             - Thời gian tạo
UpdatedAt             - Thời gian cập nhật
```

**Quy tắc nghiệp vụ:**
- Lịch chạy là danh sách chuyến xe tài xế chạy (QĐ8)
- Tài xế làm việc tối đa 10 giờ/ngày, tối đa 4 tiếng liên tục

---

### 3. NHÓM BẢNG QUẢN LÝ TUYẾN ĐƯỜNG

#### 3.1. Bảng `Locations` - Địa điểm
```
LocationID (PK)       - ID địa điểm
LocationName          - Tên địa điểm
Province              - Tỉnh/Thành phố
Address               - Địa chỉ
Latitude              - Vĩ độ
Longitude             - Kinh độ
CreatedAt             - Thời gian tạo
```

#### 3.2. Bảng `Routes` - Lộ trình
```
RouteID (PK)          - ID lộ trình
RouteName             - Tên lộ trình
OriginID (FK)         - Điểm đi
DestinationID (FK)    - Điểm đến
Distance              - Khoảng cách (km)
EstimatedDuration     - Thời gian dự kiến (phút)
Status                - Trạng thái (Hoạt động, Bảo trì, Dừng)
CreatedAt             - Thời gian tạo
UpdatedAt             - Thời gian cập nhật
```

**Quy tắc nghiệp vụ:**
- Chỉ Admin được tạo, sửa, xóa lộ trình (QĐ5)
- Lộ trình có 3 trạng thái: Hoạt động, Bảo trì, Dừng (QĐ5)
- Chỉ lộ trình "Hoạt động" mới dùng để lập lịch chuyến

#### 3.3. Bảng `RouteStops` - Điểm dừng trên lộ trình
```
StopID (PK)           - ID điểm dừng
RouteID (FK)          - Lộ trình
LocationID (FK)       - Địa điểm
StopOrder             - Thứ tự dừng
StopType              - Loại điểm (Khởi hành, Dừng chân, Đến)
DistanceFromOrigin    - Khoảng cách từ điểm xuất phát
EstimatedTime         - Thời gian dự kiến (phút)
```

**Quy tắc nghiệp vụ:**
- Danh sách điểm phải theo đúng thứ tự thực tế (QĐ5)

---

### 4. NHÓM BẢNG QUẢN LÝ CHUYẾN XE

#### 4.1. Bảng `Trips` - Chuyến xe
```
TripID (PK)           - ID chuyến xe
RouteID (FK)          - Lộ trình
VehicleID (FK)        - Xe
DriverID (FK)         - Tài xế
DepartureTime         - Giờ khởi hành
ArrivalTime           - Giờ đến dự kiến
BasePrice             - Giá vé cơ bản
Status                - Trạng thái (Chờ, Đang chạy, Hoàn thành, Hủy, Trễ)
StatusNote            - Ghi chú trạng thái
CreatedBy (FK)        - Admin tạo
CreatedAt             - Thời gian tạo
UpdatedAt             - Thời gian cập nhật
```

**Quy tắc nghiệp vụ:**
- Chỉ Admin được tạo, sửa, xóa chuyến xe (QĐ6)
- Xe/Tài xế không được trùng lịch
- Trạng thái hợp lệ: Chờ, Đang chạy, Hoàn thành, Hủy, Trễ (QĐ20)
- Mọi thay đổi trạng thái phải ghi log (QĐ20)

#### 4.2. Bảng `TripSeats` - Sơ đồ ghế chuyến xe
```
SeatID (PK)           - ID ghế
TripID (FK)           - Chuyến xe
SeatNumber            - Số ghế (A1, A2, B1...)
SeatType              - Loại ghế (Thường, VIP, Giường)
Status                - Trạng thái (Trống, Đang giữ, Đã đặt)
HoldExpiry            - Thời gian hết hạn giữ chỗ
CreatedAt             - Thời gian tạo
```

**Quy tắc nghiệp vụ:**
- Danh sách ghế thuộc về xe của chuyến đó (QĐ6)
- Chỉ cho phép chọn ghế "Trống" (QĐ10)
- Ghế được giữ trong 10 phút (QĐ18)
- Hết thời gian chưa thanh toán → tự động giải phóng

---

### 5. NHÓM BẢNG ĐẶT VÉ VÀ THANH TOÁN

#### 5.1. Bảng `Bookings` - Đặt vé
```
BookingID (PK)        - ID đặt vé
BookingCode (UNIQUE)  - Mã đặt vé (tự động)
CustomerID (FK)       - ID khách hàng (NULL nếu vãng lai)
CustomerName          - Tên khách hàng
CustomerPhone         - SĐT khách hàng
CustomerEmail         - Email khách hàng
TripID (FK)           - Chuyến xe
TotalAmount           - Tổng tiền
BookingStatus         - Trạng thái (Đang giữ, Đã thanh toán, Đã hủy, Đã hoàn thành)
BookingType           - Loại đặt (Online, Tại quầy)
CreatedBy (FK)        - Nhân viên tạo (nếu tại quầy)
CreatedAt             - Thời gian tạo
UpdatedAt             - Thời gian cập nhật
```

#### 5.2. Bảng `Tickets` - Vé
```
TicketID (PK)         - ID vé
TicketCode (UNIQUE)   - Mã vé (tự động)
BookingID (FK)        - Đơn đặt vé
SeatID (FK)           - Ghế
Price                 - Giá vé
TicketStatus          - Trạng thái (Chưa xác nhận, Đã xác nhận, Đã sử dụng, Đã hủy)
PrintedBy (FK)        - Nhân viên in vé
PrintedAt             - Thời gian in
CreatedAt             - Thời gian tạo
```

**Quy tắc nghiệp vụ:**
- Vé chỉ được in khi đã thanh toán thành công (QĐ12)
- Sau khi in, cập nhật trạng thái "Đã xác nhận" và lưu nhân viên thực hiện (QĐ12)

#### 5.3. Bảng `Payments` - Thanh toán
```
PaymentID (PK)        - ID thanh toán
BookingID (FK)        - Đơn đặt vé
Amount                - Số tiền
PaymentMethod         - Phương thức (Tiền mặt, Chuyển khoản, Thẻ, Ví điện tử)
PaymentStatus         - Trạng thái (Chờ xử lý, Thành công, Thất bại, Đã hoàn tiền)
TransactionID         - Mã giao dịch từ cổng thanh toán
PaymentGateway        - Tên cổng thanh toán
PaymentNote           - Ghi chú
PaidAt                - Thời gian thanh toán
CreatedAt             - Thời gian tạo
```

**Quy tắc nghiệp vụ:**
- Chỉ xác nhận vé thành công khi nhận mã thành công từ cổng thanh toán (QĐ11)

#### 5.4. Bảng `Refunds` - Hoàn tiền
```
RefundID (PK)         - ID hoàn tiền
BookingID (FK)        - Đơn đặt vé
RefundAmount          - Số tiền hoàn
RefundReason          - Lý do hoàn tiền
RefundStatus          - Trạng thái (Đang xử lý, Đã hoàn tiền, Từ chối)
RefundMethod          - Phương thức hoàn (Chuyển khoản, Tiền mặt)
BankAccount           - Tài khoản ngân hàng
ProcessedBy (FK)      - Nhân viên xử lý
ProcessedAt           - Thời gian xử lý
CreatedAt             - Thời gian tạo
```

**Quy tắc nghiệp vụ:**
- Vé chỉ được hủy trước giờ khởi hành tối thiểu 2 giờ (QĐ13)
- Hoàn 90% nếu hủy trước 4 giờ, 50% nếu 2-4 giờ

---

### 6. NHÓM BẢNG ĐÁNH GIÁ VÀ PHẢN HỒI

#### 6.1. Bảng `Reviews` - Đánh giá
```
ReviewID (PK)         - ID đánh giá
TripID (FK)           - Chuyến xe
CustomerID (FK)       - Khách hàng
TicketID (FK)         - Vé
Rating                - Số sao (1-5)
Comment               - Bình luận
ReviewDate            - Thời gian đánh giá
```

**Quy tắc nghiệp vụ:**
- Mỗi khách hàng chỉ đánh giá 1 lần cho mỗi chuyến đi (QĐ15)
- Chỉ đánh giá được chuyến đã hoàn thành

---

### 7. NHÓM BẢNG THÔNG BÁO

#### 7.1. Bảng `Notifications` - Thông báo
```
NotificationID (PK)   - ID thông báo
NotificationType      - Loại (Tự động, Thủ công)
IncidentType          - Loại sự cố (Hủy chuyến, Đổi giờ, Đổi xe, Nhắc nhở, Khác)
Title                 - Tiêu đề
Content               - Nội dung
TargetAudience        - Đối tượng (Khách hàng, Tài xế, Tất cả)
TripID (FK)           - Chuyến xe (NULL nếu thông báo chung)
CreatedBy (FK)        - Người tạo
SentAt                - Thời gian gửi
```

**Quy tắc nghiệp vụ:**
- Chỉ Admin được gửi thông báo thủ công (QĐ17)
- Nội dung phải rõ ràng, đúng mã chuyến

#### 7.2. Bảng `UserNotifications` - Thông báo người dùng
```
ID (PK)               - ID
NotificationID (FK)   - Thông báo
UserID (FK)           - Người dùng
IsRead                - Đã đọc chưa
ReadAt                - Thời gian đọc
CreatedAt             - Thời gian tạo
```

---

### 8. NHÓM BẢNG CHI PHÍ VÀ BÁO CÁO

#### 8.1. Bảng `OperatingCosts` - Chi phí vận hành
```
CostID (PK)           - ID chi phí
CostType              - Loại chi phí (Xăng dầu, Bảo trì, Bảo hiểm, Lương, Khác)
Description           - Mô tả
Amount                - Số tiền
TripID (FK)           - Chuyến xe (NULL nếu chi phí chung)
VehicleID (FK)        - Xe
DriverID (FK)         - Tài xế
CostDate              - Ngày phát sinh
CreatedBy (FK)        - Người tạo
CreatedAt             - Thời gian tạo
```

#### 8.2. Bảng `Cargo` - Hàng hóa vận chuyển
```
CargoID (PK)          - ID hàng hóa
BookingID (FK)        - Đơn đặt (có thể NULL)
CustomerID (FK)       - Khách hàng
TripID (FK)           - Chuyến xe
Description           - Mô tả hàng hóa
Weight                - Cân nặng (kg)
CargoFee              - Phí vận chuyển
Status                - Trạng thái (Đang chờ, Đang vận chuyển, Đã giao, Hủy)
CreatedAt             - Thời gian tạo
```

---

### 9. NHÓM BẢNG AUDIT LOG

#### 9.1. Bảng `AuditLogs` - Nhật ký hệ thống
```
LogID (PK)            - ID log
UserID (FK)           - Người thực hiện
Action                - Hành động (CREATE, UPDATE, DELETE, LOGIN, LOGOUT)
TableName             - Tên bảng
RecordID              - ID bản ghi
OldValue              - Giá trị cũ
NewValue              - Giá trị mới
IPAddress             - Địa chỉ IP
CreatedAt             - Thời gian tạo
```

---

## 🔍 VIEWS (Các truy vấn có sẵn)

### 1. `vw_TripDetails` - Chi tiết chuyến xe
Hiển thị thông tin đầy đủ về chuyến xe bao gồm lộ trình, xe, tài xế, ghế trống

### 2. `vw_DailyRevenue` - Thống kê doanh thu theo ngày
Tổng hợp doanh thu, số booking, số vé theo ngày

### 3. `vw_PassengerManifest` - Danh sách hành khách
Hiển thị danh sách hành khách theo chuyến xe

---

## ⚙️ STORED PROCEDURES

### 1. `sp_GenerateSeatsForTrip(p_TripID)`
Tự động tạo ghế cho chuyến xe dựa trên loại xe

**Cách sử dụng:**
```sql
CALL sp_GenerateSeatsForTrip(1);
```

### 2. `sp_CancelTicket(p_BookingID, p_RefundReason)`
Hủy vé và xử lý hoàn tiền theo quy định

**Cách sử dụng:**
```sql
CALL sp_CancelTicket(123, 'Khách hàng có việc đột xuất');
```

**Quy tắc:**
- Phải hủy trước giờ khởi hành tối thiểu 2 giờ
- Hoàn 90% nếu hủy trước 4 giờ
- Hoàn 50% nếu hủy trong khoảng 2-4 giờ

### 3. `sp_ReleaseExpiredSeats()`
Tự động giải phóng ghế hết hạn giữ chỗ (chạy định kỳ)

**Cách sử dụng:**
```sql
CALL sp_ReleaseExpiredSeats();
```

---

## 🔔 TRIGGERS

### 1. `trg_GenerateBookingCode`
Tự động tạo mã booking khi insert (format: BK20231201XXXXX)

### 2. `trg_GenerateTicketCode`
Tự động tạo mã vé khi insert (format: TK20231201XXXXX)

### 3. `trg_UpdateSeatStatus`
Tự động cập nhật trạng thái ghế thành "Đã đặt" khi tạo vé

### 4. `trg_AuditUserChanges`
Tự động ghi log khi thông tin user thay đổi

---

## 📈 INDEX (Tối ưu hiệu suất)

```sql
-- Tìm kiếm chuyến xe
idx_trips_search          ON Trips(RouteID, DepartureTime, Status)
idx_routes_locations      ON Routes(OriginID, DestinationID)

-- Quản lý vé
idx_bookings_customer     ON Bookings(CustomerID, BookingStatus)
idx_tickets_booking       ON Tickets(BookingID, TicketStatus)

-- Thanh toán
idx_payments_booking      ON Payments(BookingID, PaymentStatus)

-- Đánh giá
idx_reviews_trip          ON Reviews(TripID, Rating)

-- Thông báo
idx_user_notifications    ON UserNotifications(UserID, IsRead)
```

---

## 🔐 YÊU CẦU BẢO MẬT

1. **Mã hóa mật khẩu:** Sử dụng bcrypt/SHA-256 hash
2. **SSL/TLS:** Mã hóa dữ liệu cá nhân và thanh toán
3. **Phân quyền:** Chỉ Admin xem báo cáo (QĐ21)
4. **Audit Log:** Ghi nhận mọi thay đổi quan trọng
5. **Xác thực OTP/Email:** Hỗ trợ xác thực 2 lớp

---

## 📊 CÁC TRUY VẤN MẪU

### Tìm chuyến xe
```sql
SELECT * FROM vw_TripDetails
WHERE Origin LIKE '%Hà Nội%'
AND Destination LIKE '%Hải Phòng%'
AND DATE(DepartureTime) = '2023-12-01'
AND TripStatus = 'Chờ'
AND AvailableSeats > 0;
```

### Thống kê doanh thu tháng
```sql
SELECT 
    DATE_FORMAT(RevenueDate, '%Y-%m') AS Month,
    SUM(TotalRevenue) AS MonthlyRevenue,
    SUM(TotalTickets) AS TotalTickets
FROM vw_DailyRevenue
WHERE YEAR(RevenueDate) = 2023
GROUP BY DATE_FORMAT(RevenueDate, '%Y-%m');
```

### Danh sách vé của khách hàng
```sql
SELECT 
    b.BookingCode,
    b.CustomerName,
    t.TicketCode,
    tr.DepartureTime,
    r.RouteName,
    ts.SeatNumber,
    t.TicketStatus
FROM Tickets t
JOIN Bookings b ON t.BookingID = b.BookingID
JOIN TripSeats ts ON t.SeatID = ts.SeatID
JOIN Trips tr ON ts.TripID = tr.TripID
JOIN Routes r ON tr.RouteID = r.RouteID
WHERE b.CustomerID = 123
ORDER BY tr.DepartureTime DESC;
```

### Top 5 tuyến đường doanh thu cao nhất
```sql
SELECT 
    r.RouteName,
    COUNT(DISTINCT t.TripID) AS TotalTrips,
    COUNT(DISTINCT tk.TicketID) AS TotalTickets,
    SUM(p.Amount) AS TotalRevenue
FROM Routes r
JOIN Trips t ON r.RouteID = t.RouteID
JOIN Bookings b ON t.TripID = b.TripID
JOIN Tickets tk ON b.BookingID = tk.BookingID
JOIN Payments p ON b.BookingID = p.BookingID
WHERE p.PaymentStatus = 'Thành công'
GROUP BY r.RouteID
ORDER BY TotalRevenue DESC
LIMIT 5;
```

---

## 🚀 HƯỚNG DẪN SỬ DỤNG

### Bước 1: Tạo Database
```bash
mysql -u root -p < create_database.sql
```

### Bước 2: Kiểm tra cấu trúc
```sql
USE BusTicketManagement;
SHOW TABLES;
```

### Bước 3: Thêm dữ liệu mẫu
Dữ liệu mẫu đã có trong script (Roles, VehicleTypes, Admin)

### Bước 4: Chạy stored procedure tạo ghế
```sql
-- Giả sử TripID = 1 đã tồn tại
CALL sp_GenerateSeatsForTrip(1);
```

### Bước 5: Thiết lập cron job giải phóng ghế
```sql
-- Chạy mỗi phút để giải phóng ghế hết hạn
CREATE EVENT ReleaseExpiredSeats
ON SCHEDULE EVERY 1 MINUTE
DO CALL sp_ReleaseExpiredSeats();
```

---

## 📞 HỖ TRỢ

Nếu cần hỗ trợ hoặc có câu hỏi về database, vui lòng liên hệ team phát triển.

**Tài liệu này được tạo tự động từ yêu cầu hệ thống bán vé xe khách.**
