# So sánh chi tiết: mihomo-proxy-ros vs nfqws-mikrotik (bypass DPI trên MikroTik)

> Hai cách chạy DPI-bypass trong container RouterOS, quan sát thực tế khi triển khai trên CHR x86_64 / RouterOS 7.21.5.
> - **mihomo** = image `medium1992/mihomo-proxy-ros` (engine Clash.Meta/mihomo, Go)
> - **nfqws** = image `wiktorbgu/nfqws-mikrotik` (bol-van/zapret nfqws, C)

---

## TL;DR

- Chỉ cần **bypass DPI thuần** (kể cả IPv6), muốn **nhẹ**: chọn **nfqws**.
- Cần **proxy / định tuyến qua VPN / geo-split / dashboard đẹp**: chọn **mihomo** (nhưng **không làm được IPv6** ở image này).

---

## Bảng so sánh nhanh

| Tiêu chí | **mihomo-proxy-ros** | **nfqws-mikrotik** |
|---|---|---|
| Engine | Clash.Meta (mihomo), **Go** | bol-van/zapret **nfqws**, **C** |
| Cơ chế | **TProxy + proxy đầy đủ** (cắt & tạo lại kết nối) | **NFQUEUE desync** (sửa vài packet đầu) |
| RAM điển hình | **~30 MB** | **~2–8 MB** |
| RAM tăng theo lưu lượng | Có (giữ state mọi conn) | Gần như không |
| Đụng vào data | **Toàn bộ luồng** qua userspace | **~9 packet đầu**/conn, phần còn lại chạy trong kernel |
| IPv4 | ✅ | ✅ |
| **IPv6** | ❌ **tắt cứng** (disable_ipv6=1, khó sửa) | ✅ **dual-stack** (bật `DISABLE_IPV6=0`) |
| Cấu hình | **RouterOS envs** (`ZAPRET_CMD1`) + config.yaml generate | **File disk** `/opt/zapret/config` (`NFQWS_OPT`) |
| WebUI | ✅ **dashboard metacubexd** (:9090) + UI :80 | ⚙️ **blockcheck** (CLI + web đơn giản) |
| Proxy (VLESS/SS/Trojan/WG) | ✅ | ❌ |
| Định tuyến qua VPN / geo-split | ✅ (re-originate → route được) | ❌ (chỉ desync, không định tuyến) |
| DNS fake-ip / rule engine | ✅ | ❌ |
| Cài đặt | Kéo image là chạy | Lần đầu **tự tải zapret từ GitHub** (cần `dns=`) |
| Độ trễ thêm | Cao hơn (proxy cả luồng) | Rất thấp |

---

## 1. Cơ chế hoạt động — gốc rễ mọi khác biệt

### mihomo — proxy trong suốt (TProxy)
Client → traffic bị TProxy **chuyển hết vào mihomo** → mihomo **cắt kết nối client, mở kết nối MỚI ra ngoài**, chuyển tiếp 2 chiều. Vì nó là **một đầu cuối proxy thật**:
- Mọi byte của mọi kết nối đi **qua userspace** (kernel → mihomo → kernel).
- Phải **giữ state + buffer** cho từng kết nối.
- Vì "mở kết nối mới từ chính nó" nên **định tuyến được** (đẩy non-VN qua NordVPN, chọn outbound theo rule…).

### nfqws — sửa gói ở hàng đợi kernel (NFQUEUE)
nftables đẩy **vài packet đầu** của mỗi kết nối (thường 1–9 gói handshake 80/443) lên nfqws qua **NFQUEUE**. nfqws **chèn fake packet / cắt (split) / gây desync** để làm rối DPI, rồi trả gói lại kernel. Phần còn lại của luồng **chạy thẳng trong kernel**, nfqws không đụng.
- Không giữ kết nối, không buffer, không proxy → **cực nhẹ**.
- Chỉ **desync**, **không định tuyến** được (không thể tự đẩy qua VPN).

> Chính điểm này giải thích: **mihomo nặng vì làm nhiều hơn** (proxy + route), **nfqws nhẹ vì làm đúng 1 việc** (desync).

---

## 2. RAM / CPU / độ trễ

| | mihomo | nfqws |
|---|---|---|
| Base RAM | ~30 MB (Go runtime + GC + goroutine/conn + geodata) | ~2–8 MB (binary C tĩnh) |
| Theo lưu lượng | Tăng (buffer + state) | Phẳng |
| CPU | Cao hơn (copy toàn bộ data qua userspace) | Thấp (chỉ xử lý gói đầu) |
| Trễ | Thêm 1 hop proxy | Gần như không |

Trên router yếu / CHR RAM thấp, khác biệt này **rõ rệt**.

---

## 3. IPv6 — khác biệt lớn nhất trong 2 image này

- **nfqws**: **dual-stack sẵn**. Chỉ cần `DISABLE_IPV6=0` trong `/opt/zapret/config` → zapret tạo nft cho cả `ip` lẫn `ip6`, **một tiến trình nfqws xử lý cả hai họ**. Cấp IPv6 cho veth (`gateway6`) là chạy.
- **mihomo (image này)**: **tắt IPv6 cứng** — entrypoint chạy `sysctl disable_ipv6=1` vô điều kiện mỗi lần start, config sinh ra `ipv6: false`, nft TProxy chỉ IPv4. Bản thân **engine mihomo hỗ trợ IPv6**, nhưng **image** chặn; muốn mở phải viết lại entrypoint (fragile, mất khi update image). → **Không nên**.

---

## 4. Cấu hình

| | mihomo | nfqws |
|---|---|---|
| Nơi lưu | RouterOS `envlist` (ENV) + config.yaml generate mỗi start | File `/opt/zapret/config` trên disk container |
| Chiến lược desync | ENV `ZAPRET_CMD1="..."` | Dòng `NFQWS_OPT="..."` |
| Sửa strategy | Sửa env (nhớ `list=` từ ROS 7.20+) → restart | `/container/shell` → sửa file → `zapret restart` |
| Bền vững | Env ghi đè config lúc start | File persist trên disk |

nfqws **không cần RouterOS envs** — gọn hơn về mặt quản trị (mọi thứ trong container).

---

## 5. Chức năng kèm theo

**mihomo có nhiều thứ ngoài DPI-bypass** (đây cũng là lý do nó nặng):
- Dashboard **metacubexd** (:9090) — theo dõi connection/băng thông/log realtime.
- WebUI riêng (:80) sinh lệnh RouterOS + blockcheck.
- **Proxy đủ giao thức**: VLESS / Shadowsocks / Trojan / WireGuard / Hysteria…
- **DNS fake-ip**, rule engine, geosite/geoip, subscription, health-check.
- **Định tuyến/geo-split**: đẩy traffic theo đích ra outbound/VPN khác nhau (vd non-VN → NordVPN).

**nfqws chỉ có**:
- **blockcheck** — công cụ tự dò chiến lược desync hiệu quả cho từng domain.
- (không proxy, không VPN routing, không dashboard nặng.)

---

## 6. Cài đặt & cập nhật

| | mihomo | nfqws |
|---|---|---|
| Lần đầu | Kéo image → chạy ngay | Container **tự tải bol-van/zapret từ GitHub** rồi cài (cần `dns=`, ~1–2 phút) |
| Cập nhật lõi | Re-pull image | Xoá `/opt/zapret` → restart (tải bản zapret mới) |
| Bẫy thường gặp | ENV `name=` vs `list=` (đổi ở ROS 7.20+) | Thiếu `dns=` → fail cài (không resolve GitHub) |

---

## 7. Khi nào chọn cái nào

**Chọn nfqws khi:**
- Chỉ cần **bypass DPI** (chống bóp/chặn) cho **v4 và/hoặc v6**.
- Muốn **nhẹ**, ít RAM/CPU, router yếu.
- Thích cấu hình gọn 1 chỗ, không cần dashboard.

**Chọn mihomo khi:**
- Cần **proxy / VPN / geo-split** (định tuyến theo đích, đổi outbound, đi qua NordVPN/WARP…).
- Cần **dashboard trực quan** theo dõi kết nối.
- **Chỉ IPv4** là đủ (image này không làm IPv6).

**Kết hợp:** dùng **nfqws lo DPI (v4+v6)** + làm **định tuyến VPN ở tầng RouterOS** (mangle → bảng route VPN) → có cả hai lợi ích mà **không cần mihomo**. Đây thường là hướng gọn nhất cho "vừa bypass DPI vừa split-tunnel".

---

## 8. Tóm tắt một dòng

> **nfqws** = "con dao nhỏ sắc" — sửa vài gói ở kernel, nhẹ, làm được IPv6, chỉ để bypass DPI.
> **mihomo** = "bộ đồ nghề đầy đủ" — proxy thật, định tuyến/VPN/dashboard, nhưng nặng và (ở image này) không IPv6.
