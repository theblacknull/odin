# Phân tích chi tiết tầng 4: stealer + RAT (phiên bản nội bộ 1.6.31)

Nguồn: `stage4_stealer_rat/stage4_sessions_payload.js.txt` (272.299 byte, một dòng, JS đã rút gọn, tên biến bị đổi nhưng không bị mã hóa chuỗi).
Phân tích tĩnh bằng Python (không chạy). Hàm được lập chỉ mục: 455 đơn vị (`deobfuscated/index4.txt.txt`).
Mức độ đọc: các module chính (C2, lệnh từ xa, upload, ABE, SSH, persistence, tự cập nhật) đã đọc trực tiếp từng đoạn; các phần phụ (nhánh macOS/Linux, nội bộ lớp phiên SQL/Docker) chỉ đọc ở mức hành vi. Xem mục 11.

## 1. Tổng quan

- Ngôn ngữ/chạy: Node.js, chạy trong bộ nhớ qua `new Function` (không ghi file). Đa nền tảng: Windows (chính), macOS, Linux.
- Mã khách: `client_id = 101`, phiên bản `y = "1.6.31"`. Thư mục làm việc `~/.vs_cache` (biến `w`).
- Mục tiêu kép: (1) **stealer**: lấy cookie/mật khẩu/ví/khóa SSH/file; (2) **RAT**: chạy lệnh, duyệt file, điều khiển keylogger và cài NetSupport.
- Tên giả để lẫn vào hệ thống: Windows `Microsoft Realtime Agreegator`, macOS `AppleAccountSync`, Linux `unixkernalloader`. Trong log/giao diện của họ, cơ chế upload hàng loạt được gọi là "FileZilla" (nhãn giả, không phải FileZilla thật).
- Hai chế độ chạy cùng một mã: tiến trình chính (có socket điều khiển) và tiến trình con "backup" (`VS_BACKUP_CHILD=1`, socket bị vô hiệu, chỉ làm việc quét + upload).
- `process.exit` bị ghi đè: chương trình chỉ dừng khi nhận lệnh `ss_del` hoặc `ss_force_upd`; mọi lỗi bị nuốt (`uncaughtException`, `unhandledRejection` chỉ log).

## 2. Hạ tầng và giao thức trong code

| Hằng số | Giá trị | Vai trò |
|---|---|---|
| `b` | `http://famoustreamservice.store:80` | `/register`, `/file-upload`, `/content-upload`, `/download-*` |
| Socket `Me` | `http://famoustreamservice.store:80/client` | Kênh điều khiển (websocket + polling, tự kết nối lại vô hạn, trễ 2–60 giây) |
| `K` (= `pu.serverUrl`) | `http://37.27.51.109:7576` | Máy chủ nhận file quét hàng loạt: `/upload`, `/check`, `/status`, `/complete` |
| `q` | `http://famoustreamservice.store:443` | `SERVER_URL` truyền cho keylogger; `/download-app`, `/download-win-key` |
| Keylogger | `http://195.66.213.162:443` | socket.io riêng của `win_key_agent` |

Tất cả dùng **HTTP không mã hóa** (kể cả cổng 443). Tiêu đề nhận dạng nạn nhân: `X-Machine-ID`, `X-Username`, `X-Client-ID`; file có `X-File-Path` (đường dẫn gốc mã hóa base64), `X-File-Size`, `X-Upload-ID`, `X-Is-Priority`.

**Tự đổi địa chỉ (origin refresh):** khi socket rớt quá một khoảng thời gian, hàm `ze()` chạy lại `main.js` (tầng 2 → 3) để "lấy IP máy chủ đã công bố" rồi thoát; nhờ vậy họ đổi hạ tầng mà không cần sửa mã khách.

## 3. Đăng ký nạn nhân (`POST /register`)

Gửi: `username`, `customId` (client_id), `instanceUsername`, `osType`, `platform`, `osRelease`, `osVersionLabel`, `osVersionText`, `clientVersion` và `ipInfo`. `ipInfo` gồm kết quả phân loại máy:
- **Cloud/VM**: hỏi các endpoint metadata AWS/Azure/GCP/Alibaba/Oracle/OpenStack/Hetzner/DigitalOcean (`169.254.169.254`, `metadata.google.internal`, `100.100.100.200`), cùng `systemd-detect-virt`, `sysctl hw.model`, DMI (`sys_vendor`, `product_name`, `bios_vendor`…), WMI `Win32_ComputerSystem/BIOS`.
- **Tham gia domain (AD)**: `Win32_ComputerSystem.PartOfDomain`, `wmic computersystem`, `wbinfo`, `sssctl` (Linux) và công cụ tương ứng trên macOS.
- **IP/địa lý**: `ip-api.com`, `freeipapi.com`, `ipwho.is`, loại hosting/mobile/proxy.

Server trả về `storageKey` và nhãn; mọi dữ liệu sau đó gắn với khóa này. (Suy luận: kết quả phân loại giúp "admin" quyết định máy nào đáng khai thác; mã không tự dừng theo kết quả.)

## 4. Lệnh từ "admin" (socket.io `/client`)

Sự kiện: `exec`, `dir`, `read_file`, `env`, `imp`, `pat`, `upload`, `client:ping/pong`.

**`exec`** nhận một chuỗi lệnh (hoặc đối tượng `{instruction, cwd, sshPassword, sshKeyPath, sshWrap, sqlSession, dockerSession, adminId, execId}`):
- Nếu bắt đầu bằng `ss_...`: lệnh quản trị nội bộ (bảng dưới).
- `cd`: đổi thư mục làm việc ảo.
- **Mọi chuỗi khác chạy như lệnh shell** (`Hs`), kết quả trả về qua `exec_result`; nếu là `ssh/scp/sftp` thì thêm sự kiện `ssh_status` (đích, đã kết nối hay chưa).
- `sqlSession`: phiên tương tác `mysql`/`psql` (có thể bọc qua SSH). `dockerSession`: phiên docker tương tác.
- **SSH từ máy nạn nhân**: lệnh chạy `ssh` cục bộ với `-o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL -o GlobalKnownHostsFile=NUL -o ConnectTimeout=15 ...`; mật khẩu được đưa qua `SSH_ASKPASS` (+ `SSH_ASKPASS_REQUIRE=force`, file tạm `ssh_askpass_<pid>_*`, `ssh_pw_<pid>_*`, tự xóa sau). Đích (`target`, `port`, `remotePath`), khóa (`-i`) và mật khẩu do operator cấp lúc chạy, **không có trong code**. Việc đặt `UserKnownHostsFile=NUL` khiến `known_hosts` của nạn nhân không ghi lại.

**Lệnh `ss_*`:**

| Lệnh | Tác dụng |
|---|---|
| `ss_del` | Tự gỡ: gọi hàm dọn dẹp `kd()` (nội dung chưa đọc chi tiết), ngắt kết nối rồi thoát |
| `ss_force_upd` | Buộc cập nhật: tải `main.js`/client mới rồi khởi động lại (qua `_vs_upd_*.vbs`) |
| `ss_upd <thư mục>` / `ss_upf <file>` | Upload cả thư mục / một file ngay (`isManualUpload`) |
| `ss_pause`, `ss_resume`, `ss_stop`, `ss_cancel` | Điều khiển các job upload |
| `ss_backup_pause/resume/status`, `ss_harvest_status` | Điều khiển và theo dõi đợt quét "backup" và thu thập |
| `ss_kill_browsers` | `taskkill`/`pkill` các trình duyệt (để mở khóa file SQLite) |
| `ss_key_start/stop/status` | Bật/tắt keylogger `win_key_agent` |
| `ss_key_log_upload` | Gửi log keylogger lên server dưới tên `event_log` |
| `ss_take_pc` | Cài NetSupport Manager (xem mục 7) |
| `ss_mac_autostart_install/remove/status` | Autostart trên macOS |

**`dir`**: `cd`, `list`, `search` (duyệt/tìm file, có "bigFilesOnly"). **`read_file`**: đọc file văn bản gửi về `file_content`. **`env`/`imp`/`pat`**: quét file theo mẫu (.env, "important", theo pattern admin gửi). Các lệnh duyệt có cờ `allowVsCache`: mặc định **chặn truy cập thư mục `.vs_cache`** để operator không vô tình lộ/xóa mã độc (trừ khi cờ này bật).

**`upload`**: kích hoạt toàn bộ đợt thu thập ở mục 5 (ghi lại `storageKey`).

## 5. Thu thập dữ liệu

### 5.1 Trình duyệt (Windows)
Danh sách trình duyệt: Chrome, Edge, Brave, Opera, Yandex, Vivaldi, Tor, Firefox/LibreWolf/Waterfox, Chromium (qua nhận diện thư mục profile `dn()`).
Với mỗi profile, **tải thẳng file thô lên server để giải mã ngoại tuyến**, kèm khóa:
- `Local State` (tải với tên `logkc-edb`), `Login Data` (`uld_<n>`), `Login Data For Account` (`ulda_<n>`), `Cookies` (`pck_<n>`), `History`, `Web Data` (thẻ, autofill), `Bookmarks`, danh sách tải xuống.
- Văn bản đã giải mã (mật khẩu, cookie dạng JSON/Netscape, thẻ, địa chỉ) gửi qua `/content-upload`.

**Lấy khóa giải mã (lần lượt, nếu bước trước thất bại):**
1. v10/v11: `@primno/dpapi` (DPAPI người dùng) hoặc PowerShell `CryptUnprotectData` biên dịch C# (`VsDpapi`).
2. **v20 (App-Bound Encryption):**
   - `rr()`: PowerShell `Add-Type` gọi COM `IElevator` trực tiếp (CLSID Chrome `708860E0-…`, Edge `1FCBE96C-…`, Brave `576B31AF-…`).
   - `nr()`: **tiêm DLL**. PowerShell/C# tạo tiến trình trình duyệt ở trạng thái treo (`CreateProcessW`, cờ `0x08000004` = không cửa sổ + treo), `VirtualAllocEx` + `WriteProcessMemory` ghi đường dẫn `abe_payload.dll`, `CreateRemoteThread(LoadLibraryA)`, sau đó đọc khóa từ file `ABE_OUT` (log `…status.txt`), rồi `TerminateProcess`. DLL chạy trong ngữ cảnh trình duyệt nên vượt kiểm tra đường dẫn của dịch vụ elevation (đây là DLL `abe_payload.dll`).
   - Khóa/thông số v20 cố định (công khai của Chrome) có sẵn trong code (AES-256-GCM, ChaCha20-Poly1305, XOR) để giải mã cookie ngoại tuyến.
3. **DPAPI cấp SYSTEM** (`Gn`): đọc `C:\Windows\ServiceProfiles\LocalService\AppData\Roaming\Microsoft\Crypto\Keys` và `C:\ProgramData\Microsoft\Crypto\SystemKeys`, tạo tác vụ `schtasks` chạy `CryptUnprotectData` (cần admin; thất bại báo `schtasks_access_denied`/`need_admin`).
4. Firefox/Tor: giải mã NSS (`nssPrivate`, 3DES/AES-CBC).
5. Nếu file bị khóa: đợi, hoặc operator ra lệnh `ss_kill_browsers`.
6. macOS: Keychain (`Chrome Safe Storage`…) và **CDP**: khởi chạy trình duyệt headless (`--headless=new --remote-debugging-port=9223/9224 --remote-allow-origins=*`, thư mục `/tmp/hc_chrome|hc_brave`), đọc file qua DevTools Protocol. Linux: libsecret/KWallet.

### 5.2 Extension ví, ví offline, password manager
- Quét dữ liệu extension (thư mục `Local Extension Settings`, `Sync Extension Settings`, IndexedDB) cho **74 ID extension ví** (`Ze`) cộng thêm danh sách `Ge` (4 ID), gửi bằng bước `ext:<trình duyệt>:<id>` và `wallet:`.
- Ví offline: **Exodus** (`exodus.wallet`, `Local Storage\leveldb`, thư mục `Desktop\exodus-exports`), **Electrum** (`wallets`, CSV/JSON trong home), **Guarda**, **Atomic**.
- **1Password**: sao chép CSDL desktop (`1Password_desktop_*`, `*_1pass_sync_*`, `*_1pass_idb_*`); trước đó đóng `1Password.exe`/`BrowserHelper` nếu cần.

### 5.3 Hệ thống, SSH, shell
- `system.txt`: hostname, machine, platform, release, type, username, version, UUID, IP/địa lý.
- **SSH (`Oc`)**: đọc `.bash_history`, `.zsh_history`, `fish_history`, `PSReadLine\ConsoleHost_history.txt`; lọc dòng chứa `ssh/scp/sftp/rsync`; gửi thành `ssh_history.txt` (`/content-upload`); với mỗi khóa trong tham số `-i`, gửi file `sshkey_<tên>` (`/file-upload`). Thư mục `.ssh` bị loại khỏi đợt quét hàng loạt (nằm trong `excludePatterns`) nhưng operator vẫn đọc được bằng `read_file`/`ss_upf`.
- Có thêm cờ "cloud sync roots" (OneDrive, Dropbox, MEGA) được quét như thư mục thường.

### 5.4 Quét file hàng loạt ("backup" / "FileZilla")
Chạy tự động khi khởi động (tiến trình con), upload tới `K/upload`.
- **Phạm vi:** `Desktop`, `Documents`, `Downloads`, mọi thư mục khác trong profile (trừ `AppData`, `Contacts`, `Favorites`, `Links`, `Music`, `Pictures`, `Videos`, `Saved Games`, `Searches`, `Start Menu`, `Templates`, `3D Objects`), thư mục dấu chấm (`.xxx`, trừ danh sách loại trừ), file lẻ ở thư mục gốc profile, thư mục cấp một của ổ **D: đến Z:**, thư mục đồng bộ đám mây.
- **Điều kiện gửi:** kích thước ≤ 10MB; không có đuôi/thành phần đường dẫn trong `excludePatterns` (276 mẫu: ví dụ `.exe .dll .mp4 .mov .mp3 .avi .sh .bin .jar .yml .yaml .ppt .psd .key .sol .sys .msi .apk .dat .sqlite .pkg .dmg`, `*.log *.tmp *.bak`, và các từ trong tên thư mục như `node_modules cache temp tmp dist build lib library public vendor example extension media image(s) fonts plugin module package android debug`, cùng `.git .vscode .docker .ssh .aws .azure .config .gnupg .cargo .npm .cache`); server chưa có file (`HEAD /check`).
- **Thứ tự ưu tiên (`Mu`):** tên chứa từ khóa (`wallet private public seed mnemonic phrase pass cred secret metamask phantom trust crypto bitcoin ethereum solana polygon base optimism arbitrum avalanche fantom litecoin dogecoin ripple stellar cardano polkadot terra near cosmos eos tezos flow aptos ton kusama coin coinbase binance bitfinex bitstamp` và 3 ID extension ví) → `.env*` → `env.example` → `config*` → `.json` → ảnh chụp màn hình `.png` → `.png` → `.js/.jsx/.ts/.tsx` → còn lại.
- **Nhịp độ:** cửa sổ làm việc 5 phút rồi nghỉ 1 phút (`[FileZilla] 5 min work window reached`), 7 upload song song (Windows; macOS 4), khối 32KB, thử lại có lùi, tạm dừng khi "socket-priority work" (khi đang xử lý lệnh/thu thập). Server đánh dấu hoàn tất qua `POST /complete`; nếu `GET /status` báo `completed` thì không quét lại trừ khi `VS_BACKUP_FORCE_RUN=1`.
- Trước khi quét chính có bước `env search` (`pa`/`fa`) tìm file `.env` và cấu hình trên các ổ.

### 5.5 Keylogger
`jl()` tải `win_key_agent.js` (`/download-win-key`) vào `.vs_cache`, chạy bằng node với `NODE_PATH=.vs_cache\node_modules`, `CLIENT_ID`, `SERVER_URL`; có hẹn giờ tự kiểm tra và chạy sau 3–7 phút kể từ lúc khởi động (và theo lệnh `ss_key_start`). Agent đặt hook bàn phím mức thấp bằng `koffi`/`user32.dll`, ghi phím + tiêu đề cửa sổ + clipboard, log cục bộ `%LOCALAPPDATA%\Microsoft\RealPrior\Cache\7f25-2c61-6824-bf72`, nhịp tim 30 giây tới `195.66.213.162:443`. Có dọn các log "keyboard-agent" của bản cũ (đánh dấu `.legacy_keyboard_logs_cleaned_1.6.14`).

## 6. Duy trì, cập nhật, gỡ

| Việc | Windows |
|---|---|
| Persistence | `reg add HKCU\...\Run /v "Microsoft Realtime Agreegator" /d wscript.exe "...\main.vbs"`; bản dự phòng file `.vbs`/`.cmd` trong thư mục Startup |
| Khởi động lại/cập nhật | Ghi `%TEMP%\_vs_upd_<pid>_<ms>.vbs` (chờ vài giây, chép `main.js.pending` đè `main.js`, chạy `wscript //B main.vbs`, tự xóa) |
| Gỡ | `%TEMP%\_vs_rm_<pid>_<ms>.vbs` xóa thư mục/file rồi tự xóa; `ss_del` |
| Thư mục lạ | `_force_client.pending`, `_force_bootstrap.js`, `main.js.pending`, `client.pid` |

macOS: `LaunchAgent com.<tên>.plist` + bundle `.app`; Linux: `~/.config/autostart/*.desktop`.

## 7. TakePC: cài NetSupport Manager (`ss_take_pc`, chỉ Windows)
Tải `client-portable.zip` (`/download-client-portable`) vào `%TEMP%\vs_takepc_<pid>_<ms>\`, `Expand-Archive`, chạy `deploy-to-target.cmd /nopause` (hoặc tự triển khai): sao chép `Service.exe` + DLL + `client32.ini` vào `%LOCALAPPDATA%\Microsoft\Windows\SettingSync\Caches\{8F7A2C1E-…}\Win32\ResCache\`, vá `Filename=` trong `client32.ini`, chạy `cksini.exe` sửa checksum, tạo `ResCache.vbs` và Run key `SettingSyncResCache`, khởi động client. Cổng: `chainmanamgentagency.info:443`. Staging bị xóa sau đó.

## 8. Che giấu và chống phát hiện
- Chạy trong bộ nhớ; tên giả mạo Microsoft/Apple; thư mục `.vs_cache`; mọi tiến trình ẩn (`windowsHide`, `//B`, `detached`, `stdio: ignore`).
- Chặn operator xem thư mục `.vs_cache`; nuốt lỗi; không thoát khi gặp lỗi.
- Giảm dấu vết SSH (`UserKnownHostsFile=NUL`), xóa file mật khẩu tạm.
- Tự giới hạn tốc độ để tránh đột biến mạng; tạm dừng khi có việc ưu tiên.
- Phân loại máy (VM/cloud/AD) gửi lên server trước khi khai thác.
- Giao tiếp HTTP thường: dễ bị chặn/giám sát ở mạng, nhưng họ không dùng TLS.

## 9. Dấu vết để săn tìm
- Windows: sự kiện Application `VBScriptDeprecationAlert` (ID 4096) có `ProcessTreeEnhanced` chứa `main.vbs`, `_vs_upd_*.vbs`, `_vs_rm_*.vbs`, `ResCache.vbs`; tiến trình `wscript.exe //B ... main.vbs` có cha là `node.exe`.
- File/thư mục: `~\.vs_cache\{main.js,main.vbs,package.json,log.log,loader.log,abe_payload.dll,win_key_agent.js,_abe_el_*,_abe_ksp_*,main.js.pending,_force_*,client.pid}`; `%TEMP%\{vs_takepc_*,_vs_upd_*,_vs_rm_*,ssh_askpass_*,ssh_pw_*,event_log_*,nsm-run.cmd}`; `%LOCALAPPDATA%\Microsoft\RealPrior\Cache\`; `...\SettingSync\Caches\{8F7A2C1E-9B4D-4E6A-A3C8-1D5E7F90B2C4}\`.
- Registry: Run `Microsoft Realtime Agreegator`, `SettingSyncResCache`.
- Mạng: HTTP thường tới `famoustreamservice.store:80/443`, `37.27.51.109:7576`, `195.66.213.162:443`, `chainmanamgentagency.info:443`; header `X-Machine-ID`, `X-File-Path`, `X-Upload-ID`, `X-Client-ID`; đường dẫn `/client`, `/register`, `/file-upload`, `/content-upload`, `/upload`, `/check`, `/status`, `/complete`.
- Gọi metadata cloud (`169.254.169.254`) và `ip-api.com/ipwho.is/freeipapi.com` từ tiến trình `node.exe` lạ.

## 10. Hệ quả cho nạn nhân (phạm vi dữ liệu có thể bị lấy)
Mật khẩu/cookie/thẻ/lịch sử của mọi profile trình duyệt, dữ liệu ví (extension và ví offline), cơ sở dữ liệu 1Password, lịch sử shell và khóa SSH dùng với `-i`, mọi file ≤10MB ngoài danh sách loại trừ trong các thư mục người dùng (ưu tiên file chứa từ khóa), tổ hợp phím và clipboard (nếu keylogger bật), và quyền chạy lệnh/SSH/SQL/Docker từ máy nạn nhân.

## 11. Giới hạn của phân tích này
- Mã đọc ở mức hành vi cho nhánh macOS/Linux (Keychain, libsecret, LaunchAgent, systemd) và nội bộ các lớp `Tc`/`Ic` (phiên SQL/Docker).
- Một số hàm phụ (định dạng log, tiện ích) chỉ được lập chỉ mục, chưa đọc từng dòng.
- Đây là bản `1.6.31` tải ngày 18/11/2026; bản chạy trên máy nạn nhân ngày 17/11 không còn để so sánh.
- Phần phía máy chủ (Express, bảng quản trị, nơi lưu dữ liệu đánh cắp) không có trong dữ liệu.
