# PHÂN TÍCH BỔ SUNG & CẢI TIẾN DATABASE
## Hệ Thống Quản Lý Bán Vé Xe Khách - Version 2.0

---

## 📊 CÁC YÊU CẦU MỚI CẦN BỔ SUNG

### 1. QUẢN LÝ ĐẶT NHIỀU VÉ (Multi-Ticket Booking)

#### Vấn đề hiện tại:
- Database hiện tại chưa phân biệt rõ "người mua" và "người đi"
- Khi 1 người mua nhiều vé cho nhiều người khác → cần quản lý thông tin từng hành khách

#### Giải pháp:

**Bảng mới: `Passengers` - Hành khách**
```sql
CREATE TABLE Passengers (
    PassengerID INT PRIMARY KEY AUTO_INCREMENT,
    TicketID INT NOT NULL UNIQUE,
    FullName VARCHAR(100) NOT NULL,
    PhoneNumber VARCHAR(20),
    IdentityNumber VARCHAR(20), -- CMND/CCCD (tùy chọn)
    DateOfBirth DATE,
    Gender ENUM('Nam', 'Nữ', 'Khác'),
    PickupPoint VARCHAR(200), -- Điểm đón cụ thể
    DropoffPoint VARCHAR(200), -- Điểm trả cụ thể
    SpecialNote TEXT, -- Ghi chú đặc biệt (hành lý nhiều, cần hỗ trợ...)
    CheckInStatus ENUM('Chưa lên xe', 'Đã lên xe', 'Đã xuống xe') DEFAULT 'Chưa lên xe',
    CheckInTime DATETIME,
    CheckInMethod ENUM('Quét mã', 'Điểm danh tay') DEFAULT 'Quét mã',
    CheckedInBy INT, -- Nhân viên/Tài xế thực hiện
    CreatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (TicketID) REFERENCES Tickets(TicketID) ON DELETE CASCADE,
    FOREIGN KEY (CheckedInBy) REFERENCES Users(UserID) ON DELETE SET NULL
);
```

**Lợi ích:**
- Phân biệt rõ: Booking (người mua) vs Passengers (người đi)
- 1 Booking có nhiều Tickets, mỗi Ticket có 1 Passenger
- Quản lý điểm đón/trả riêng cho từng hành khách
- Kiểm soát lên/xuống xe chi tiết

---

### 2. QUY TRÌNH HỦY VÉ VÀ ĐỔI CHUYẾN

#### Yêu cầu mới:
- Hủy toàn bộ booking hoặc hủy từng vé riêng lẻ
- Đổi chuyến (chuyển sang chuyến khác)
- Quản lý ảnh hưởng của việc hủy

#### Cải tiến bảng `Refunds`:
```sql
ALTER TABLE Refunds 
ADD COLUMN RefundType ENUM('Hủy toàn bộ', 'Hủy 1 vé', 'Đổi chuyến') DEFAULT 'Hủy toàn bộ',
ADD COLUMN AffectedTicketIDs TEXT, -- Danh sách TicketID bị ảnh hưởng (JSON array)
ADD COLUMN NewTripID INT, -- Nếu đổi chuyến
ADD COLUMN PriceDifference DECIMAL(15,2), -- Chênh lệch giá (nếu đổi chuyến)
ADD FOREIGN KEY (NewTripID) REFERENCES Trips(TripID) ON DELETE SET NULL;
```

**Bảng mới: `TicketChanges` - Lịch sử đổi vé**
```sql
CREATE TABLE TicketChanges (
    ChangeID INT PRIMARY KEY AUTO_INCREMENT,
    TicketID INT NOT NULL,
    OldTripID INT NOT NULL,
    NewTripID INT NOT NULL,
    OldSeatID INT NOT NULL,
    NewSeatID INT NOT NULL,
    PriceDifference DECIMAL(15,2),
    ChangeReason TEXT,
    ChangedBy INT, -- Nhân viên thực hiện
    ChangeDate DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (TicketID) REFERENCES Tickets(TicketID) ON DELETE CASCADE,
    FOREIGN KEY (OldTripID) REFERENCES Trips(TripID),
    FOREIGN KEY (NewTripID) REFERENCES Trips(TripID),
    FOREIGN KEY (ChangedBy) REFERENCES Users(UserID) ON DELETE SET NULL
);
```

---

### 3. BLOCK THỜI GIAN ĐẶT VÉ ONLINE GẦN GIỜ KHỞI HÀNH

#### Yêu cầu:
- Không cho đặt vé online trong vòng 1 giờ trước giờ khởi hành
- Để tránh tranh chấp với khách mua tại quầy

#### Cải tiến bảng `Trips`:
```sql
ALTER TABLE Trips
ADD COLUMN OnlineBookingCutoff INT DEFAULT 60, -- Phút trước giờ khởi hành (mặc định 60 phút)
ADD COLUMN AllowOfflineBooking BOOLEAN DEFAULT TRUE; -- Cho phép bán tại quầy hay không
```

**Logic kiểm tra trong code:**
```sql
-- Stored Procedure kiểm tra thời gian đặt vé
DELIMITER //
CREATE PROCEDURE sp_CheckBookingEligibility(
    IN p_TripID INT,
    IN p_BookingType ENUM('Online', 'Tại quầy'),
    OUT p_CanBook BOOLEAN,
    OUT p_Message VARCHAR(255)
)
BEGIN
    DECLARE v_DepartureTime DATETIME;
    DECLARE v_OnlineCutoff INT;
    DECLARE v_TimeDiff INT;
    
    SELECT DepartureTime, OnlineBookingCutoff 
    INTO v_DepartureTime, v_OnlineCutoff
    FROM Trips WHERE TripID = p_TripID;
    
    SET v_TimeDiff = TIMESTAMPDIFF(MINUTE, NOW(), v_DepartureTime);
    
    IF p_BookingType = 'Online' AND v_TimeDiff < v_OnlineCutoff THEN
        SET p_CanBook = FALSE;
        SET p_Message = CONCAT('Không thể đặt vé online. Vui lòng đặt tại quầy (còn ', v_TimeDiff, ' phút tới giờ khởi hành)');
    ELSE
        SET p_CanBook = TRUE;
        SET p_Message = 'Có thể đặt vé';
    END IF;
END //
DELIMITER ;
```

---

### 4. ĐIỂM ĐÓN / ĐIỂM TRẢ CHI TIẾT

#### Yêu cầu:
- Mỗi lộ trình có nhiều điểm đón/trả
- Khách hàng chọn điểm đón/trả khi đặt vé
- Tài xế biết đón/trả từng khách ở đâu

#### Cải tiến bảng `RouteStops`:
```sql
ALTER TABLE RouteStops
ADD COLUMN StopName VARCHAR(200), -- Tên điểm dừng cụ thể
ADD COLUMN IsPickupPoint BOOLEAN DEFAULT TRUE, -- Có phải điểm đón không
ADD COLUMN IsDropoffPoint BOOLEAN DEFAULT TRUE, -- Có phải điểm trả không
ADD COLUMN StopAddress TEXT,
ADD COLUMN Latitude DECIMAL(10, 8),
ADD COLUMN Longitude DECIMAL(11, 8);
```

**Bảng `Passengers` đã có:**
- `PickupPoint` - Điểm đón
- `DropoffPoint` - Điểm trả

---

### 5. MÔ PHỎNG ĐỊNH VỊ TÀI XẾ VÀ QUY TRÌNH BÁO TRỄ

#### Yêu cầu:
- Theo dõi vị trí xe thời gian thực
- Tài xế báo trễ và lý do
- Hệ thống thông báo khách hàng

**Bảng mới: `TripTracking` - Theo dõi hành trình**
```sql
CREATE TABLE TripTracking (
    TrackingID INT PRIMARY KEY AUTO_INCREMENT,
    TripID INT NOT NULL,
    CurrentLatitude DECIMAL(10, 8),
    CurrentLongitude DECIMAL(11, 8),
    Speed DECIMAL(5,2), -- km/h
    EstimatedArrival DATETIME,
    DelayMinutes INT DEFAULT 0,
    DelayReason TEXT,
    TrafficStatus ENUM('Bình thường', 'Kẹt xe', 'Tai nạn', 'Khác'),
    RecordedAt DATETIME DEFAULT CURRENT_TIMESTAMP,
    RecordedBy INT, -- Tài xế hoặc hệ thống tự động
    FOREIGN KEY (TripID) REFERENCES Trips(TripID) ON DELETE CASCADE,
    FOREIGN KEY (RecordedBy) REFERENCES Users(UserID) ON DELETE SET NULL,
    INDEX idx_trip_tracking (TripID, RecordedAt)
);
```

**Bảng mới: `DelayNotifications` - Thông báo trễ chuyến**
```sql
CREATE TABLE DelayNotifications (
    NotificationID INT PRIMARY KEY AUTO_INCREMENT,
    TripID INT NOT NULL,
    DelayMinutes INT NOT NULL,
    DelayReason TEXT,
    CompensationType ENUM('Hoàn tiền', 'Voucher', 'Miễn phí chuyến sau', 'Không đền bù'),
    CompensationAmount DECIMAL(15,2),
    NotifiedAt DATETIME DEFAULT CURRENT_TIMESTAMP,
    NotifiedBy INT,
    FOREIGN KEY (TripID) REFERENCES Trips(TripID) ON DELETE CASCADE,
    FOREIGN KEY (NotifiedBy) REFERENCES Users(UserID) ON DELETE SET NULL
);
```

---

### 6. TÍNH TOÁN CHI PHÍ, LỢI NHUẬN VÀ ĐỀ XUẤT GIÁ VÉ

#### Yêu cầu:
- Tính chi phí mỗi chuyến (xăng, phí cao tốc, lương...)
- Tính lợi nhuận
- Đề xuất giá vé tối ưu

**Bảng mới: `TripCosts` - Chi phí từng chuyến**
```sql
CREATE TABLE TripCosts (
    CostID INT PRIMARY KEY AUTO_INCREMENT,
    TripID INT NOT NULL,
    
    -- Chi phí cố định
    FuelCost DECIMAL(15,2) DEFAULT 0, -- Xăng dầu
    TollFeeCost DECIMAL(15,2) DEFAULT 0, -- Phí cao tốc
    DriverSalary DECIMAL(15,2) DEFAULT 0, -- Lương tài xế
    MaintenanceCost DECIMAL(15,2) DEFAULT 0, -- Bảo trì
    InsuranceCost DECIMAL(15,2) DEFAULT 0, -- Bảo hiểm
    OtherCosts DECIMAL(15,2) DEFAULT 0, -- Chi phí khác
    
    -- Tổng chi phí
    TotalCost DECIMAL(15,2) GENERATED ALWAYS AS 
        (FuelCost + TollFeeCost + DriverSalary + MaintenanceCost + InsuranceCost + OtherCosts) STORED,
    
    -- Doanh thu
    Revenue DECIMAL(15,2) DEFAULT 0, -- Doanh thu từ vé bán ra
    
    -- Lợi nhuận
    Profit DECIMAL(15,2) GENERATED ALWAYS AS (Revenue - TotalCost) STORED,
    ProfitMargin DECIMAL(5,2), -- % lợi nhuận
    
    -- Metadata
    CalculatedAt DATETIME DEFAULT CURRENT_TIMESTAMP,
    UpdatedAt DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (TripID) REFERENCES Trips(TripID) ON DELETE CASCADE,
    INDEX idx_trip_profit (TripID, Profit)
);
```

**Bảng mới: `TollStations` - Trạm thu phí cao tốc**
```sql
CREATE TABLE TollStations (
    StationID INT PRIMARY KEY AUTO_INCREMENT,
    StationName VARCHAR(200) NOT NULL,
    LocationID INT,
    FeeAmount DECIMAL(10,2) NOT NULL,
    VehicleTypeID INT, -- Loại xe (theo VehicleTypes)
    IsActive BOOLEAN DEFAULT TRUE,
    CreatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (LocationID) REFERENCES Locations(LocationID),
    FOREIGN KEY (VehicleTypeID) REFERENCES VehicleTypes(TypeID)
);
```

**Bảng mới: `RouteTollStations` - Trạm thu phí trên tuyến**
```sql
CREATE TABLE RouteTollStations (
    ID INT PRIMARY KEY AUTO_INCREMENT,
    RouteID INT NOT NULL,
    StationID INT NOT NULL,
    StationOrder INT NOT NULL, -- Thứ tự qua trạm
    IsMandatory BOOLEAN DEFAULT TRUE, -- Bắt buộc đi qua hay không
    FOREIGN KEY (RouteID) REFERENCES Routes(RouteID) ON DELETE CASCADE,
    FOREIGN KEY (StationID) REFERENCES TollStations(StationID),
    UNIQUE KEY unique_route_station (RouteID, StationID)
);
```

**View: Tính toán đề xuất giá vé**
```sql
CREATE VIEW vw_PriceSuggestion AS
SELECT 
    t.TripID,
    r.RouteName,
    tc.TotalCost,
    COUNT(ts.SeatID) AS TotalSeats,
    COUNT(CASE WHEN ts.Status = 'Đã đặt' THEN 1 END) AS OccupiedSeats,
    -- Giá vé tối thiểu để hòa vốn
    CEILING(tc.TotalCost / COUNT(ts.SeatID)) AS MinPriceToBreakEven,
    -- Giá vé đề xuất (lợi nhuận 20%)
    CEILING(tc.TotalCost * 1.2 / COUNT(ts.SeatID)) AS SuggestedPrice,
    -- Tỷ lệ lấp đầy
    ROUND(COUNT(CASE WHEN ts.Status = 'Đã đặt' THEN 1 END) * 100.0 / COUNT(ts.SeatID), 2) AS OccupancyRate,
    -- Doanh thu hiện tại
    SUM(CASE WHEN tk.TicketStatus IN ('Đã xác nhận', 'Đã sử dụng') THEN tk.Price ELSE 0 END) AS CurrentRevenue
FROM Trips t
JOIN Routes r ON t.RouteID = r.RouteID
LEFT JOIN TripCosts tc ON t.TripID = tc.TripID
LEFT JOIN TripSeats ts ON t.TripID = ts.TripID
LEFT JOIN Tickets tk ON ts.SeatID = tk.SeatID
GROUP BY t.TripID;
```

---

### 7. ĐÁNH GIÁ CHI TIẾT (Tài xế, Xe, Dịch vụ)

#### Yêu cầu ban đầu: Đánh giá riêng tài xế, xe, dịch vụ
#### Đề xuất đơn giản hóa: Đánh giá chung 1 lần

**Cải tiến bảng `Reviews`:**
```sql
ALTER TABLE Reviews
-- Đánh giá tổng thể
MODIFY COLUMN Rating INT NOT NULL CHECK (Rating BETWEEN 1 AND 5),
ADD COLUMN Comment TEXT,

-- Đánh giá chi tiết (tùy chọn)
ADD COLUMN DriverRating INT CHECK (DriverRating BETWEEN 1 AND 5),
ADD COLUMN VehicleRating INT CHECK (VehicleRating BETWEEN 1 AND 5),
ADD COLUMN ServiceRating INT CHECK (ServiceRating BETWEEN 1 AND 5),
ADD COLUMN PunctualityRating INT CHECK (PunctualityRating BETWEEN 1 AND 5),

-- Phản hồi cụ thể
ADD COLUMN Feedback TEXT, -- Phản hồi chi tiết cho nhà xe

-- Trạng thái
ADD COLUMN ReviewStatus ENUM('Chờ duyệt', 'Đã duyệt', 'Bị ẩn') DEFAULT 'Đã duyệt',
ADD COLUMN AdminResponse TEXT, -- Phản hồi từ nhà xe
ADD COLUMN RespondedAt DATETIME;
```

---

### 8. GIỚI HẠN HOẠT ĐỘNG TÀI XẾ HỢP PHÁP

#### Yêu cầu:
- Tài xế làm tối đa 10 giờ/ngày
- Tối đa 4 giờ liên tục
- Tính công theo tháng/chuyến/giờ

**Bảng mới: `DriverWorklog` - Nhật ký làm việc tài xế**
```sql
CREATE TABLE DriverWorklog (
    LogID INT PRIMARY KEY AUTO_INCREMENT,
    DriverID INT NOT NULL,
    TripID INT,
    WorkDate DATE NOT NULL,
    StartTime DATETIME NOT NULL,
    EndTime DATETIME,
    TotalHours DECIMAL(4,2),
    TripCount INT DEFAULT 0, -- Số chuyến trong ngày
    SalaryType ENUM('Theo giờ', 'Theo chuyến', 'Cố định tháng') DEFAULT 'Theo chuyến',
    SalaryAmount DECIMAL(15,2),
    Status ENUM('Đang làm việc', 'Hoàn thành', 'Nghỉ giữa ca') DEFAULT 'Đang làm việc',
    ViolationNote TEXT, -- Ghi chú vi phạm (nếu vượt giờ)
    CreatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (DriverID) REFERENCES Drivers(DriverID) ON DELETE CASCADE,
    FOREIGN KEY (TripID) REFERENCES Trips(TripID) ON DELETE SET NULL,
    INDEX idx_driver_date (DriverID, WorkDate)
);
```

**Stored Procedure: Kiểm tra giờ làm việc hợp pháp**
```sql
DELIMITER //
CREATE PROCEDURE sp_CheckDriverWorkHours(
    IN p_DriverID INT,
    IN p_TripID INT,
    OUT p_CanWork BOOLEAN,
    OUT p_Message VARCHAR(255)
)
BEGIN
    DECLARE v_TodayHours DECIMAL(4,2);
    DECLARE v_TripDuration INT;
    DECLARE v_LastEndTime DATETIME;
    DECLARE v_BreakTime INT;
    
    -- Tổng giờ làm hôm nay
    SELECT COALESCE(SUM(TotalHours), 0) INTO v_TodayHours
    FROM DriverWorklog
    WHERE DriverID = p_DriverID 
    AND WorkDate = CURDATE();
    
    -- Thời gian chuyến mới
    SELECT TIMESTAMPDIFF(MINUTE, DepartureTime, ArrivalTime) / 60.0 
    INTO v_TripDuration
    FROM Trips WHERE TripID = p_TripID;
    
    -- Kiểm tra 10 giờ/ngày
    IF (v_TodayHours + v_TripDuration) > 10 THEN
        SET p_CanWork = FALSE;
        SET p_Message = 'Vượt giới hạn 10 giờ/ngày';
    
    -- Kiểm tra 4 giờ liên tục
    ELSE
        SELECT EndTime INTO v_LastEndTime
        FROM DriverWorklog
        WHERE DriverID = p_DriverID
        ORDER BY EndTime DESC LIMIT 1;
        
        SET v_BreakTime = TIMESTAMPDIFF(MINUTE, v_LastEndTime, NOW());
        
        IF v_TripDuration > 4 THEN
            SET p_CanWork = FALSE;
            SET p_Message = 'Chuyến xe vượt 4 giờ liên tục';
        ELSEIF v_BreakTime < 30 THEN
            SET p_CanWork = FALSE;
            SET p_Message = 'Chưa đủ thời gian nghỉ (cần 30 phút)';
        ELSE
            SET p_CanWork = TRUE;
            SET p_Message = 'Tài xế có thể nhận chuyến';
        END IF;
    END IF;
END //
DELIMITER ;
```

---

### 9. PHÂN QUYỀN NGƯỜI DÙNG CHƯA ĐĂNG NHẬP

#### Yêu cầu:
- Người chưa đăng nhập có thể xem thông tin và đặt vé
- Không lưu lịch sử
- Nhắc đăng nhập ở bước thanh toán

**Cải tiến bảng `Bookings`:**
```sql
-- Đã có sẵn CustomerID NULL cho khách vãng lai
-- Không cần thay đổi cấu trúc

-- Thêm trường để phân biệt
ALTER TABLE Bookings
ADD COLUMN IsGuestBooking BOOLEAN DEFAULT FALSE, -- Khách vãng lai
ADD COLUMN GuestSessionID VARCHAR(100); -- Session ID để tracking
```

**Logic xử lý:**
- Nếu `CustomerID = NULL` → Khách vãng lai
- Hiển thị popup "Đăng nhập để lưu lịch sử" ở bước thanh toán
- Sau thanh toán, gửi email/SMS kèm link tạo tài khoản

---

### 10. TẠO NHANH LỊCH TRÌNH VÀ CHÈN CHUYẾN

#### Yêu cầu:
- Tạo nhiều chuyến cùng lúc (cùng tuyến, lặp theo ngày)
- Chèn thêm chuyến cùng giờ hoặc lệch 5 phút

**Stored Procedure: Tạo hàng loạt chuyến**
```sql
DELIMITER //
CREATE PROCEDURE sp_CreateMultipleTrips(
    IN p_RouteID INT,
    IN p_StartDate DATE,
    IN p_EndDate DATE,
    IN p_DepartureTime TIME,
    IN p_VehicleIDs TEXT, -- Danh sách VehicleID (comma separated)
    IN p_DriverIDs TEXT,  -- Danh sách DriverID
    IN p_BasePrice DECIMAL(15,2),
    IN p_TimeOffset INT -- Khoảng cách phút giữa các xe (0 = cùng giờ, 5 = lệch 5 phút)
)
BEGIN
    DECLARE v_CurrentDate DATE;
    DECLARE v_VehicleID INT;
    DECLARE v_DriverID INT;
    DECLARE v_DepartureDateTime DATETIME;
    DECLARE v_Duration INT;
    DECLARE v_Counter INT DEFAULT 0;
    
    -- Lấy thời gian dự kiến
    SELECT EstimatedDuration INTO v_Duration
    FROM Routes WHERE RouteID = p_RouteID;
    
    SET v_CurrentDate = p_StartDate;
    
    WHILE v_CurrentDate <= p_EndDate DO
        SET v_Counter = 0;
        
        -- Lặp qua danh sách xe
        vehicle_loop: LOOP
            -- Parse VehicleID và DriverID
            -- (Cần implement hàm split string hoặc xử lý ở application layer)
            
            SET v_DepartureDateTime = TIMESTAMP(v_CurrentDate, p_DepartureTime);
            
            IF v_Counter > 0 THEN
                SET v_DepartureDateTime = DATE_ADD(v_DepartureDateTime, INTERVAL (v_Counter * p_TimeOffset) MINUTE);
            END IF;
            
            -- Tạo chuyến xe
            INSERT INTO Trips (RouteID, VehicleID, DriverID, DepartureTime, ArrivalTime, BasePrice, Status)
            VALUES (
                p_RouteID,
                v_VehicleID,
                v_DriverID,
                v_DepartureDateTime,
                DATE_ADD(v_DepartureDateTime, INTERVAL v_Duration MINUTE),
                p_BasePrice,
                'Chờ'
            );
            
            -- Tạo ghế cho chuyến vừa tạo
            CALL sp_GenerateSeatsForTrip(LAST_INSERT_ID());
            
            SET v_Counter = v_Counter + 1;
            
            -- Điều kiện thoát loop
            IF v_Counter >= ...(số lượng xe) THEN
                LEAVE vehicle_loop;
            END IF;
        END LOOP;
        
        SET v_CurrentDate = DATE_ADD(v_CurrentDate, INTERVAL 1 DAY);
    END WHILE;
END //
DELIMITER ;
```

---

## 📊 TỔNG KẾT CÁC BẢNG MỚI CẦN THÊM

1. ✅ **Passengers** - Quản lý thông tin hành khách chi tiết
2. ✅ **TicketChanges** - Lịch sử đổi vé
3. ✅ **TripTracking** - Theo dõi hành trình GPS
4. ✅ **DelayNotifications** - Thông báo trễ chuyến và đền bù
5. ✅ **TripCosts** - Chi phí và lợi nhuận từng chuyến
6. ✅ **TollStations** - Trạm thu phí cao tốc
7. ✅ **RouteTollStations** - Trạm thu phí trên tuyến
8. ✅ **DriverWorklog** - Nhật ký làm việc tài xế

---

## 🔧 CÁC BẢNG CẦN CẢI TIẾN

1. ✅ **Bookings** - Thêm trường phân biệt khách vãng lai
2. ✅ **Refunds** - Thêm loại hoàn tiền và đổi chuyến
3. ✅ **Trips** - Thêm thời gian block online booking
4. ✅ **RouteStops** - Chi tiết điểm đón/trả
5. ✅ **Reviews** - Đánh giá chi tiết hơn

---

## 🚀 CÁC STORED PROCEDURES MỚI

1. ✅ **sp_CheckBookingEligibility** - Kiểm tra thời gian đặt vé
2. ✅ **sp_CheckDriverWorkHours** - Kiểm tra giờ làm tài xế
3. ✅ **sp_CreateMultipleTrips** - Tạo hàng loạt chuyến
4. ✅ **sp_ChangeTicket** - Đổi vé sang chuyến khác
5. ✅ **sp_CheckInPassenger** - Check-in hành khách

---

## 📝 GHI CHÚ TRIỂN KHAI

### Ưu tiên cao (Cần làm ngay):
1. Bảng `Passengers` - Quản lý hành khách
2. Cải tiến `Bookings` - Khách vãng lai
3. Block thời gian online booking
4. Đánh giá chi tiết

### Ưu tiên trung bình:
1. Bảng `TripCosts` - Chi phí lợi nhuận
2. Bảng `TicketChanges` - Đổi vé
3. Nhật ký tài xế

### Ưu tiên thấp (Có thể làm sau):
1. GPS tracking
2. Trạm thu phí
3. Tạo hàng loạt chuyến

---

**Tài liệu này phân tích chi tiết các yêu cầu bổ sung và đề xuất cải tiến database.**
