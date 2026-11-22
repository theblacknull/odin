# Giao thức C2 và phía máy chủ (dựng lại từ mã khách + quan sát)

Phía máy chủ của kẻ tấn công **không có** trong dữ liệu. Tài liệu này ghi lại (1) phần máy chủ phát ra đã thu thập được, (2) "hợp đồng" giao thức mà mã khách mong đợi (suy ra từ code tầng 4), (3) bằng chứng về cách kích hoạt. Chưa đăng ký hay gửi dữ liệu giả vào máy chủ của họ.

## 1. Kiểm kê phần phát payload (đã thu thập đủ mọi endpoint có trong code)

| Endpoint (máy chủ trung tâm) | Nội dung | Kích thước | SHA256 (đầu) | Lưu trong odin |
|---|---|---|---|---|
| (blockchain) TRON `THKrGQ3E…` → BSC tx | Tầng 3 (4 phiên bản) | 5070 / 4734 | 6c466fe7… / 312432a6… | `stage3_c2_bootstrap/variants/` |
| `GET :80/session/<token>` | JSON giả `package.json` (khóa `sessions` = tầng 4) | 305.441 | (xem `SHA256SUMS.txt`) | `stage4_stealer_rat/c2_response_fake_vscode_package.json.txt` |
| `GET :80/download-client/101` | Tầng 4 (trùng nội dung khóa `sessions`) | 272.299 | 9b5bac06… | `stage4_stealer_rat/stage4_sessions_payload.js.txt` |
| `GET :80/download-main/101` | Tầng 2 `main.js` (trùng bản trên máy nạn nhân) | 8.697 | 54ea75e4… | `stage2_blockchain_loader/main.js.txt` |
| `GET :80/download-abe-payload` | `abe_payload.dll` (trùng bản trên máy nạn nhân) | 9.728 | 47a76888… | `stage5_modules/abe_payload.dll.bin` |
| `GET :443/download-win-key` | `win_key_agent.js` (keylogger) | 91.184 | f59fdaeb… | `stage5_modules/win_key_agent_keylogger.js.txt` |
| `GET :80/download-client-portable` | `client-portable.zip` (NetSupport) | 2.298.289 | 90b09c9f… | `stage5_modules/netsupport_client_portable.zip.bin` |
| `GET :443/download-app` | 404 "Binary n…" (không có file) | 16 | | không có |

Quan sát khi truy vấn ngày 18/11/2026 06:34 GMT: máy chủ chạy Express (`X-Powered-By: Express`), trả `200` cho yêu cầu không có header nhận dạng nào, không thấy lọc theo User-Agent hay IP ở mức cơ bản. Lọc nâng cao (theo `client_id`, địa lý, thời gian) **chưa biết**; `client_id` khác `101` chưa được thử.

## 2. Đăng ký nạn nhân và "khóa" (`storageKey`)

`POST http://famoustreamservice.store:80/register`, JSON:
`{username, customId, instanceUsername, platform, osType, osRelease, osVersionLabel, osVersionText, clientVersion, ipInfo?}` (xem `REPORT_STAGE4_DETAIL.md`, mục 3).
Mã khách mong đợi trả về `{storageKey, label}` (hàm `_o()` ghi đè nhãn bằng giá trị server nếu có). Mã khách lặp vòng (1 giây/lần) cho tới khi nhận được `storageKey` hợp lệ (không chứa `Unknown`) trước khi upload.

**`storageKey` không phải khóa bí mật.** Mã khách tự dựng nhãn mặc định:
`storageKey = "<customId>_" + <hostname>_<nhãnOS>_<tênNgườiDùng>`, trong đó `customId = 101`, hostname bỏ đuôi `.local` và đổi khoảng trắng thành `_`, nhãn OS Windows là `Win11` (build ≥ 22000) hoặc `Win10`, ký tự `< > : " / \ | ? *` đổi thành `_`.
Ví dụ dạng: `101_<hostname>_Win11_<user>`. Nhãn này là tên thư mục lưu dữ liệu của nạn nhân trên máy chủ (và là giá trị header `X-Machine-ID`, `folderName`, `storageKey`). Máy chủ có thể ghi đè bằng giá trị riêng; khi đó nhãn thật chỉ có ở phía họ.
Giá trị cụ thể của máy bị nhiễm cần ghi vào báo cáo abuse để nhà cung cấp lưu/tìm dữ liệu; tôi không ghi sẵn tên máy/người dùng vào tài liệu này.

Hằng số khác đã lưu (xem `README.md`): token phiên `6c3368b5fa77237f`, `client_id=101`, khóa XOR dead-drop `02e4f3e7-9442-4f9c-a6f4-4fa136605d1c`, khóa API TronGrid `0abf12d7-601d-4565-bba3-7f94d807468c`. Các khóa giải mã Chrome (AES/ChaCha20/XOR v20) nằm sẵn trong code tầng 4 (công khai của Chrome), không do máy chủ cấp.

## 3. Cách kích hoạt đợt thu thập (đã lưu phía khách; phía máy chủ chưa có)

**Hợp đồng:** máy chủ gửi sự kiện socket.io `upload` (namespace `/client`) với **một đối số chuỗi** (tên thư mục/`storageKey`) tới máy nạn nhân. Bộ xử lý ở tầng 4 chạy tuần tự:
1. `extensions`: với mỗi trình duyệt và mỗi ID trong danh sách 74 ID ví → `Uc()` (thư mục `Local Extension Settings`, `Sync Extension Settings`, IndexedDB).
2. `system` → `system.txt` (thông tin máy + IP).
3. `ssh` → `ssh_history.txt` + khóa SSH trong tham số `-i`.
4. `logins:<trình duyệt>` → `Login Data`, `Local State`, `Cookies`, `History`, `Web Data`…; nếu bị khóa thì đánh dấu thử lại.
5. `logins-retry` / `cookies-retry`: **chờ tối đa 30 phút** (`Tn(browser, 30*60*1000)`) cho trình duyệt thoát, rồi thử lại.
6. `wallet:<Exodus|Electrum|Guarda|atomic>` → ví offline.
7. `1password` → CSDL 1Password.
8. `flush` → thử lại các file thất bại; báo `exec_result: upload finished…`.

**Bằng chứng khớp trên máy nạn nhân (17/11):**
- 09:19:29 khởi chạy; **09:19:47** (sau ~18 giây) dữ liệu extension MetaMask bị đọc → bước 1 bắt đầu ngay sau khi kết nối.
- 09:24:37 tạo thư mục keylogger (hẹn giờ 3–7 phút sau khởi động).
- **09:53:50** `Login Data` của 4 profile Chrome đọc cùng một giây (bước 4, xử lý toàn bộ profile một trình duyệt trong một lần).
- **10:25:58** `Cookies` đọc cùng một giây, **32 phút** sau bước 4 (bước 5: chờ tối đa 30 phút cho Chrome thoát rồi thử lại).
- 09:27–09:32 quét file hàng loạt (tiến trình con "backup", chạy độc lập, khởi động tự động).

**Kết luận về tự động hóa:** việc bắt đầu thu thập chỉ ~18 giây sau khi mã khởi chạy không có độ trễ của con người, và toàn bộ trình tự sau đó (thời gian chờ 30 phút, xử lý cả profile cùng giây) khớp với mã. Điều này **ủng hộ mạnh** giả thuyết kịch bản tự động ở phía máy chủ (máy chủ phát `upload` ngay khi nạn nhân kết nối). Nó chưa chứng minh gì về các lệnh `exec`/SSH sau đó; những lệnh này cần quyết định ở phía họ và không có dấu vết trên máy nạn nhân.

Phần **chưa có**: mã phía máy chủ quyết định gửi `upload` (và các lệnh `ss_*`/`exec`/`dir`) cho từng nạn nhân, điều kiện kích hoạt, hàng đợi, và bảng quản trị.

## 4. Hợp đồng HTTP của các máy chủ nhận dữ liệu (suy ra từ client)

**Máy chủ file quét hàng loạt `37.27.51.109:7576`** (header chung: `X-Machine-ID`, `X-Username`, `X-Client-ID`)
| Yêu cầu | Ý nghĩa mong đợi |
|---|---|
| `HEAD /check` với `X-File-Path` (base64), `X-File-Size` | `200` = server đã có file → bỏ qua |
| `GET /status?machineId=<key>` | JSON `{completed: bool}`; `true` = không quét lại |
| `POST /upload`, thân file thô, header `X-File-Path`, `X-File-Size`, `X-Upload-ID`, `X-Is-Priority`, `X-Overwrite?` | `200` thành công, `409` đã tồn tại, `503` bận (thử lại có lùi) |
| `POST /complete` JSON `{username, machineId}` | `200` = đánh dấu hoàn tất |

**Máy chủ trung tâm `famoustreamservice.store:80`**
| Yêu cầu | Ý nghĩa mong đợi |
|---|---|
| `POST /register` (JSON) | Trả `{storageKey, label}` |
| `POST /file-upload` (multipart: `folderName`, `fileName`, `filePath`, `file`, `client_id`, `isUpload`, `storageKey`; header `X-File-Size`, `X-Client-Id`) | `{state:"success"}` hoặc `503/{state:"busy"}` |
| `POST /content-upload` (JSON: `username`, `folderName`, `fileName`, `fileContent`, `client_id`, `storageKey`) | `{state:"success"}` hoặc `503` |
| WebSocket `/client` (query: `storageKey`, `username`, `customId`, `instanceUsername`, `platform`, `osType`, `osRelease`, `osVersionLabel`, `osVersionText`) | Kênh điều khiển; heartbeat `client:ping` mỗi 60 giây ↔ `client:pong` |

Tên tệp gửi lên (`fileName`) mà server nhận: `logkc-edb` (Local State), `uld_<n>` (Login Data), `ulda_<n>`, `pck_<n>` (Cookies), `ssh_history.txt`, `sshkey_<tên>`, `system.txt`, `event_log` (log keylogger), `bash_history`, kèm tiền tố `ext_`/`wallet_` cho extension/ví.

## 5. Điều khiển và quét file (đã lưu)
- Bộ lệnh `exec`/`ss_*`/`dir`/`read_file`/`env`/`imp`/`pat` và logic thực thi: `REPORT_STAGE4_DETAIL.md` mục 4.
- Quy tắc quét file (phạm vi, 276 mẫu loại trừ, ưu tiên, nhịp): mục 5.4.
- Điều **không** có: danh sách lệnh thực tế mà "admin" hay kịch bản đã gửi cho máy bạn và logic chọn đích SSH/SQL/Docker.

## 6. Tóm tắt còn thiếu (cần phía máy chủ)
Mã máy chủ Express; logic chọn payload theo điều kiện; nơi/cách lưu dữ liệu theo `storageKey`; kịch bản phát `upload`/lệnh; bảng quản trị; danh sách các nạn nhân khác. Các thứ này chỉ có thể do nhà cung cấp hosting/cơ quan chức năng thu giữ.
