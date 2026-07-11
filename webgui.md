# Hướng dẫn sử dụng WebGUI zapret (Mihomo Proxy RoS)

> Container `medium1992/mihomo-proxy-ros` có **2 giao diện web**. Truy cập từ máy trong **mạng admin**
> (đã định tuyến tới subnet container `10.10.0.0/24`). **Không** mở các port này ra WAN.

| Giao diện | URL | Dùng để |
|---|---|---|
| **WebUI cấu hình zapret** | `http://10.10.0.2/` | Xem config, sửa env, sinh lệnh MikroTik, **blockcheck** (test chiến lược) |
| **Dashboard mihomo** (metacubexd) | `http://10.10.0.2:9090/ui/` | Theo dõi kết nối, băng thông, log realtime |

> IP `10.10.0.2` = container. Nếu đổi setup thì thay bằng IP veth container của bạn.
> Giao diện mặc định **tiếng Nga** → dùng Chrome "Dịch trang" (auto-translate) sang Việt/Anh.

---

## 1. WebUI cấu hình — `http://10.10.0.2/`

Trang **"Mihomo Proxy RoS"** có 3 mục chính:

- **Обзор конфигурации** (Tổng quan cấu hình) — xem config đang chạy.
- **Карта env** (Bản đồ env) — xem/chỉnh các biến môi trường (`ZAPRET_CMD1`, `TPROXY`, `ZAPRET_PACKETS`…) qua form.
- **Команды для MikroTik** (Lệnh cho MikroTik) — sinh sẵn các lệnh RouterOS (`/container/envs/add …`) để **copy dán** vào router.

### ⚠️ Lưu ý quan trọng về nguồn cấu hình

Setup hiện tại quản lý chiến lược zapret bằng **RouterOS envlist `zapret`** (biến `ZAPRET_CMD1` set qua CLI — xem hướng dẫn cài đặt). Đây mới là **nguồn chuẩn, tồn tại qua reboot**.

Nếu bạn sửa env **trực tiếp trong WebUI**, nó chỉ đổi cấu hình runtime trong container; khi container **restart** (hoặc auto-update hàng tuần), RouterOS bơm lại `envlist=zapret` → **ghi đè** thay đổi từ WebUI.

👉 **Cách dùng đúng:**
- Dùng WebUI chủ yếu để **blockcheck** (tìm chiến lược) + **xem dashboard**.
- Khi tìm được chiến lược tốt, **áp bằng CLI** để bền vững:
  ```
  /container/envs/set [find list=zapret key=ZAPRET_CMD1] value="...chiến lược mới..."
  /container/stop [find comment~"zapret DPI"]; /container/start [find comment~"zapret DPI"]
  ```

---

## 2. Blockcheck — tìm chiến lược desync hoạt động (tính năng quan trọng nhất)

Đây là công cụ trả lời câu "chưa biết site nào / strategy nào chạy": nó **tự thử hàng loạt chiến lược** desync với 1 domain và báo cái nào vượt được DPI.

Trong WebUI có **blockcheck1**, **blockcheck2** (và **byedpi-check**). Cách dùng:

1. Nhập **domain** cần test (ví dụ `youtube.com`, `discord.com`, hoặc site bạn thấy bị bóp/chặn).
2. Chọn giao thức (TLS/HTTPS, QUIC) và nhấn **chạy** (Start).
3. Công cụ chạy qua nhiều tổ hợp `--dpi-desync=…` và in kết quả **PASS/FAIL** cho từng chiến lược.
4. Lấy chiến lược **PASS** đưa vào `ZAPRET_CMD1` (qua CLI như mục 1).

> Có nút cancel/status để theo dõi và dừng giữa chừng. Blockcheck test **từ chính container**, đúng đường mạng thật của ISP bạn → kết quả sát thực tế.

---

## 3. Dashboard mihomo — `http://10.10.0.2:9090/ui/`

Giao diện **metacubexd** (dashboard chuẩn của mihomo). Xem:

- **Connections** — kết nối realtime: host/đích, app, upload/download, thời gian.
- **Traffic** — biểu đồ băng thông up/down.
- **Logs** — log realtime (đặt mức `error`/`info`/`debug`).
- **Rules / Proxies** — luật định tuyến và outbound đang dùng (setup hiện tại là DIRECT + nfqws desync).

Khi client trong VLAN đang chạy, mở tab **Connections** để xác nhận traffic thực sự đi qua container.

> API điều khiển ở `http://10.10.0.2:9090` (ví dụ `/version`, `/configs`). Dashboard tự gọi API này.

---

## 4. Quy trình dùng thực tế

```
1. Mở http://10.10.0.2/  → Blockcheck → nhập vài site (youtube, discord, site bị bóp)
2. Chạy → xem chiến lược nào PASS
3. Áp chiến lược PASS vào ZAPRET_CMD1 bằng CLI (mục 1) → restart container
4. Mở http://10.10.0.2:9090/ui/ → tab Connections → kiểm tra traffic chạy qua, mượt
5. Trải nghiệm thực tế trên máy client trong VLAN
```

---

## 5. Bảo mật (đọc kỹ)

- Dashboard 9090 có `secret` **rỗng** và bind `0.0.0.0` → **bất kỳ ai vào được `10.10.0.2` đều điều khiển được mihomo**. Hiện chỉ **mạng admin** định tuyến tới được nên tương đối an toàn, nhưng nên đặt mật khẩu:
  ```
  /container/envs/add list=zapret key=UI_SECRET value="mat-khau-manh-cua-ban"
  /container/stop [find comment~"zapret DPI"]; /container/start [find comment~"zapret DPI"]
  ```
  Sau đó dashboard sẽ hỏi secret khi đăng nhập.
- **TUYỆT ĐỐI không** NAT/mở port 80 hoặc 9090 ra internet (WAN). Chỉ truy cập nội bộ.
- WebUI/dashboard không thay thế việc cấu hình bền vững qua RouterOS envlist (mục 1).

---

## 6. Không vào được WebUI?

- Kiểm tra container chạy: `/container/print` (status R).
- Từ router thử: `/tool/fetch url=http://10.10.0.2/ mode=http dst-path=test.html` (phải tải được).
- Máy bạn phải ở **mạng admin** (hoặc mạng có route tới `10.10.0.0/24`). Máy ngoài VLAN admin sẽ không tới được — đúng theo thiết kế.
