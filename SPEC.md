# Sổ Thu Chi Ba Mảng — Đặc tả để phát triển tiếp

> Tài liệu bàn giao cho AI coding agent (Antigravity) viết tiếp. File mã nguồn hiện có: `so-thu-chi.html` (một file duy nhất, HTML + CSS + JavaScript thuần, không framework, không build step).

## 1. Mục tiêu nghiệp vụ

Công cụ quản lý thu chi cho doanh nghiệp có **3 mảng hoạt động**:

| Mã | Mảng | Vai trò trong báo cáo |
|---|---|---|
| `sm` | Sơn mài | Trung tâm lợi nhuận — có thu, có chi |
| `fnb` | F&B (cà phê, cơm trưa) | Trung tâm lợi nhuận — có thu, có chi |
| `vp` | Văn phòng | Trung tâm chi phí — **không phân bổ** cho 2 mảng kia |

Người dùng: nhiều người dùng chung một bộ số liệu (kế toán nhập, quản lý xem báo cáo). Đơn vị tiền: **VND**, hiển thị dấu chấm phân cách hàng nghìn (`vi-VN`). Giao diện tiếng Việt.

Nhu cầu chính:
1. **Sơn mài**: ghi từng khoản thu tiền và theo dõi tình trạng xuất hóa đơn (đã xuất / chưa xuất, số HĐ, ngày HĐ).
2. **F&B**: đối soát tiền thu hằng ngày: `Doanh thu KiotViet = Tiền MBB nhận trong ngày + Tiền mặt nộp`. Nhập tay tổng theo ngày.
3. **Chi phí**: chi tiền mặt nhập tay từng dòng; chi chuyển khoản lấy từ sao kê ngân hàng MBB bằng cách **dán nhiều dòng từ Excel** rồi gán mảng + khoản mục.
4. **Bảng lương**: chi tiết từng nhân viên, mỗi nhân viên thuộc một mảng.
5. **Báo cáo hiệu quả** theo tháng: thu, chi, lãi góp, biên theo mảng.

## 2. Cấu trúc giao diện (6 tab)

| Tab | Chức năng |
|---|---|
| Báo cáo hiệu quả | KPI tháng, thẻ 3 mảng, bảng thu chi theo mảng × khoản mục, biểu đồ cột chênh lệch 6 tháng, phần "Cách tính" |
| Sơn mài · Thu & hóa đơn | Form ghi khoản thu; bảng kê; lọc "chỉ khoản chưa xuất HĐ (mọi kỳ)"; nút "Cập nhật HĐ" |
| F&B · Đối soát thu | Form nhập ngày (chọn lại ngày cũ sẽ nạp số để sửa); bảng đối soát có trạng thái Khớp / Thiếu tiền / Thừa tiền; danh sách ngày chưa nhập |
| Chi phí | Form ghi chi; ô dán sao kê MBB → bảng xem trước (chọn dòng, gán mảng, khoản mục) → nhập hàng loạt; bảng kê lọc theo mảng |
| Bảng lương | Danh sách nhân viên (thêm, đổi mảng, cho nghỉ); lưới lương theo tháng; "Chép từ tháng trước"; "Lưu bảng lương" |
| Danh mục | Thêm / xóa / sắp xếp khoản mục chi phí |

Bộ chọn **Kỳ báo cáo** (`<input type="month">`) ở đầu trang áp dụng cho tất cả tab.

Khi kho dữ liệu trống có nút **"Xem thử với số liệu mẫu"** — sinh dữ liệu mẫu trong bộ nhớ (hàm `demoData()`), không ghi vào kho, có banner cảnh báo.

## 3. Mô hình dữ liệu

Kho tài liệu dạng NoSQL (document store). 4 collection + 1 document cấu hình.

### 3.1 `txn` — mọi khoản thu/chi (trừ doanh thu F&B và lương)
```json
{
  "id": "t_lx3k9a2bc",
  "kind": "thu" | "chi",
  "seg": "sm" | "fnb" | "vp",
  "date": "2026-10-09",
  "src": "tm" | "ck",            // tiền mặt / chuyển khoản
  "cat": "Điện nước",            // khoản mục, rỗng với khoản thu
  "content": "Cty ABC – tranh sơn mài đợt 1",
  "amount": 15000000,            // số dương, VND
  "inv": "0000123",              // số hóa đơn (chỉ dùng cho thu Sơn mài), rỗng = chưa xuất
  "invDate": "2026-10-10",
  "updatedAt": 1791518715000
}
```
Hiện tại form chỉ tạo `kind:"thu"` cho `seg:"sm"`. Trạng thái hóa đơn suy ra từ `inv`: có giá trị = Đã xuất.

### 3.2 `fnbday` — đối soát F&B, **id = ngày** (`YYYY-MM-DD`, mỗi ngày 1 document)
```json
{ "date": "2026-10-09", "kv": 5200000, "mbb": 3000000, "cash": 2200000, "note": "", "updatedAt": 0 }
```

### 3.3 `employees`
```json
{ "id": "e_...", "name": "Nguyễn Văn An", "seg": "sm", "active": true }
```

### 3.4 `payroll` — **id = `{YYYY-MM}_{empId}`**
```json
{ "month": "2026-10", "empId": "e_...", "name": "...", "seg": "sm",
  "gross": 11000000,   // tổng thu nhập (lương + phụ cấp)
  "ins": 2365000,      // BHXH, BHYT, BHTN phần doanh nghiệp đóng
  "note": "" }
```
`seg` được chụp tại thời điểm lưu → đổi mảng nhân viên không làm thay đổi bảng lương các tháng đã lưu.

### 3.5 `settings/main`
```json
{ "cats": ["NVL / Hàng hóa","Thuê mặt bằng","Điện nước","Vận chuyển","Marketing","Công cụ dụng cụ","Thuế, phí","Phí ngân hàng","Khác"] }
```
Lương **không** nằm trong danh mục vì lấy từ `payroll`.

## 4. Logic tính toán (hàm `compute(month)`)

Với mỗi mảng `s` trong tháng `m`:

- `thu[sm]` = Σ `txn.amount` có `kind="thu"`, `seg="sm"` — **cơ sở tiền mặt, đã gồm VAT** nếu khách trả gồm VAT.
- `thu[fnb]` = Σ `fnbday.kv` (doanh thu KiotViet) + Σ `txn` thu của fnb (nếu có).
  Tiền MBB và tiền mặt nộp **chỉ dùng đối soát**, không cộng vào thu (tránh tính 2 lần).
- `thu[vp]` = Σ `txn` thu của vp (hiện chưa có form).
- `chi[s]` = Σ `txn` chi theo `cat` + `luong[s]`
- `luong[s]` = Σ (`gross` + `ins`) của `payroll` tháng `m` có `seg = s`
- `net[s]` = `thu[s]` − `chi[s]` (lãi góp)
- `biên[s]` = `net[s]` ÷ `thu[s]`
- **Toàn công ty** = net[sm] + net[fnb] + net[vp] (net[vp] = −chi văn phòng)

Đối soát F&B mỗi ngày: `chênh lệch = kv − (mbb + cash)`. Dương → **Thiếu tiền** (đỏ), âm → **Thừa tiền** (vàng), 0 → **Khớp** (xanh).

### Tránh tính trùng lương
Khi dán sao kê, dòng có nội dung chứa "lương / luong" được **bỏ chọn sẵn**, vì lương đã tính từ Bảng lương.

### Đọc dữ liệu dán từ sao kê (hàm trong handler `parseBtn`)
- Tách dòng theo `\n`, tách cột theo `\t`.
- Ngày: cột đầu tiên khớp `dd/mm/yyyy`, `dd-mm-yyyy` hoặc `yyyy-mm-dd`.
- Số tiền: cột số **cuối cùng** không phải ngày; bỏ dấu, lấy trị tuyệt đối; bỏ phần thập phân `.00` / `,00`.
- Nội dung: cột văn bản dài nhất còn lại.
- Mặc định `seg="vp"`, `cat="Khác"`, `src="ck"`.

## 5. Lớp lưu trữ — CẦN THAY KHI CHẠY NGOÀI claude.ai

Bản hiện tại chạy trên Claude Artifact và dùng kho dùng chung của nền tảng qua `window.claude.use("db")` (API kiểu Firestore: `collection().doc().set()`, `.delete()`, `.onSnapshot()`). **Ngoài claude.ai, đối tượng này không tồn tại** → trang tự chuyển sang chế độ bộ nhớ tạm (mất dữ liệu khi đóng trang, chip trạng thái báo "Chưa kết nối kho chung").

Toàn bộ truy cập kho gom vào 4 điểm, chỉ cần viết lại các điểm này:

| Hàm / đoạn | Việc làm |
|---|---|
| `put(coll, id, data)` | Ghi đè 1 document |
| `del(coll, id)` | Xóa 1 document |
| `saveCats()` | Ghi `settings/main` |
| Khối khởi động cuối file (`bind(...)`) | Đăng ký nghe thay đổi realtime từng collection, cập nhật `S.txn`, `S.fnb`, `S.emp`, `S.pay`, `S.cats` rồi gọi `renderAll()` |

Đề xuất thay bằng (chọn 1):
- **Firebase Firestore** — API gần như giống hệt, đổi ít nhất; dùng Firebase Auth để phân quyền.
- **Supabase** (Postgres + Realtime) — nếu muốn SQL, xuất báo cáo dễ hơn. Tạo 4 bảng tương ứng mục 3.
- **Google Sheets + Apps Script** — nếu muốn kế toán xem trực tiếp trên Sheet.

## 6. Việc nên làm tiếp (backlog gợi ý)

1. Thay lớp lưu trữ (mục 5) + đăng nhập, phân quyền: Kế toán (nhập), Quản lý (chỉ xem báo cáo).
2. Nhập file Excel xuất từ KiotViet và sao kê MBB (.xlsx) thay vì nhập tay / dán — dùng SheetJS.
3. Xuất báo cáo tháng ra Excel/PDF.
4. Tách VAT cho khoản thu Sơn mài (doanh thu chưa thuế + thuế GTGT đầu ra) để khớp sổ kế toán.
5. Theo dõi công nợ phải thu Sơn mài theo khách hàng / hợp đồng (giá trị HĐ, đã thu, còn phải thu).
6. Form "Thu khác" cho Văn phòng.
7. Nhân viên làm nhiều mảng: chia % lương theo mảng.
8. Khóa sổ theo tháng (không cho sửa kỳ đã khóa) và nhật ký thay đổi (ai sửa, lúc nào).
9. Kiểm tra trùng khi dán sao kê (cùng ngày + số tiền + nội dung).

## 7. Quy ước kỹ thuật cần giữ

- Mọi màu là CSS token trong `:root`, có bộ dark mode (`prefers-color-scheme` và `[data-theme]`).
- Font: Lora (tiêu đề), Be Vietnam Pro (nội dung) — đều hỗ trợ tiếng Việt.
- Ô tiền có class `money`: tự định dạng dấu chấm nghìn khi gõ; đọc lại bằng `num()`.
- Ngày lưu dạng ISO `YYYY-MM-DD`, hiển thị `dd/mm/yyyy` qua `dmy()`.
- Mọi chuỗi người dùng nhập đi qua `esc()` trước khi đưa vào `innerHTML`.
- Xóa cần bấm 2 lần (không dùng `confirm()`).
- Responsive tới ~400px, bảng rộng cuộn ngang trong khung riêng.
