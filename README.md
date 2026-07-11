# Hướng dẫn cài đặt zapret (DPI bypass) trên MikroTik RouterOS

> Container `medium1992/mihomo-proxy-ros` chạy **zapret / nfqws** để vượt DPI.
> Đã triển khai và verify trên router **Home-CHR** (CHR x86_64, RouterOS 7.21.5).
> Traffic một VLAN được rẽ qua container bằng policy-routing; nfqws desync gói TLS/QUIC.

---

## 0. Yêu cầu

- RouterOS **7.4+** (bản này dùng 7.21.5). Từ 7.20 trở lên cú pháp env đổi `name=` → **`list=`**.
- Package **container** đã cài (`/system/package/print`).
- Device-mode bật container:
  ```
  /system/device-mode/print          # xem container: yes chưa
  # nếu chưa: /system/device-mode/update container=yes  (cần bấm nút/power-cycle để xác nhận)
  ```
- Kiến trúc container phải khớp CPU: x86 CHR → image amd64 (image này multi-arch, tự chọn).
- CHR không cần disk ngoài — container lưu trên ổ hệ thống (cần vài chục MB trống).

---

## 1. Kiến trúc

```
Client VLAN ──(policy-route)──> veth 10.10.0.2 (container)
                                   │  nftables TProxy (tcp/udp)
                                   ▼
                                 mihomo ──> nfqws (queue 301, desync fake TLS/QUIC)
                                   │
                                   ▼  default gw 10.10.0.1 (host)
                                 Host ──(masquerade)──> WAN
```

- Client DNS đi thẳng qua router (không qua container) → phân giải IP thật.
- Chỉ traffic **ra internet** của VLAN mới rẽ qua container; traffic nội bộ + tới router giữ local.

---

## 2. Tạo veth + IP host

```
/interface/veth/add name=veth-zapret address=10.10.0.2/24 gateway=10.10.0.1 comment="zapret DPI container"
/ip/address/add address=10.10.0.1/24 interface=veth-zapret comment="zapret container gw"
```

- `10.10.0.2` = IP container, `10.10.0.1` = IP host (gateway của container).
- NAT: chỉ cần rule masquerade `out-interface-list=WAN` sẵn có là container ra net được. Nếu chưa có:
  ```
  /ip/firewall/nat/add chain=srcnat action=masquerade src-address=10.10.0.0/24 out-interface-list=WAN
  ```

---

## 3. Cài container

```
/container/config/set tmpdir=zapret/tmp registry-url=https://registry-1.docker.io
/container/add remote-image=medium1992/mihomo-proxy-ros:latest \
  interface=veth-zapret root-dir=zapret/root hostname=zapret logging=yes \
  comment="zapret DPI - test vlan121"
```

Chờ tải/giải nén xong (status chuyển `E` → `S`):
```
/container/print detail       # đợi status = S (stopped)
/log/print where topics~"container"
```

---

## 4. Cấu hình chiến lược zapret (env)

> **QUAN TRỌNG:** RouterOS 7.20+ dùng `list=` (không phải `name=`).
> `ZAPRET_CMD1` = nguyên chuỗi tham số nfqws. KHÔNG set `--hostlist` → áp dụng **mọi host** (tất cả dịch vụ).

Chiến lược chung TLS(tcp443) + QUIC(udp443) — dán **1 dòng**:
```
/container/envs/add list=zapret key=ZAPRET_CMD1 value="--filter-tcp=443 --dpi-desync=fake,multidisorder --dpi-desync-split-pos=midsld --dpi-desync-fooling=badseq --dpi-desync-fake-tls=/zapret-fakebin/tls_clienthello_www_google_com.bin --new --filter-udp=443 --dpi-desync=fake --dpi-desync-repeats=6 --dpi-desync-fake-quic=/zapret-fakebin/quic_1.bin"
```

Gắn env vào container, bật auto-start, khởi động:
```
/container/set [find comment~"zapret DPI"] envlist=zapret start-on-boot=yes
/container/stop [find comment~"zapret DPI"]
/container/start [find comment~"zapret DPI"]
```

Kiểm tra nfqws chạy (log phải có `Starting ZAPRET_CMD1 on queue 301` và `2 user defined desync profile(s)`):
```
/log/print where topics~"container"
```

> Fakebin (tls/quic) và list mẫu nằm sẵn trong container: `/zapret-fakebin/`, `/zapret-lists/`.
> Nếu muốn dùng bản fakebin/list riêng thì mount đè:
> `/container/mounts/add name=fb src=zapret-fb dst=/zapret-fakebin` (đặt file .bin vào thư mục host trước).

---

## 5. Rẽ traffic một VLAN qua zapret (ví dụ vlan100_admin)

> Thay `vlan100_admin` và address-list `"IP Admin"` bằng VLAN / subnet của bạn.
> Rule "skip" giữ traffic nội bộ + tới router **không** vòng qua container (tránh loop, giữ quản trị được router).

```
# bảng định tuyến riêng
/routing/table/add name=via_zapret fib comment=zapret

# skip traffic nội bộ (đích nằm trong LAN của mình) - đặt TRƯỚC 2 rule mark
/ip/firewall/mangle/add chain=prerouting in-interface=vlan100_admin \
  dst-address-list="IP Admin" action=accept comment="zapret: skip intra-lan"

# mark connection + mark routing (rẽ ra container). Đặt DƯỚI các rule QoS.
/ip/firewall/mangle/add chain=prerouting in-interface=vlan100_admin connection-state=new \
  action=mark-connection new-connection-mark=zapret_conn passthrough=yes comment="zapret: mark conn"
/ip/firewall/mangle/add chain=prerouting in-interface=vlan100_admin connection-mark=zapret_conn \
  action=mark-routing new-routing-mark=via_zapret passthrough=no comment="zapret: mark routing"

# default route của bảng riêng trỏ vào container
/ip/route/add dst-address=0.0.0.0/0 routing-table=via_zapret gateway=10.10.0.2 comment="zapret: via container"
```

**Thứ tự rule mangle:** đặt zapret **DƯỚI** các rule QoS. `mark-routing passthrough=no` cắt chuỗi — nếu để lên trên, các rule QoS phía dưới bị bỏ qua. Thứ tự chuẩn: `mark-connection → mark-packet (QoS) → mark-routing (zapret)`.

---

## 6. Kiểm tra thành công

```
/ip/route/print where routing-table=via_zapret          # route phải ACTIVE (cờ A)
/ping 10.10.0.2 interface=veth-zapret count=3            # container reachable
/ip/firewall/mangle/print stats where comment~"zapret"   # counter mark-routing phải tăng khi có traffic
```
Từ máy trong VLAN đó: mở web HTTPS bình thường (Google/YouTube phải vào, code 200). Traffic của máy giờ đi qua zapret.

> Không test được từ trong container bằng wget (fake-ip/tproxy chỉ xử lý traffic forward, không xử lý traffic tự phát của container). Ping 8.8.8.8 trong container chỉ để xác nhận container có internet.

---

## 7. Tinh chỉnh chiến lược (khi 1 site vẫn bị chặn)

Chiến lược desync phụ thuộc DPI của ISP — không có 1 lệnh đúng cho mọi mạng. Nếu site nào không vào được, đổi `ZAPRET_CMD1` rồi restart container. Vài biến thể hay dùng cho phần TLS:

- `--dpi-desync=fake,disorder2 --dpi-desync-split-pos=1`
- `--dpi-desync=fake,split2 --dpi-desync-split-pos=1 --dpi-desync-fooling=badseq`
- Đổi fake TLS: `--dpi-desync-fake-tls=/zapret-fakebin/tls_clienthello_iana_org.bin`
- Thêm HTTP(80): `--new --filter-tcp=80 --dpi-desync=fake,multisplit --dpi-desync-split-pos=method+2`

Đổi env rồi:
```
/container/envs/set [find list=zapret key=ZAPRET_CMD1] value="...chiến lược mới..."
/container/stop [find comment~"zapret DPI"]; /container/start [find comment~"zapret DPI"]
```

---

## 8. Auto-update (đã cài sẵn)

Script `zapret-update` + scheduler chạy **hàng tuần 04:00** recreate container để lấy image `:latest` mới nhất. Có kiểm tra internet, chờ extract xong mới start, gắn lại env + start-on-boot.

```
/system/scheduler/print where name=zapret-update     # xem lịch
/system/script/run zapret-update                     # chạy update thủ công ngay (giờ thấp điểm)
/log/print where message~"zapret-update"             # xem kết quả
```

- Đổi tần suất: `/system/scheduler/set zapret-update interval=30d` (hàng tháng) v.v.
- **Lưu ý downtime:** lúc update, internet của VLAN đang rẽ qua zapret gián đoạn ~30–60s (container bị recreate). Quản trị router (SSH tới IP router) **không** bị ảnh hưởng vì giữ local.
- **Ceiling:** re-pull `:latest` mỗi lần dù chưa có bản mới (RouterOS không có "pull if newer" native). Chấp nhận được với tần suất tuần/tháng.

---

## 9. Gỡ sạch (rollback)

```
/ip/firewall/mangle/remove [find comment~"zapret"]
/ip/route/remove [find comment~"zapret: via container"]
/routing/table/remove [find name=via_zapret]
/container/stop [find comment~"zapret DPI"]; /container/remove [find comment~"zapret DPI"]
/interface/veth/remove veth-zapret
/ip/address/remove [find interface=veth-zapret]
/container/envs/remove [find list=zapret]
/system/scheduler/remove [find name=zapret-update]
/system/script/remove [find name=zapret-update]
```

---

## 10. Ghi chú vận hành

- VLAN quản trị rẽ qua zapret → phụ thuộc container. Đã bật `start-on-boot=yes` nên tự chạy lại sau reboot; nếu container chết, chỉ internet VLAN đó gián đoạn, **vẫn vào được router** để xử lý.
- QoS: traffic rẽ qua container không được queue tree hiện tại shape (đổi path + đổi src IP). Với VLAN quản trị không sao; nếu áp lên VLAN cần QoS thật thì cần thiết kế lại.
- Không ảnh hưởng các tunnel L2TP/khách khác — mọi rule đều scope theo `in-interface` của 1 VLAN.
