# ⚠️ Cảnh báo bảo mật nghiêm trọng (Critical Security Alerts)

Tài liệu này tổng hợp các vấn đề bảo mật nghiêm trọng thường gặp trong quá trình phát triển và vận hành Cardano node, cardano-cli, smart contracts (Plutus/Aiken) và các dịch vụ liên quan. Mỗi cảnh báo gồm:

* **Mô tả vấn đề**
* **Rủi ro**
* **Cách phát hiện**
* **Biện pháp khắc phục & phòng tránh**

---

## Mục lục

1. [Lưu Private Key trong Git](#1-lưu-private-key-trong-git)
2. [File `.skey` hoặc `.mkey` không mã hoá](#2-file-skey-hoặc-mkey-không-mã-hoá)
3. [Node lắng nghe trên Public IP mà không firewall](#3-node-lắng-nghe-trên-public-ip-mà-không-firewall)
4. [Hardcode seed phrase trong source code](#4-hardcode-seed-phrase-trong-source-code)
5. [Thiếu giới hạn trong API gọi `cardano-cli`/Lucid](#5-thiếu-giới-hạn-trong-api-gọi-cardano-clilucid)
6. [Không rotate KES key đúng chu kỳ](#6-không-rotate-kes-key-đúng-chu-kỳ)
7. [Lưu `--signing-key-file` trong script tự động](#7-lưu--signing-key-file-trong-script-tự-động)
8. [Expose debug info ra public log](#8-expose-debug-info-ra-public-log)
9. [CSRF/CORS khi triển khai REST API](#9-csrfcors-khi-triển-khai-rest-api)
10. [Replay Attack & Metadata Injection](#10-replay-attack--metadata-injection)

---

## 1. Lưu Private Key trong Git

### Mô tả vấn đề

Nhiều dev vô tình commit file chứa private key (`*.skey`, `*.hkey`, `*.mkey`) hoặc seed phrase vào repository.

### Rủi ro

* Kẻ tấn công có thể clone repo và chiếm quyền sở hữu wallet, đánh cắp ADA.
* Lộ danh tính staking pool/DRep.

### Cách phát hiện

* Chạy `git log --pretty=format:"%H %an %ad" -- **/*.skey`
* Dùng công cụ quét lộ thông tin: `gitleaks detect`, `truffleHog`

### Khắc phục & Phòng tránh

1. Thêm vào `.gitignore`:

   ```gitignore
   # Private keys
   *.skey
   *.hkey
   *.mkey
   mnemonic.txt
   ```
2. Xoá lịch sử nếu đã commit: `git filter-repo --invert-paths --path '*.skey'`
3. Dùng thành phần secrets manager (HashiCorp Vault, AWS KMS) hoặc `.env` để lưu.
4. Audit định kỳ với `gitleaks`, `truffleHog`.

---

## 2. File `.skey` hoặc `.mkey` không mã hoá

### Mô tả vấn đề

Private key lưu dưới dạng plaintext trên hệ thống hoặc cloud storage.

### Rủi ro

* Nếu máy chủ bị xâm nhập, attacker dễ dàng lấy khóa.

### Cách phát hiện

* Kiểm tra file tồn tại: `find ~/ -type f \( -name "*.skey" -o -name "*.mkey" \)`

### Khắc phục & Phòng tránh

1. Mã hoá file với GPG:

   ```bash
   gpg --symmetric --cipher-algo AES256 wallet.skey
   ```
2. Sử dụng HSM / hardware wallet (Ledger, Trezor) hoặc YubiKey.
3. Không lưu khóa trên máy chủ; yêu cầu người dùng nhập passphrase khi cần ký.

---

## 3. Node lắng nghe trên Public IP mà không firewall

### Mô tả vấn đề

`cardano-node` hoặc ứng dụng API bind `0.0.0.0` mà không giới hạn access.

### Rủi ro

* Port P2P, RPC, REST bị quét và tấn công brute-force.

### Cách phát hiện

* `netstat -tuln | grep 3001`
* Kiểm tra UPF/CORS rules.

### Khắc phục & Phòng tránh

1. Chỉ bind localhost:

   ```json
   {
     "ListenAddress": "127.0.0.1",
     // ...
   }
   ```
2. Cấu hình firewall (ufw):

   ```bash
   sudo ufw allow from 127.0.0.1 to any port 3001
   sudo ufw enable
   ```
3. Sử dụng reverse proxy (nginx, traefik) kèm TLS và access control.

---

## 4. Hardcode seed phrase trong source code

### Mô tả vấn đề

Seed phrase được viết trực tiếp trong mã để phục vụ test.

### Rủi ro

* Khi deploy, code có thể chứa seed phrase.

### Cách phát hiện

* `grep -R "[0-9a-zA-Z]{24}" src/`

### Khắc phục & Phòng tránh

1. Dùng biến môi trường hoặc file `.env`:

   ```dotenv
   WALLET_MNEMONIC="..."
   ```
2. Thường xuyên audit để xoá các hardcode.
3. Code review nghiêm ngặt.

---

## 5. Thiếu giới hạn trong API gọi `cardano-cli`/Lucid

### Mô tả vấn đề

REST API cho phép người dùng gọi trực tiếp `cardano-cli` hoặc Lucid mà không kiểm soát input.

### Rủi ro

* Injection lệnh độc hại, thao túng tx, đánh cắp asset.

### Cách phát hiện

* Kiểm tra route handler nhận tham số từ client và chèn vào lệnh shell.

### Khắc phục & Phòng tránh

1. Sử dụng thư viện binding (Lucid) thay vì gọi shell.
2. Validate & whitelist các tham số (address, amount).
3. Chạy API trong sandbox user không root.

---

## 6. Không rotate KES key đúng chu kỳ

### Mô tả vấn đề

KES key hết hạn nhưng không thay mới, node mất khả năng ký block.

### Rủi ro

* Node bị ngừng sản xuất block, mất phần thưởng và uy tín.

### Cách phát hiện

* Chạy `cardano-cli node issue-op-cert --kes-verification-key-file kes.vkey --cold-signing-key-file cold.skey --operational-certificate-issue-counter kes.counter` và xem epoch.

### Khắc phục & Phòng tránh

1. Thiết lập cron job:

   ```cron
   0 0 * * * /usr/local/bin/rotate-kes.sh
   ```
2. Script `rotate-kes.sh` kiểm tra `kes-period-info` và tự động generate ops cert.

---

## 7. Lưu `--signing-key-file` trong script tự động

### Mô tả vấn đề

Script deploy tx lưu đường dẫn và passphrase key.

### Rủi ro

* Ai có script có thể đọc key và passphrase.

### Cách phát hiện

* Xem script CI/CD hoặc cron job.

### Khắc phục & Phòng tránh

1. Tách bước ký, yêu cầu nhập passphrase thủ công.
2. Dùng agent (ssh-agent, gpg-agent) để load key trong phiên làm việc.

---

## 8. Expose debug info ra public log

### Mô tả vấn đề

Chạy node hoặc app với `--debug`/`--trace` và lưu log công cộng.

### Rủi ro

* Lộ đường dẫn, tham số nhạy cảm, thông tin môi trường.

### Cách phát hiện

* Kiểm tra file log hoặc console output.

### Khắc phục & Phòng tránh

1. Tắt debug trước khi deploy.
2. Sử dụng `logger` với cơ chế redact (ví dụ: chỉ show `xxxx..xxxx`).

---

## 9. CSRF/CORS khi triển khai REST API

### Mô tả vấn đề

Frontend có thể gửi yêu cầu độc hại qua CSRF hoặc CORS mở (`*`).

### Rủi ro

* Site bị tấn công CSRF, user bị ép ký tx.

### Cách phát hiện

* Kiểm tra header `Access-Control-Allow-Origin` và CSRF token.

### Khắc phục & Phòng tránh

1. Đặt CSRF token cho các endpoint thay đổi state.
2. Cấu hình CORS chỉ cho phép domain tin cậy.
3. Kết hợp SameSite cookie.

---

## 10. Replay Attack & Metadata Injection

### Mô tả vấn đề

Không kiểm soát nonce/UTxO, metadata được inject dữ liệu độc hại.

### Rủi ro

* Kẻ tấn công tái sử dụng tx, tạo tx giả.

### Cách phát hiện

* Kiểm tra tx history, metadata format.

### Khắc phục & Phòng tránh

1. Sử dụng UTxO model đúng: lock trước, unlock sau.
2. Validate schema metadata (JSON Schema).
3. Thêm nonce/timestamp vào metadata và kiểm tra uniqueness.

---

*Bản cập nhật ngày: 2025-05-19*
*Maintained by [@Minhcardanian](https://github.com/Minhcardanian)*
