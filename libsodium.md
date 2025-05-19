# Ghi chú Debug: Biên dịch `libsodium` với Hỗ trợ VRF cho Cardano Node

## Tóm tắt Cài đặt Cuối cùng

* **Repository sử dụng**: [IntersectMBO/libsodium](https://github.com/IntersectMBO/libsodium)
* **Nhánh**: `iquerejeta/vrf_batchverify`
* **Commit đã dùng**: `dbb48ccdd3b6a6c6e49ff7721f65cb0572c2a771`
* **Biểu tượng cần thiết xác nhận**: `crypto_vrf_ietfdraft13_*` có trong `/usr/local/lib/libsodium.so`
* **Tương thích với**: `cardano-crypto-praos` (phụ thuộc của `cardano-node`)
* **Phiên bản cardano-node đã biên dịch**: `10.3.1`, sử dụng GHC 9.6

---

## Các Lỗi Chính và Cách Đã Giải Quyết

### 1. Sử dụng sai nguồn repository ban đầu

* **Vấn đề**: Đã clone từ `input-output-hk/libsodium`, thiếu các hàm VRF cần thiết.
* **Hệ quả**: Lỗi liên kết khi build `cardano-crypto-praos`.
* **Giải pháp**: Chuyển sang dùng `IntersectMBO/libsodium` có chứa các hàm `crypto_vrf_ietfdraft13_*`.

### 2. Kiểm tra tệp `.so` quá sớm

* **Sai sót**: Kiểm tra biểu tượng trong `.so` trước khi build đúng source.
* **Hệ quả**: Kết quả sai làm chẩn đoán lệch hướng.
* **Bài học**: Luôn xác minh source và build hoàn tất trước khi kiểm tra `.so`.

### 3. Lỗi cấu hình script từ Savannah

* **Lỗi**: `configure: error: cannot guess build type; you must specify one`
* **Nguyên nhân**:

  * `wget` tải về trang HTML lỗi thay vì script shell.
  * Server Savannah tạm thời lỗi 502 Bad Gateway.
* **Khắc phục**:

```bash
curl -o build-aux/config.guess https://git.savannah.gnu.org/gitweb/?p=config.git\;a=blob_plain\;f=config.guess\;hb=HEAD
curl -o build-aux/config.sub https://git.savannah.gnu.org/gitweb/?p=config.git\;a=blob_plain\;f=config.sub\;hb=HEAD
chmod +x build-aux/config.*
```

### 4. Nhầm lẫn phiên bản cardano-node

* **Vấn đề**: Lệnh kiểm tra hiển thị phiên bản cũ `10.1.4`
* **Nguyên nhân**: Path hệ thống bị lẫn hoặc tồn dư binary cũ.
* **Khắc phục**:

```
cardano-node 10.3.1 - linux-x86_64 - ghc-9.6
git rev b3f237b75e64f4d8142af95b053e2828221d707f
```

Xác nhận phiên bản đúng sau khi rebuild.

---

## Các bước dọn dẹp hữu ích

```bash
sudo rm -rf /usr/local/lib/libsodium* /usr/local/include/sodium
find /usr/local/lib -name "libsodium*"  # kết quả chỉ nên còn .pc nếu có
```

---

## Lệnh xác nhận cuối cùng

```bash
nm -D /usr/local/lib/libsodium.so | grep ietfdraft13
```

* Kết quả mong đợi:

  * `crypto_vrf_ietfdraft13_batch_verify`
  * `crypto_vrf_ietfdraft13_publickeybytes`
  * ...v.v.

---

## Định nghĩa

### ✅ Thư viện **libsodium** là gì?

**libsodium** là thư viện mã hóa đa nền tảng, cung cấp các thao tác mã hóa cấp cao:

* Mã hóa đối xứng & khóa công khai,
* Mã hóa xác thực (Authenticated encryption),
* Trao đổi khóa và hàm băm (Blake2b, SHA),
* Hỗ trợ VRF (trong các fork tùy chỉnh như IntersectMBO).

Đây là thành phần thiết yếu trong hạ tầng node Cardano — đặc biệt cho giao tiếp bảo mật và tạo số ngẫu nhiên trong giao thức đồng thuận.

### 🔐 **VRF (Verifiable Random Function)** là gì?

VRF là một hàm:

* Sinh ra kết quả ngẫu nhiên từ một khóa bí mật,
* Cho phép bất kỳ ai xác minh rằng kết quả đó là hợp lệ.

Trong Cardano (Ouroboros Praos), VRF được dùng để:

* Xác minh riêng tư ai có quyền lãnh đạo slot,
* Đảm bảo ngẫu nhiên trong giao thức là có thể xác minh nhưng không bị điều chỉnh.

Dòng hàm `crypto_vrf_ietfdraft13_*` triển khai theo chuẩn IETF draft 13, có hỗ trợ thêm cho xác minh batch và tương thích ngược.

---

## Khuyến nghị cho các lần build sau

* Luôn xác nhận repository có chứa `crypto_vrf_ietfdraft13_*` trước khi tiếp tục.
* Chỉ kiểm tra `.so` sau khi đã build và cài đặt thành công.
* Kiểm tra file `config.guess` và `config.sub` tải về (không phải HTML).
* Gỡ sạch các phiên bản libsodium cũ trước khi cài lại.
* Xác minh phiên bản cuối cùng của `cardano-node` và tương thích GHC.

---

