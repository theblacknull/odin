# odin: mẫu mã độc từ repo TradeMind-MVP (phân tích 17–18/11/2026)

> **CẢNH BÁO: đây là MÃ ĐỘC THẬT.** Mọi file đã được đổi đuôi (`.txt`, `.bin`) để không chạy nhầm.
> Đừng đổi lại đuôi `.js` / `.vbs` / `.dll` / `.exe`, đừng `node`, `wscript`, `rundll32`, `regsvr32` hay giải nén
> `netsupport_client_portable.zip.bin` rồi chạy. Chỉ mở bằng trình soạn thảo văn bản, tốt nhất trên máy cô lập.
> Thư mục này chỉ dành cho phân tích, lưu chứng cứ và báo cáo abuse.

## Cấu trúc thư mục

| Thư mục | Nội dung |
|---|---|
| `stage1_loader/` | Tầng 0–1: `tailwind.config.js` (nạp file "ảnh"), `tailwind.gif` (thực chất JS bị làm rối), `.vscode/settings.json` (ẩn file) |
| `stage2_blockchain_loader/` | Tầng 2: `main.js`, `main.vbs`, `package.json` do loader ghi vào `~/.vs_cache` |
| `stage3_c2_bootstrap/` | Tầng 3: JS giải ra từ giao dịch BSC (dead-drop), trỏ tới C2 |
| `stage4_stealer_rat/` | Tầng 4: phản hồi C2 (`package.json` giả extension VS Code) và payload `sessions` (stealer + RAT, 272KB) |
| `stage5_modules/` | Module phụ: `abe_payload.dll.bin`, `win_key_agent_keylogger.js.txt`, `netsupport_client_portable.zip.bin` (+ `netsupport_text_configs/`, chỉ các file văn bản, KHÔNG giải nén binary) |
| `victim_artifacts/` | Dấu vết trên máy bị nhiễm: `loader.log`, log keylogger, bản DLL trên máy |
| `deobfuscated/` | Bản đã giải mã chuỗi (tầng 1, 2, 3, win_key), chỉ mục hàm tầng 4, bảng chuỗi |
| `analysis_tools/` | Script Python tĩnh đã dùng để giải mã (không thực thi JS) |
| `stage5_modules/netsupport_extracted/` | Các file NetSupport giải nén từ zip, mỗi file thêm đuôi `.bin` (kể cả `Service.exe.bin`, `PCICL32.DLL.bin`…) |
| `REPORT_STAGE4_DETAIL.md` | **Phân tích chi tiết tầng 4**: giao thức, lệnh `ss_*`/`exec`/SSH, thu thập trình duyệt/ví/file, ABE, persistence, IOC săn tìm |
| `INVESTIGATION_FINDINGS.md` | Phát hiện điều tra: GitHub của kẻ tấn công, dòng thời gian trên máy nạn nhân, lệnh `exec`, log mạng, phiên bản tầng 3, đính chính |
| `PROTOCOL_AND_SERVER_SIDE.md` | Phần phát payload (kiểm kê), đăng ký và `storageKey`, cách kích hoạt đợt thu thập, hợp đồng HTTP của máy chủ, phần còn thiếu phía máy chủ |
| `wallet_lookup_2026-11-18.json.txt` | Dữ liệu blockchain công khai của ví TRON/BSC của kẻ tấn công và chuỗi cấp tiền |
| `domain_lookup_2026-11-18.json.txt` | Dữ liệu RDAP/DNS thô của hai domain (registrar, ngày đăng ký, nameserver) |
| `github_lookup_2026-11-18.json.txt` | Dữ liệu API GitHub công khai (tài khoản/repo) |
| `RENAME_MAP.txt` | **Bảng đổi lại tên gốc**: tên đã lưu → tên gốc, nguồn/vị trí gốc, loại, SHA256 |
| `SHA256SUMS.txt` | Hash toàn bộ file |

## Chuỗi tấn công

1. `npm start` → Tailwind nạp `tailwind.config.js` → `require("./src/assets/images/dist/tailwind.gif")` (JS làm rối).
2. Tầng 1 tạo `%USERPROFILE%\.vs_cache\` (`main.js`, `main.vbs`, `package.json`), chạy `npm install` ẩn (axios, socket.io-client, better-sqlite3, @primno/dpapi, koffi, node-machine-id), chạy `wscript main.vbs` → `node main.js`, ghi Run key `Microsoft Realtime Agreegator`.
3. Tầng 2 đọc ví TRON (memo → hash BSC), lấy `input` giao dịch BSC, giải XOR + gzip → tầng 3, chạy bằng `new Function`.
4. Tầng 3 gọi `GET http://famoustreamservice.store:80/session/<token>` → nhận JSON giả `package.json` ms-python, khóa `sessions` chứa tầng 4.
5. Tầng 4 (stealer + RAT): trình duyệt (DPAPI, ABE qua `abe_payload.dll`, CDP), ví crypto, 1Password, lịch sử SSH và khóa `-i` → `famoustreamservice.store:80/file-upload` và `/content-upload`; quét file toàn máy (≤10MB, loại trừ theo danh sách, ưu tiên từ khóa) → `37.27.51.109:7576/upload`; RAT qua socket.io `famoustreamservice.store:80/client`; tải keylogger `win_key` và có thể cài NetSupport.
6. Persistence: Run key + `main.vbs`; chạy lại chuỗi mỗi lần đăng nhập.

## IOC (chỉ báo thỏa hiệp)

**Mạng**
- `famoustreamservice.store` (195.66.213.162, port 80/443): **máy chủ trung tâm**: phân phối payload (`/session/<token>`, `/download-main/101`, `/download-client/101`, `/download-abe-payload`, `/download-win-key`, `/download-client-portable`; `/download-app` trả 404), đăng ký nạn nhân (`/register`), nhận dữ liệu trình duyệt/ví/SSH (`/file-upload` multipart, `/content-upload` JSON), và **kênh điều khiển RAT**: websocket/socket.io `http://famoustreamservice.store:80/client` (sự kiện `exec`, `dir`, `read_file`, `env`, `imp`, `pat`, `upload`, `client:ping`, kể cả lệnh `ssh` chạy từ máy nạn nhân)
- `37.27.51.109:7576` (Hetzner, Phần Lan): **máy chủ nhận file quét hàng loạt**: HTTP `POST /upload` (thân file thô, đường dẫn base64 trong header `X-File-Path`), `HEAD /check` (chống trùng), `GET /status`, `POST /complete`. KHÔNG phải kênh điều khiển RAT.
- `195.66.213.162:443` (keylogger socket.io)
- Giai đoạn 12–14/11 tầng 3 trỏ vào IP trần `195.66.213.162:80`; từ 14/11 20:06 chuyển sang domain (xem `stage3_c2_bootstrap/VARIANTS_NOTES.txt`)
- `chainmanamgentagency.info:443` (82.158.193.218): cổng NetSupport RAT

**Blockchain**
- Ví TRON: `THKrGQ3EuFscDfCmcWFwWPtFbi5YWNQKNZ`
- BSC tx: `0x319ee72a0b202c3d1f7aac1df38cf932d3e7354e259380ff1e7b16bc61c0743f` (ví gửi `0xa7195f95318851eeb46768ac6c2743cdcb3391c5`)
- Khóa XOR: `02e4f3e7-9442-4f9c-a6f4-4fa136605d1c`, tiền tố payload `BCGZ1|`
- Session token: `6c3368b5fa77237f`, `client_id=101`

**Máy bị nhiễm**
- `%USERPROFILE%\.vs_cache\` (`main.js`, `main.vbs`, `abe_payload.dll`, `loader.log`)
- Run key `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\Microsoft Realtime Agreegator = wscript.exe "...\.vs_cache\main.vbs"`
- `%LOCALAPPDATA%\Microsoft\RealPrior\Cache\7f25-2c61-6824-bf72` (log keylogger)
- (nếu có NetSupport) `%LOCALAPPDATA%\Microsoft\Windows\SettingSync\Caches\{8F7A2C1E-9B4D-4E6A-A3C8-1D5E7F90B2C4}\Win32\ResCache\` + Run key `SettingSyncResCache`

**Nguồn phát tán (GitHub, của kẻ tấn công):**
- Tài khoản (Organization, tạo 12/11/2026 09:56 UTC): `https://github.com/finodcs26` (dạng an toàn: `hxxps://github[.]com/finodcs26`)
- Repo mồi: `https://github.com/finodcs26/TradeMind-MVP` (dạng an toàn: `hxxps://github[.]com/finodcs26/TradeMind-MVP`). Ngày 18/11 trả 404 (đã xóa hoặc chuyển riêng tư). Đừng mở/clone; hãy báo cho GitHub.
- Commit chứa mã độc: `4e5e577` (18/03/2026), tên tác giả hiển thị `Dolpheus89` (tài khoản GitHub có thật nhưng metadata git dễ giả mạo, không kết luận). Remote `origin` của bản clone trỏ vào repo trên; remote `theblacknull/TradeMind` là nơi đẩy commit của bạn (cũng đang chứa mã độc).
- Chi tiết: `INVESTIGATION_FINDINGS.md`, mục 1; dữ liệu thô `github_lookup_2026-11-18.json.txt`.

## Ghi chú phương pháp

- Mọi giải mã chuỗi làm bằng Python thuần (bảng chuỗi base64 xoay + checksum), không chạy JS của mã độc.
- Tầng 3 và 4 lấy bằng truy vấn chỉ đọc (TronGrid, RPC BSC, `GET` C2 ngày 18/11/2026) rồi lưu dạng văn bản. Tầng 4 là bản `1.6.31` tại thời điểm đó, có thể khác bản đã chạy trên máy nạn nhân ngày 17/11.
- Các file `*_deob*.txt` là bản đọc được, đã thay chuỗi và chia dòng; có thể có sai lệch nhỏ về cú pháp, chỉ dùng để đọc.
