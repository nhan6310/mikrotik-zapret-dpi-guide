# Hướng dẫn nfqws (zapret) dual-stack IPv4 + IPv6 trên MikroTik RouterOS

> Chạy **bol-van/zapret (nfqws)** trong container để bypass DPI cho **cả IPv4 lẫn IPv6** bằng **một container duy nhất**.
> Dùng NFQUEUE desync (không proxy) → **nhẹ (~vài MB RAM)**, chỉ đụng vài packet đầu mỗi kết nối.
> Đã kiểm chứng trên CHR x86_64, RouterOS 7.21.5.

> **Khác với image mihomo**: nfqws **không dùng RouterOS envs**. Toàn bộ cấu hình nằm ở file `/opt/zapret/config` trong container (bền vững trên disk).

---

## 0. Yêu cầu

- RouterOS **≥ 7.21** (cần TProxy/nftables mới; nfqws dùng NFQUEUE).
- Package **container** đã bật (`/system/device-mode/print` → `container: yes`).
- Kiến trúc khớp CPU (x86 CHR → image amd64).
- **IPv6**: chỉ cần nếu muốn bypass v6. Cần WAN có IPv6 (prefix delegation). Nếu ISP cấp **/60 hoặc lớn hơn** thì cấp được /64 global riêng cho container → **không cần NAT66**. Nếu chỉ có /64 thì phải NAT66 (không nêu ở đây).

Ký hiệu trong hướng dẫn (thay bằng giá trị của bạn):
- `<LAN>` = interface VLAN client (vd `vlan100_admin`)
- `<PFX>::/60` = prefix IPv6 ISP cấp cho router (xem `/ipv6/dhcp-client/print detail` → `prefix=`)
- `<CT6>` = 1 /64 rảnh trong `<PFX>::/60` dành cho container (vd `<PFX+1>::/64`)

---

## 1. Kiến trúc

```
Client VLAN (v4 + v6 global)
  → mangle mark-routing (v4: bảng to_zapret · v6: bảng via_zapret6)
  → container nfqws (v4: 10.10.6.2 · v6: <CT6>::2)
      nfqws NFQUEUE desync (fake/split TLS 443, multisplit 80, fake QUIC)
  → forward ra WAN
      v4: masquerade (container SNAT → 10.10.6.2, rồi router NAT ra WAN)
      v6: KHÔNG NAT (global, giữ src client)
```

Một tiến trình nfqws xử lý **cả hai họ** (v4+v6) trên cùng queue — chỉ cần bật `DISABLE_IPV6=0`.

---

## 2. Tạo veth (kèm cả IPv4 + IPv6) + IP host

```
/interface/veth/add name=veth-z6 \
  address=10.10.6.2/24,<CT6>::2/64 gateway=10.10.6.1 gateway6=<CT6>::1 \
  comment="zapret nfqws dual-stack"

# IP host (đầu còn lại của veth)
/ip/address/add   address=10.10.6.1/24  interface=veth-z6
/ipv6/address/add address=<CT6>::1/64   interface=veth-z6 advertise=no
```

- `address`/`gateway6` của veth **tự gán IPv4+IPv6 cho container**, bền vững qua restart.
- NAT: rule masquerade `out-interface-list=WAN` sẵn có lo phần IPv4 ra WAN. IPv6 global không NAT.

---

## 3. Tạo container

> **BẮT BUỘC set `dns=`** — image này không tự cấu hình DNS; thiếu là **fail lúc cài** (không resolve được GitHub để tải zapret).

```
/container/config/set tmpdir=zapret6/tmp registry-url=https://registry-1.docker.io
/container/add remote-image=wiktorbgu/nfqws-mikrotik:latest \
  interface=veth-z6 root-dir=zapret6/root hostname=zapret6 \
  dns=8.8.8.8,1.1.1.1 logging=yes start-on-boot=yes \
  comment="zapret nfqws dual-stack"
/container/start [find comment~"nfqws"]
```

Lần đầu chạy nó **tự tải bol-van/zapret từ GitHub + cài** (chờ 1–2 phút). Xem log:
```
/log/print where topics~"container"
# đợi tới "Set default config" và "NFQWS github version ..."
```

---

## 4. Bật IPv6 + cấu hình strategy (trong container)

nfqws cấu hình bằng **file** `/opt/zapret/config`, không phải RouterOS envs.

```
/container/shell [find comment~"nfqws"]
```
Trong shell container:
```sh
# 1) Bật IPv6 (mặc định tắt)
sed -i 's/^DISABLE_IPV6=1/DISABLE_IPV6=0/' /opt/zapret/config

# 2) Bật IPv6 forwarding mỗi lần start (hook bền vững)
printf '#!/bin/sh\nsysctl -w net.ipv6.conf.all.forwarding=1 >/dev/null 2>&1\n' \
  > /opt/zapret/init.d/sysv/custom.d/10-ipv6-forward
chmod +x /opt/zapret/init.d/sysv/custom.d/10-ipv6-forward

# 3) (tuỳ chọn) chỉnh chiến lược desync: sửa dòng NFQWS_OPT trong /opt/zapret/config
#    Default đã tốt: fake/multidisorder 443 + multisplit 80 + fake QUIC

# 4) Áp dụng
sysctl -w net.ipv6.conf.all.forwarding=1
/opt/zapret/init.d/sysv/zapret restart
exit
```

Kiểm tra (trong container): `nft list ruleset | grep -c 'ip6 daddr'` phải > 0, và `ps | grep nfqws` phải có tiến trình.

> File `/opt/zapret/config` và hook nằm trên disk (`root-dir=zapret6/root`) → **bền vững** qua restart. Muốn cập nhật zapret: xoá `/opt/zapret` rồi restart container (nó tải lại bản mới).

---

## 5. Định tuyến traffic client vào container

Thay `<LAN>` bằng interface VLAN của bạn. Rule "skip" giữ traffic nội bộ + tới router không vòng qua container.

### IPv4
```
/routing/table/add name=to_zapret fib comment=zapret
/ip/firewall/mangle/add chain=prerouting in-interface=<LAN> connection-state=new \
  dst-address-list=!IP_Admin action=mark-connection new-connection-mark=zapret_conn \
  passthrough=yes comment="zapret v4: mark conn"
/ip/firewall/mangle/add chain=prerouting in-interface=<LAN> connection-mark=zapret_conn \
  action=mark-routing new-routing-mark=to_zapret passthrough=no comment="zapret v4: mark routing"
/ip/route/add dst-address=0.0.0.0/0 routing-table=to_zapret gateway=10.10.6.2 comment="zapret v4: via container"
```
(`IP_Admin` = address-list chứa subnet nội bộ cần đi thẳng.)

### IPv6
```
/routing/table/add name=via_zapret6 fib comment=zapret6
# forward cho traffic từ container ra ngoài
/ip/firewall/mangle/add chain=prerouting in-interface=<LAN> dst-address=<PFX>::/60 \
  action=accept comment="zapret v6: skip intra-site"
/ipv6/firewall/mangle/add chain=prerouting in-interface=<LAN> dst-address=fe80::/10 action=accept comment="zapret v6: skip ll"
/ipv6/firewall/mangle/add chain=prerouting in-interface=<LAN> dst-address=ff00::/8  action=accept comment="zapret v6: skip mcast"
/ipv6/firewall/mangle/add chain=prerouting in-interface=<LAN> connection-state=new \
  action=mark-connection new-connection-mark=zapret6_conn passthrough=yes comment="zapret v6: mark conn"
/ipv6/firewall/mangle/add chain=prerouting in-interface=<LAN> connection-mark=zapret6_conn \
  action=mark-routing new-routing-mark=via_zapret6 passthrough=no comment="zapret v6: mark routing"
/ipv6/route/add dst-address=::/0 routing-table=via_zapret6 gateway=<CT6>::2 comment="zapret v6: via container"
# cho phép forward traffic của container
/ipv6/firewall/filter/add chain=forward action=accept in-interface=veth-z6 \
  place-before=0 comment="zapret v6: container fwd"
```

> ⚠️ Rule "skip intra-site" IPv4 để ở `/ip/firewall/mangle`, IPv6 để ở `/ipv6/firewall/mangle` (menu riêng).

---

## 6. Kiểm tra

```
/ip/route/print   where routing-table=to_zapret     # ACTIVE, gw 10.10.6.2
/ipv6/route/print where routing-table=via_zapret6   # ACTIVE, gw <CT6>::2
```
Từ máy client trong VLAN:
```
curl -4 https://www.google.com -o /dev/null -w "v4 %{http_code}\n"
curl -6 https://www.youtube.com -o /dev/null -w "v6 %{http_code}\n"   # cả hai phải 200
```
Counter `mark-routing` (v4 + v6) phải tăng khi có traffic.

> Không test bypass từ trong container (masquerade/forward chỉ áp cho traffic client, không áp traffic tự phát của container). Test từ máy client thật.

---

## 7. Tinh chỉnh strategy khi 1 site vẫn bị chặn

nfqws có sẵn **blockcheck** — tự thử hàng loạt chiến lược với 1 domain và báo cái nào PASS:
```
/container/shell [find comment~"nfqws"]
/opt/zapret/blockcheck.sh          # làm theo hướng dẫn, nhập domain cần test
```
Lấy chiến lược PASS đưa vào dòng `NFQWS_OPT` trong `/opt/zapret/config`, rồi `zapret restart`.

---

## 8. Gỡ sạch (rollback)

```
/ip/firewall/mangle/remove   [find comment~"zapret v4"]
/ipv6/firewall/mangle/remove [find comment~"zapret v6"]
/ipv6/firewall/filter/remove [find comment~"container fwd"]
/ip/route/remove   [find comment~"zapret v4: via container"]
/ipv6/route/remove [find comment~"zapret v6: via container"]
/routing/table/remove [find name=to_zapret] ; /routing/table/remove [find name=via_zapret6]
/container/stop [find comment~"nfqws"] ; /container/remove [find comment~"nfqws"]
/interface/veth/remove veth-z6
/ip/address/remove   [find interface=veth-z6]
/ipv6/address/remove [find interface=veth-z6]
```

---

## 9. Ghi chú

- **nfqws không dùng RouterOS envs** — mọi cấu hình ở `/opt/zapret/config` (disk container, bền vững).
- VLAN rẽ qua container phụ thuộc container. `start-on-boot=yes` → tự chạy lại sau reboot. Nếu container chết, chỉ VLAN đó gián đoạn internet; truy cập router vẫn OK (giữ local nhờ rule skip).
- IPv6 global **không NAT** → giữ nguyên IP client. Cần `<PFX>` là prefix routed tới router (PD ≥ /60 để có /64 rảnh).
- Rất nhẹ vì NFQUEUE chỉ đụng ~vài packet đầu mỗi kết nối, phần còn lại chạy thẳng trong kernel.
