# BÀI TẬP VỀ NHÀ 03 - THIẾT KẾ VÀ CÀI ĐẶT CSDL QUẢN LÝ CẦM ĐỒ

**Môn học:** Hệ Quản Trị Cơ Sở Dữ Liệu  
**Lớp:** 59KMT  
**Giảng viên:** Đỗ Duy Cốp

**Họ tên sinh viên:** Từ Văn Hải

**MSSV:** K235480106022 

## 1. Giới thiệu bài toán

Hệ thống **Quản lý Cầm đồ** được xây dựng nhằm số hóa quy trình vay tiền thế chấp tài sản tại các tiệm cầm đồ. Đây là bài toán quản lý có độ phức tạp cao do các đặc thù nghiệp vụ sau:

### Đặc điểm nghiệp vụ chính

| Đặc điểm | Mô tả |
|---|---|
| **Quan hệ nhiều-nhiều** | Một hợp đồng có thể bao gồm nhiều tài sản thế chấp; một khách hàng có thể có nhiều hợp đồng song song |
| **Cơ chế lãi suất kép** | Lãi đơ sn trước Deadline 1 (5.000đ/1.000.000đ/ngày), lãi kép sau Deadline 1 tính trên (Gốc + Lãi đơn tích lũy) |
| **Quản lý trạng thái phức tạp** | Hợp đồng và từng tài sản đều có vòng đời riêng với các trạng thái chuyển đổi tự động |
| **Trả nợ từng phần** | Hệ thống hỗ trợ trả góp, kèm logic nghiệp vụ để xác định tài sản nào được phép rút về |
| **Audit Log** | Mọi giao dịch tiền đều được ghi lại để đảm bảo tính minh bạch và traceable |

### Luồng nghiệp vụ tổng quan

```
[Khách đến cầm đồ]
       │
       ▼
[Tạo hợp đồng + Định giá tài sản]
       │
       ▼
[Thiết lập Deadline1 & Deadline2]
       │
       ├─── Trước Deadline1 ──► Tính LÃI ĐƠN (5k/1tr/ngày)
       │
       ├─── Sau Deadline1 ────► Tính LÃI KÉP (trên gốc + lãi đơn tích lũy)
       │                        Hợp đồng chuyển → "Qua han"
       │
       └─── Sau Deadline2 ────► Tài sản chuyển → "San sang thanh ly"
                                Hợp đồng → "Da thanh ly"
```

---

## 2. Thiết kế Cơ sở Dữ liệu (Nhiệm vụ 1)

### 2.1. Sơ đồ ERD

Sơ đồ thực thể - quan hệ (ERD) thể hiện toàn bộ các thực thể, thuộc tính, khóa chính, khóa ngoại và lực lượng quan hệ của hệ thống.

<img width="1377" height="650" alt="Untitled (1)" src="https://github.com/user-attachments/assets/96ccc4c8-c6bd-4a44-8dc3-652789189823" />
Sơ đồ ERD - Hệ thống Quản lý Cầm đồ
Ảnh trên là sơ đồ ERD được vẽ bằng công cụ dbdiagram.io. Các đường nối thể hiện quan hệ 1-N và N-N giữa các thực thể.

**Phân tích ERD:**

Từ sơ đồ, ta xác định được 5 thực thể chính và các quan hệ sau:

- `KhachHang` **1 — N** `HopDong`: Một khách hàng có thể ký nhiều hợp đồng cầm cố qua các thời điểm khác nhau.
- `HopDong` **N — N** `TaiSan`: Một hợp đồng có thể bao gồm nhiều tài sản; quan hệ này được phân rã qua bảng trung gian `HopDong_TaiSan`.
- `HopDong` **1 — N** `LichSuGiaoDich`: Một hợp đồng phát sinh nhiều lần trả tiền, mỗi lần là một dòng trong bảng Log.
- `NhanVien` **1 — N** `LichSuGiaoDich`: Mỗi giao dịch được ghi nhận nhân viên thực hiện.

---

### 2.2. Các bảng dữ liệu và chuẩn hóa 3NF

> **Lưu ý:** SQL Server không có kiểu `ENUM`. Thay vào đó, ta dùng `NVARCHAR` kết hợp `CHECK CONSTRAINT` để giới hạn tập giá trị hợp lệ.

#### Bảng 1: `KhachHang` — Thông tin khách hàng

| Cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---|---|---|---|
| `KhachHangID` | INT | PK, IDENTITY(1,1) | Mã khách hàng, tự tăng |
| `HoTen` | NVARCHAR(100) | NOT NULL | Họ và tên đầy đủ |
| `CMND_CCCD` | NVARCHAR(20) | UNIQUE, NOT NULL | Số CMND/CCCD |
| `SoDienThoai` | NVARCHAR(15) | NOT NULL | Số điện thoại liên hệ |
| `DiaChi` | NVARCHAR(255) | NULL | Địa chỉ thường trú |
| `NgayTao` | DATETIME | DEFAULT GETDATE() | Ngày tạo hồ sơ |

#### Bảng 2: `NhanVien` — Nhân viên tiệm cầm đồ

| Cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---|---|---|---|
| `NhanVienID` | INT | PK, IDENTITY(1,1) | Mã nhân viên |
| `HoTen` | NVARCHAR(100) | NOT NULL | Họ và tên |
| `SoDienThoai` | NVARCHAR(15) | NULL | Số điện thoại |
| `ChucVu` | NVARCHAR(50) | NULL | Chức vụ |

#### Bảng 3: `HopDong` — Hợp đồng vay tiền

| Cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---|---|---|---|
| `HopDongID` | INT | PK, IDENTITY(1,1) | Mã hợp đồng |
| `KhachHangID` | INT | FK → KhachHang | Khách hàng ký hợp đồng |
| `NhanVienID` | INT | FK → NhanVien | Nhân viên tiếp nhận |
| `NgayVay` | DATE | NOT NULL | Ngày bắt đầu hợp đồng |
| `SoTienVayGoc` | DECIMAL(18,0) | NOT NULL | Số tiền vay gốc (đồng) |
| `SoTienDaTra` | DECIMAL(18,0) | DEFAULT 0 | Tổng tiền khách đã trả |
| `Deadline1` | DATE | NOT NULL | Hạn chót tính lãi đơn |
| `Deadline2` | DATE | NOT NULL | Hạn chót trước khi thanh lý |
| `TrangThai` | NVARCHAR(20) | NOT NULL, CHECK | Trạng thái hợp đồng |

> **Giá trị CHECK TrangThai HopDong:** `'Dang vay'`, `'Qua han'`, `'Dang tra gop'`, `'Da thanh toan'`, `'Da thanh ly'`

#### Bảng 4: `TaiSan` — Tài sản thế chấp

| Cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---|---|---|---|
| `TaiSanID` | INT | PK, IDENTITY(1,1) | Mã tài sản |
| `TenTaiSan` | NVARCHAR(200) | NOT NULL | Tên / mô tả tài sản |
| `LoaiTaiSan` | NVARCHAR(100) | NULL | Loại (điện thoại, xe máy, vàng...) |
| `GiaTriDinhGia` | DECIMAL(18,0) | NOT NULL | Giá trị định giá tại thời điểm cầm |
| `TrangThai` | NVARCHAR(25) | NOT NULL, CHECK | Trạng thái tài sản hiện tại |

> **Giá trị CHECK TrangThai TaiSan:** `'Dang cam co'`, `'Da tra khach'`, `'San sang thanh ly'`, `'Da ban thanh ly'`

#### Bảng 5: `HopDong_TaiSan` — Bảng trung gian (quan hệ N-N)

| Cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---|---|---|---|
| `HopDongID` | INT | FK → HopDong | Mã hợp đồng |
| `TaiSanID` | INT | FK → TaiSan | Mã tài sản |

> **Khóa chính:** `(HopDongID, TaiSanID)` — Composite PK đảm bảo không trùng lặp.

#### Bảng 6: `LichSuGiaoDich` — Audit Log giao dịch

| Cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---|---|---|---|
| `GiaoDichID` | INT | PK, IDENTITY(1,1) | Mã giao dịch |
| `HopDongID` | INT | FK → HopDong | Hợp đồng liên quan |
| `NhanVienID` | INT | FK → NhanVien | Nhân viên thu tiền |
| `NgayGiaoDich` | DATETIME | DEFAULT GETDATE() | Ngày giờ giao dịch |
| `SoTienTra` | DECIMAL(18,0) | NOT NULL | Số tiền khách trả trong lần này |
| `DuNoTruocKhiTra` | DECIMAL(18,0) | NOT NULL | Tổng nợ tại thời điểm giao dịch |
| `DuNoSauKhiTra` | DECIMAL(18,0) | NOT NULL | Dư nợ còn lại sau khi trả |
| `LoaiGiaoDich` | NVARCHAR(50) | NULL | `'Tra no'`, `'Gia han'`, `'Thanh ly'` |
| `GhiChu` | NVARCHAR(500) | NULL | Ghi chú thêm |

---

### 2.2.1. Giải thích chuẩn hóa 3NF

**1NF (Dạng chuẩn 1):** Tất cả các bảng đều có khóa chính rõ ràng, không có thuộc tính đa trị hay thuộc tính lồng nhau. Danh sách tài sản trong hợp đồng được tách riêng sang bảng `HopDong_TaiSan` thay vì lưu dạng chuỗi.

**2NF (Dạng chuẩn 2):** Bảng `HopDong_TaiSan` có composite PK là `(HopDongID, TaiSanID)`. Mọi thuộc tính không khóa đều phụ thuộc đầy đủ vào toàn bộ khóa chính — không có phụ thuộc bộ phận.

**3NF (Dạng chuẩn 3):** Loại bỏ mọi phụ thuộc bắc cầu. Ví dụ: thông tin khách hàng (`HoTen`, `SoDienThoai`) không được lưu trong bảng `HopDong` mà tham chiếu qua `KhachHangID`. Tương tự, thông tin nhân viên thu tiền được tham chiếu qua `NhanVienID` trong `LichSuGiaoDich`.

---

### 2.3. Script tạo bảng (DDL)

```sql
-- ============================================================
-- DDL: TẠO CẤU TRÚC CSDL QUẢN LÝ CẦM ĐỒ (SQL SERVER / T-SQL)
-- ============================================================

-- Kiểm tra database 'QuanLyCamDo' đã tồn tại chưa bằng cách truy vấn sys.databases
-- Nếu chưa có thì tạo mới database
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = N'K235480106022_QuanLyCamDo')
    CREATE DATABASE K235480106022_QuanLyCamDo;
GO  -- Kết thúc batch; SQL Server bắt đầu thực thi batch mới từ đây

-- Chuyển ngữ cảnh làm việc sang database QuanLyCamDo
-- Tất cả lệnh sau sẽ chạy trong database này
USE K235480106022_QuanLyCamDo;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/30aae3e0-73a9-40cb-b6a4-9dae02783740" />


Tạo cơ sở dữ liệu

```sql
-- -------------------------------------------------------
-- BẢNG 1: KhachHang — lưu thông tin khách hàng vay tiền
-- -------------------------------------------------------
CREATE TABLE KhachHang (
    -- Khóa chính: số nguyên, tự tăng từ 1, bước nhảy 1; NOT NULL bắt buộc
    KhachHangID     INT             NOT NULL IDENTITY(1,1),
    -- Họ và tên đầy đủ của khách hàng; NVARCHAR hỗ trợ Unicode (tiếng Việt)
    HoTen           NVARCHAR(100)   NOT NULL,
    -- Số CMND hoặc CCCD 12 số; dùng làm định danh duy nhất
    CMND_CCCD       NVARCHAR(20)    NOT NULL,
    -- Số điện thoại liên hệ; lưu dạng chuỗi để giữ số 0 đầu
    SoDienThoai     NVARCHAR(15)    NOT NULL,
    -- Địa chỉ thường trú; NULL cho phép để trống nếu không có
    DiaChi          NVARCHAR(255)   NULL,
    -- Ngày tạo hồ sơ; DEFAULT GETDATE() tự động lấy thời điểm INSERT
    NgayTao         DATETIME        NOT NULL DEFAULT GETDATE(),
    -- Khai báo khóa chính trên cột KhachHangID
    CONSTRAINT PK_KhachHang      PRIMARY KEY (KhachHangID),
    -- Ràng buộc UNIQUE: mỗi số CMND/CCCD chỉ được xuất hiện một lần
    CONSTRAINT UQ_KhachHang_CMND UNIQUE (CMND_CCCD)
);
GO

-- -------------------------------------------------------
-- BẢNG 2: NhanVien — lưu thông tin nhân viên tiệm cầm đồ
-- -------------------------------------------------------
CREATE TABLE NhanVien (
    -- Khóa chính: số nguyên, tự tăng từ 1, bước nhảy 1
    NhanVienID      INT             NOT NULL IDENTITY(1,1),
    -- Họ và tên nhân viên; bắt buộc không được để trống
    HoTen           NVARCHAR(100)   NOT NULL,
    -- Số điện thoại nhân viên; NULL cho phép để trống
    SoDienThoai     NVARCHAR(15)    NULL,
    -- Chức vụ (Quản lý, Thu ngân...); NULL cho phép để trống
    ChucVu          NVARCHAR(50)    NULL,
    -- Khai báo khóa chính trên cột NhanVienID
    CONSTRAINT PK_NhanVien PRIMARY KEY (NhanVienID)
);
GO

-- -------------------------------------------------------
-- BẢNG 3: TaiSan — lưu thông tin tài sản thế chấp
-- Phải tạo TRƯỚC HopDong vì HopDong_TaiSan sẽ tham chiếu bảng này
-- -------------------------------------------------------
CREATE TABLE TaiSan (
    -- Khóa chính: số nguyên, tự tăng từ 1, bước nhảy 1
    TaiSanID        INT             NOT NULL IDENTITY(1,1),
    -- Tên hoặc mô tả chi tiết của tài sản; bắt buộc
    TenTaiSan       NVARCHAR(200)   NOT NULL,
    -- Phân loại tài sản (dien thoai, xe may, vang...); có thể để trống
    LoaiTaiSan      NVARCHAR(100)   NULL,
    -- Giá trị định giá tại thời điểm cầm cố (đơn vị đồng VND); bắt buộc
    GiaTriDinhGia   DECIMAL(18,0)   NOT NULL,
    -- Trạng thái hiện tại của tài sản; mặc định là 'Dang cam co' khi nhập
    TrangThai       NVARCHAR(25)    NOT NULL DEFAULT N'Dang cam co',
    -- Khai báo khóa chính trên cột TaiSanID
    CONSTRAINT PK_TaiSan           PRIMARY KEY (TaiSanID),
    -- CHECK CONSTRAINT: chỉ chấp nhận đúng 4 giá trị trạng thái hợp lệ
    -- 'Dang cam co'      → tài sản đang được giữ tại tiệm cầm đồ
    -- 'Da tra khach'     → đã hoàn trả về tay khách hàng
    -- 'San sang thanh ly'→ quá Deadline2, chuẩn bị bán thanh lý
    -- 'Da ban thanh ly'  → đã bán thanh lý thành công
    CONSTRAINT CK_TaiSan_TrangThai CHECK (TrangThai IN (
        N'Dang cam co', N'Da tra khach',
        N'San sang thanh ly', N'Da ban thanh ly'
    ))
);
GO

-- -------------------------------------------------------
-- BẢNG 4: HopDong — lưu hợp đồng vay tiền thế chấp tài sản
-- Phải tạo SAU KhachHang và NhanVien (có FK tới 2 bảng đó)
-- -------------------------------------------------------
CREATE TABLE HopDong (
    -- Khóa chính: số nguyên, tự tăng từ 1, bước nhảy 1
    HopDongID       INT             NOT NULL IDENTITY(1,1),
    -- Khóa ngoại liên kết tới khách hàng vay; bắt buộc
    KhachHangID     INT             NOT NULL,
    -- Khóa ngoại liên kết tới nhân viên tiếp nhận hợp đồng; bắt buộc
    NhanVienID      INT             NOT NULL,
    -- Ngày ký kết hợp đồng; kiểu DATE chỉ lưu ngày không lưu giờ
    NgayVay         DATE            NOT NULL,
    -- Số tiền vay gốc ban đầu; DECIMAL(18,0) đủ lớn, không có phần thập phân
    SoTienVayGoc    DECIMAL(18,0)   NOT NULL,
    -- Tổng số tiền khách đã nộp (tích lũy qua mọi lần trả); mặc định 0 khi tạo
    SoTienDaTra     DECIMAL(18,0)   NOT NULL DEFAULT 0,
    -- Hạn chót áp dụng lãi đơn; sau ngày này tính lãi kép
    Deadline1       DATE            NOT NULL,
    -- Hạn chót trước khi tài sản bị thanh lý; sau ngày này tài sản có thể bị bán
    Deadline2       DATE            NOT NULL,
    -- Trạng thái hợp đồng; mặc định 'Dang vay' ngay khi tạo mới
    TrangThai       NVARCHAR(20)    NOT NULL DEFAULT N'Dang vay',
    -- Khai báo khóa chính trên cột HopDongID
    CONSTRAINT PK_HopDong           PRIMARY KEY (HopDongID),
    -- Khóa ngoại: KhachHangID phải tồn tại trong bảng KhachHang
    -- Ngăn tạo hợp đồng cho khách hàng chưa đăng ký
    CONSTRAINT FK_HopDong_KhachHang FOREIGN KEY (KhachHangID)
        REFERENCES KhachHang(KhachHangID),
    -- Khóa ngoại: NhanVienID phải tồn tại trong bảng NhanVien
    -- Ngăn gán hợp đồng cho nhân viên không tồn tại
    CONSTRAINT FK_HopDong_NhanVien  FOREIGN KEY (NhanVienID)
        REFERENCES NhanVien(NhanVienID),
    -- CHECK CONSTRAINT: chỉ chấp nhận 5 giá trị trạng thái hợp lệ
    -- 'Dang vay'    → hợp đồng đang hiệu lực, chưa quá hạn
    -- 'Qua han'     → đã qua Deadline1, tính lãi kép (nợ xấu)
    -- 'Dang tra gop'→ khách đã trả một phần, chưa trả hết
    -- 'Da thanh toan'→ khách đã trả đủ, hợp đồng kết thúc bình thường
    -- 'Da thanh ly' → tài sản đã bị bán thanh lý
    CONSTRAINT CK_HopDong_TrangThai CHECK (TrangThai IN (
        N'Dang vay', N'Qua han', N'Dang tra gop',
        N'Da thanh toan', N'Da thanh ly'
    ))
);
GO

-- -------------------------------------------------------
-- BẢNG 5: HopDong_TaiSan — bảng trung gian phá vỡ quan hệ N-N
-- Một hợp đồng ↔ nhiều tài sản; một tài sản ↔ nhiều hợp đồng (qua thời gian)
-- Phải tạo SAU HopDong và TaiSan
-- -------------------------------------------------------
CREATE TABLE HopDong_TaiSan (
    -- Khóa ngoại trỏ tới bảng HopDong; bắt buộc
    HopDongID       INT     NOT NULL,
    -- Khóa ngoại trỏ tới bảng TaiSan; bắt buộc
    TaiSanID        INT     NOT NULL,
    -- Composite PK: cặp (HopDongID, TaiSanID) phải là duy nhất
    -- Đảm bảo một tài sản không bị liên kết 2 lần vào cùng một hợp đồng
    CONSTRAINT PK_HopDong_TaiSan PRIMARY KEY (HopDongID, TaiSanID),
    -- Khóa ngoại: HopDongID phải tồn tại trong HopDong
    CONSTRAINT FK_HDT_HopDong    FOREIGN KEY (HopDongID)
        REFERENCES HopDong(HopDongID),
    -- Khóa ngoại: TaiSanID phải tồn tại trong TaiSan
    CONSTRAINT FK_HDT_TaiSan     FOREIGN KEY (TaiSanID)
        REFERENCES TaiSan(TaiSanID)
);
GO

-- -------------------------------------------------------
-- BẢNG 6: LichSuGiaoDich — Audit Log; ghi lại MỌI biến động tiền
-- Không bao giờ ghi đè, chỉ INSERT thêm dòng mới mỗi lần phát sinh
-- Phải tạo SAU HopDong và NhanVien
-- -------------------------------------------------------
CREATE TABLE LichSuGiaoDich (
    -- Khóa chính: số nguyên, tự tăng từ 1, bước nhảy 1
    GiaoDichID          INT             NOT NULL IDENTITY(1,1),
    -- Khóa ngoại: giao dịch thuộc về hợp đồng nào; bắt buộc
    HopDongID           INT             NOT NULL,
    -- Khóa ngoại: nhân viên nào thực hiện thu tiền; bắt buộc
    NhanVienID          INT             NOT NULL,
    -- Thời điểm phát sinh giao dịch; DEFAULT GETDATE() tự điền khi INSERT
    NgayGiaoDich        DATETIME        NOT NULL DEFAULT GETDATE(),
    -- Số tiền khách nộp trong lần giao dịch này (không phải tổng cộng)
    SoTienTra           DECIMAL(18,0)   NOT NULL,
    -- Dư nợ tính TRƯỚC khi trừ số tiền trả lần này; dùng để đối chiếu
    DuNoTruocKhiTra     DECIMAL(18,0)   NOT NULL,
    -- Dư nợ CÒN LẠI sau khi trừ số tiền trả lần này
    DuNoSauKhiTra       DECIMAL(18,0)   NOT NULL,
    -- Phân loại giao dịch: 'Tra no', 'Gia han', 'Thanh ly'; NULL nếu không phân loại
    LoaiGiaoDich        NVARCHAR(50)    NULL,
    -- Ghi chú tự do thêm thông tin; NULL nếu không có
    GhiChu              NVARCHAR(500)   NULL,
    -- Khai báo khóa chính trên cột GiaoDichID
    CONSTRAINT PK_LichSuGiaoDich PRIMARY KEY (GiaoDichID),
    -- Khóa ngoại: HopDongID phải tồn tại trong HopDong
    CONSTRAINT FK_LSGD_HopDong   FOREIGN KEY (HopDongID)
        REFERENCES HopDong(HopDongID),
    -- Khóa ngoại: NhanVienID phải tồn tại trong NhanVien
    CONSTRAINT FK_LSGD_NhanVien  FOREIGN KEY (NhanVienID)
        REFERENCES NhanVien(NhanVienID)
);
GO
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/e1ecf7da-f9ed-4296-9a04-1e81e0e4664b" />

Tạo bảng thành công trong SSMS

> **Phân tích:** Sau khi chạy script DDL trong SSMS, cửa sổ "Messages" hiển thị `Command(s) completed successfully` cho từng batch. Object Explorer liệt kê đủ 6 bảng dưới schema `dbo`. Các `CHECK CONSTRAINT`, `FOREIGN KEY` và `UNIQUE` đều được tạo đúng — có thể kiểm tra qua node `Constraints` của từng bảng.

---
```sql
-- =========================================
-- CHÈN DỮ LIỆU MẪU CHO BẢNG KhachHang
-- =========================================
INSERT INTO KhachHang (HoTen, CMND_CCCD, SoDienThoai, DiaChi)
VALUES
(N'Nguyễn Văn An',  N'001203456789', N'0912345678', N'Hà Nội'),
(N'Trần Thị Bình',  N'001203456790', N'0988123456', N'Hải Phòng'),
(N'Lê Minh Cường',  N'001203456791', N'0977666555', N'Đà Nẵng'),
(N'Phạm Thu Dung',  N'001203456792', N'0966888999', N'TP Hồ Chí Minh'),
(N'Hoàng Quốc Em',  N'001203456793', N'0933555777', N'Cần Thơ');
GO

-- =========================================
-- CHÈN DỮ LIỆU MẪU CHO BẢNG NhanVien
-- =========================================
INSERT INTO NhanVien (HoTen, SoDienThoai, ChucVu)
VALUES
(N'Nguyễn Thành Công', N'0909000001', N'Quản lý'),
(N'Trần Mỹ Linh',      N'0909000002', N'Thu ngân'),
(N'Phạm Quốc Bảo',     N'0909000003', N'Nhân viên');
GO

-- =========================================
-- CHÈN DỮ LIỆU MẪU CHO BẢNG TaiSan
-- =========================================
INSERT INTO TaiSan (TenTaiSan, LoaiTaiSan, GiaTriDinhGia, TrangThai)
VALUES
(N'iPhone 15 Pro Max', N'Điện thoại', 30000000, N'Dang cam co'),
(N'Xe máy Honda SH',   N'Xe máy',     85000000, N'Dang cam co'),
(N'Laptop Dell XPS',   N'Laptop',     25000000, N'Dang cam co'),
(N'Nhẫn vàng 24K',     N'Vàng',       15000000, N'Dang cam co'),
(N'MacBook Pro M3',    N'Laptop',     45000000, N'Dang cam co');
GO

-- =========================================
-- CHÈN DỮ LIỆU MẪU CHO BẢNG HopDong
-- =========================================
INSERT INTO HopDong
(
    KhachHangID,
    NhanVienID,
    NgayVay,
    SoTienVayGoc,
    SoTienDaTra,
    Deadline1,
    Deadline2,
    TrangThai
)
VALUES
(1, 1, '2026-05-01', 20000000, 5000000, '2026-06-01', '2026-07-01', N'Dang tra gop'),
(2, 2, '2026-05-03', 50000000, 0,       '2026-06-03', '2026-07-03', N'Dang vay'),
(3, 2, '2026-04-15', 15000000, 15000000,'2026-05-15', '2026-06-15', N'Da thanh toan'),
(4, 3, '2026-03-10', 10000000, 2000000, '2026-04-10', '2026-05-10', N'Qua han'),
(5, 1, '2026-05-08', 30000000, 0,       '2026-06-08', '2026-07-08', N'Dang vay');
GO

-- =========================================
-- CHÈN DỮ LIỆU MẪU CHO BẢNG HopDong_TaiSan
-- =========================================
INSERT INTO HopDong_TaiSan (HopDongID, TaiSanID)
VALUES
(1, 1),
(2, 2),
(3, 4),
(4, 3),
(5, 5);
GO

-- =========================================
-- CHÈN DỮ LIỆU MẪU CHO BẢNG LichSuGiaoDich
-- =========================================
INSERT INTO LichSuGiaoDich
(
    HopDongID,
    NhanVienID,
    NgayGiaoDich,
    SoTienTra,
    DuNoTruocKhiTra,
    DuNoSauKhiTra,
    LoaiGiaoDich,
    GhiChu
)
VALUES
(1, 2, GETDATE(), 3000000, 20000000, 17000000, N'Tra no', N'Khách trả lần 1'),
(1, 2, GETDATE(), 2000000, 17000000, 15000000, N'Tra no', N'Khách trả lần 2'),
(3, 1, GETDATE(), 15000000,15000000, 0,        N'Tra no', N'Tất toán hợp đồng'),
(4, 3, GETDATE(), 2000000, 10000000, 8000000,  N'Tra no', N'Trả một phần'),
(2, 2, GETDATE(), 0,        50000000,50000000, N'Gia han', N'Gia hạn thêm 30 ngày');
GO
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/6dd00511-8763-430a-98e9-b8e6412ee971" />

Chèn dữ liệu vào các bảng

## 3. Cài đặt các tính năng cốt lõi (Nhiệm vụ 2)

### Event 1: Đăng ký hợp đồng mới

Store Procedure `sp_TiepNhanHopDong` xử lý toàn bộ luồng đăng ký: tạo hợp đồng, liên kết danh sách tài sản và thiết lập hai mốc deadline.

**Tham số đầu vào:**

| Tham số | Kiểu | Mô tả |
|---|---|---|
| `@KhachHangID` | INT | ID khách hàng (đã tồn tại) |
| `@NhanVienID` | INT | ID nhân viên tiếp nhận |
| `@SoTienVay` | DECIMAL(18,0) | Số tiền vay gốc |
| `@NgayVay` | DATE | Ngày ký hợp đồng |
| `@SoNgayDeadline1` | INT | Số ngày đến Deadline1 (ví dụ: 30) |
| `@SoNgayDeadline2` | INT | Số ngày từ Deadline1 đến Deadline2 (ví dụ: 30) |
| `@TaiSanIDs` | NVARCHAR(MAX) | Chuỗi TaiSanID cách nhau bởi dấu phẩy, ví dụ `N'1,2,3'` |

```sql
-- Tạo Stored Procedure tiếp nhận hợp đồng mới
-- Lưu ý: CREATE PROCEDURE phải là câu lệnh ĐẦU TIÊN trong batch → cần GO trước đó
CREATE PROCEDURE sp_TiepNhanHopDong
    @KhachHangID        INT,            -- ID khách hàng vay tiền (phải tồn tại trong bảng KhachHang)
    @NhanVienID         INT,            -- ID nhân viên tiếp nhận hợp đồng
    @SoTienVay          DECIMAL(18,0),  -- Số tiền vay gốc (đơn vị: đồng VND)
    @NgayVay            DATE,           -- Ngày ký hợp đồng (chỉ lưu ngày)
    @SoNgayDeadline1    INT,            -- Số ngày kể từ NgayVay tới Deadline1 (ví dụ: 30)
    @SoNgayDeadline2    INT,            -- Số ngày kể từ Deadline1 tới Deadline2 (ví dụ: 30)
    @TaiSanIDs          NVARCHAR(MAX)   -- Danh sách TaiSanID phân cách bởi dấu phẩy, ví dụ N'1,2,3'
AS
BEGIN
    SET NOCOUNT ON;  -- Tắt thông báo "n row(s) affected" để tránh nhiễu kết quả trả về

    -- Khai báo biến nội bộ dùng trong procedure
    DECLARE @HopDongID      INT;                    -- Sẽ lưu ID hợp đồng vừa tạo
    DECLARE @Deadline1      DATE;                   -- Ngày hết hạn lãi đơn
    DECLARE @Deadline2      DATE;                   -- Ngày cuối trước khi thanh lý
    DECLARE @TongGiaTri     DECIMAL(18,0) = 0;      -- Tổng giá trị định giá các tài sản

    -- Kiểm tra đầu vào: số tiền vay phải dương
    IF @SoTienVay <= 0
    BEGIN
        -- THROW: ném lỗi có mã 50001, thông báo lỗi, trạng thái 1
        -- Mã lỗi người dùng phải >= 50000
        THROW 50001, N'So tien vay phai lon hon 0.', 1;
        RETURN;  -- Thoát procedure ngay lập tức sau khi ném lỗi
    END

    BEGIN TRY  -- Bắt đầu khối xử lý an toàn; nếu có lỗi sẽ nhảy vào BEGIN CATCH
        BEGIN TRANSACTION;  -- Bắt đầu transaction; mọi thay đổi sẽ chưa lưu thật cho đến COMMIT

        -- Tính Deadline1: cộng thêm @SoNgayDeadline1 ngày vào NgayVay
        SET @Deadline1 = DATEADD(DAY, @SoNgayDeadline1, @NgayVay);
        -- Tính Deadline2: cộng thêm @SoNgayDeadline2 ngày vào Deadline1
        SET @Deadline2 = DATEADD(DAY, @SoNgayDeadline2, @Deadline1);

        -- Chèn bản ghi hợp đồng mới vào bảng HopDong
        INSERT INTO HopDong (
            KhachHangID, NhanVienID, NgayVay,   -- Thông tin cơ bản
            SoTienVayGoc, SoTienDaTra,           -- Số tiền vay và số tiền đã trả (bắt đầu = 0)
            Deadline1, Deadline2, TrangThai      -- Hai mốc deadline và trạng thái ban đầu
        ) VALUES (
            @KhachHangID, @NhanVienID, @NgayVay,
            @SoTienVay, 0,                       -- SoTienDaTra = 0 vì mới vay, chưa trả gì
            @Deadline1, @Deadline2, N'Dang vay'  -- Trạng thái mặc định là 'Dang vay'
        );

        -- Lấy ID của hợp đồng vừa INSERT (an toàn trong cùng session, không bị ảnh hưởng bởi session khác)
        SET @HopDongID = SCOPE_IDENTITY();

        -- Liên kết các tài sản vào hợp đồng qua bảng trung gian HopDong_TaiSan
        -- STRING_SPLIT: hàm tách chuỗi theo ký tự phân cách (yêu cầu SQL Server 2016+)
        -- TRIM: xóa khoảng trắng thừa quanh mỗi giá trị (ví dụ '1, 2 ,3' → '1','2','3')
        -- CAST(... AS INT): chuyển chuỗi sang số nguyên để khớp kiểu TaiSanID
        INSERT INTO HopDong_TaiSan (HopDongID, TaiSanID)
        SELECT @HopDongID, CAST(TRIM(value) AS INT)
        FROM STRING_SPLIT(@TaiSanIDs, ',')      -- Tách chuỗi @TaiSanIDs theo dấu phẩy
        WHERE TRIM(value) <> N'';               -- Bỏ qua phần tử rỗng (nếu có dấu phẩy dư)

        -- Cập nhật trạng thái các tài sản được liên kết thành 'Dang cam co'
        -- Dùng subquery với STRING_SPLIT để lọc đúng danh sách TaiSanID
        UPDATE TaiSan
        SET TrangThai = N'Dang cam co'
        WHERE TaiSanID IN (
            SELECT CAST(TRIM(value) AS INT)
            FROM STRING_SPLIT(@TaiSanIDs, ',')
            WHERE TRIM(value) <> N''
        );

        -- Tính tổng giá trị định giá của tất cả tài sản thuộc hợp đồng vừa tạo
        -- ISNULL(..., 0): nếu SUM trả về NULL (không có tài sản) thì dùng 0
        SELECT @TongGiaTri = ISNULL(SUM(ts.GiaTriDinhGia), 0)
        FROM TaiSan ts
        INNER JOIN HopDong_TaiSan hdt ON ts.TaiSanID = hdt.TaiSanID
        WHERE hdt.HopDongID = @HopDongID;  -- Chỉ tính tài sản của hợp đồng này

        -- Kiểm tra nghiệp vụ: tổng giá trị tài sản phải >= số tiền vay
        -- (Không thể vay 30tr mà chỉ thế chấp tài sản trị giá 10tr)
        IF @TongGiaTri < @SoTienVay
        BEGIN
            ROLLBACK TRANSACTION;  -- Huỷ toàn bộ thay đổi đã làm trong transaction này
            THROW 50002,
                N'Tong gia tri tai san nho hon so tien vay, hop dong khong hop le.',
                1;
            RETURN;  -- Thoát procedure
        END

        COMMIT TRANSACTION;  -- Lưu thật tất cả thay đổi vào database

        -- Trả về kết quả thông báo tạo hợp đồng thành công
        SELECT
            @HopDongID      AS HopDongID,       -- ID hợp đồng vừa tạo
            @SoTienVay      AS SoTienVayGoc,    -- Số tiền vay gốc
            @Deadline1      AS Deadline1,        -- Ngày hết lãi đơn
            @Deadline2      AS Deadline2,        -- Ngày cuối trước thanh lý
            @TongGiaTri     AS TongGiaTriTaiSan, -- Tổng giá trị tài sản thế chấp
            N'Tao hop dong thanh cong' AS ThongBao;  -- Thông báo thành công
    END TRY
    BEGIN CATCH  -- Bắt mọi lỗi xảy ra trong khối TRY ở trên
        -- Nếu vẫn còn transaction đang mở (@@TRANCOUNT > 0) thì huỷ nó
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        -- Re-throw lỗi gốc ra ngoài để caller biết nguyên nhân
        THROW;
    END CATCH
END;
GO
```
<img width="1918" height="1073" alt="image" src="https://github.com/user-attachments/assets/6596fa77-2a89-4a0c-9682-4b667c87f5c7" />

Tạo sp_TiepNhanHopDong
**Cách gọi Store Procedure:**

```sql
EXEC sp_TiepNhanHopDong
    @KhachHangID     = 1,
    @NhanVienID      = 1,
    @SoTienVay       = 15000000,
    @NgayVay         = '2026-04-11',
    @SoNgayDeadline1 = 30,
    @SoNgayDeadline2 = 30,
    @TaiSanIDs       = N'1,2';
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/292918e6-d0db-4a22-8b09-dc0925e2031d" />
Kết quả chạy sp_TiepNhanHopDong trong SSMS

> **Phân tích kết quả:** SSMS hiển thị result set với `HopDongID` vừa được cấp (ví dụ: 1), `Deadline1 = 2026-05-11`, `Deadline2 = 2026-06-10`, tổng giá trị tài sản và thông báo thành công. Tab "Messages" in `(1 row(s) affected)` cho từng INSERT. Kiểm tra `SELECT * FROM HopDong_TaiSan WHERE HopDongID = 1` sẽ thấy 2 dòng tương ứng với 2 tài sản được liên kết.

---

### Event 2: Tính toán công nợ

#### 2.1. Giải thích thuật toán tính lãi

**Lãi đơn (Simple Interest)** — áp dụng từ `NgayVay` đến hết `Deadline1`:

```
LaiDon = SoTienVayGoc × 0.005 × SoNgayLaiDon
       (vì 5.000đ / 1.000.000đ = 0.005 mỗi ngày)
```

**Lãi kép (Compound Interest)** — áp dụng từ ngày sau `Deadline1` đến `TargetDate`:

```
GocKep = SoTienVayGoc + LaiDon_tichLuy_denDeadline1
LaiKep = GocKep × ( POWER(1.005, SoNgayKep) − 1 )
```

Hàm `POWER()` trong T-SQL tính lũy thừa — chính xác cho công thức lãi kép theo ngày.

#### 2.2. Function `fn_CalcMoney_Transaction`

Trả về tổng số tiền gốc + lãi của một hợp đồng tính đến `@TargetDate`.

```sql
-- Tạo scalar function tính tổng nợ (gốc + lãi) của một hợp đồng đến một ngày cụ thể
-- Scalar function: trả về một giá trị duy nhất kiểu DECIMAL(18,2)
-- CREATE FUNCTION phải là câu lệnh đầu tiên trong batch → cần GO trước đó
CREATE FUNCTION dbo.fn_CalcMoney_Transaction (
    @HopDongID  INT,   -- ID hợp đồng cần tính nợ
    @TargetDate DATE   -- Ngày cần tính nợ đến (có thể là ngày hôm nay hoặc tương lai)
)
RETURNS DECIMAL(18,2)  -- Kiểu trả về: số thực 18 chữ số tổng, 2 chữ số thập phân
AS
BEGIN
    -- Khai báo các biến cần dùng trong function
    DECLARE @SoTienVayGoc   DECIMAL(18,0);  -- Số tiền vay gốc lấy từ bảng HopDong
    DECLARE @NgayVay        DATE;           -- Ngày bắt đầu hợp đồng
    DECLARE @Deadline1      DATE;           -- Ngày hết hạn lãi đơn
    DECLARE @NgayLaiDon     INT;            -- Số ngày tính lãi đơn
    DECLARE @NgayLaiKep     INT;            -- Số ngày tính lãi kép (sau Deadline1)
    DECLARE @LaiDon         DECIMAL(18,2);  -- Tiền lãi đơn tính được
    DECLARE @GocKep         DECIMAL(18,2);  -- Gốc dùng để tính lãi kép = Gốc + LãiĐơn tích lũy
    DECLARE @LaiKep         DECIMAL(18,2);  -- Tiền lãi kép tính được
    DECLARE @KetQua         DECIMAL(18,2);  -- Tổng kết quả trả về (Gốc + Lãi)

    -- Lấy thông tin cơ bản của hợp đồng từ bảng HopDong
    SELECT
        @SoTienVayGoc = SoTienVayGoc,  -- Gán số tiền vay gốc
        @NgayVay      = NgayVay,        -- Gán ngày vay
        @Deadline1    = Deadline1       -- Gán mốc deadline 1
    FROM HopDong
    WHERE HopDongID = @HopDongID;      -- Chỉ lấy đúng hợp đồng cần tính

    -- Rẽ nhánh logic tính lãi dựa vào vị trí của TargetDate so với Deadline1
    IF @TargetDate <= @Deadline1
    -- NHÁNH 1: TargetDate nằm TRƯỚC hoặc ĐÚNG Deadline1 → chỉ tính lãi đơn
    BEGIN
        -- Tính số ngày đã vay (từ ngày vay đến ngày cần tính)
        SET @NgayLaiDon = DATEDIFF(DAY, @NgayVay, @TargetDate);
        -- Công thức lãi đơn: Gốc × 0.005 × số ngày
        -- 0.005 = 5.000đ / 1.000.000đ (tỷ lệ lãi theo đề bài)
        SET @LaiDon     = @SoTienVayGoc * 0.005 * @NgayLaiDon;
        -- Tổng = Gốc + Lãi đơn
        SET @KetQua     = @SoTienVayGoc + @LaiDon;
    END
    ELSE
    -- NHÁNH 2: TargetDate nằm SAU Deadline1 → tính lãi đơn đến Deadline1 + lãi kép từ đó
    BEGIN
        -- Bước 1: Tính lãi đơn từ NgayVay đến Deadline1
        SET @NgayLaiDon = DATEDIFF(DAY, @NgayVay, @Deadline1);
        SET @LaiDon     = @SoTienVayGoc * 0.005 * @NgayLaiDon;

        -- Bước 2: Tính gốc mới dùng để nhân lãi kép = Gốc gốc + toàn bộ lãi đơn tích lũy
        SET @GocKep     = @SoTienVayGoc + @LaiDon;

        -- Bước 3: Tính số ngày tính lãi kép (từ Deadline1 đến TargetDate)
        SET @NgayLaiKep = DATEDIFF(DAY, @Deadline1, @TargetDate);

        -- Bước 4: Áp dụng công thức lãi kép: A = P × (1 + r)^n - P = P × ((1+r)^n - 1)
        -- POWER(base, exponent): tính lũy thừa trong T-SQL
        -- CAST(1.005 AS FLOAT): ép kiểu sang FLOAT để POWER tính chính xác số thực
        SET @LaiKep = @GocKep * (POWER(CAST(1.005 AS FLOAT), @NgayLaiKep) - 1);

        -- Tổng cuối = Gốc kép + Lãi kép = tổng số phải trả đến TargetDate
        SET @KetQua = @GocKep + @LaiKep;
    END

    -- Làm tròn kết quả đến hàng đồng (0 chữ số thập phân) và trả về
    RETURN ROUND(@KetQua, 0);
END;
GO
```

<img width="1917" height="1078" alt="image" src="https://github.com/user-attachments/assets/14097009-6436-44d6-a948-d3470653ae1c" />
Tính toán công nợ

#### 2.3. Function `fn_CalcMoneyContract`

Trả về **dư nợ thực tế còn phải trả** (tổng nợ trừ đi phần đã nộp).

```sql
-- Tạo scalar function tính DƯ NỢ THỰC TẾ còn lại của hợp đồng đến ngày chỉ định
-- = Tổng nợ (gốc + lãi) tính đến TargetDate - Số tiền khách đã trả
-- Phụ thuộc vào fn_CalcMoney_Transaction → phải tạo function đó trước
CREATE FUNCTION dbo.fn_CalcMoneyContract (
    @ContractID INT,   -- ID hợp đồng cần tra cứu dư nợ
    @TargetDate DATE   -- Ngày cần tính đến
)
RETURNS DECIMAL(18,2)  -- Trả về số thực (dư nợ còn lại, đơn vị đồng)
AS
BEGIN
    -- Khai báo biến lưu kết quả trung gian
    DECLARE @TongNo     DECIMAL(18,2);  -- Tổng nợ gốc + lãi đến TargetDate
    DECLARE @DaTra      DECIMAL(18,0);  -- Tổng số tiền khách đã trả từ trước đến nay
    DECLARE @DuNo       DECIMAL(18,2);  -- Dư nợ thực tế = TổngNợ - ĐãTrả

    -- Gọi function đã tạo ở trên để tính tổng nợ (gốc + lãi) đến TargetDate
    SET @TongNo = dbo.fn_CalcMoney_Transaction(@ContractID, @TargetDate);

    -- Lấy số tiền khách đã trả từ cột SoTienDaTra trong bảng HopDong
    -- ISNULL(..., 0): nếu không tìm thấy hợp đồng hoặc cột NULL thì dùng 0
    SELECT @DaTra = ISNULL(SoTienDaTra, 0)
    FROM HopDong
    WHERE HopDongID = @ContractID;

    -- Tính dư nợ = Tổng nợ - Đã trả
    SET @DuNo = @TongNo - @DaTra;
    -- Đảm bảo dư nợ không âm (trường hợp khách trả dư)
    IF @DuNo < 0 SET @DuNo = 0;

    -- Làm tròn đến hàng đồng và trả về
    RETURN ROUND(@DuNo, 0);
END;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/a7883620-9a65-44ad-b64d-f911042aa358" />
Tạo function tính dư nợ thực tế


**Kiểm thử hai function:**

```sql
-- Tính tổng nợ và dư nợ thực tế của hợp đồng ID=1 tính đến ngày hôm nay
-- CAST(GETDATE() AS DATE): chuyển DATETIME hiện tại sang DATE (bỏ phần giờ:phút:giây)
SELECT
    dbo.fn_CalcMoney_Transaction(1, CAST(GETDATE() AS DATE)) AS TongNoGocLai,  -- Tổng gốc + lãi
    dbo.fn_CalcMoneyContract(1,     CAST(GETDATE() AS DATE)) AS DuNoCon;       -- Dư nợ còn lại

-- Dự phóng tổng nợ và dư nợ sau 30 ngày nữa kể từ hôm nay
-- DATEADD(DAY, 30, ...): cộng thêm 30 ngày vào ngày hiện tại
SELECT
    dbo.fn_CalcMoney_Transaction(1, DATEADD(DAY, 30, CAST(GETDATE() AS DATE))) AS No_Sau30Ngay,     -- Tổng nợ sau 30 ngày
    dbo.fn_CalcMoneyContract(1,     DATEADD(DAY, 30, CAST(GETDATE() AS DATE))) AS DuNo_Sau30Ngay;   -- Dư nợ sau 30 ngày
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/9ba39032-13d5-4b8e-9a08-e2e9a3de0be0" />

Kết quả kiểm thử hai function tính lãi
> **Phân tích kết quả:**
>
> **Ví dụ minh họa:** Vay 15.000.000đ ngày 11/04/2026, Deadline1 = 11/05/2026. Kiểm tra ngày 01/05/2026 (20 ngày lãi đơn):
> - Lãi đơn = 15.000.000 × 0.005 × 20 = **1.500.000đ**
> - Tổng phải trả = **16.500.000đ**
>
> Kiểm tra ngày 20/05/2026 (đã qua Deadline1 thêm 9 ngày):
> - Lãi đơn đến Deadline1 = 15.000.000 × 0.005 × 30 = **2.250.000đ**
> - Gốc kép = 15.000.000 + 2.250.000 = **17.250.000đ**
> - Lãi kép 9 ngày = 17.250.000 × (POWER(1.005, 9) − 1) ≈ **789.000đ**
> - Tổng ≈ **18.039.000đ**
>
> SSMS hiển thị hai cột `TongNoGocLai` và `DuNoCon` dạng số nguyên (đã làm tròn qua `ROUND`).

---

### Event 3: Xử lý trả nợ & Hoàn trả tài sản

Store Procedure `sp_XuLyTraNo` xử lý toàn bộ luồng khi khách mang tiền đến: kiểm tra tài sản, cập nhật nợ, ghi Audit Log và gợi ý tài sản có thể rút về.

```sql
-- Tạo Stored Procedure xử lý khi khách hàng đến trả tiền
-- Xử lý 3 tình huống: tài sản đã thanh lý / trả hết nợ / trả góp một phần
CREATE PROCEDURE sp_XuLyTraNo
    @HopDongID      INT,            -- ID hợp đồng cần xử lý trả nợ
    @NhanVienID     INT,            -- ID nhân viên thực hiện thu tiền
    @SoTienTra      DECIMAL(18,0)   -- Số tiền khách nộp trong lần này
AS
BEGIN
    SET NOCOUNT ON;  -- Tắt thông báo "n row(s) affected"

    -- Khai báo biến nội bộ
    DECLARE @CoTaiSanDaBan  INT;            -- Đếm số tài sản đã bán thanh lý
    DECLARE @DuNoCu         DECIMAL(18,2);  -- Dư nợ TRƯỚC khi trả lần này
    DECLARE @DuNoMoi        DECIMAL(18,2);  -- Dư nợ SAU khi trả lần này
    -- Lấy ngày hiện tại một lần, dùng xuyên suốt procedure để nhất quán
    DECLARE @NgayHomNay     DATE = CAST(GETDATE() AS DATE);

    -- Kiểm tra xem có tài sản nào của hợp đồng này đã bị bán thanh lý chưa
    -- Nếu có → không thu tiền, không trả đồ (theo quy tắc nghiệp vụ)
    SELECT @CoTaiSanDaBan = COUNT(*)
    FROM HopDong_TaiSan hdt
    INNER JOIN TaiSan ts ON hdt.TaiSanID = ts.TaiSanID
    WHERE hdt.HopDongID = @HopDongID           -- Chỉ xét tài sản của hợp đồng này
      AND ts.TrangThai = N'Da ban thanh ly';   -- Đếm những tài sản đã bị bán

    -- Nếu có ít nhất 1 tài sản đã bán thanh lý → từ chối giao dịch
    IF @CoTaiSanDaBan > 0
    BEGIN
        -- Trả về thông báo từ chối; không thực hiện bất kỳ cập nhật nào
        SELECT
            N'KHONG_THU_TIEN' AS KetQua,
            N'Tai san da bi thanh ly. Khong the thu tien hay hoan tra do.' AS ThongBao;
        RETURN;  -- Thoát procedure ngay lập tức
    END

    BEGIN TRY  -- Bắt đầu khối xử lý an toàn
        BEGIN TRANSACTION;  -- Mở transaction; mọi thay đổi chưa lưu thật

        -- Tính dư nợ hiện tại của hợp đồng tính đến ngày hôm nay
        -- Gọi function đã tạo ở Event 2
        SET @DuNoCu  = dbo.fn_CalcMoneyContract(@HopDongID, @NgayHomNay);
        -- Tính dư nợ còn lại sau khi trừ số tiền khách vừa trả
        SET @DuNoMoi = @DuNoCu - @SoTienTra;
        -- Đảm bảo dư nợ không âm (trường hợp khách trả dư)
        IF @DuNoMoi < 0 SET @DuNoMoi = 0;

        -- Cộng dồn số tiền vừa trả vào tổng đã trả của hợp đồng
        UPDATE HopDong
        SET SoTienDaTra = SoTienDaTra + @SoTienTra  -- Cộng thêm, không ghi đè
        WHERE HopDongID = @HopDongID;               -- Chỉ cập nhật đúng hợp đồng

        -- Ghi Audit Log: mỗi lần trả tiền tạo một dòng mới trong LichSuGiaoDich
        -- Không bao giờ xóa hoặc sửa log → đảm bảo tính truy vết
        INSERT INTO LichSuGiaoDich (
            HopDongID, NhanVienID, NgayGiaoDich,
            SoTienTra, DuNoTruocKhiTra, DuNoSauKhiTra, LoaiGiaoDich
        ) VALUES (
            @HopDongID,     -- Hợp đồng liên quan
            @NhanVienID,    -- Nhân viên thu tiền
            GETDATE(),      -- Thời điểm giao dịch (có cả giờ:phút:giây)
            @SoTienTra,     -- Số tiền trả trong lần này
            @DuNoCu,        -- Dư nợ trước khi trả (dùng đối chiếu sau này)
            @DuNoMoi,       -- Dư nợ sau khi trả
            N'Tra no'       -- Phân loại giao dịch
        );

        -- Rẽ nhánh: kiểm tra đã trả hết nợ chưa
        IF @DuNoMoi = 0
        BEGIN
            -- TRƯỜNG HỢP ĐÃ TRẢ HẾT: đóng hợp đồng và hoàn tất tài sản

            -- Chuyển trạng thái hợp đồng sang 'Da thanh toan'
            UPDATE HopDong
            SET TrangThai = N'Da thanh toan'
            WHERE HopDongID = @HopDongID;

            -- Chuyển tất cả tài sản thuộc hợp đồng này sang 'Da tra khach'
            -- Subquery lấy danh sách TaiSanID của hợp đồng từ bảng trung gian
            UPDATE TaiSan
            SET TrangThai = N'Da tra khach'
            WHERE TaiSanID IN (
                SELECT TaiSanID FROM HopDong_TaiSan
                WHERE HopDongID = @HopDongID
            );

            COMMIT TRANSACTION;  -- Lưu thật tất cả thay đổi

            -- Trả về thông báo hoàn tất
            SELECT
                N'TRA_HET' AS KetQua,
                N'Khach da tra du. Tat ca tai san duoc hoan tra.' AS ThongBao,
                @DuNoMoi AS DuNoCon;  -- = 0
        END
        ELSE
        BEGIN
            -- TRƯỜNG HỢP CHƯA TRẢ HẾT: ghi nhận trả góp và gợi ý tài sản rút về

            -- Chuyển trạng thái hợp đồng sang 'Dang tra gop'
            UPDATE HopDong
            SET TrangThai = N'Dang tra gop'
            WHERE HopDongID = @HopDongID;

            COMMIT TRANSACTION;  -- Lưu thật các thay đổi phía trên

            -- Trả về thông báo dư nợ còn lại
            -- FORMAT(..., N'N0'): định dạng số có dấu phân cách hàng nghìn (ví dụ: 5.000.000)
            SELECT
                N'TRA_GOP' AS KetQua,
                N'Con no: ' + FORMAT(@DuNoMoi, N'N0') + N' dong' AS ThongBao,
                @DuNoMoi AS DuNoCon;

            -- Truy vấn gợi ý: liệt kê tài sản KHÁCH CÓ THỂ RÚT VỀ
            -- Điều kiện nghiệp vụ: tổng giá trị tài sản CÒN LẠI (sau khi rút tài sản này)
            -- phải >= dư nợ hiện tại → đảm bảo tài sản còn lại vẫn bảo đảm được khoản nợ
            SELECT
                ts.TaiSanID,
                ts.TenTaiSan,
                FORMAT(ts.GiaTriDinhGia, N'N0') AS GiaTriDinhGia,  -- Hiển thị có dấu phẩy
                N'Co the rut ve' AS GopY
            FROM TaiSan ts
            INNER JOIN HopDong_TaiSan hdt ON ts.TaiSanID = hdt.TaiSanID
            WHERE hdt.HopDongID = @HopDongID          -- Chỉ xét tài sản của hợp đồng này
              AND ts.TrangThai = N'Dang cam co'        -- Chỉ tài sản chưa trả, chưa thanh lý
              AND (
                  -- Subquery: tính tổng giá trị tài sản CÒN LẠI nếu rút tài sản ts này về
                  -- ts2 là tập tài sản còn lại = tất cả tài sản của hợp đồng trừ ts
                  SELECT ISNULL(SUM(ts2.GiaTriDinhGia), 0)
                  FROM TaiSan ts2
                  INNER JOIN HopDong_TaiSan hdt2
                      ON ts2.TaiSanID = hdt2.TaiSanID
                  WHERE hdt2.HopDongID = @HopDongID      -- Cùng hợp đồng
                    AND ts2.TrangThai = N'Dang cam co'   -- Tài sản đang còn cầm
                    AND ts2.TaiSanID <> ts.TaiSanID      -- Loại trừ chính tài sản đang xét
              ) >= @DuNoMoi;  -- Điều kiện: phần còn lại phải >= dư nợ
        END
    END TRY
    BEGIN CATCH  -- Bắt lỗi xảy ra trong khối TRY
        -- Nếu còn transaction đang mở thì huỷ để đảm bảo tính toàn vẹn dữ liệu
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;  -- Re-throw lỗi ra ngoài
    END CATCH
END;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/d438f37f-ae72-4752-ae25-6287c3480941" />

Tạo sp sp_XuLyTraNo

**Cách gọi:**

```sql
EXEC sp_XuLyTraNo
    @HopDongID  = 1,
    @NhanVienID = 1,
    @SoTienTra  = 5000000;
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/51b3ae4e-5ea9-411e-9747-55f502fde644" />

Kết quả sp_XuLyTraNo - Trường hợp trả góp và Danh sách tài sản gợi ý rút về

> **Phân tích kết quả:**
>
> - **Chưa trả đủ:** SSMS trả về 2 result set. Result set 1 thông báo dư nợ còn lại. Result set 2 liệt kê tài sản đủ điều kiện rút về — tức là sau khi rút tài sản đó, phần còn lại vẫn đủ bảo đảm cho dư nợ.
> - **Đã trả đủ:** Hợp đồng chuyển `'Da thanh toan'`, mọi tài sản liên kết chuyển `'Da tra khach'`, chỉ có 1 result set thông báo hoàn tất.
> - **Tài sản đã thanh lý:** Procedure trả về cảnh báo và `RETURN` ngay — không thực hiện bất kỳ cập nhật nào.

---

### Event 4: Truy vấn nợ xấu

```sql
-- Truy vấn danh sách khách hàng NỢ XẤU: đã quá Deadline1 nhưng chưa thanh toán xong
-- Kết quả gồm 6 cột theo yêu cầu đề bài
SELECT
    kh.HoTen            AS TenKhachHang,     -- Tên khách hàng lấy từ bảng KhachHang
    kh.SoDienThoai      AS SoDienThoai,      -- Số điện thoại liên hệ
    -- FORMAT(..., N'N0'): hiển thị số tiền với dấu phân cách hàng nghìn, 0 chữ số thập phân
    FORMAT(hd.SoTienVayGoc, N'N0')           AS SoTienVayGoc,
    -- DATEDIFF(DAY, ...): tính số ngày từ Deadline1 đến hôm nay → số ngày đã quá hạn
    DATEDIFF(DAY, hd.Deadline1, CAST(GETDATE() AS DATE))  AS SoNgayQuaHan,
    -- Gọi function tính dư nợ thực tế đến ngày hôm nay
    -- CAST(GETDATE() AS DATE): lấy ngày hôm nay không có phần giờ
    FORMAT(
        dbo.fn_CalcMoneyContract(hd.HopDongID, CAST(GETDATE() AS DATE)),
        N'N0'
    )                                        AS TongTienPhaiTraHienTai,
    -- Dự phóng: tính dư nợ nếu thêm 30 ngày nữa không trả
    -- DATEADD(DAY, 30, ...): cộng thêm 30 ngày vào ngày hôm nay
    FORMAT(
        dbo.fn_CalcMoneyContract(
            hd.HopDongID,
            DATEADD(DAY, 30, CAST(GETDATE() AS DATE))
        ),
        N'N0'
    )                                        AS TongTienPhaiTraSau1Thang
FROM HopDong hd
-- JOIN để lấy thông tin khách hàng (tên, SĐT) qua khóa ngoại KhachHangID
INNER JOIN KhachHang kh ON hd.KhachHangID = kh.KhachHangID
-- Lọc chỉ lấy hợp đồng đã quá Deadline1 (Deadline1 < ngày hôm nay)
WHERE hd.Deadline1 < CAST(GETDATE() AS DATE)
  -- Chỉ lấy hợp đồng chưa kết thúc (chưa thanh toán đủ, chưa thanh lý hết)
  AND hd.TrangThai IN (N'Qua han', N'Dang tra gop', N'Dang vay')
-- Sắp xếp giảm dần theo số ngày quá hạn: ca nặng nhất (quá lâu nhất) lên đầu
ORDER BY DATEDIFF(DAY, hd.Deadline1, CAST(GETDATE() AS DATE)) DESC;
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/4ef175b7-5e4a-4927-80ca-3079d3e850ad" />

Kết quả query danh sách nợ xấu

> **Phân tích kết quả:** Bảng kết quả đủ 6 cột theo yêu cầu đề bài. `FORMAT(..., 'N0')` hiển thị số tiền có dấu phân cách hàng nghìn. Kết quả sắp xếp giảm dần theo `SoNgayQuaHan` — giúp nhân viên ưu tiên xử lý ca nặng nhất trước. Cột `TongTienPhaiTraSau1Thang` đã phản ánh lãi kép tích lũy thêm 30 ngày qua `POWER(1.005, n)` bên trong function.

---

### Event 5: Quản lý thanh lý tài sản (Triggers)

> **Lưu ý quan trọng về Trigger trong SQL Server:** SQL Server không có `FOR EACH ROW`. Trigger chạy một lần cho toàn bộ câu lệnh. Phải dùng bảng ảo `inserted` (giá trị mới) và `deleted` (giá trị cũ) thay cho `NEW` / `OLD`. Logic phải viết theo kiểu set-based để xử lý đúng khi UPDATE ảnh hưởng nhiều dòng cùng lúc.

#### Trigger 1: Tự động chuyển hợp đồng sang "Qua han"

```sql
-- Tạo trigger tự động phát hiện và chuyển hợp đồng sang trạng thái 'Qua han'
-- Kích hoạt mỗi khi có lệnh UPDATE trên bảng HopDong
-- AFTER UPDATE: chạy SAU khi lệnh UPDATE đã hoàn thành
CREATE TRIGGER trg_AutoQuaHan
ON HopDong          -- Trigger gắn vào bảng HopDong
AFTER UPDATE        -- Kích hoạt sau lệnh UPDATE (không phải INSERT hay DELETE)
AS
BEGIN
    SET NOCOUNT ON;  -- Tắt thông báo số dòng bị ảnh hưởng

    -- Cập nhật trạng thái sang 'Qua han' cho các hợp đồng thỏa cả hai điều kiện:
    -- 1. Trạng thái trong bảng inserted (= giá trị MỚI sau UPDATE) là 'Dang vay'
    -- 2. Ngày hiện tại đã vượt quá Deadline1
    -- Dùng set-based: UPDATE một lúc nhiều dòng nếu có nhiều hợp đồng được UPDATE cùng lúc
    UPDATE hd
    SET hd.TrangThai = N'Qua han'
    FROM HopDong hd
    -- INNER JOIN với bảng ảo inserted để lấy danh sách các dòng vừa bị UPDATE
    INNER JOIN inserted i ON hd.HopDongID = i.HopDongID
    -- Chỉ áp dụng khi trạng thái MỚI (sau UPDATE) vẫn là 'Dang vay'
    WHERE i.TrangThai = N'Dang vay'
      -- Và ngày hiện tại đã qua Deadline1 → đủ điều kiện chuyển sang 'Qua han'
      AND CAST(GETDATE() AS DATE) > i.Deadline1;
END;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/d80c814d-3a76-4d68-8f46-3941751d651e" />
Tạo trigger Tự động chuyển hợp đồng sang "Qua han"


#### Trigger 2: Tự động chuyển tài sản sang "San sang thanh ly"

Khi hợp đồng chuyển sang `'Qua han'` **và** ngày hiện tại đã vượt `Deadline2`.

```sql
-- Tạo trigger tự động chuyển tài sản sang 'San sang thanh ly'
-- Kích hoạt mỗi khi có UPDATE trên bảng HopDong (cùng bảng với trigger 1)
CREATE TRIGGER trg_AutoSanSangThanhLy
ON HopDong          -- Gắn vào bảng HopDong
AFTER UPDATE        -- Kích hoạt sau lệnh UPDATE
AS
BEGIN
    SET NOCOUNT ON;  -- Tắt thông báo số dòng bị ảnh hưởng

    -- Cập nhật trạng thái tài sản sang 'San sang thanh ly'
    -- Áp dụng cho tài sản thuộc hợp đồng thỏa ĐỦ BA điều kiện:
    -- 1. Trạng thái hợp đồng MỚI (inserted) là 'Qua han'
    -- 2. Ngày hiện tại đã vượt quá Deadline2
    -- 3. Tài sản đang ở trạng thái 'Dang cam co' (chưa xử lý)
    UPDATE ts
    SET ts.TrangThai = N'San sang thanh ly'
    FROM TaiSan ts
    -- JOIN bảng trung gian để biết tài sản thuộc hợp đồng nào
    INNER JOIN HopDong_TaiSan hdt ON ts.TaiSanID  = hdt.TaiSanID
    -- JOIN bảng ảo inserted để lấy thông tin hợp đồng vừa được UPDATE
    INNER JOIN inserted i          ON hdt.HopDongID = i.HopDongID
    -- Chỉ khi trạng thái hợp đồng MỚI là 'Qua han'
    WHERE i.TrangThai = N'Qua han'
      -- Và ngày hiện tại đã vượt Deadline2 → tài sản đủ điều kiện thanh lý
      AND CAST(GETDATE() AS DATE) > i.Deadline2
      -- Chỉ xử lý tài sản đang ở trạng thái 'Dang cam co', tránh cập nhật 2 lần
      AND ts.TrangThai = N'Dang cam co';
END;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/92057be8-e906-4676-ab8b-70d22b5876e7" />
Tự động chuyển tài sản sang "San sang thanh ly"

#### Trigger 3: Tự động chuyển tài sản sang "Da ban thanh ly"

Khi hợp đồng được cập nhật thành `'Da thanh ly'`, toàn bộ tài sản còn lại chuyển sang đã bán. Điều kiện `deleted.TrangThai <> 'Da thanh ly'` đảm bảo trigger chỉ kích hoạt **đúng một lần** khi trạng thái vừa chuyển.

```sql
-- Tạo trigger tự động chuyển tài sản sang 'Da ban thanh ly'
-- khi hợp đồng vừa chuyển sang trạng thái 'Da thanh ly'
CREATE TRIGGER trg_AutoDaBanThanhLy
ON HopDong          -- Gắn vào bảng HopDong
AFTER UPDATE        -- Kích hoạt sau lệnh UPDATE
AS
BEGIN
    SET NOCOUNT ON;  -- Tắt thông báo số dòng bị ảnh hưởng

    -- Cập nhật tài sản sang 'Da ban thanh ly' với điều kiện chặt chẽ:
    -- 1. Trạng thái MỚI (inserted) của hợp đồng là 'Da thanh ly'
    -- 2. Trạng thái CŨ (deleted) của hợp đồng KHÔNG PHẢI 'Da thanh ly'
    --    → Đảm bảo chỉ kích hoạt khi trạng thái VỪA chuyển sang 'Da thanh ly'
    --    → Tránh trigger chạy lặp nếu UPDATE hợp đồng nhiều lần mà vẫn 'Da thanh ly'
    -- 3. Tài sản đang ở trạng thái 'Dang cam co' hoặc 'San sang thanh ly'
    UPDATE ts
    SET ts.TrangThai = N'Da ban thanh ly'
    FROM TaiSan ts
    -- JOIN bảng trung gian để biết tài sản thuộc hợp đồng nào
    INNER JOIN HopDong_TaiSan hdt ON ts.TaiSanID   = hdt.TaiSanID
    -- JOIN bảng ảo inserted: giá trị MỚI sau UPDATE
    INNER JOIN inserted i          ON hdt.HopDongID = i.HopDongID
    -- JOIN bảng ảo deleted: giá trị CŨ trước UPDATE
    -- Trong AFTER UPDATE trigger, deleted chứa snapshot trước khi UPDATE
    INNER JOIN deleted  d          ON d.HopDongID   = i.HopDongID
    -- Điều kiện 1: trạng thái mới phải là 'Da thanh ly'
    WHERE i.TrangThai = N'Da thanh ly'
      -- Điều kiện 2: trạng thái cũ không được là 'Da thanh ly' (tránh chạy lặp)
      AND d.TrangThai <> N'Da thanh ly'
      -- Điều kiện 3: chỉ cập nhật tài sản chưa xử lý (đang cầm hoặc chờ thanh lý)
      AND ts.TrangThai IN (N'Dang cam co', N'San sang thanh ly');
END;
GO
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/06a899ec-6910-4c4e-b2de-a3bf4ec7299e" />

Tự động chuyển tài sản sang "Da ban thanh ly"

**Kiểm thử Triggers:**

```sql
-- Giả lập hợp đồng ID=2 đã quá cả hai deadline để kiểm thử trigger
-- Đặt Deadline1 và Deadline2 về quá khứ xa
UPDATE HopDong
SET Deadline1 = '2026-01-01',   -- Deadline1 đã qua từ lâu
    Deadline2 = '2026-02-01'    -- Deadline2 cũng đã qua từ lâu
WHERE HopDongID = 2;

-- Kích hoạt trigger trg_AutoQuaHan bằng cách UPDATE cột TrangThai
-- Trigger sẽ phát hiện ngày hiện tại > Deadline1 và tự đổi thành 'Qua han'
-- Đồng thời trg_AutoSanSangThanhLy cũng sẽ chạy vì ngày hiện tại > Deadline2
UPDATE HopDong SET TrangThai = N'Dang vay' WHERE HopDongID = 2;

-- Kiểm tra kết quả: trạng thái hợp đồng sau khi trigger chạy
SELECT HopDongID, TrangThai, Deadline1, Deadline2
FROM HopDong WHERE HopDongID = 2;

-- Kiểm tra trạng thái tài sản liên kết với hợp đồng ID=2
SELECT ts.TaiSanID, ts.TenTaiSan, ts.TrangThai
FROM TaiSan ts
INNER JOIN HopDong_TaiSan hdt ON ts.TaiSanID = hdt.TaiSanID
WHERE hdt.HopDongID = 2;
```
<img width="1915" height="1071" alt="image" src="https://github.com/user-attachments/assets/ced5eeff-aed1-4e0e-8afb-2149f76809a7" />

Trạng thái tự động chuyển đổi

> **Phân tích kết quả:** Sau lệnh UPDATE, `trg_AutoQuaHan` kích hoạt và đổi hợp đồng ID=2 thành `'Qua han'`. Ngay trong cùng batch AFTER UPDATE, `trg_AutoSanSangThanhLy` cũng chạy vì `CAST(GETDATE() AS DATE) > Deadline2`, đổi tài sản liên kết sang `'San sang thanh ly'`. Khi UPDATE hợp đồng thành `'Da thanh ly'`, `trg_AutoDaBanThanhLy` kích hoạt và đổi tài sản sang `'Da ban thanh ly'` — điều kiện `deleted.TrangThai <> 'Da thanh ly'` ngăn trigger chạy lặp lại không cần thiết.



## 4. Các sự kiện bổ sung

### 4.1. Gia hạn hợp đồng

Khách hàng trả toàn bộ lãi tính đến hiện tại, đổi lại hai mốc Deadline được dời sang kỳ hạn mới — không còn bị tính lãi kép tiếp.

```sql
-- Tạo Stored Procedure xử lý gia hạn hợp đồng
-- Khách trả toàn bộ lãi hiện tại → deadline được dời thêm @SoNgayGiaHan ngày
-- Sau gia hạn: lãi tính lại từ đầu (lãi đơn), không còn lãi kép
CREATE PROCEDURE sp_GiaHanHopDong
    @HopDongID      INT,   -- ID hợp đồng cần gia hạn
    @NhanVienID     INT,   -- ID nhân viên xử lý giao dịch gia hạn
    @SoNgayGiaHan   INT    -- Số ngày muốn gia hạn thêm (ví dụ: 30)
AS
BEGIN
    SET NOCOUNT ON;  -- Tắt thông báo số dòng bị ảnh hưởng

    -- Khai báo biến nội bộ dùng trong procedure
    DECLARE @TrangThai      NVARCHAR(20);   -- Trạng thái hiện tại của hợp đồng
    DECLARE @LaiHienTai     DECIMAL(18,2);  -- Tổng lãi khách phải trả để gia hạn
    DECLARE @Deadline1Cu    DATE;           -- Deadline1 cũ trước khi gia hạn
    DECLARE @Deadline2Cu    DATE;           -- Deadline2 cũ trước khi gia hạn
    DECLARE @Deadline1Moi   DATE;           -- Deadline1 mới sau khi gia hạn
    DECLARE @Deadline2Moi   DATE;           -- Deadline2 mới sau khi gia hạn
    DECLARE @KhoangDD1DD2   INT;            -- Số ngày khoảng cách giữa Deadline1 và Deadline2
    -- Lấy ngày hôm nay một lần để dùng xuyên suốt
    DECLARE @NgayHomNay     DATE = CAST(GETDATE() AS DATE);

    -- Lấy thông tin hiện tại của hợp đồng cần gia hạn
    SELECT
        @TrangThai   = TrangThai,   -- Trạng thái hợp đồng
        @Deadline1Cu = Deadline1,   -- Deadline1 hiện tại
        @Deadline2Cu = Deadline2    -- Deadline2 hiện tại
    FROM HopDong
    WHERE HopDongID = @HopDongID;

    -- Kiểm tra điều kiện: chỉ gia hạn được hợp đồng đang vay hoặc đã quá hạn
    -- Không gia hạn được hợp đồng đã đóng ('Da thanh toan') hoặc đã thanh lý ('Da thanh ly')
    IF @TrangThai NOT IN (N'Dang vay', N'Qua han')
    BEGIN
        THROW 50003,
            N'Hop dong khong the gia han (da dong hoac da thanh ly).',
            1;
        RETURN;  -- Thoát procedure
    END

    BEGIN TRY  -- Bắt đầu khối xử lý an toàn
        BEGIN TRANSACTION;  -- Mở transaction

        -- Tính dư nợ hiện tại (lãi khách phải trả để gia hạn) tính đến hôm nay
        SET @LaiHienTai  = dbo.fn_CalcMoneyContract(@HopDongID, @NgayHomNay);
        -- Tính khoảng cách ngày giữa Deadline1 cũ và Deadline2 cũ (để giữ nguyên khoảng cách)
        SET @KhoangDD1DD2 = DATEDIFF(DAY, @Deadline1Cu, @Deadline2Cu);
        -- Deadline1 mới = hôm nay + số ngày gia hạn
        SET @Deadline1Moi = DATEADD(DAY, @SoNgayGiaHan, @NgayHomNay);
        -- Deadline2 mới = Deadline1 mới + khoảng cách cũ (giữ nguyên khoảng cách DD1-DD2)
        SET @Deadline2Moi = DATEADD(DAY, @KhoangDD1DD2, @Deadline1Moi);

        -- Cập nhật hợp đồng với deadline mới và đặt lại trạng thái về 'Dang vay'
        UPDATE HopDong
        SET Deadline1  = @Deadline1Moi,   -- Ghi deadline1 mới
            Deadline2  = @Deadline2Moi,   -- Ghi deadline2 mới
            TrangThai  = N'Dang vay'      -- Đặt lại trạng thái (xóa 'Qua han' nếu có)
        WHERE HopDongID = @HopDongID;

        -- Ghi Audit Log cho giao dịch gia hạn
        -- SoTienTra = @LaiHienTai: khách trả toàn bộ lãi để gia hạn
        -- DuNoSauKhiTra = 0: sau khi trả lãi, gốc vẫn còn nhưng lãi = 0 (tính lại từ đầu)
        INSERT INTO LichSuGiaoDich (
            HopDongID, NhanVienID, NgayGiaoDich,
            SoTienTra, DuNoTruocKhiTra, DuNoSauKhiTra,
            LoaiGiaoDich, GhiChu
        ) VALUES (
            @HopDongID,             -- Hợp đồng liên quan
            @NhanVienID,            -- Nhân viên xử lý
            GETDATE(),              -- Thời điểm gia hạn (có giờ:phút:giây)
            @LaiHienTai,            -- Số tiền lãi khách đã trả để gia hạn
            @LaiHienTai,            -- Dư nợ trước khi trả (bằng lãi hiện tại)
            0,                      -- Dư nợ sau khi trả = 0 (lãi đã trả xong, tính lại)
            N'Gia han',             -- Phân loại giao dịch
            -- Ghi chú thêm chi tiết: số ngày gia hạn và deadline mới
            -- CONVERT(..., 103): định dạng ngày theo kiểu dd/mm/yyyy (format 103)
            N'Gia han ' + CAST(@SoNgayGiaHan AS NVARCHAR(10))
                + N' ngay. Deadline1 moi: '
                + CONVERT(NVARCHAR(10), @Deadline1Moi, 103)
        );

        COMMIT TRANSACTION;  -- Lưu thật tất cả thay đổi

        -- Trả về kết quả thông báo gia hạn thành công
        SELECT
            @Deadline1Moi   AS Deadline1Moi,            -- Deadline1 mới
            @Deadline2Moi   AS Deadline2Moi,            -- Deadline2 mới
            @LaiHienTai     AS SoTienLaiDaThanhToan,    -- Số tiền lãi khách đã trả
            N'Gia han thanh cong' AS ThongBao;          -- Thông báo
    END TRY
    BEGIN CATCH  -- Bắt lỗi trong khối TRY
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;  -- Huỷ transaction nếu có lỗi
        THROW;  -- Re-throw lỗi ra ngoài
    END CATCH
END;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/6250ea2b-b200-4f3e-873a-7670370d8e9d" />
Tạo sp Gia hạn hợp đồng
**Cách gọi:**

```sql
EXEC sp_GiaHanHopDong
    @HopDongID    = 1,
    @NhanVienID   = 1,
    @SoNgayGiaHan = 30;
```

<img width="1917" height="1078" alt="image" src="https://github.com/user-attachments/assets/b72445f9-32da-432c-871f-ab6eaa4b0674" />

Kết quả gia hạn hợp đồng

> **Phân tích kết quả:** SSMS trả về `Deadline1Moi` và `Deadline2Moi` tính từ hôm nay + số ngày gia hạn. Khoảng cách `Deadline2 - Deadline1` giữ nguyên so với hợp đồng gốc. Lãi đơn bắt đầu tính lại từ đầu — khách không còn bị tính lãi kép nếu gia hạn đúng hạn. Log gia hạn được ghi rõ ngày, số tiền lãi đã thanh toán và Deadline mới.

---

### 4.2. Lịch sử hợp đồng (Audit Log)

Bảng `LichSuGiaoDich` là **Audit Log** ghi nhận mọi biến động tiền tệ của từng hợp đồng, đảm bảo tính minh bạch và truy vết đầy đủ.

**Query xem lịch sử một hợp đồng:**

```sql
-- Truy vấn toàn bộ lịch sử giao dịch của hợp đồng ID=1
-- Sắp xếp tăng dần theo thời gian để xem dòng tiền theo thứ tự
SELECT
    lsg.GiaoDichID,                                          -- Số thứ tự giao dịch
    -- CONVERT(..., 120): định dạng ngày giờ kiểu 'yyyy-mm-dd hh:mi:ss'
    CONVERT(NVARCHAR(20), lsg.NgayGiaoDich, 120) AS NgayGiaoDich,
    nv.HoTen                                     AS NhanVienThu,       -- Tên nhân viên thu tiền
    FORMAT(lsg.SoTienTra,       N'N0')           AS SoTienTra,         -- Tiền trả lần này (có dấu phẩy)
    FORMAT(lsg.DuNoTruocKhiTra, N'N0')           AS DuNoTruoc,         -- Dư nợ trước khi trả
    FORMAT(lsg.DuNoSauKhiTra,   N'N0')           AS DuNoSau,           -- Dư nợ sau khi trả
    lsg.LoaiGiaoDich,                                        -- Loại: 'Tra no', 'Gia han', 'Thanh ly'
    lsg.GhiChu                                               -- Ghi chú bổ sung
FROM LichSuGiaoDich lsg
-- JOIN để lấy tên nhân viên thu tiền
INNER JOIN NhanVien nv ON lsg.NhanVienID = nv.NhanVienID
-- Lọc theo hợp đồng cụ thể; thay số 1 bằng ID hợp đồng cần xem
WHERE lsg.HopDongID = 1
-- Sắp xếp từ cũ đến mới để thấy được dòng tiền theo thứ tự thời gian
ORDER BY lsg.NgayGiaoDich ASC;
```
<img width="1917" height="1078" alt="image" src="https://github.com/user-attachments/assets/f63cb472-a186-4743-a7aa-ea9ea0539e62" />
Lịch sử hợp đồng (Audit Log)

**Tại sao cần Audit Log thay vì chỉ cập nhật tổng nợ?**

| Chỉ ghi đè tổng nợ | Dùng Audit Log |
|---|---|
| Không biết khách trả bao nhiêu lần | Mỗi lần trả = 1 dòng riêng biệt |
| Không biết ai thu tiền | Ghi rõ `NhanVienID` thu tiền |
| Không thể đối chiếu khi tranh chấp | Có timestamp chính xác từng giao dịch |
| Dễ bị gian lận nội bộ | Không thể sửa xóa log mà không để lại dấu vết |
| Mất dấu vết dòng tiền | Tính lại toàn bộ lịch sử bất kỳ lúc nào |

Bảng lịch sử hiển thị từng giao dịch theo thứ tự thời gian. Tính đúng đắn xác minh được bằng bất biến: `DuNoSau` của dòng thứ n phải bằng `DuNoTruoc` của dòng thứ n+1. Cột `LoaiGiaoDich` phân biệt rõ `'Tra no'`, `'Gia han'` và `'Thanh ly'`.



*Báo cáo được viết bởi [Họ tên sinh viên] — MSSV: [MSSV] — Lớp 59KMT*
