# TỔNG HỢP THAY ĐỔI - DATABASE V2.0
## Cập nhật dựa trên yêu cầu bổ sung

---

## ✅ DANH SÁCH THAY ĐỔI CHÍNH

### 1. BẢNG MỚI (8 bảng)

| Tên bảng | Mục đích | Ưu tiên |
|----------|----------|---------|
| **Passengers** | Quản lý thông tin hành khách chi tiết, điểm đón/trả, check-in | ⭐⭐⭐ Cao |
| **TicketChanges** | Lịch sử đổi vé, tracking thay đổi | ⭐⭐ Trung bình |
| **TripTracking** | Theo dõi GPS, vị trí xe thời gian thực | ⭐ Thấp |
| **DelayNotifications** | Quản lý trễ chuyến, đền bù | ⭐⭐ Trung bình |
| **TripCosts** | Tính chi phí, lợi nhuận, đề xuất giá | ⭐⭐⭐ Cao |
| **TollStations** | Trạm thu phí cao tốc | ⭐ Thấp |
| **TollFees** | Phí theo loại xe | ⭐ Thấp |
| **RouteTollStations** | Trạm trên từng tuyến | ⭐ Thấp |
| **DriverWorklog** | Nhật ký làm việc, tính lương tài xế | ⭐⭐⭐ Cao |

---

### 2. CẢI TIẾN BẢNG CŨ

#### Bảng `Bookings`
```sql
+ IsGuestBooking          -- Phân biệt khách vãng lai
+ GuestSessionID          -- Tracking session
+ InvitedToRegister       -- Đã mời đăng ký chưa
+ InvitationSentAt        -- Thời gian gửi lời mời
```
**Mục đích:** Hỗ trợ khách đặt vé không cần đăng nhập

#### Bảng `Refunds`
```sql
+ RefundType              -- Hủy toàn bộ/1 vé/Đổi chuyến
+ AffectedTicketIDs       -- Danh sách vé bị ảnh hưởng
+ NewTripID               -- Chuyến mới nếu đổi
+ PriceDifference         -- Chênh lệch giá
```
**Mục đích:** Quản lý chi tiết quy trình hủy/đổi vé

#### Bảng `Trips`
```sql
+ OnlineBookingCutoff     -- Block đặt online (mặc định 60 phút)
+ AllowOfflineBooking     -- Cho phép bán tại quầy
+ IsFullyBooked           -- Đánh dấu đã đầy
+ MinPassengers           -- Số khách tối thiểu
+ AutoCancelIfNotEnough   -- Tự động hủy nếu không đủ
```
**Mục đích:** Kiểm soát thời gian đặt vé và quản lý chuyến

#### Bảng `RouteStops`
```sql
+ StopName                -- Tên điểm dừng cụ thể
+ IsPickupPoint           -- Có phải điểm đón
+ IsDropoffPoint          -- Có phải điểm trả
+ StopAddress             -- Địa chỉ chi tiết
+ Latitude, Longitude     -- Tọa độ GPS
+ StopNote                -- Ghi chú
```
**Mục đích:** Quản lý chi tiết điểm đón/trả

#### Bảng `Reviews`
```sql
+ DriverRating            -- Đánh giá tài xế
+ VehicleRating           -- Đánh giá xe
+ ServiceRating           -- Đánh giá dịch vụ
+ PunctualityRating       -- Đánh giá đúng giờ
+ Feedback                -- Phản hồi chi tiết
+ ReviewStatus            -- Chờ duyệt/Đã duyệt/Bị ẩn
+ AdminResponse           -- Phản hồi từ nhà xe
+ RespondedBy, RespondedAt
```
**Mục đích:** Đánh giá chi tiết và quản lý phản hồi

---

### 3. VIEWS MỚI (3 views)

#### vw_PriceSuggestion
Đề xuất giá vé dựa trên chi phí và tỷ lệ lấp đầy
```sql
- Giá hòa vốn
- Giá đề xuất (lợi nhuận 20%, 30%)
- Tỷ lệ lấp đầy hiện tại
- Dự đoán doanh thu
```

#### vw_DriverPerformance
Hiệu suất làm việc của tài xế
```sql
- Tổng chuyến, chuyến hoàn thành
- Trung bình delay
- Đánh giá trung bình
- Tổng giờ làm, số lần vi phạm
- Tổng lương
```

#### vw_PassengerList
Danh sách hành khách chi tiết theo chuyến
```sql
- Thông tin người mua vs người đi
- Điểm đón/trả cụ thể
- Trạng thái check-in
- Ghi chú đặc biệt
```

---

### 4. STORED PROCEDURES MỚI (3 procedures)

#### sp_CheckBookingEligibility
Kiểm tra có thể đặt vé không
```sql
INPUT:  TripID, BookingType (Online/Tại quầy)
OUTPUT: CanBook (TRUE/FALSE), Message
LOGIC:  
  - Kiểm tra thời gian còn lại
  - So sánh với OnlineBookingCutoff
  - Kiểm tra đã đầy chỗ chưa
```

#### sp_CheckInPassenger
Check-in hành khách lên xe
```sql
INPUT:  TicketCode, CheckInMethod, CheckedInBy
OUTPUT: Success, Message
LOGIC:
  - Tìm vé theo mã
  - Kiểm tra trạng thái chuyến
  - Cập nhật trạng thái check-in
```

#### sp_ChangeTicket
Đổi vé sang chuyến khác
```sql
INPUT:  TicketID, NewTripID, NewSeatNumber, Reason, ChangedBy
OUTPUT: Success, Message
LOGIC:
  - Kiểm tra ghế mới còn trống
  - Giải phóng ghế cũ
  - Đặt ghế mới
  - Tính chênh lệch giá
  - Lưu lịch sử thay đổi
```

---

### 5. TRIGGERS MỚI (2 triggers)

#### trg_CreatePassengerOnTicket
Tự động tạo bản ghi Passenger khi có Ticket mới
```sql
AFTER INSERT ON Tickets
- Lấy thông tin từ Booking
- Tạo Passenger với thông tin cơ bản
```

#### trg_UpdateTripFullStatus
Tự động cập nhật trạng thái đầy chỗ
```sql
AFTER UPDATE ON TripSeats
- Đếm số ghế đã đặt
- Cập nhật IsFullyBooked nếu full
```

---

## 🎯 GIẢI QUYẾT CÁC YÊU CẦU

### ✅ Quản lý nhiều vé
**Giải pháp:** Bảng `Passengers`
- Phân biệt "người mua" (Bookings) vs "người đi" (Passengers)
- 1 Booking → N Tickets → N Passengers
- Mỗi passenger có thông tin riêng, điểm đón/trả riêng

### ✅ Hủy vé và đổi chuyến
**Giải pháp:** 
- Cải tiến `Refunds` với RefundType
- Bảng `TicketChanges` tracking lịch sử
- Procedure `sp_ChangeTicket` xử lý logic

### ✅ Block thời gian đặt online
**Giải pháp:** 
- Thêm `OnlineBookingCutoff` vào Trips
- Procedure `sp_CheckBookingEligibility` kiểm tra
- Mặc định block 60 phút trước giờ khởi hành

### ✅ Kiểm soát lên/xuống xe
**Giải pháp:** Bảng `Passengers`
- `CheckInStatus`: Chưa lên/Đã lên/Đã xuống
- `CheckInMethod`: Quét mã QR/Vạch/Điểm danh tay
- `CheckInTime`: Thời gian check-in
- Procedure `sp_CheckInPassenger`

### ✅ Điểm đón/trả chi tiết
**Giải pháp:**
- Cải tiến `RouteStops` với tọa độ GPS
- Mỗi Passenger chọn pickup/dropoff point
- Tài xế xem danh sách qua `vw_PassengerList`

### ✅ Theo dõi GPS và báo trễ
**Giải pháp:** 
- Bảng `TripTracking`: Lưu vị trí real-time
- Bảng `DelayNotifications`: Quản lý trễ và đền bù
- Lưu lý do, phương án đền bù

### ✅ Tính chi phí và đề xuất giá
**Giải pháp:** 
- Bảng `TripCosts`: Chi tiết từng loại chi phí
- Generated columns tự động tính tổng
- View `vw_PriceSuggestion`: Đề xuất giá thông minh

### ✅ Giới hạn giờ làm tài xế
**Giải pháp:** 
- Bảng `DriverWorklog`: Nhật ký từng ca
- Procedure kiểm tra 10h/ngày, 4h liên tục
- Tính lương theo giờ/chuyến/tháng
- Tracking vi phạm

### ✅ Đánh giá chi tiết
**Giải pháp:** 
- Cải tiến `Reviews` với rating riêng
- Driver/Vehicle/Service/Punctuality
- Admin có thể phản hồi
- Trạng thái duyệt

### ✅ Khách vãng lai
**Giải pháp:** 
- `IsGuestBooking` trong Bookings
- `CustomerID = NULL` cho khách vãng lai
- Mời đăng ký sau thanh toán

### ✅ Trạm thu phí cao tốc
**Giải pháp:** 
- Bảng `TollStations`, `TollFees`
- `RouteTollStations`: Gắn trạm vào tuyến
- Tự động tính phí vào `TripCosts`

---

## 📊 SO SÁNH TRƯỚC VÀ SAU

| Tính năng | Version 1.0 | Version 2.0 |
|-----------|-------------|-------------|
| Số bảng | 20 | 28 (+8) |
| Views | 3 | 6 (+3) |
| Stored Procedures | 3 | 6 (+3) |
| Triggers | 4 | 6 (+2) |
| Quản lý hành khách | Cơ bản | Chi tiết (điểm đón/trả, check-in) |
| Hủy/Đổi vé | Chỉ hủy | Hủy + Đổi chuyến |
| Block online | ❌ | ✅ (60 phút) |
| GPS tracking | ❌ | ✅ |
| Tính chi phí | ❌ | ✅ (chi tiết từng khoản) |
| Đề xuất giá | ❌ | ✅ (dựa trên chi phí) |
| Nhật ký tài xế | ❌ | ✅ (kiểm soát giờ làm) |
| Khách vãng lai | Hạn chế | ✅ (đầy đủ) |
| Đánh giá | Tổng quát | Chi tiết (4 tiêu chí) |

---



## ⚠️ LƯU Ý QUAN TRỌNG

### 1. Migration dữ liệu cũ
Nếu đã có dữ liệu trong database v1.0, cần:
- Tạo Passengers cho các Tickets đã tồn tại
- Cập nhật RouteStops với thông tin GPS
- Backfill dữ liệu cho các trường mới

### 2. Application code cần update
- Check logic đặt vé: Gọi `sp_CheckBookingEligibility`
- Form đặt vé: Thu thập thông tin Passenger
- Check-in: Tích hợp quét mã QR/vạch
- Đổi vé: Sử dụng `sp_ChangeTicket`
- Dashboard: Sử dụng các view mới

### 3. Performance
- Bảng `TripTracking` có thể lớn nhanh → Cần index tốt
- Nên archive dữ liệu cũ định kỳ
- Cân nhắc partition cho bảng lớn

### 4. Tích hợp bên ngoài
- GPS tracking: Cần API từ thiết bị định vị
- QR code: Cần thư viện generate/scan
- Payment gateway: Xử lý đổi vé có chênh lệch giá

---

## 📈 KẾ HOẠCH TRIỂN KHAI

### Phase 1 (Tuần 1-2) - Ưu tiên cao
- [x] Tạo bảng Passengers
- [x] Cải tiến Bookings (khách vãng lai)
- [x] Block thời gian online
- [x] Bảng TripCosts
- [x] View PriceSuggestion

### Phase 2 (Tuần 3-4) - Ưu tiên trung
- [x] Bảng TicketChanges
- [x] Procedure đổi vé
- [x] Cải tiến Reviews
- [x] Bảng DriverWorklog
- [x] Procedure check giờ làm

### Phase 3 (Tuần 5-6) - Ưu tiên thấp
- [x] GPS tracking
- [x] Delay notifications
- [x] Toll stations
- [x] Performance optimization

---

## ✅ CHECKLIST SAU KHI CẬP NHẬT

- [ ] Database cập nhật thành công
- [ ] Tất cả bảng mới đã có
- [ ] Views trả về dữ liệu đúng
- [ ] Stored procedures chạy OK
- [ ] Triggers hoạt động
- [ ] Indexes được tạo
- [ ] Test basic workflows:
  - [ ] Đặt vé với thông tin passenger
  - [ ] Check-in hành khách
  - [ ] Đổi vé sang chuyến khác
  - [ ] Tính chi phí và lợi nhuận
  - [ ] Kiểm tra giờ làm tài xế

---

**Tài liệu này tóm tắt toàn bộ thay đổi trong Database V2.0**
