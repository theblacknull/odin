# Phát hiện điều tra trên máy nạn nhân và hạ tầng (cập nhật 18/11/2026)

Mọi nội dung dưới đây chỉ dựa trên quan sát đọc-only. Chỗ nào là suy luận được ghi rõ.

## 1. Nguồn mã độc: tài khoản GitHub của kẻ tấn công

| Mục | Giá trị |
|---|---|
| Tài khoản | `hxxps://github[.]com/finodcs26` (dạng thật: `https://github.com/finodcs26`) |
| Repo phát tán | `hxxps://github[.]com/finodcs26/TradeMind-MVP` (dạng thật: `https://github.com/finodcs26/TradeMind-MVP`), là `origin` của bản clone trên máy nạn nhân |
| Loại | **Organization**, id `355745993`, tạo **12/11/2026 09:56 UTC** (cùng ngày giao dịch TRON đầu tiên của dead-drop, 12/11 20:28 UTC) |
| Tình trạng ngày 18/11 | 0 repo công khai; `TradeMind-MVP` trả **404** (đã xóa hoặc chuyển riêng tư). Không ai xem được qua GitHub API công khai |
| Kết luận | Tổ chức mới lập ngay trước chiến dịch, dùng làm "mồi" (repo giả dự án giao dịch crypto) |

**Lưu ý về tác giả commit:** commit đầu tiên `4e5e577` (18/03/2026) mang tên tác giả `Dolpheus89`. Đó là một tài khoản GitHub có thật (tạo 2023-10-02, 38 repo học tập, không có repo TradeMind công khai). Metadata git (tên, email, ngày) rất dễ bị giả, nên **không kết luận** người này liên quan; có thể kẻ tấn công sao chép hoặc mạo danh. Không dùng thông tin này để quy trách nhiệm.

Dữ liệu API thô lưu ở `github_lookup_2026-11-18.json.txt`.

Khuyến nghị: báo cáo cả hai URL trên cho GitHub (mục "Report abuse/malware"), kèm các commit và hash trong `SHA256SUMS.txt`.

## 2. Dòng thời gian trên máy nạn nhân (17/11/2026, giờ địa phương)

Nguồn chính: nhật ký Application của Windows, sự kiện `VBScriptDeprecationAlert` (ID 4096). Mỗi lần `wscript` chạy VBScript, Windows ghi cây tiến trình và tên script.

| Giờ | Sự kiện | Nguồn |
|---|---|---|
| 09:19:05–17 | Chạy `npm run start:frontend` → `react-scripts start.js` | Tiến trình / cây tiến trình |
| 09:19:20 | `wmic path Win32_NetworkAdapterConfiguration …` do dev server gọi (không liên quan mã độc) | Application log |
| 09:19:22–29 | Tạo `.vs_cache`, ghi `main.js`, `package.json`, `package-lock.json`, `main.vbs`, cài node_modules | Dấu thời gian file |
| 09:19:29 | `wscript //B main.vbs` chạy; **cha của nó là `node react-scripts start.js`** (chứng minh trực tiếp tầng 1 đã khởi chạy mã độc) | Application log |
| 09:19:47 | Dữ liệu extension MetaMask (profile Chrome `Default`) bị đọc | LastAccess |
| 09:24:37 | Tạo thư mục `RealPrior\Cache` (keylogger) | Dấu thời gian |
| 09:27:04–09:32 | Quét ~8.900 file ≤10MB trong Desktop/Downloads (khớp cửa sổ 5 phút của "backup") | LastAccess |
| 09:53:02 | `abe_payload.dll` được thả xuống | Dấu thời gian |
| 09:53:50 | `Login Data` của 4 profile Chrome bị đọc **cùng một giây** | LastAccess |
| 10:25:58 | `Cookies` của 3 profile Chrome bị đọc cùng một giây | LastAccess |
| 10:26:03 | Ghi lần cuối vào log keylogger | Dấu thời gian |
| **10:39:02** | `main.vbs` chạy lại; cùng giây có `_vs_upd_39620_1794901141347.vbs` (tiến trình tầng 4 PID 39620 tự khởi động lại) | Application log |
| 10:44:00 | `loader.log`: "tiny loader: socket hang up" (tải tầng 3 thất bại) | `loader.log` |
| sau 10:44 | Không có hoạt động mã độc; máy chưa khởi động lại kể từ 14/11, nên Run key chưa chạy | Không có dấu vết |

Diễn giải:
- Tầng 4 chạy khoảng **1 giờ 20 phút** (09:19:29 → 10:39:02).
- Lần khởi động lại 10:39:02 do `ss_force_upd` (lệnh từ "admin") **hoặc** do cơ chế origin refresh khi socket rớt. Chưa phân biệt được; chi tiết `loader.log` 10:44 (lỗi mạng) nghiêng về máy chủ không phản hồi lúc đó.
- Dòng `~/.ssh` bị đọc lúc 13:34:58 nằm **sau** khi mã độc đã ngừng; nhiều khả năng do người dùng/công cụ của họ (suy luận).
- Các sự kiện `scrcons.exe` (VBScript) lặp mỗi vài giờ từ trước 14/11 là **Dell Command Power Manager** (đăng ký WMI hợp lệ), không phải mã độc.

**Bổ sung (tự động hóa):** đợt thu thập trình duyệt/ví/SSH chỉ chạy khi máy chủ phát sự kiện socket `upload`; trên máy nạn nhân, việc đọc dữ liệu MetaMask xảy ra lúc 09:19:47, chỉ ~18 giây sau khi mã chạy, và việc đọc Cookies lúc 10:25:58 cách đợt đọc `Login Data` 32 phút khớp với thời gian chờ 30 phút trong code. Điều này ủng hộ mạnh việc máy chủ kích hoạt **tự động** thay vì do con người ra lệnh. Chi tiết: `PROTOCOL_AND_SERVER_SIDE.md`. (Ý "có thể là người vận hành vì cách nhau hơn 30 phút" trong trao đổi trước được rút lại.)

## 3. Đã có lệnh `exec` từ xa chưa

Không tìm thấy bằng chứng của lệnh `exec` tùy ý; cũng **không thể chứng minh là không có**, vì Security 4688/Sysmon không bật.

| Lệnh của họ | Dấu vết trên máy | Kết luận |
|---|---|---|
| `ss_take_pc` (cài NetSupport) | Không có `SettingSync`, không có Run key `SettingSyncResCache`, không có `vs_takepc_*` | Không xảy ra |
| `ss_del` (tự gỡ) | Run key và `.vs_cache` còn nguyên | Không xảy ra |
| `ss_force_upd` | Có thể là nguồn của lần khởi động lại 10:39:02 | Có thể (hoặc origin refresh) |
| `upload` (thu thập) | Đọc 4 profile Chrome 09:53:50, Cookies 10:25:58 | Rất có thể |
| `exec`/SSH | Không có `ssh_askpass_*`, `ssh_pw_*` còn sót; Chrome chạy liên tục từ 16/11 03:01 (không bị `taskkill`); không có tài khoản/dịch vụ/tác vụ lạ; `hosts` không bị sửa | Không thấy, nhưng lệnh đơn lẻ không để lại dấu vết |

Cần tự xác nhận: dịch vụ `UltraViewService` (UltraViewer, phần mềm điều khiển từ xa hợp lệ) đang chạy trên máy; nếu không phải do bạn cài thì cần xem lại.

## 4. Log mạng: xem được gì

- Mọi đường gửi dữ liệu của mã độc là **HTTP/WebSocket không mã hóa**. Một bản bắt gói (pcap) sẽ chứa **nguyên nội dung** file đã gửi (thân thô tới `37.27.51.109:7576/upload`, multipart tới `famoustreamservice.store:80/file-upload`) và các lệnh/kết quả trong khung WebSocket.
- Log luồng (router/firewall/NetFlow) chỉ cho IP, cổng, thời gian, số byte. DNS log chỉ cho tên miền tra cứu. SRUM (Windows, cần admin) cho số byte mỗi ứng dụng theo giờ.
- Trên máy này: **không có pcap**; firewall không bật ghi log (cả 3 profile `Disable`); không cài Wireshark/Npcap; `SRUDB.dat` chưa đọc vì cần quyền admin.
- Máy dùng **Happ Tunnel** (TUN, `xray`) cùng Windscribe/PlanetVPN, nên router/ISP chỉ thấy luồng mã hóa tới máy chủ VPN; đích thật (và nội dung HTTP) chỉ hiện ở máy chủ VPN/đích. Việc hợp lý còn lại: đọc SRUM bằng quyền admin để ước lượng lượng dữ liệu `node.exe` đã gửi trong 09:00–11:00 ngày 17/11, và yêu cầu nhà cung cấp hosting của hai IP lưu log.

## 5. Phiên bản tầng 3 (dead-drop)

Ví TRON `THKrGQ3EuFscDfCmcWFwWPtFbi5YWNQKNZ` có đúng 4 giao dịch gửi đi:

| # | Thời gian (UTC) | Kích thước | SHA256 (đầu) | C2 trong payload |
|---|---|---|---|---|
| 0 | 12/11 20:28 | 5070 | 6c466fe79276 | `http://195.66.213.162:80` |
| 1 | 12/11 20:33 | 5070 | 6c466fe79276 | `http://195.66.213.162:80` |
| 2 | 14/11 13:35 | 5070 | 6c466fe79276 | `http://195.66.213.162:80` |
| 3 | 14/11 20:06 | 4734 | 312432a6a4d4 | `http://famoustreamservice.store:80` |

Bản #3 (mới nhất, trước ngày nhiễm 17/11) là bản máy bạn đã chạy. Chi tiết: `stage3_c2_bootstrap/VARIANTS_NOTES.txt`.

## 6. Hạ tầng (RDAP, ngày 18/11/2026)

| Địa chỉ | Chủ dải / ghi chú |
|---|---|
| `195.66.213.162` | Dải `VSYS-AMS` (Hà Lan), nhà cung cấp hosting VSYS |
| `82.158.193.218` | Dải `VPS_USA`, cùng nhà cung cấp VSYS (cổng NetSupport) |
| `37.27.51.109` | Hetzner Online GmbH (dải Phần Lan) |
| `famoustreamservice.store` | Registrar **Hostinger operations, UAB** (Vilnius, abuse@hostinger.com). Đăng ký **12/11/2026 20:09:25 UTC** (19 phút trước giao dịch TRON đầu tiên 20:28 UTC), hết hạn 12/11/2027, thay đổi lần cuối 17/11 20:10 UTC. Nameserver `aurora/nebula.dns-parking.com`; A → 195.66.213.162. Người đăng ký không công khai (bị ẩn) |
| `chainmanamgentagency.info` | Registrar Hostinger, đăng ký 05/11/2026, đổi 10/11, nameserver `orbit/horizon.dns-parking.com`; A → 82.158.193.218. Người đăng ký không công khai |

Dữ liệu thô: `domain_lookup_2026-11-18.json.txt`. Tra cứu nhật ký chứng chỉ (crt.sh) lỗi 502 ngày 18/11, chưa có dữ liệu.

Tên chủ dải là của nhà cung cấp hosting, **không chứng minh** họ là kẻ tấn công.

## 7. Đính chính các sai sót trước đó

- `37.27.51.109:7576` **không phải** kênh điều khiển RAT mà là máy chủ nhận file quét hàng loạt. Kênh RAT là `famoustreamservice.store:80/client`.
- Thư mục `.ssh` không bị quét hàng loạt (nằm trong danh sách loại trừ); khóa SSH chỉ bị lấy qua `ssh_history`/tham số `-i` hoặc lệnh trực tiếp của operator.
- Đợt truy cập 09:27–09:32 thật sự khớp cửa sổ làm việc 5 phút của crawler (xem `REPORT_STAGE4_DETAIL.md`).

## 8. Ví crypto của kẻ tấn công (dữ liệu công khai trên blockchain, tra ngày 18/11/2026)

| Ví | Mạng | Thông tin |
|---|---|---|
| `THKrGQ3EuFscDfCmcWFwWPtFbi5YWNQKNZ` | TRON | Ví dead-drop. Tạo **12/11 20:07:12 UTC**, nhận 7,907 TRX lúc 20:07:15 UTC, chỉ có 4 giao dịch gửi đi (12/11 20:28 → 14/11 20:06), số dư hiện ~3,58 TRX. Không có chuyển khoản TRC20 |
| `0xa7195f95318851eeb46768ac6c2743cdcb3391c5` | BSC | Ví gửi các giao dịch chứa payload tới địa chỉ `0x…dead`. Số dư ~0,00145 BNB, đã gửi 9 giao dịch. Chưa lấy được lịch sử/nguồn tiền (explorer công khai cần khóa API) |

**Chuỗi cấp tiền ngược (TRON):**
`TCeiLNTWE783EUhErT7s423GLZbQwFpWR6` (tạo 31/10/2025, 67 giao dịch gửi đi tới 21 địa chỉ, có khoản lớn tới ~3.160 TRX; nhiều khả năng ví của người vận hành) → chuyển 8 TRX (03/09/2026) và 10 TRX (15/09/2026) →
`TBMcSRezuxdoNEC8gD3uS4GQVGvybYU4UU` (tạo 03/09/2026, chỉ 9 giao dịch gửi đi: 8 khoản 1 "sun" vô nghĩa tới `T9yD14Nj9j7xAB4dbGeiX9h8unkKHxuWwb` và 7,907 TRX tới ví dead-drop lúc 12/11 20:07) → ví dead-drop.
Các khoản 1–8 "sun" và token lạ nhận vào `TBMc…` là rác/giả mạo địa chỉ (address poisoning), không phải của họ. Địa chỉ `TEAZf3Jy…` gửi 4 "sun" cho ví dead-drop cũng chỉ là rác (gửi tới 118 địa chỉ).
**Chưa lần được tới sàn hay dịch vụ nào** (không có nhãn công khai; TronScan yêu cầu khóa API). Việc truy tiếp (nguồn của `TCeiLNTW…`) nên giao cho công cụ chuyên dụng hoặc cơ quan chức năng; một địa chỉ không chứng minh danh tính.
Dữ liệu thô: `wallet_lookup_2026-11-18.json.txt`.
