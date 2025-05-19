# 📚 Tổng hợp lỗi cấu hình phổ biến trong môi trường Cardano

Đây là tài liệu tổng hợp các lỗi cấu hình thường gặp trong quá trình cài đặt và phát triển với Cardano Node, cardano-cli, Aiken, Haskell, GHC, Cabal, cũng như khi sử dụng môi trường WSL trên Windows. Tài liệu này nhằm giúp các nhà phát triển tiết kiệm thời gian khi xử lý lỗi hệ thống, build tool và đồng bộ node.

---

## 🧱 1. Lỗi liên quan đến `cabal`, `ghcup`, và `ghc`

| ❗ Lỗi | 🔍 Nguyên nhân | ✅ Cách khắc phục |
|------|----------------|------------------|
| `cabal: Could not resolve dependencies` | Sai resolver hoặc thiếu ràng buộc trong `.cabal` | Chạy `cabal update`, kiểm tra lại file `.cabal`, dùng `cabal build all -w ghc-x.xx.x` |
| `ghc: Could not load module` | Thiếu module như `bytestring`, `aeson`, v.v. | Cài thư viện thiếu bằng `cabal install --lib <module>` |
| `ghcup: Not in PATH` | Biến môi trường PATH không được xuất đúng | Thêm `~/.ghcup/bin` vào `.bashrc` hoặc `.zshrc` |
| GHC version mismatch | Dự án yêu cầu GHC 9.2.8 hoặc 9.6.4 nhưng đang dùng bản khác | Dùng `ghcup set ghc 9.6.4` để chọn đúng phiên bản |

---

## 🐧 2. Lỗi phổ biến trong môi trường WSL (Windows Subsystem for Linux)

| ❗ Lỗi | 🔍 Nguyên nhân | ✅ Cách khắc phục |
|------|----------------|------------------|
| `Permission denied` ở `/mnt/c/...` | Truy cập phân vùng Windows bằng quyền root | Không dùng `sudo` với `/mnt/c`, hoặc `sudo chown` lại thư mục |
| System clock desync (lệch thời gian) | Đồng bộ giữa WSL và Windows kém | Dùng `sudo hwclock -s` hoặc khởi động lại WSL |
| Không có IPv6 hoặc bị chặn mạng P2P | WSL2 không có bridge networking đầy đủ | Dùng WSL2 với VPNKit hoặc NAT thủ công qua port mapping |

---

## 🔗 3. Lỗi thường gặp khi sử dụng Cardano Node & CLI

| ❗ Lỗi | 🔍 Nguyên nhân | ✅ Cách khắc phục |
|------|----------------|------------------|
| `cardano-node: symbol lookup error` | Thư viện liên kết động không khớp | Dùng `cabal clean`, build lại node với đúng GHC |
| `Failed reading: conversion error` khi dùng `cardano-cli` | `cardano-cli` cũ hơn `cardano-node` | Cập nhật `cardano-cli` để khớp version |
| `KES signature failed` | KES key hết hạn hoặc dùng sai KES period | Dùng `kes-period-info` kiểm tra, rotate lại KES key |
| `cardano-cli: Error decoding bech32` | Địa chỉ không hợp lệ hoặc bị cắt | Kiểm tra kỹ đoạn bech32, dùng `cardano-cli address key-hash` để validate |
| `Connection refused` port 3001 | Sai config hoặc firewall chặn | Kiểm tra `EnableP2P` trong `config.json`, mở port firewall |

---

## 🧪 4. Lỗi khi dùng Aiken, Plutus, hoặc môi trường `nix`

| ❗ Lỗi | 🔍 Nguyên nhân | ✅ Cách khắc phục |
|------|----------------|------------------|
| `error: file 'iohkNix' was not found` | Thiếu channel trong `nix` | Chạy `nix-channel --add https://github.com/input-output-hk/iohk-nix.git iohkNix` |
| `aiken: command not found` | PATH chưa được export | Thêm `~/.aiken/bin` vào biến PATH |
| `missing input 'haskell.nix'` | Sai hoặc thiếu input trong `flake.nix` | Kiểm tra lại `flake.nix`, chạy `nix flake update` để sửa |

---

## 🗃️ 5. Dọn rác hệ thống & xử lý lỗi bộ nhớ

| ❗ Lỗi | 🔍 Nguyên nhân | ✅ Cách khắc phục |
|------|----------------|------------------|
| Ổ đĩa đầy vì `.cabal/store` hoặc `/var/cache/apt` | Không dọn cache giữa các lần build | Dùng `cabal clean`, `sudo apt clean && sudo apt autoremove` |
| Gặp lỗi `Out of memory` khi build Haskell | WSL mặc định bị giới hạn RAM | Tăng RAM trong `.wslconfig`, hoặc dùng flag `+RTS -M` khi build |

---

## 📌 Đóng góp thêm

Nếu bạn gặp lỗi mới hoặc có giải pháp hiệu quả, hãy tạo PR hoặc issue tại [https://github.com/Minhcardanian/common-config-error-crd](https://github.com/Minhcardanian/common-config-error-crd). Mọi đóng góp đều giúp hệ sinh thái Cardano phát triển vững mạnh hơn!

---

*Maintained by [@Minhcardanian](https://github.com/Minhcardanian)*  
*Last updated: 2025-05*
