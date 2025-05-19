# Debug Note – Network Failure During Cardano EZ Installer Execution

**Thời gian**: 10/09/2024
**Môi trường**:

* Hệ điều hành: Windows 11 với WSL2
* Phân phối: Ubuntu 22.04
* Trình cài: `cardano-ez-installer v0.3`
* Mạng: FPT (Việt Nam), nghi ngờ NAT quy mô lớn (Carrier-Grade NAT / CG-NAT)

---

## ❗ Tóm tắt lỗi gặp

Khi chạy:

```bash
./install.sh
```

Trình cài báo lỗi không tải được repository `cardano-node`:

```bash
fatal: unable to access 'http://github.com/input-output-hk/cardano-node/': 
Could not resolve host: github.com
```

Thử ping đến hạ tầng Cardano:

```bash
ping book.world.dev.cardano.org
```

Kết quả:

```
100% packet loss (không truy cập được IP 3.73.96.239:443)
```

Tất cả các cài đặt DNS và proxy đều đã được kiểm tra.

---

## 🧠 Phân tích nguyên nhân

### 🔍 Vấn đề DNS & NAT trong WSL (Windows 11)

* Việc phân giải DNS trong WSL bị lỗi mặc dù đã cấu hình đúng.
* Ping tới server Cardano chỉ bị lỗi khi sử dụng mạng FPT; hoạt động bình thường khi chuyển sang mạng khác.
* WSL2 thực chất sử dụng NAT từ Windows host, bị ảnh hưởng bởi router và hạ tầng mạng bên ngoài.

### 🚫 CG-NAT / IPNAT từ ISP (FPT)

* Sau khi liên hệ FPT, nhân viên xác nhận tuyến mạng đang dùng IPNAT/CG-NAT:

  * Chia sẻ IP công cộng với nhiều người dùng.
  * Port P2P (để node Cardano đồng bộ) không hoạt động định tuyến.
  * Truy cập DNS và GitHub/CDN có thể bị lọc hoặc delay.

---

## ✅ Giải pháp đã áp dụng

### 1. ✅ Gỡ DNS mặc định trong WSL, dùng DNS của Google

```bash
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```

→ DNS GitHub và Cardano đồng bộ vẫn không truy cập được do NAT.

---

### 2. 📲 Liên hệ ISP FPT để yêu cầu gỡ IPNAT

> Gọi trực tiếp tổng đài kỹ thuật FPT theo hướng dẫn:
>
> **"Gọi 19006600, báo cần gỡ IPNAT để mở port truy cập P2P cho dự án blockchain (Cardano preview node)."**

* Sau khi gỡ IPNAT:

  * Ping Cardano server thành công.
  * Đồng bộ `cardano-node` hoàn tất.
  * Outbound traffic ổn định.

---

## 📌 Khuyến nghị dành cho dev blockchain dùng WSL

* Nếu không thể chuyển sang Linux native, có thể áp dụng các biện pháp hỗ trợ sau để cải thiện kết nối và khả năng đồng bộ node trong môi trường WSL:

### 🔧 Hỗ trợ kỹ thuật cho WSL:

1. **Dùng VPN có IP công hoặc split-tunnel**

   * Lựa chọn VPN cung cấp IP công (public IP) hoặc thiết lập split-tunnel để các kết nối đến GitHub, relay node có thể bypass CG-NAT.
   * Các VPN đề xuất: Mullvad, ProtonVPN, NordVPN (có wireguard hỗ trợ port forwarding).

2. **Chuyển sang WSL version 2 và bật chế độ networking nâng cao**

   * Cập nhật Windows và WSL:

     ```bash
     wsl --update
     wsl --set-default-version 2
     ```
   * Cho phép forwarding port và DNS từ Windows host vào WSL:

     ```powershell
     netsh interface portproxy add v4tov4 listenport=3001 connectaddress=127.0.0.1 connectport=3001
     ```

3. **Thiết lập mạng Bridged hoặc NAT riêng biệt nếu chạy qua Hyper-V**

   * Nếu dùng WSL2 với Hyper-V, có thể tạo một switch bridge qua card mạng thật thay vì NAT mặc định.
   * Điều này cho phép IP nội bộ trong WSL truy cập internet trực tiếp qua modem, tăng tính ổn định.

4. **Sử dụng tunneling service như Cloudflare Tunnel hoặc Tailscale**

   * Nếu node cần đồng bộ P2P hoặc cần phản hồi từ bên ngoài, có thể dùng tunneling để tránh chặn từ CG-NAT.

5. **Kiểm tra log bằng `journalctl`, `netstat` hoặc `tcpdump` để xác định port bị chặn**

   * Theo dõi `cardano-node` log với `journalctl -u cardano-node` để xác minh inbound/outbound.
   * Sử dụng `netstat -tnp` hoặc `tcpdump` để theo dõi gói tin khi sync.

6. **Cấu hình proxy từ Windows cho phép WSL truy cập qua `http_proxy`, `https_proxy`**

   * Đặt trong `.bashrc` hoặc `.profile`:

     ```bash
     export http_proxy=http://host.docker.internal:3128
     export https_proxy=http://host.docker.internal:3128
     ```

---

* Nếu dùng blockchain trong môi trường WSL:

  * **Kiểm tra NAT / IP công cộng** trên modem/router.
  * **Yêu cầu gỡ IPNAT / NAT chia sẻ** với ISP.
  * Hoặc dùng VPN với IP công cố định.
  * Nếu có yêu cầu về hiệu năng cao, độ ổn định lâu dài hoặc truy cập mạng phức tạp, nên cân nhắc cài trực tiếp Linux native trong môi trường production. Tuy nhiên, WSL hiện tại cũng đã hỗ trợ khá tốt cho hầu hết nhu cầu phát triển blockchain cá nhân và thử nghiệm.

---
