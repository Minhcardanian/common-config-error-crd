# Debug socket

## Thông báo lỗi

```
Network.Socket.connect: <socket: …>: does not exist (Connection refused)
```

## Nguyên nhân có thể

* **Phiên bản node lỗi thời** (1.35.x) không tương thích với mạng `preview` hiện tại (phiên bản giao thức ≥10).

## Các bước khắc phục nhanh

1. **Cập nhật `cardano-node`** lên phiên bản được hỗ trợ (khuyến nghị `10.4.1`):

   ```bash
   curl -L https://github.com/IntersectMBO/cardano-node/releases/download/10.4.1/cardano-node-10.4.1-linux-x86_64.tar.gz \
     | tar xz
   sudo mv bin/cardano-* /usr/local/bin/
   cardano-node --version  # phải hiển thị 10.4.1
   ```

2. **Đảm bảo các file cấu hình `preview` chính xác** (`config.json`, `topology.json`, các file genesis) trong thư mục `$HOME/.cardano/preview`.

3. **Khởi chạy node** với đường dẫn socket đúng:

   ```bash
   cardano-node run \
     --config ~/.cardano/preview/config.json \
     --topology ~/.cardano/preview/topology.json \
     --database-path ~/.cardano/preview/db \
     --socket-path ~/.cardano/preview/node.socket \
     --host-addr 0.0.0.0 --port 3001
   ```

4. **Thiết lập biến môi trường socket**:

   ```bash
   export CARDANO_NODE_SOCKET_PATH=$HOME/.cardano/preview/node.socket
   ls -l $CARDANO_NODE_SOCKET_PATH  # kiểm tra file socket đã tồn tại
   ```

5. **Kiểm tra bằng `cardano-cli`**:

   ```bash
   cardano-cli query tip --testnet-magic 2
   ```

> Khi socket đã tồn tại, lệnh trên sẽ trả về JSON bao gồm trường `syncProgress`. Nếu vẫn gặp lỗi, kiểm tra log của node và quyền truy cập file socket.

## Hướng giải quyết bổ sung

* **Xây dựng và chạy từ nguồn**: Khi binary gặp lỗi không mong muốn, clone repo `input-output-hk/cardano-node` và build bằng Cabal (GHC 9.2/9.6, cabal 3.10+).

* **Chạy trong Docker chính thức**: Sử dụng image `ghcr.io/inputoutput/cardano-node:latest` hoặc tag tương ứng, tránh Docker image cũ hoặc build nhầm.

* **Kiểm tra firewall và port**: Đảm bảo port `3001` (hoặc port đã cấu hình) không bị iptables, ufw hoặc firewall cloud provider chặn.

* **Kiểm tra kết nối đến peers**: Dùng lệnh `telnet <relay>:3001` hoặc `nc -vz <relay> 3001` để xác thực kết nối đến các relay node trong `topology.json`.

* **Xóa và đồng bộ lại DB**: Nếu nghi ngờ database bị corrupt, dừng node, xóa thư mục `~/.cardano/preview/db`, khởi động lại để sync từ đầu.

* **Chuyển sang mạng testnet**: Thử cấu hình `--testnet-magic 1097911063` với bộ config testnet, xác định vấn đề chỉ xảy ra trên mạng preview hay chung.

* **Thêm relay nodes uy tín**: Cập nhật `topology.json` với các relay của IOHK hoặc Intersect để tăng cơ hội kết nối.

* **Giám sát tài nguyên hệ thống**: Kiểm tra CPU, RAM, I/O disk, tránh tình trạng node bị kill bởi OOM hoặc I/O quá tải.

* **Cập nhật Dev Container**:

  Nếu sau khi cập nhật binary mà version vẫn hiển thị **1.35.5** khi kiểm tra trong Dev Container, có thể bạn chưa rebuild container hoặc image cũ vẫn được dùng. Hãy làm theo các bước:

  1. **Dừng và xóa container cũ**

     ```bash
     docker ps -a | grep cardano-node               # xác định container
     docker stop <container_id_or_name>             # dừng
     docker rm   <container_id_or_name>             # xóa
     ```
  2. **Cập nhật `devcontainer.json`**

     ```json
     {
       "image": "ghcr.io/intersectmbo/cardano-node:10.4.1",
       "runArgs": [
         "-v", "${localWorkspaceFolder}/ipc:/ipc"
       ]
     }
     ```
  3. **Rebuild Dev Container**

     * Trong VS Code: mở Command Palette (Ctrl+Shift+P) → chọn **Dev Containers: Rebuild Container**.
     * Hoặc dùng CLI:

       ```bash
       devcontainer rebuild --workspace-folder .
       ```
  4. **Kiểm tra lại version**

     ```bash
     cardano-node --version   # phải hiển thị 10.4.1
     ```
  5. **Chạy node và kiểm tra socket** như các bước trước:

     ```bash
     cardano-node run \
       --config /data/config.json \
       --topology /data/topology.json \
       --database-path /data/db \
       --socket-path /ipc/node.socket \
       --host-addr 0.0.0.0 --port 3001
     export CARDANO_NODE_SOCKET_PATH=/ipc/node.socket
     ls -l /ipc/node.socket
     cardano-cli query tip --testnet-magic 2 --socket-path /ipc/node.socket
     ```

  Nếu vẫn không thành công, gửi log **node.log** và file **devcontainer.json** để debug chi tiết hơn.
